---
trigger: always_on
---


# Integration test guidelines
- for integration test involving calling this application's REST API, use `org.springframework.test.web.servlet.client.RestTestClient`
- use testcontainers with PostgreSQL
- use the same test container between tests to speed up test execution by extending `AbstractIntegrationTest` class
  - e.g.
```java
// AbstractIntegrationTest.java
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

public abstract class AbstractIntegrationTest {

    @ServiceConnection
    protected static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:15");

    static {
        postgres.start(); // force start once at class load
    }
}

// UserControllerTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerTest extends AbstractIntegrationTest {

    @LocalServerPort
    int port;

    private RestTestClient restTestClient;

    @Autowired
    UserRepository repository;

    @Autowired
    JdbcTemplate jdbcTemplate;


    @BeforeEach
    void setUp() {
        jdbcTemplate.execute("TRUNCATE TABLE users RESTART IDENTITY CASCADE");
        repository.deleteAll(); // clean up before each test

        this.restTestClient = RestTestClient.bindToServer()
                .baseUrl("http://localhost:" + port)
                .build();
    }

  @Test
    void shouldCreateUser() {
        String body = """
                {
                  "name": "John",
                  "age": 25
                }
                """;

        restTestClient.post()
                .uri("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .body(body)
                .exchange()
                .expectStatus().isOk()
                .expectBody()
                .jsonPath("$.name").isEqualTo("John")
                .jsonPath("$.age").isEqualTo(25)
                .jsonPath("$.id").isEqualTo(1);
    }
```