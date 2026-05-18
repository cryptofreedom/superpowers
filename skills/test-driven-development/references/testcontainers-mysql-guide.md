# Testcontainers MySQL Guide

> Read SKILL.md "Project test architecture" first. This guide shows **how** to wire up the integration test against a real Dockerized MySQL — the only kind of test this skill covers when the code under test touches a `DataSource` / mapper.
>
> **The same pattern generalizes to other middleware.** Redis (cache, TTL, distributed locks, Lua), message brokers (Kafka, RabbitMQ — ack, redelivery, partition routing), and any future middleware follow the same shape: a base class (`AbstractRedisIT`, `AbstractKafkaIT`, ...) that owns a JVM-shared container, exposes a client factory, and provides a per-test isolation hook (the equivalent of `truncate(table)`). The hard rules below — real container over mock, surgical scope per test, isolation hook in `@BeforeEach`, no Spring on top, distinct literal values for seed and assertion — apply equally. **When you need a Redis or broker variant and an `AbstractXxxIT` base does not yet exist, build one mirroring `AbstractMySqlIT` rather than reaching for a mock.** Mocking the middleware client to simulate its semantics is the anti-pattern this whole approach exists to avoid.

## Why a real MySQL

A test with a mocked DataSource cannot catch bugs that live in the SQL itself:

- MyBatis-Plus `BaseMapper.updateById` only updates non-null fields (`FieldStrategy.NOT_NULL`) — but does it really? Only real MySQL execution proves it.
- Camel-case (`memsourceProjectId`) ↔ snake-case (`memsource_project_id`) mapping
- JOINs / GROUP BY / window functions returning the wrong rows
- JSON columns + `@TableField(typeHandler=…)` round-trip
- Soft-delete `is_delete = 0` filter accidentally dropped from a new query
- Unique constraints / FK violations the mock-level seam can't see

For those bugs the *only* fake that's faithful enough is a real MySQL. Testcontainers is the cleanest way to get one without standing up shared infrastructure. The base class `com.eci.system.connector.integration.AbstractMySqlIT` makes this a one-liner per test.

## Two example skeletons

The integration test extends `AbstractMySqlIT`. Pick **what to point it at** based on the bug surface — see SKILL.md → "Project test architecture" and "When does the SQL deserve its own integration test" for the decision flow. The skeletons below differ only in scope; everything else (base class, isolation hook, distinct-value rule, no-Spring rule) is identical.

### Example: scoped to a mapper (when the SQL is the unit being pinned)

Use when the SQL behavior deserves a focused red/green independent of any caller — partial-update FieldStrategy, JSON_SET edges, FIND_IN_SET CSV semantics, multi-table UPDATE cross-row isolation, cross-DB / SP invocation, dynamic-SQL EXISTS branches, nested-subquery pagination, correlated-subquery / GROUP-MAX shapes.

```java
@DisplayName("XxxMapper.someQuery() on real MySQL")
class XxxMapperSomeQueryTest extends AbstractMySqlIT {

    private XxxMapper mapper;

    @BeforeEach
    void cleanAndWire() throws Exception {
        truncate("xxx_table");          // isolation per test (container is JVM-shared)
        mapper = getMapper(XxxMapper.class);
    }

    @Test
    @DisplayName("explicit-named scenario — what the SQL must do, not what the code does")
    void scenario() {
        // 1. Seed via the same mapper or via dataSource() raw JDBC
        // 2. Exercise the query under test
        // 3. Assert observable persistence outcome (reload + check fields)
    }
}
```

### Example: scoped to a service method (the connector-code default)

Use when a service method's business decision depends on what a non-trivial mapper actually returns (JOIN ≥2 tables, ≥15-column projection, JSON typeHandler, enum / bit / NOT NULL DEFAULT columns). The mapper is real, the SQL is real; only the true external collaborators (Feign / `ICATPlatformService` / `IInputPlatformService` / `RedisTemplate` / brokers / `MockedStatic` for static utilities like `RequestsUtil`) stay mocked.

```java
@DisplayName("XxxServiceImpl.doThing() — business decision over real SQL")
class XxxServiceImplDoThingTest extends AbstractMySqlIT {

    private XxxServiceImpl service;
    private XxxMapper xxxMapper;
    private YyyMapper yyyMapper;
    private SomeFeignClient feignClient;            // true external — mocked
    private MockedStatic<RequestsUtil> requestsUtilMock;

    @BeforeEach
    void cleanAndWire() throws Exception {
        truncate("xxx");                            // every table the test seeds
        truncate("yyy");

        xxxMapper = getMapper(XxxMapper.class);     // real mapper, real SQL
        yyyMapper = getMapper(YyyMapper.class);
        feignClient = mock(SomeFeignClient.class);
        requestsUtilMock = Mockito.mockStatic(RequestsUtil.class);
        requestsUtilMock.when(RequestsUtil::getUser).thenReturn(testUser());

        service = new XxxServiceImpl();
        ReflectionTestUtils.setField(service, "xxxMapper", xxxMapper);
        ReflectionTestUtils.setField(service, "yyyMapper", yyyMapper);
        ReflectionTestUtils.setField(service, "feignClient", feignClient);
    }

    @AfterEach
    void closeStatics() {
        if (requestsUtilMock != null) requestsUtilMock.close();
    }

    @Test
    @DisplayName("decision branch X — describe the business outcome, not the if-statement")
    void branchX() {
        // 1. Seed real rows via dataSource() raw JDBC — not via mapper.insert (avoid coupling to writes-under-test)
        try (var conn = dataSource().getConnection(); var st = conn.createStatement()) {
            st.executeUpdate("INSERT INTO xxx (id, client_id, status) VALUES (1, 7, 'A')");
            st.executeUpdate("INSERT INTO yyy (id, xxx_id, name) VALUES (10, 1, 'seeded-name')");
        }

        // 2. Stub external collaborators — only their contract-level behaviour
        when(feignClient.fetchSomething(eq(7))).thenReturn(R.success(new Foo("ext")));

        // 3. Exercise the service method (one path per test — see anti-patterns below)
        Result result = service.doThing(/* request */);

        // 4. Assert: service return + DB reload (Distinct Test Values rule applies — seed and assert literals
        //    should not share the same values, so a wrong-column read can't pass)
        assertThat(result.getResolvedName()).isEqualTo("seeded-name");
        Xxx reloaded = xxxMapper.selectById(1);
        assertThat(reloaded.getStatus()).isEqualTo("PROCESSED");
    }
}
```

Notes on the service-scoped example:
- **No Spring**. Instantiate the service with `new`, inject collaborators by reflection. Bringing up `ApplicationContext` defeats the seam and pulls in the whole connector universe. See "Hard rules" below.
- **Seed via raw JDBC, not via the mapper-under-test's own write methods.** Otherwise a bug in the write half hides the read half.
- **A mapper-scoped and a service-scoped test can co-exist for the same SQL when each carries weight independently** — see SKILL.md → "When does the SQL deserve its own integration test". When the mapper-scoped test has become a strict subset of what the service-scoped test exercises, fold it in (see "When to delete or rewrite" below).

## Hard rules

1. **File path follows the package of the code under test.** **Do not create a separate `integration/` tier.** Reference: `TaskMapperPartialUpdateTest` lives at `src/test/java/com/eci/system/connector/domain/mapper/`, next to `TaskMapper`, not under any `integration/` folder. See SKILL.md "Project test architecture".
2. **File name must end with `Test.java`** (not `IT.java`) — Surefire's default test pattern picks up `*Test.java`. Industry convention names integration tests `*IT.java`, but this project doesn't run a separate Failsafe phase; everything goes through Surefire. Follow the project, not the convention.
3. **Class must extend `AbstractMySqlIT`** — provides the container, DataSource, and mapper factory. If the base class doesn't fit your need, fix the base; don't fork.
4. **Always `truncate(table)` in `@BeforeEach`** — the container is shared across the whole JVM, no implicit isolation. Forgetting this is the #1 cause of flaky tests.
5. **Distinct literal values for seed and assertion.** Same Distinct Test Values rule from `inner-loop-guide.md`. If you seed `client_id = 7` and assert `clientId == 7`, a wrong-column read still passes. Ideally seed via one literal and assert against the row's reload, not against the test variable.
6. **Don't add Spring infrastructure on top.** No `@SpringBootTest`, no `@Autowired`, no `ApplicationContext`. The whole point is a focused integration test that swapped one mock for a container — adding Spring drags in the entire connector universe and defeats it.

## Adding a new table to the IT schema

The IT schema in `src/test/resources/it/init-schema-XXX.sql` is hand-written and minimal — only contains tables current tests reference.

1. Write `src/test/resources/it/init-schema-XXX.sql` (one file per table is fine).
2. Add the resource path to `AbstractMySqlIT.SCHEMA_FILES` in apply order.
3. **Mirror column types from the test environment exactly.** Permissive types (`TEXT` instead of `VARCHAR(255)`, missing `NOT NULL`) make the IT pass while the test environment throws. Use the mysql MCP (`mcp__mcp_server_mysql__mysql_query`) to run `SHOW CREATE TABLE` against the test environment and copy the DDL verbatim, instead of hand-writing it from memory.

Long-term improvement: replace hand-written schemas with `mysqldump --no-data` from a freshly migrated test-environment database, so the IT schema and `sql/*.sql` migrations cannot drift.

## Running

```bash
mvn test                              # all tests
mvn test -Dtest='**/XxxTest'          # one class
```

Requires Docker daemon. First run pulls `mysql:8.0` (~150MB). Subsequent runs are ~5s startup.

## Anti-patterns

- ❌ **Mocking the DataSource here.** If you'd do that, the code under test isn't actually touching middleware — write a plain JUnit test instead. This skeleton's only reason to exist is the real DB.
- ❌ **Mocking a real external boundary as if it were middleware.** Feign clients to partner systems, `ICATPlatformService` / `IInputPlatformService`, `RestTemplate` to a third-party — these stay mocked even in a service-scoped test. Do not pull them into the container. They are the boundary this skill deliberately doesn't cross; pinning their wire format belongs to E2E.
- ❌ **Wrapping `AbstractMySqlIT` in `@SpringBootTest`.** Adds the entire Nacos / Sa-Token / Redis / Feign universe back. If you genuinely need Spring wiring, that's a tier the project decided not to have — re-examine whether the seam belongs at unit level instead.
- ❌ **Chaining multiple service methods in one test** (e.g., `service.create(...) → service.update(...) → service.delete(...)` and asserting at the end). A service-scoped test is built around exercising **one** service method per test along **one** business path; chaining turns it into a multi-step flow that is slow, hard to diagnose, and tends to hide which step actually broke. *Note:* a service-scoped test deliberately covers a **whole feature path through one service method** — that is the unit; what's forbidden is stitching multiple methods together.
- ❌ **Reusing the same literal in seed and assertion.** Forces you to verify the column you wrote actually round-tripped — not "some 7 came back".
- ❌ **Asserting on `count(*)` only.** Always reload the row(s) and assert specific fields; `count == 1` passes for the wrong row.
- ❌ **Sleeping or polling.** This is deterministic by construction — if you're tempted to sleep, you're testing the wrong thing.
- ❌ **Hand-rolling complex DTOs as `when(mapper.xxx()).thenReturn(...)` returns** to stub a JOIN result for a service test that lives next to the service. The hand-rolled DTO is almost certainly the wrong shape — convert it to a service-scoped integration test against the real container instead.

## When to delete or rewrite

- Schema migration changes a column the test exercises → update the schema sql + test in the same PR
- The mapper method is removed → delete the test
- The test starts mocking the persistence layer → it's drifted out of scope; either restore the real container or move it out and stop calling it an integration test
- **A service-scoped test lands on top of a mapper-scoped test and the mapper-scoped test becomes a strict subset of what the service-scoped test now exercises** → fold the mapper-scoped test in. (Per SKILL.md → "When does the SQL deserve its own integration test", state 2.) The flow is:
  1. **Migrate**: copy the mapper-scoped test's assertion intent (specific edges, null guards, count boundaries, bug-pin comments) into the service-scoped test — usually as additional `@Test` methods or a `@Nested` block. Distinct Test Values rule still applies.
  2. **Verify**: run `mvn test` and confirm the service-scoped test actually fails when the migrated assertions are violated (e.g., temporarily break the SQL and watch the right test go red). Equivalence proven, not assumed.
  3. **Delete**: only now remove the migrated methods from the mapper-scoped test (or the whole file if every method moved). If any method tests a behaviour that is on the independent-anchor list (FieldStrategy boundaries, JSON_SET edges, etc.) and doesn't naturally fit the service path, leave it as the standalone anchor.
