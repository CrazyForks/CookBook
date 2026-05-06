# 单元测试基础与 JUnit 5

## 1. 单元测试概述

```
为什么需要单元测试？

┌─────────────────────────────────────────────────────────────┐
│                    测试金字塔                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                      /\                                     │
│                     /  \        E2E测试                     │
│                    / E2E\       (少量)                      │
│                   /______\                                  │
│                  /        \     集成测试                    │
│                 / Integration\   (适量)                     │
│                /______________\                             │
│               /                \   单元测试                 │
│              /   Unit Tests     \  (大量)                   │
│             /____________________\                          │
│                                                             │
│  单元测试优势:                                              │
│  ├── 快速反馈: 毫秒级执行                                   │
│  ├── 早期发现Bug: 开发阶段即可发现问题                      │
│  ├── 重构信心: 有测试保护，放心重构                         │
│  ├── 文档作用: 测试即代码的使用示例                         │
│  └── 设计驱动: TDD驱动良好设计                              │
│                                                             │
│  单元测试原则:                                              │
│  ├── FAST: Fast(快), Independent(独立), Self-validating(自验证), Timely(及时) │
│  ├── AAA: Arrange(准备), Act(执行), Assert(断言)            │
│  ├── FIRST: Fast(快), Independent(独), Repeatable(可重复), Self-validating(自验), Timely(及时) │
│  └── 一个测试只验证一个行为                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. JUnit 5 架构

```
JUnit 5 架构:

┌─────────────────────────────────────────────────────────────┐
│                    JUnit Platform                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              JUnit Launcher                          │   │
│  │  - 启动测试框架                                     │   │
│  │  - 发现和执行测试                                   │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                    JUnit Jupiter                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  编程模型 (API)              扩展模型 (Extension)   │   │
│  │  ├── @Test                   ├── BeforeAllCallback  │   │
│  │  ├── @BeforeEach             ├── AfterAllCallback   │   │
│  │  ├── @AfterEach              ├── BeforeEachCallback │   │
│  │  ├── @DisplayName            ├── AfterEachCallback  │   │
│  │  ├── @ParameterizedTest      ├── TestWatcher        │   │
│  │  ├── @RepeatedTest           └── ...                │   │
│  │  └── ...                                            │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                    JUnit Vintage                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  兼容 JUnit 3/4 的测试引擎                          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 3. 环境配置

```xml
<!-- pom.xml -->
<properties>
    <junit.version>5.10.1</junit.version>
    <mockito.version>5.8.0</mockito.version>
</properties>

<dependencies>
    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>${junit.version}</version>
        <scope>test</scope>
    </dependency>
    
    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>${mockito.version}</version>
        <scope>test</scope>
    </dependency>
    
    <!-- Mockito JUnit 5 集成 -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>${mockito.version}</version>
        <scope>test</scope>
    </dependency>
    
    <!-- AssertJ（可选，更优雅的断言） -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.24.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.2</version>
        </plugin>
    </plugins>
</build>
```

## 4. 基础注解

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    
    // ========== 生命周期方法 ==========
    
    @BeforeAll
    static void beforeAll() {
        // 整个测试类执行前执行一次
        // 必须是 static 方法
        System.out.println("Before All");
    }
    
    @AfterAll
    static void afterAll() {
        // 整个测试类执行后执行一次
        // 必须是 static 方法
        System.out.println("After All");
    }
    
    @BeforeEach
    void beforeEach() {
        // 每个测试方法执行前执行
        System.out.println("Before Each");
    }
    
    @AfterEach
    void afterEach() {
        // 每个测试方法执行后执行
        System.out.println("After Each");
    }
    
    // ========== 测试方法 ==========
    
    @Test
    @DisplayName("测试加法")  // 自定义测试名称
    void testAdd() {
        Calculator calc = new Calculator();
        assertEquals(5, calc.add(2, 3), "2 + 3 应该等于 5");
    }
    
    @Test
    @Disabled("暂时禁用")  // 禁用测试
    void disabledTest() {
        // 不会执行
    }
    
    @Test
    @Timeout(1)  // 超时测试（秒）
    void timeoutTest() {
        // 超过1秒会失败
    }
    
    @Test
    void testWithAssertions() {
        // ========== 断言方法 ==========
        
        // 相等断言
        assertEquals(5, calculator.add(2, 3));
        
        // 对象相等（引用比较）
        assertSame(obj1, obj2);
        
        // 非空断言
        assertNotNull(result);
        
        // 布尔断言
        assertTrue(list.isEmpty());
        assertFalse(list.isEmpty());
        
        // 异常断言
        assertThrows(IllegalArgumentException.class, () -> {
            calculator.divide(1, 0);
        });
        
        // 超时断言
        assertTimeout(Duration.ofSeconds(1), () -> {
            // 耗时操作
        });
        
        // 所有断言（收集所有失败）
        assertAll("group",
            () -> assertEquals(1, result.getX()),
            () -> assertEquals(2, result.getY()),
            () -> assertEquals(3, result.getZ())
        );
    }
}
```

## 5. 参数化测试

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;

class ParameterizedTestExample {
    
    // ========== 值来源 ==========
    
    @ParameterizedTest
    @ValueSource(ints = {1, 2, 3, 4, 5})
    void testWithValueSource(int number) {
        assertTrue(number > 0 && number < 10);
    }
    
    @ParameterizedTest
    @ValueSource(strings = {"racecar", "radar", "level"})
    void testPalindrome(String word) {
        assertTrue(isPalindrome(word));
    }
    
    // ========== 枚举来源 ==========
    
    @ParameterizedTest
    @EnumSource(value = DayOfWeek.class, names = {"MONDAY", "FRIDAY"})
    void testWithEnumSource(DayOfWeek day) {
        assertNotNull(day);
    }
    
    // ========== 方法来源 ==========
    
    @ParameterizedTest
    @MethodSource("provideNumbers")
    void testWithMethodSource(int number) {
        assertTrue(number > 0);
    }
    
    static Stream<Integer> provideNumbers() {
        return Stream.of(1, 2, 3, 4, 5);
    }
    
    // ========== CSV 来源 ==========
    
    @ParameterizedTest
    @CsvSource({
        "1, 2, 3",
        "10, 20, 30",
        "100, 200, 300"
    })
    void testWithCsvSource(int a, int b, int expected) {
        assertEquals(expected, calculator.add(a, b));
    }
    
    // ========== CSV 文件来源 ==========
    
    @ParameterizedTest
    @CsvFileSource(resources = "/test-data.csv", numLinesToSkip = 1)
    void testWithCsvFileSource(int a, int b, int expected) {
        assertEquals(expected, calculator.add(a, b));
    }
    
    // ========== 参数化显示名称 ==========
    
    @ParameterizedTest(name = "计算 {0} + {1} = {2}")
    @CsvSource({
        "1, 2, 3",
        "10, 20, 30"
    })
    void testWithCustomName(int a, int b, int expected) {
        assertEquals(expected, calculator.add(a, b));
    }
}
```

## 6. 嵌套测试

```java
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

class CalculatorTest {
    
    private Calculator calculator = new Calculator();
    
    @Nested
    class AdditionTests {
        @Test
        void testPositiveNumbers() {
            assertEquals(5, calculator.add(2, 3));
        }
        
        @Test
        void testNegativeNumbers() {
            assertEquals(-5, calculator.add(-2, -3));
        }
    }
    
    @Nested
    class DivisionTests {
        @Test
        void testNormalDivision() {
            assertEquals(2, calculator.divide(6, 3));
        }
        
        @Test
        void testDivisionByZero() {
            assertThrows(ArithmeticException.class, 
                () -> calculator.divide(1, 0));
        }
    }
}
```

## 7. 重复测试

```java
import org.junit.jupiter.api.RepeatedTest;

class RepeatedTestExample {
    
    @RepeatedTest(5)
    void repeatedTest() {
        // 重复执行5次
        assertTrue(true);
    }
    
    @RepeatedTest(value = 5, name = "第 {currentRepetition} 次，共 {totalRepetitions} 次")
    void repeatedTestWithCustomName() {
        assertTrue(true);
    }
}
```

## 8. 条件测试

```java
import org.junit.jupiter.api.condition.*;

class ConditionalTestExample {
    
    @Test
    @EnabledOnOs(OS.WINDOWS)
    void onlyOnWindows() {
        // 只在 Windows 上执行
    }
    
    @Test
    @EnabledOnOs(OS.MAC)
    void onlyOnMac() {
        // 只在 Mac 上执行
    }
    
    @Test
    @EnabledOnJre(JRE.JAVA_17)
    void onlyOnJava17() {
        // 只在 Java 17 上执行
    }
    
    @Test
    @EnabledIfEnvironmentVariable(named = "ENV", matches = "prod")
    void onlyInProd() {
        // 只在生产环境执行
    }
    
    @Test
    @EnabledIfSystemProperty(named = "os.arch", matches = "amd64")
    void onlyOnAmd64() {
        // 只在 amd64 架构执行
    }
}
```

## 9. 测试最佳实践

```
单元测试 Checklist:

□ 测试命名规范:
  - 方法名: should_预期行为_when_条件
  - 或: 测试_方法名_场景_预期结果
  - 使用 @DisplayName 提供可读名称

□ 测试结构:
  - Arrange: 准备测试数据
  - Act: 执行被测方法
  - Assert: 验证结果

□ 测试原则:
  - 一个测试只验证一个行为
  - 测试之间相互独立
  - 测试可重复执行
  - 测试快速执行
  - 测试自验证（无需人工判断）

□ 测试覆盖:
  - 正常流程测试
  - 边界条件测试
  - 异常场景测试
  - 空值/null 测试

□ 避免:
  - 测试中包含业务逻辑
  - 测试依赖外部系统
  - 测试依赖执行顺序
  - 测试过于复杂
```
