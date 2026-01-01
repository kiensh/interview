# Testing Interview Questions & Answers

---

## Table of Contents

### [Testing Fundamentals](#testing-fundamentals)
- [Q1: What are the different types of testing?](#q1)
- [Q2: What is the testing pyramid?](#q2)
- [Q3: What is the difference between mocking and stubbing?](#q3)

### [JUnit](#junit)
- [Q4: What are the main JUnit 5 annotations?](#q4)
- [Q5: What is the test lifecycle in JUnit?](#q5)
- [Q6: How do you write parameterized tests?](#q6)
- [Q7: What are assertions in JUnit?](#q7)

### [Mockito](#mockito)
- [Q8: What is Mockito and how do you use it?](#q8)
- [Q9: What is the difference between @Mock and @InjectMocks?](#q9)
- [Q10: How do you verify method calls with Mockito?](#q10)
- [Q11: What is ArgumentCaptor?](#q11)
- [Q12: How do you mock void methods?](#q12)

### [Spring Boot Testing](#spring-boot-testing)
- [Q13: What are the different test slices in Spring Boot?](#q13)
- [Q14: How do you test REST controllers?](#q14)
- [Q15: How do you test with a database?](#q15)

---

## Testing Fundamentals

<a id="q1"></a>
### Q1: What are the different types of testing?
**Answer:**

| Type | Scope | Speed | Purpose |
|------|-------|-------|---------|
| Unit Test | Single class/method | Fast | Test logic in isolation |
| Integration Test | Multiple components | Medium | Test component interaction |
| E2E Test | Entire application | Slow | Test user workflows |
| Contract Test | API boundaries | Medium | Ensure API compatibility |
| Performance Test | System | Slow | Test under load |

```java
// Unit Test - tests single method in isolation
@Test
void calculateDiscount_shouldReturnCorrectAmount() {
    PricingService service = new PricingService();
    BigDecimal result = service.calculateDiscount(new BigDecimal("100"), 10);
    assertEquals(new BigDecimal("10.00"), result);
}

// Integration Test - tests multiple components together
@SpringBootTest
@Transactional
class OrderServiceIntegrationTest {
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void createOrder_shouldPersistToDatabase() {
        Order order = orderService.createOrder(new OrderRequest(...));
        assertNotNull(orderRepository.findById(order.getId()));
    }
}

// E2E Test - tests entire flow
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class OrderE2ETest {
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void createOrder_fullFlow() {
        OrderRequest request = new OrderRequest(...);
        ResponseEntity<Order> response = restTemplate.postForEntity(
            "/api/orders", request, Order.class);
        assertEquals(HttpStatus.CREATED, response.getStatusCode());
    }
}
```

<a id="q2"></a>
### Q2: What is the testing pyramid?
**Answer:**

```
        /\
       /  \       E2E Tests (few, slow, expensive)
      /────\
     /      \     Integration Tests (some)
    /────────\
   /          \   Unit Tests (many, fast, cheap)
  /────────────\
```

| Level | Quantity | Speed | Maintenance |
|-------|----------|-------|-------------|
| Unit | Many (70%) | Fast | Low |
| Integration | Some (20%) | Medium | Medium |
| E2E | Few (10%) | Slow | High |

**Best practices:**
- Write many fast unit tests
- Use integration tests for critical paths
- Minimize slow E2E tests
- Follow F.I.R.S.T principles: Fast, Independent, Repeatable, Self-validating, Timely

<a id="q3"></a>
### Q3: What is the difference between mocking and stubbing?
**Answer:**

| Stub | Mock |
|------|------|
| Returns canned responses | Records interactions |
| State verification | Behavior verification |
| "When X happens, return Y" | "Verify X was called with Y" |
| Focus on output | Focus on how methods are called |

```java
// Stub - provides canned answers
@Test
void testWithStub() {
    UserRepository stubRepository = mock(UserRepository.class);
    
    // Stubbing: define return value
    when(stubRepository.findById(1L)).thenReturn(Optional.of(new User("John")));
    
    UserService service = new UserService(stubRepository);
    User user = service.getUser(1L);
    
    // Only verify state/output
    assertEquals("John", user.getName());
}

// Mock - verifies behavior
@Test
void testWithMock() {
    EmailService mockEmailService = mock(EmailService.class);
    
    NotificationService service = new NotificationService(mockEmailService);
    service.notifyUser(new User("john@email.com"));
    
    // Verify interaction/behavior
    verify(mockEmailService).sendEmail(
        eq("john@email.com"),
        eq("Welcome"),
        anyString()
    );
}
```

---

## JUnit

<a id="q4"></a>
### Q4: What are the main JUnit 5 annotations?
**Answer:**

| Annotation | Purpose |
|------------|---------|
| `@Test` | Marks a test method |
| `@BeforeEach` | Runs before each test |
| `@AfterEach` | Runs after each test |
| `@BeforeAll` | Runs once before all tests (static) |
| `@AfterAll` | Runs once after all tests (static) |
| `@DisplayName` | Custom test name |
| `@Disabled` | Skip this test |
| `@Nested` | Group related tests |
| `@Tag` | Tag for filtering tests |
| `@ParameterizedTest` | Run with different parameters |
| `@RepeatedTest` | Run test multiple times |
| `@Timeout` | Fail if takes too long |

```java
@DisplayName("User Service Tests")
class UserServiceTest {
    
    private UserService userService;
    
    @BeforeAll
    static void setupClass() {
        // Run once before all tests
    }
    
    @BeforeEach
    void setup() {
        userService = new UserService();
    }
    
    @Test
    @DisplayName("Should find user by ID")
    void findById_shouldReturnUser() {
        User user = userService.findById(1L);
        assertNotNull(user);
    }
    
    @Test
    @Disabled("Feature not implemented yet")
    void futureFeature() {
        // Skipped
    }
    
    @Nested
    @DisplayName("When user exists")
    class WhenUserExists {
        @Test
        void shouldReturnUserDetails() {
            // Grouped tests
        }
    }
    
    @AfterEach
    void teardown() {
        // Cleanup after each test
    }
}
```

<a id="q5"></a>
### Q5: What is the test lifecycle in JUnit?
**Answer:**

```
@BeforeAll (once)
    │
    ├─── Test 1:
    │    @BeforeEach → @Test → @AfterEach
    │
    ├─── Test 2:
    │    @BeforeEach → @Test → @AfterEach
    │
    └─── Test 3:
         @BeforeEach → @Test → @AfterEach
    │
@AfterAll (once)
```

```java
class LifecycleDemo {
    
    @BeforeAll
    static void beforeAll() {
        System.out.println("1. Before All");
    }
    
    @BeforeEach
    void beforeEach() {
        System.out.println("2. Before Each");
    }
    
    @Test
    void test1() {
        System.out.println("3. Test 1");
    }
    
    @Test
    void test2() {
        System.out.println("3. Test 2");
    }
    
    @AfterEach
    void afterEach() {
        System.out.println("4. After Each");
    }
    
    @AfterAll
    static void afterAll() {
        System.out.println("5. After All");
    }
}

// Output:
// 1. Before All
// 2. Before Each
// 3. Test 1
// 4. After Each
// 2. Before Each
// 3. Test 2
// 4. After Each
// 5. After All
```

<a id="q6"></a>
### Q6: How do you write parameterized tests?
**Answer:**

```java
class ParameterizedDemo {
    
    // ValueSource - simple values
    @ParameterizedTest
    @ValueSource(strings = {"racecar", "radar", "level"})
    void isPalindrome(String word) {
        assertTrue(StringUtils.isPalindrome(word));
    }
    
    // ValueSource with nulls and empty
    @ParameterizedTest
    @NullAndEmptySource
    @ValueSource(strings = {"  ", "\t", "\n"})
    void isBlank(String input) {
        assertTrue(StringUtils.isBlank(input));
    }
    
    // EnumSource
    @ParameterizedTest
    @EnumSource(Month.class)
    void monthsHaveValidValue(Month month) {
        assertTrue(month.getValue() >= 1 && month.getValue() <= 12);
    }
    
    // CsvSource - multiple parameters
    @ParameterizedTest
    @CsvSource({
        "1, 1, 2",
        "2, 3, 5",
        "10, 20, 30"
    })
    void add(int a, int b, int expected) {
        assertEquals(expected, calculator.add(a, b));
    }
    
    // MethodSource - complex objects
    @ParameterizedTest
    @MethodSource("provideUsers")
    void validateUser(User user, boolean expected) {
        assertEquals(expected, validator.isValid(user));
    }
    
    private static Stream<Arguments> provideUsers() {
        return Stream.of(
            Arguments.of(new User("John", "john@email.com"), true),
            Arguments.of(new User("", "invalid"), false),
            Arguments.of(new User(null, null), false)
        );
    }
}
```

<a id="q7"></a>
### Q7: What are assertions in JUnit?
**Answer:**

```java
import static org.junit.jupiter.api.Assertions.*;

class AssertionDemo {
    
    @Test
    void basicAssertions() {
        // Equality
        assertEquals(expected, actual);
        assertEquals(expected, actual, "Custom message");
        
        // Boolean
        assertTrue(condition);
        assertFalse(condition);
        
        // Null checks
        assertNull(object);
        assertNotNull(object);
        
        // Same reference
        assertSame(expected, actual);
        assertNotSame(expected, actual);
        
        // Arrays
        assertArrayEquals(expected, actual);
        
        // Iterable
        assertIterableEquals(expected, actual);
    }
    
    @Test
    void exceptionAssertions() {
        // Assert exception is thrown
        Exception ex = assertThrows(IllegalArgumentException.class, () -> {
            service.process(null);
        });
        assertEquals("Input cannot be null", ex.getMessage());
        
        // Assert no exception
        assertDoesNotThrow(() -> service.process("valid"));
    }
    
    @Test
    void groupedAssertions() {
        User user = service.getUser(1L);
        
        // All assertions run, reports all failures
        assertAll("user",
            () -> assertEquals("John", user.getName()),
            () -> assertEquals("john@email.com", user.getEmail()),
            () -> assertNotNull(user.getCreatedAt())
        );
    }
    
    @Test
    @Timeout(value = 500, unit = TimeUnit.MILLISECONDS)
    void timeoutAssertion() {
        // Or inline
        assertTimeout(Duration.ofSeconds(1), () -> {
            // Code that should complete within 1 second
            service.process();
        });
    }
}

// AssertJ (more fluent API)
import static org.assertj.core.api.Assertions.*;

@Test
void assertJExamples() {
    assertThat(user.getName()).isEqualTo("John");
    assertThat(list).hasSize(3).contains("a", "b");
    assertThat(number).isGreaterThan(5).isLessThan(10);
    assertThat(optional).isPresent().contains("value");
}
```

---

## Mockito

<a id="q8"></a>
### Q8: What is Mockito and how do you use it?
**Answer:**
Mockito is a mocking framework for unit tests that allows you to create mock objects, stub methods, and verify interactions.

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private EmailService emailService;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void createUser_shouldSaveAndSendEmail() {
        // Arrange - stub the repository
        User user = new User("John", "john@email.com");
        when(userRepository.save(any(User.class))).thenReturn(user);
        
        // Act
        User result = userService.createUser("John", "john@email.com");
        
        // Assert - verify state
        assertEquals("John", result.getName());
        
        // Verify interactions
        verify(userRepository).save(any(User.class));
        verify(emailService).sendWelcomeEmail("john@email.com");
    }
    
    @Test
    void getUser_whenNotFound_shouldThrowException() {
        // Stub to return empty
        when(userRepository.findById(anyLong())).thenReturn(Optional.empty());
        
        // Assert exception
        assertThrows(UserNotFoundException.class, () -> {
            userService.getUser(999L);
        });
    }
}
```

<a id="q9"></a>
### Q9: What is the difference between @Mock and @InjectMocks?
**Answer:**

| @Mock | @InjectMocks |
|-------|--------------|
| Creates a mock object | Creates the real object |
| Used for dependencies | Used for the class under test |
| Returns null/defaults by default | Injects mocks into this object |

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock  // Creates mock of dependency
    private OrderRepository orderRepository;
    
    @Mock  // Creates mock of dependency
    private PaymentService paymentService;
    
    @Mock  // Creates mock of dependency
    private NotificationService notificationService;
    
    @InjectMocks  // Creates real OrderService with mocks injected
    private OrderService orderService;
    
    @Test
    void createOrder_shouldProcessPaymentAndNotify() {
        // All dependencies are mocks, orderService is real
        when(orderRepository.save(any())).thenReturn(new Order(1L));
        when(paymentService.process(any())).thenReturn(true);
        
        orderService.createOrder(new OrderRequest());
        
        verify(paymentService).process(any());
        verify(notificationService).sendOrderConfirmation(any());
    }
}
```

**Manual equivalent:**
```java
@BeforeEach
void setup() {
    orderRepository = mock(OrderRepository.class);
    paymentService = mock(PaymentService.class);
    notificationService = mock(NotificationService.class);
    
    orderService = new OrderService(
        orderRepository, 
        paymentService, 
        notificationService
    );
}
```

<a id="q10"></a>
### Q10: How do you verify method calls with Mockito?
**Answer:**

```java
@Test
void verificationExamples() {
    // Basic verification
    verify(mock).method();
    
    // Verify with specific argument
    verify(mock).method("specific");
    
    // Verify with argument matchers
    verify(mock).method(anyString());
    verify(mock).method(eq("value"));
    verify(mock).method(argThat(s -> s.startsWith("prefix")));
    
    // Verify number of invocations
    verify(mock, times(3)).method();
    verify(mock, atLeast(2)).method();
    verify(mock, atMost(5)).method();
    verify(mock, never()).method();
    verify(mock, atLeastOnce()).method();
    
    // Verify call order
    InOrder inOrder = inOrder(mock1, mock2);
    inOrder.verify(mock1).firstMethod();
    inOrder.verify(mock2).secondMethod();
    
    // Verify no more interactions
    verifyNoMoreInteractions(mock);
    verifyNoInteractions(mock);
    
    // Verify with timeout (for async)
    verify(mock, timeout(1000)).asyncMethod();
}
```

<a id="q11"></a>
### Q11: What is ArgumentCaptor?
**Answer:**
ArgumentCaptor captures arguments passed to mock methods for later assertions.

```java
@Test
void argumentCaptorExample() {
    // Create captor
    ArgumentCaptor<User> userCaptor = ArgumentCaptor.forClass(User.class);
    ArgumentCaptor<String> stringCaptor = ArgumentCaptor.forClass(String.class);
    
    // Execute
    userService.createUser("John", "john@email.com");
    
    // Capture and verify
    verify(userRepository).save(userCaptor.capture());
    verify(emailService).sendEmail(eq("john@email.com"), stringCaptor.capture());
    
    // Assert captured values
    User capturedUser = userCaptor.getValue();
    assertEquals("John", capturedUser.getName());
    assertEquals("john@email.com", capturedUser.getEmail());
    
    String capturedSubject = stringCaptor.getValue();
    assertTrue(capturedSubject.contains("Welcome"));
}

// Capture multiple calls
@Test
void captureMultipleCalls() {
    service.process(Arrays.asList("a", "b", "c"));
    
    ArgumentCaptor<String> captor = ArgumentCaptor.forClass(String.class);
    verify(handler, times(3)).handle(captor.capture());
    
    List<String> allValues = captor.getAllValues();
    assertEquals(Arrays.asList("a", "b", "c"), allValues);
}
```

**With annotation:**
```java
@Captor
private ArgumentCaptor<User> userCaptor;
```

<a id="q12"></a>
### Q12: How do you mock void methods?
**Answer:**

```java
@Test
void mockingVoidMethods() {
    // doNothing - do nothing (default for void)
    doNothing().when(mock).voidMethod();
    
    // doThrow - throw exception
    doThrow(new RuntimeException("error"))
        .when(mock).voidMethod();
    
    // doAnswer - custom behavior
    doAnswer(invocation -> {
        String arg = invocation.getArgument(0);
        System.out.println("Called with: " + arg);
        return null;
    }).when(mock).voidMethod(anyString());
    
    // doCallRealMethod - call actual implementation (for spy)
    doCallRealMethod().when(spy).voidMethod();
    
    // Multiple behaviors
    doNothing()
        .doThrow(new RuntimeException())
        .when(mock).voidMethod();
    
    mock.voidMethod();  // Does nothing
    assertThrows(RuntimeException.class, () -> mock.voidMethod());  // Throws
}

// For consecutive calls with different arguments
@Test
void differentBehaviorPerCall() {
    doAnswer(invocation -> {
        String arg = invocation.getArgument(0);
        if ("fail".equals(arg)) {
            throw new RuntimeException("Failed");
        }
        return null;
    }).when(mock).process(anyString());
    
    mock.process("success");  // OK
    assertThrows(RuntimeException.class, () -> mock.process("fail"));
}
```

---

## Spring Boot Testing

<a id="q13"></a>
### Q13: What are the different test slices in Spring Boot?
**Answer:**

| Annotation | Purpose | Loads |
|------------|---------|-------|
| `@SpringBootTest` | Full integration test | Entire context |
| `@WebMvcTest` | Controller layer | MVC components only |
| `@DataJpaTest` | JPA repository | JPA components |
| `@WebFluxTest` | WebFlux controllers | WebFlux components |
| `@JsonTest` | JSON serialization | JSON components |
| `@RestClientTest` | REST client | RestTemplate |

```java
// Full integration test
@SpringBootTest
@AutoConfigureMockMvc
class FullIntegrationTest {
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void fullStack() throws Exception {
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk());
    }
}

// Controller slice - only loads web layer
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean  // Mock the service
    private UserService userService;
    
    @Test
    void getUser() throws Exception {
        when(userService.findById(1L)).thenReturn(new User("John"));
        
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}

// JPA slice - only loads JPA components
@DataJpaTest
class UserRepositoryTest {
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Test
    void findByEmail() {
        entityManager.persist(new User("test@email.com"));
        
        Optional<User> found = userRepository.findByEmail("test@email.com");
        
        assertTrue(found.isPresent());
    }
}
```

<a id="q14"></a>
### Q14: How do you test REST controllers?
**Answer:**

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void getUser_success() throws Exception {
        User user = new User(1L, "John", "john@email.com");
        when(userService.findById(1L)).thenReturn(user);
        
        mockMvc.perform(get("/api/users/{id}", 1L)
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("John"))
            .andExpect(jsonPath("$.email").value("john@email.com"));
    }
    
    @Test
    void getUser_notFound() throws Exception {
        when(userService.findById(anyLong()))
            .thenThrow(new UserNotFoundException("User not found"));
        
        mockMvc.perform(get("/api/users/{id}", 999L))
            .andExpect(status().isNotFound());
    }
    
    @Test
    void createUser_success() throws Exception {
        UserRequest request = new UserRequest("John", "john@email.com");
        User created = new User(1L, "John", "john@email.com");
        
        when(userService.create(any(UserRequest.class))).thenReturn(created);
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(header().exists("Location"))
            .andExpect(jsonPath("$.id").value(1));
    }
    
    @Test
    void createUser_validationError() throws Exception {
        UserRequest invalidRequest = new UserRequest("", "invalid-email");
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalidRequest)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray());
    }
}
```

<a id="q15"></a>
### Q15: How do you test with a database?
**Answer:**

**Option 1: In-memory database (H2)**
```java
@DataJpaTest
class UserRepositoryTest {
    // Uses H2 by default
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void findByEmail() {
        User user = new User("test@email.com");
        userRepository.save(user);
        
        Optional<User> found = userRepository.findByEmail("test@email.com");
        assertTrue(found.isPresent());
    }
}
```

**Option 2: Testcontainers (real database)**
```java
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldPersistUser() {
        User user = new User("John", "john@email.com");
        User saved = userRepository.save(user);
        
        assertNotNull(saved.getId());
        assertEquals("John", userRepository.findById(saved.getId()).get().getName());
    }
}
```

**Option 3: Embedded database with @Sql**
```java
@DataJpaTest
@Sql("/test-data.sql")  // Execute before test
class UserRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @Sql(scripts = "/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
    void testWithSqlData() {
        List<User> users = userRepository.findAll();
        assertEquals(3, users.size());  // Data from test-data.sql
    }
}
```

**test-data.sql:**
```sql
INSERT INTO users (id, name, email) VALUES (1, 'John', 'john@email.com');
INSERT INTO users (id, name, email) VALUES (2, 'Jane', 'jane@email.com');
INSERT INTO users (id, name, email) VALUES (3, 'Bob', 'bob@email.com');
```

---

[← Back to Java Index](README.md)
