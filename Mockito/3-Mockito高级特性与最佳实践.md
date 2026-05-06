# Mockito 高级特性与最佳实践

## 1. Mock 注解详解

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class AnnotationTest {
    
    // @Mock - 创建 Mock 对象
    @Mock
    private UserRepository userRepository;
    
    // @Spy - 创建 Spy 对象（包装真实对象）
    @Spy
    private UserService userService;
    
    // @Captor - 创建参数捕获器
    @Captor
    private ArgumentCaptor<User> userCaptor;
    
    // @InjectMocks - 自动注入 Mock 到被测对象
    @InjectMocks
    private UserController userController;
    
    // @Mock(name = "特定名称") - 指定 Mock 名称
    @Mock(name = "primaryDataSource")
    private DataSource primaryDataSource;
    
    // @Mock(answer = Answers.RETURNS_DEEP_STUBS) - 深度 Mock
    @Mock(answer = Answers.RETURNS_DEEP_STUBS)
    private Connection connection;
    
    // @Mock(lenient = true) - 宽松模式
    @Mock(lenient = true)
    private Config config;
}
```

## 2. 深度 Mock（Deep Stubs）

```java
class DeepStubsTest {
    
    @Test
    void testDeepStubs() {
        // 普通 Mock 需要逐层设置
        Connection connection = mock(Connection.class);
        Statement statement = mock(Statement.class);
        ResultSet resultSet = mock(ResultSet.class);
        
        when(connection.createStatement()).thenReturn(statement);
        when(statement.executeQuery(anyString())).thenReturn(resultSet);
        when(resultSet.next()).thenReturn(true);
        
        // 深度 Mock 自动创建链式对象
        Connection deepConnection = mock(Connection.class, Answers.RETURNS_DEEP_STUBS);
        
        when(deepConnection.createStatement()
            .executeQuery("SELECT * FROM users")
            .next())
            .thenReturn(true);
        
        // 使用
        Statement stmt = deepConnection.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT * FROM users");
        assertTrue(rs.next());
    }
}
```

## 3. Mock 静态方法（Mockito 3.4+）

```java
import org.mockito.MockedStatic;

class StaticMockTest {
    
    @Test
    void testStaticMethod() {
        try (MockedStatic<LocalDateTime> mocked = mockStatic(LocalDateTime.class)) {
            // 固定时间
            LocalDateTime fixedTime = LocalDateTime.of(2024, 1, 1, 12, 0);
            mocked.when(LocalDateTime::now).thenReturn(fixedTime);
            
            // 使用
            LocalDateTime now = LocalDateTime.now();
            assertEquals(fixedTime, now);
        }
        // try-with-resources 自动关闭，恢复原始行为
    }
    
    @Test
    void testStaticMethodWithAnswer() {
        try (MockedStatic<UUID> mocked = mockStatic(UUID.class)) {
            mocked.when(UUID::randomUUID)
                .thenAnswer(invocation -> UUID.fromString("550e8400-e29b-41d4-a716-446655440000"));
            
            UUID uuid = UUID.randomUUID();
            assertEquals("550e8400-e29b-41d4-a716-446655440000", uuid.toString());
        }
    }
}
```

## 4. Mock 构造函数（Mockito 3.5+）

```java
import org.mockito.MockedConstruction;

class ConstructorMockTest {
    
    @Test
    void testConstructorMock() {
        try (MockedConstruction<FileInputStream> mocked = 
                mockConstruction(FileInputStream.class, 
                    (mock, context) -> {
                        when(mock.read(any(byte[].class))).thenReturn(10);
                    })) {
            
            FileInputStream fis = new FileInputStream("test.txt");
            byte[] buffer = new byte[100];
            int read = fis.read(buffer);
            
            assertEquals(10, read);
            assertEquals(1, mocked.constructed().size());
        }
    }
}
```

## 5. Mock Final 类（Mockito 2.1+）

```java
// 需要配置: src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker
// 内容: mock-maker-inline

// 或者使用注解
@ExtendWith(MockitoExtension.class)
class FinalClassMockTest {
    
    @Mock
    private FinalUserService finalUserService;  // Final 类
    
    @Test
    void testFinalClass() {
        when(finalUserService.getUser(1L)).thenReturn(new User());
        
        User user = finalUserService.getUser(1L);
        assertNotNull(user);
    }
}
```

## 6. 自定义 Answer

```java
class CustomAnswerTest {
    
    @Test
    void testCustomAnswer() {
        UserRepository repo = mock(UserRepository.class);
        
        // 自定义回答逻辑
        when(repo.save(any(User.class))).thenAnswer(invocation -> {
            User user = invocation.getArgument(0);
            // 模拟数据库自增ID
            user.setId(System.currentTimeMillis());
            user.setCreatedAt(LocalDateTime.now());
            return user;
        });
        
        User user = new User();
        user.setName("张三");
        
        User saved = repo.save(user);
        assertNotNull(saved.getId());
        assertNotNull(saved.getCreatedAt());
    }
    
    @Test
    void testConsecutiveAnswers() {
        UserRepository repo = mock(UserRepository.class);
        
        // 连续调用返回不同结果
        when(repo.findById(1L))
            .thenAnswer(inv -> Optional.of(new User(1L, "第一次")))
            .thenAnswer(inv -> Optional.of(new User(1L, "第二次")))
            .thenThrow(new RuntimeException("第三次异常"));
        
        assertEquals("第一次", repo.findById(1L).get().getName());
        assertEquals("第二次", repo.findById(1L).get().getName());
        assertThrows(RuntimeException.class, () -> repo.findById(1L));
    }
}
```

## 7. Mock 私有方法（通过反射）

```java
import java.lang.reflect.Method;

class PrivateMethodMockTest {
    
    @Test
    void testPrivateMethod() throws Exception {
        UserService service = spy(new UserService());
        
        // 获取私有方法
        Method privateMethod = UserService.class.getDeclaredMethod(
            "validateEmail", String.class);
        privateMethod.setAccessible(true);
        
        // Mock 私有方法（通过 doReturn）
        doReturn(true).when(service).validateEmail(anyString());
        
        // 但更推荐通过重构使私有方法可测试
    }
}
```

## 8. 测试异步代码

```java
import java.util.concurrent.CompletableFuture;

class AsyncTest {
    
    @Mock
    private AsyncUserService asyncUserService;
    
    @Test
    void testAsyncMethod() {
        // Mock 异步方法
        when(asyncUserService.findUserAsync(1L))
            .thenReturn(CompletableFuture.completedFuture(new User(1L, "张三")));
        
        // 测试
        CompletableFuture<User> future = asyncUserService.findUserAsync(1L);
        User user = future.join();
        
        assertEquals("张三", user.getName());
    }
    
    @Test
    void testAsyncException() {
        CompletableFuture<User> failedFuture = new CompletableFuture<>();
        failedFuture.completeExceptionally(new RuntimeException("用户不存在"));
        
        when(asyncUserService.findUserAsync(999L)).thenReturn(failedFuture);
        
        assertThrows(RuntimeException.class, 
            () -> asyncUserService.findUserAsync(999L).join());
    }
}
```

## 9. 集成 AssertJ

```java
import static org.assertj.core.api.Assertions.*;

class AssertJTest {
    
    @Test
    void testWithAssertJ() {
        User user = new User(1L, "张三", 25);
        
        // 对象断言
        assertThat(user)
            .isNotNull()
            .extracting(User::getName, User::getAge)
            .containsExactly("张三", 25);
        
        // 集合断言
        List<User> users = Arrays.asList(
            new User(1L, "张三"),
            new User(2L, "李四")
        );
        
        assertThat(users)
            .hasSize(2)
            .extracting(User::getName)
            .containsExactly("张三", "李四");
        
        // 异常断言
        assertThatThrownBy(() -> {
            throw new IllegalArgumentException("参数错误");
        }).isInstanceOf(IllegalArgumentException.class)
          .hasMessage("参数错误");
    }
}
```

## 10. 生产级测试模式

### 10.1 测试数据构建器

```java
// UserTestBuilder.java
public class UserTestBuilder {
    
    private Long id = 1L;
    private String name = "测试用户";
    private String email = "test@example.com";
    private Integer age = 25;
    
    public static UserTestBuilder aUser() {
        return new UserTestBuilder();
    }
    
    public UserTestBuilder withId(Long id) {
        this.id = id;
        return this;
    }
    
    public UserTestBuilder withName(String name) {
        this.name = name;
        return this;
    }
    
    public UserTestBuilder withEmail(String email) {
        this.email = email;
        return this;
    }
    
    public User build() {
        return new User(id, name, email, age);
    }
}

// 使用
User user = UserTestBuilder.aUser()
    .withId(100L)
    .withName("自定义用户")
    .build();
```

### 10.2 测试夹具（Fixture）

```java
class UserTestFixture {
    
    public static User createDefaultUser() {
        return new User(1L, "张三", "zhangsan@example.com", 25);
    }
    
    public static List<User> createUsers(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> new User((long)i, "User" + i, "user" + i + "@example.com", 20 + i))
            .collect(Collectors.toList());
    }
    
    public static User createUserWithInvalidEmail() {
        return new User(1L, "张三", "invalid-email", 25);
    }
}
```

## 11. 最佳实践总结

```
Mockito 高级使用 Checklist:

□ Mock 策略:
  - @Mock: 完全模拟依赖
  - @Spy: 部分模拟（包装真实对象）
  - @InjectMocks: 自动注入被测对象

□ 参数匹配:
  - 使用 any() 匹配任意值
  - 使用 eq() 匹配特定值
  - 避免在同一个调用中混用匹配器和具体值

□ 验证策略:
  - 验证重要的外部交互
  - 不要验证实现细节
  - 使用 ArgumentCaptor 验证复杂参数

□ 异常测试:
  - 使用 assertThrows 验证异常
  - 验证异常消息和类型
  - 测试边界条件

□ 测试隔离:
  - 每个测试独立设置 Mock
  - 使用 @BeforeEach 初始化
  - 使用 reset() 重置 Mock

□ 代码质量:
  - 使用 BDD 风格（given/when/then）
  - 使用测试构建器减少重复
  - 提取公共测试夹具
```
