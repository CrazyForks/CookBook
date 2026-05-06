# Spring Boot 测试实战

## 1. Spring Boot 测试概述

```
Spring Boot 测试层次:

┌─────────────────────────────────────────────────────────────┐
│                Spring Boot 测试类型                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  单元测试 (@ExtendWith(MockitoExtension.class)):           │
│  ├── 不启动 Spring 容器                                     │
│  ├── 使用 Mock 模拟依赖                                     │
│  ├── 速度快，适合纯逻辑测试                                 │
│  └── 示例: Service 层测试                                   │
│                                                             │
│  集成测试 (@SpringBootTest):                               │
│  ├── 启动完整 Spring 容器                                   │
│  ├── 加载真实 Bean 和配置                                   │
│  ├── 测试组件间交互                                         │
│  └── 示例: Controller + Service + Repository                │
│                                                             │
│  切片测试:                                                  │
│  ├── @WebMvcTest: 只加载 Web 层                            │
│  ├── @DataJpaTest: 只加载 JPA 层                           │
│  ├── @JsonTest: 只加载 JSON 序列化                         │
│  └── 示例: Controller 测试（不启动完整容器）               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 单元测试（Mockito）

```java
// UserService.java
@Service
public class UserService {
    
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    public UserService(UserRepository userRepository, 
                       EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
    
    public User createUser(CreateUserRequest request) {
        // 检查用户名是否存在
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new DuplicateUsernameException("用户名已存在");
        }
        
        // 创建用户
        User user = new User();
        user.setUsername(request.getUsername());
        user.setEmail(request.getEmail());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        
        User saved = userRepository.save(user);
        
        // 发送欢迎邮件
        emailService.sendWelcomeEmail(saved.getEmail());
        
        return saved;
    }
}
```

```java
// UserServiceTest.java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private EmailService emailService;
    
    @Mock
    private PasswordEncoder passwordEncoder;
    
    @InjectMocks
    private UserService userService;
    
    @Captor
    private ArgumentCaptor<User> userCaptor;
    
    @Test
    @DisplayName("创建用户 - 成功")
    void createUser_Success() {
        // Given
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("zhangsan");
        request.setEmail("zhangsan@example.com");
        request.setPassword("password123");
        
        when(userRepository.existsByUsername("zhangsan")).thenReturn(false);
        when(passwordEncoder.encode("password123")).thenReturn("encoded_password");
        when(userRepository.save(any(User.class)))
            .thenAnswer(inv -> {
                User user = inv.getArgument(0);
                user.setId(1L);
                return user;
            });
        
        // When
        User result = userService.createUser(request);
        
        // Then
        assertNotNull(result);
        assertEquals(1L, result.getId());
        assertEquals("zhangsan", result.getUsername());
        
        verify(userRepository).save(userCaptor.capture());
        User savedUser = userCaptor.getValue();
        assertEquals("encoded_password", savedUser.getPassword());
        
        verify(emailService).sendWelcomeEmail("zhangsan@example.com");
    }
    
    @Test
    @DisplayName("创建用户 - 用户名已存在")
    void createUser_DuplicateUsername_ThrowsException() {
        // Given
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("existing_user");
        
        when(userRepository.existsByUsername("existing_user")).thenReturn(true);
        
        // When & Then
        assertThrows(DuplicateUsernameException.class, 
            () -> userService.createUser(request));
        
        verify(userRepository, never()).save(any());
        verify(emailService, never()).sendWelcomeEmail(anyString());
    }
}
```

## 3. Web 层切片测试

```java
// UserController.java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    private final UserService userService;
    
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        User user = userService.getUserById(id);
        return ResponseEntity.ok(toResponse(user));
    }
    
    @PostMapping
    public ResponseEntity<UserResponse> createUser(@RequestBody @Valid CreateUserRequest request) {
        User user = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(toResponse(user));
    }
}
```

```java
// UserControllerTest.java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    @DisplayName("获取用户 - 成功")
    void getUser_Success() throws Exception {
        // Given
        User user = new User(1L, "zhangsan", "zhangsan@example.com");
        when(userService.getUserById(1L)).thenReturn(user);
        
        // When & Then
        mockMvc.perform(get("/api/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.username").value("zhangsan"))
            .andExpect(jsonPath("$.email").value("zhangsan@example.com"));
        
        verify(userService).getUserById(1L);
    }
    
    @Test
    @DisplayName("创建用户 - 参数校验失败")
    void createUser_ValidationFailed() throws Exception {
        // Given
        CreateUserRequest request = new CreateUserRequest();
        // 不设置必填字段
        
        // When & Then
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray());
        
        verify(userService, never()).createUser(any());
    }
}
```

## 4. Data JPA 切片测试

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Test
    @DisplayName("根据用户名查找用户")
    void findByUsername() {
        // Given
        User user = new User();
        user.setUsername("zhangsan");
        user.setEmail("zhangsan@example.com");
        entityManager.persistAndFlush(user);
        
        // When
        Optional<User> found = userRepository.findByUsername("zhangsan");
        
        // Then
        assertTrue(found.isPresent());
        assertEquals("zhangsan", found.get().getUsername());
    }
    
    @Test
    @DisplayName("检查用户名是否存在")
    void existsByUsername() {
        // Given
        User user = new User();
        user.setUsername("existing_user");
        entityManager.persistAndFlush(user);
        
        // When & Then
        assertTrue(userRepository.existsByUsername("existing_user"));
        assertFalse(userRepository.existsByUsername("new_user"));
    }
}
```

## 5. JSON 切片测试

```java
@JsonTest
class UserJsonTest {
    
    @Autowired
    private JacksonTester<User> json;
    
    @Test
    @DisplayName("序列化用户对象")
    void serialize() throws Exception {
        User user = new User(1L, "zhangsan", "zhangsan@example.com");
        
        JsonContent<User> result = json.write(user);
        
        assertThat(result).extractingJsonPathNumberValue("$.id").isEqualTo(1);
        assertThat(result).extractingJsonPathStringValue("$.username").isEqualTo("zhangsan");
        assertThat(result).extractingJsonPathStringValue("$.email").isEqualTo("zhangsan@example.com");
    }
    
    @Test
    @DisplayName("反序列化用户对象")
    void deserialize() throws Exception {
        String content = "{\"id\":1,\"username\":\"zhangsan\",\"email\":\"zhangsan@example.com\"}";
        
        User result = json.parseObject(content);
        
        assertEquals(1L, result.getId());
        assertEquals("zhangsan", result.getUsername());
    }
}
```

## 6. 集成测试

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@Testcontainers
class UserIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private UserRepository userRepository;
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }
    
    @Test
    @DisplayName("用户注册完整流程")
    void registerUser_FullFlow() throws Exception {
        // 1. 创建用户
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("zhangsan");
        request.setEmail("zhangsan@example.com");
        request.setPassword("password123");
        
        String response = mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andReturn().getResponse().getContentAsString();
        
        Long userId = JsonPath.read(response, "$.id");
        
        // 2. 查询用户
        mockMvc.perform(get("/api/users/" + userId))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.username").value("zhangsan"));
        
        // 3. 验证数据库
        assertTrue(userRepository.existsByUsername("zhangsan"));
    }
}
```

## 7. 测试配置

```java
// 测试配置类
@TestConfiguration
public class TestConfig {
    
    @Bean
    @Primary
    public EmailService testEmailService() {
        return new MockEmailService();
    }
    
    @Bean
    public Clock testClock() {
        return Clock.fixed(
            Instant.parse("2024-01-01T12:00:00Z"),
            ZoneId.of("UTC")
        );
    }
}

// 测试基类
@SpringBootTest
@ActiveProfiles("test")
@TestPropertySource(properties = {
    "spring.mail.host=localhost",
    "spring.mail.port=25"
})
public abstract class BaseIntegrationTest {
    
    @Autowired
    protected TestEntityManager entityManager;
}
```

## 8. 测试 Profile 配置

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  
  mail:
    host: localhost
    port: 25

logging:
  level:
    org.hibernate.SQL: DEBUG
```

## 9. 最佳实践

```
Spring Boot 测试 Checklist:

□ 测试分层:
  - 单元测试: Service 层（Mockito）
  - 切片测试: Controller/JPA/JSON
  - 集成测试: 完整流程

□ 测试配置:
  - 使用 @ActiveProfiles("test")
  - 使用 TestContainers 测试数据库
  - 使用 @TestConfiguration 配置测试 Bean

□ Mock 策略:
  - @MockBean: 替换 Spring Bean
  - @SpyBean: 包装真实 Bean
  - WireMock: 模拟外部 HTTP 服务

□ 数据管理:
  - 每个测试独立数据
  - 使用 @Transactional 自动回滚
  - 使用 TestEntityManager 管理测试数据

□ 性能优化:
  - 使用 @DirtiesContext 最小化容器重启
  - 使用 @Sql 初始化测试数据
  - 并行执行独立测试
```
