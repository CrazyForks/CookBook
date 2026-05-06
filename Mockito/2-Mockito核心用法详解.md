# Mockito 核心用法详解

## 1. Mockito 概述

```
Mockito 是什么？

┌─────────────────────────────────────────────────────────────┐
│                    Mockito Mock 框架                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  核心功能:                                                  │
│  ├── 创建 Mock 对象: 模拟依赖的行为                        │
│  ├── 设置返回值: when().thenReturn()                       │
│  ├── 验证调用: verify()                                    │
│  ├── 参数捕获: ArgumentCaptor                              │
│  └── 异常模拟: thenThrow()                                 │
│                                                             │
│  Mock vs Spy:                                               │
│  ┌─────────────┬─────────────┬─────────────┐               │
│  │ 特性        │ Mock        │ Spy         │               │
│  ├─────────────┼─────────────┼─────────────┤               │
│  │ 行为        │ 完全模拟    │ 包装真实对象│               │
│  │ 默认返回    │ null/0/false│ 真实返回    │               │
│  │ 使用场景    │ 隔离依赖    │ 部分模拟    │               │
│  └─────────────┴─────────────┴─────────────┘               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 创建 Mock 对象

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.InjectMocks;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)  // JUnit 5 启用 Mockito
class UserServiceTest {
    
    // ========== 方式一：注解方式 ==========
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private EmailService emailService;
    
    @InjectMocks
    private UserService userService;  // 自动注入上面的 Mock
    
    // ========== 方式二：代码方式 ==========
    
    @Test
    void testWithCode() {
        UserRepository mockRepo = mock(UserRepository.class);
        EmailService mockEmail = mock(EmailService.class);
        
        // 使用 mock 对象
        UserService service = new UserService(mockRepo, mockEmail);
    }
    
    // ========== 方式三：Mock 名称 ==========
    
    @Mock(name = "userRepo")
    private UserRepository userRepository2;
}
```

## 3. 设置返回值（Stubbing）

```java
class StubbingTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Test
    void testStubbing() {
        // ========== 基本存根 ==========
        
        // 当调用 findById(1L) 时，返回 user
        when(userRepository.findById(1L))
            .thenReturn(Optional.of(new User(1L, "张三")));
        
        // 当调用 findAll() 时，返回列表
        when(userRepository.findAll())
            .thenReturn(Arrays.asList(
                new User(1L, "张三"),
                new User(2L, "李四")
            ));
        
        // ========== 链式存根 ==========
        
        // 连续调用返回不同值
        when(userRepository.findById(anyLong()))
            .thenReturn(Optional.of(new User(1L, "第一次")))
            .thenReturn(Optional.of(new User(2L, "第二次")))
            .thenThrow(new RuntimeException("第三次异常"));
        
        // ========== 异常存根 ==========
        
        // 抛出异常
        when(userRepository.findById(999L))
            .thenThrow(new EntityNotFoundException("用户不存在"));
        
        // void 方法抛异常
        doThrow(new RuntimeException("邮件发送失败"))
            .when(emailService).sendEmail(anyString(), anyString());
        
        // ========== 无返回值方法 ==========
        
        // void 方法存根
        doNothing().when(userRepository).deleteById(1L);
        
        // void 方法执行后抛异常
        doThrow(new RuntimeException()).when(mock).someVoidMethod();
        
        // ========== Answer（自定义返回逻辑） ==========
        
        when(userRepository.save(any(User.class)))
            .thenAnswer(invocation -> {
                User user = invocation.getArgument(0);
                user.setId(1L);
                return user;
            });
    }
}
```

## 4. 参数匹配器

```java
import static org.mockito.ArgumentMatchers.*;

class ArgumentMatcherTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Test
    void testArgumentMatchers() {
        // ========== 通用匹配器 ==========
        
        // 任意值
        when(userRepository.findById(any())).thenReturn(Optional.empty());
        
        // 任意类型
        when(userRepository.save(any(User.class))).thenReturn(new User());
        
        // ========== 特定匹配器 ==========
        
        // 等于
        when(userRepository.findById(eq(1L))).thenReturn(Optional.of(user));
        
        // 字符串匹配
        when(userRepository.findByName(contains("张"))).thenReturn(users);
        when(userRepository.findByName(startsWith("张"))).thenReturn(users);
        when(userRepository.findByName(endsWith("三"))).thenReturn(users);
        when(userRepository.findByName(matches("张.*"))).thenReturn(users);
        
        // ========== 数值匹配器 ==========
        
        when(userRepository.findByAge(gt(18))).thenReturn(users);      // 大于
        when(userRepository.findByAge(lt(60))).thenReturn(users);      // 小于
        when(userRepository.findByAge(ge(18))).thenReturn(users);      // 大于等于
        when(userRepository.findByAge(le(60))).thenReturn(users);      // 小于等于
        
        // ========== 集合匹配器 ==========
        
        when(userRepository.findByIdIn(anyList())).thenReturn(users);
        when(userRepository.findByIdIn(anySet())).thenReturn(users);
        when(userRepository.findByIdIn(anyCollection())).thenReturn(users);
        
        // ========== 空值匹配器 ==========
        
        when(userRepository.findByName(isNull())).thenReturn(emptyList());
        when(userRepository.findByName(notNull())).thenReturn(users);
        
        // ========== 逻辑组合 ==========
        
        when(userRepository.findById(anyLong()))
            .thenReturn(Optional.of(user));
        
        when(userRepository.findByName(not(empty()))).thenReturn(users);
    }
}
```

## 5. 验证调用

```java
import static org.mockito.Mockito.*;

class VerificationTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Test
    void testVerification() {
        // ========== 基本验证 ==========
        
        // 验证方法被调用
        verify(userRepository).findById(1L);
        
        // 验证方法被调用次数
        verify(userRepository, times(1)).findById(1L);
        verify(userRepository, never()).findById(999L);
        verify(userRepository, atLeastOnce()).findAll();
        verify(userRepository, atLeast(2)).findAll();
        verify(userRepository, atMost(5)).findAll();
        
        // ========== 参数验证 ==========
        
        // 验证特定参数
        verify(userRepository).findById(eq(1L));
        verify(userRepository).save(any(User.class));
        
        // ========== 顺序验证 ==========
        
        InOrder inOrder = inOrder(userRepository);
        inOrder.verify(userRepository).save(any());
        inOrder.verify(userRepository).flush();
        
        // ========== 无交互验证 ==========
        
        verifyNoInteractions(userRepository);
        verifyNoMoreInteractions(userRepository);
    }
}
```

## 6. Spy 对象

```java
class SpyTest {
    
    @Test
    void testSpy() {
        // 创建 spy 对象
        List<String> spyList = spy(new ArrayList<>());
        
        // 真实方法会被调用
        spyList.add("one");
        spyList.add("two");
        
        // 验证真实行为
        assertEquals(2, spyList.size());
        
        // 部分模拟
        when(spyList.size()).thenReturn(100);
        assertEquals(100, spyList.size());
        
        // 但 get 仍然使用真实逻辑
        assertEquals("one", spyList.get(0));
    }
    
    @Test
    void testDoReturn() {
        List<String> spyList = spy(new ArrayList<>());
        
        // 使用 doReturn 而不是 when
        // when 会先调用真实方法，可能导致问题
        doReturn(100).when(spyList).size();
        
        assertEquals(100, spyList.size());
    }
}
```

## 7. ArgumentCaptor 参数捕获

```java
class ArgumentCaptorTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Captor
    private ArgumentCaptor<User> userCaptor;
    
    @Test
    void testArgumentCaptor() {
        // 执行操作
        userService.createUser("张三", "zhangsan@example.com");
        
        // 捕获参数
        verify(userRepository).save(userCaptor.capture());
        
        // 验证捕获的参数
        User capturedUser = userCaptor.getValue();
        assertEquals("张三", capturedUser.getName());
        assertEquals("zhangsan@example.com", capturedUser.getEmail());
    }
    
    @Test
    void testMultipleCaptures() {
        // 执行多次操作
        userService.createUser("张三", "a@example.com");
        userService.createUser("李四", "b@example.com");
        
        // 捕获所有参数
        verify(userRepository, times(2)).save(userCaptor.capture());
        
        List<User> allValues = userCaptor.getAllValues();
        assertEquals(2, allValues.size());
        assertEquals("张三", allValues.get(0).getName());
        assertEquals("李四", allValues.get(1).getName());
    }
}
```

## 8. BDD 风格（Behavior Driven Development）

```java
import static org.mockito.BDDMockito.*;

class BDDTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Test
    void testBDDStyle() {
        // ========== Given（准备） ==========
        
        given(userRepository.findById(1L))
            .willReturn(Optional.of(new User(1L, "张三")));
        
        given(userRepository.save(any(User.class)))
            .willAnswer(invocation -> {
                User user = invocation.getArgument(0);
                user.setId(1L);
                return user;
            });
        
        // ========== When（执行） ==========
        
        Optional<User> result = userService.getUserById(1L);
        
        // ========== Then（验证） ==========
        
        then(userRepository).should().findById(1L);
        then(userRepository).should(never()).deleteById(anyLong());
        then(userRepository).shouldHaveNoMoreInteractions();
    }
}
```

## 9. 常用场景示例

```java
class CommonScenariosTest {
    
    @Mock
    private UserService userService;
    
    @Test
    void testExceptionHandling() {
        // 场景1：模拟异常
        when(userService.getUserById(999L))
            .thenThrow(new UserNotFoundException("用户不存在"));
        
        assertThrows(UserNotFoundException.class, 
            () -> userService.getUserById(999L));
    }
    
    @Test
    void testVoidMethod() {
        // 场景2：void 方法
        doNothing().when(userService).deleteUser(1L);
        doThrow(new RuntimeException()).when(userService).deleteUser(999L);
        
        userService.deleteUser(1L);  // 正常执行
        assertThrows(RuntimeException.class, 
            () -> userService.deleteUser(999L));
    }
    
    @Test
    void testCallback() {
        // 场景3：回调
        doAnswer(invocation -> {
            Long id = invocation.getArgument(0);
            return new User(id, "User" + id);
        }).when(userService).getUserById(anyLong());
        
        User user = userService.getUserById(1L);
        assertEquals("User1", user.getName());
    }
    
    @Test
    void testReset() {
        // 场景4：重置 mock
        when(userService.getUserById(1L)).thenReturn(new User());
        reset(userService);  // 清除所有存根和验证
        
        // 重新设置
        when(userService.getUserById(1L)).thenReturn(new User(1L, "新用户"));
    }
}
```

## 10. 最佳实践

```
Mockito 使用 Checklist:

□ 何时使用 Mock:
  - 外部依赖（数据库、网络、文件）
  - 难以构造的对象
  - 非确定性行为（时间、随机数）

□ 避免过度 Mock:
  - 不要 Mock 被测类本身
  - 不要 Mock 值对象（POJO）
  - 不要 Mock 工具类

□ Mock 设置:
  - 只设置测试需要的行为
  - 使用 any() 而不是具体值（除非必要）
  - 使用 @InjectMocks 自动注入

□ 验证原则:
  - 验证重要的交互
  - 不要验证实现细节
  - 使用 ArgumentCaptor 验证参数

□ 代码风格:
  - 使用 BDD 风格（given/when/then）
  - 使用 @ExtendWith(MockitoExtension.class)
  - 使用静态导入简化代码
```
