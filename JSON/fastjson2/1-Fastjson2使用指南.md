# Fastjson2 使用指南

## 1. 概述

```
Fastjson2 特点:

┌─────────────────────────────────────────────────────────────┐
│                     Fastjson2 核心特性                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✓ 性能最优（比 Jackson 快 30-100%）                        │
│  ✓ API 简洁易用                                            │
│  ✓ 支持 JSON/JSONB 两种格式                                │
│  ✓ 强大的 JSONPath 支持                                    │
│  ✓ 安全性大幅改进（autoType 白名单）                        │
│  ✓ 兼容 Fastjson 1.x API                                   │
│  ✓ 支持 GraalVM 原生编译                                   │
│                                                             │
│  核心类:                                                    │
│  JSON           - 主要入口，静态方法                        │
│  JSONReader     - 流式读取                                 │
│  JSONWriter     - 流式写入                                 │
│  JSONObject     - JSON 对象                                │
│  JSONArray      - JSON 数组                                │
│  JSONPath       - JSON 路径查询                            │
│                                                             │
│  与 Fastjson 1.x 区别:                                     │
│  包名: com.alibaba.fastjson → com.alibaba.fastjson2        │
│  安全: autoType 默认关闭，需显式配置                        │
│  性能: 提升 30-100%                                         │
│  格式: 支持 JSONB 二进制格式                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 环境配置

```xml
<!-- Maven -->
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2</artifactId>
    <version>2.0.50</version>
</dependency>

<!-- 兼容 Fastjson 1.x API -->
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2-extension</artifactId>
    <version>2.0.50</version>
</dependency>

<!-- Spring Boot 集成 -->
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2-extension-spring6</artifactId>
    <version>2.0.50</version>
</dependency>
```

## 3. 基础使用

### 3.1 序列化

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.JSONWriter;

User user = new User(1L, "张三", "zhangsan@example.com");

// 基本序列化
String json = JSON.toJSONString(user);
// {"id":1,"name":"张三","email":"zhangsan@example.com"}

// 美化输出
String prettyJson = JSON.toJSONString(user, JSONWriter.Feature.PrettyFormat);

// 包含类型信息（用于反序列化）
String jsonWithType = JSON.toJSONString(user, JSONWriter.Feature.WriteClassName);
// {"@type":"com.example.User","id":1,"name":"张三"}

// 序列化 null 值
String jsonWithNull = JSON.toJSONString(user, JSONWriter.Feature.WriteNulls);

// 日期格式化
String json = JSON.toJSONString(user, JSONWriter.Feature.WriteDateUseDateFormat);

// 指定特征组合
String json = JSON.toJSONString(user, 
    JSONWriter.Feature.PrettyFormat,
    JSONWriter.Feature.WriteNulls,
    JSONWriter.Feature.WriteDateUseDateFormat
);
```

### 3.2 反序列化

```java
String json = "{\"id\":1,\"name\":\"张三\",\"email\":\"zhangsan@example.com\"}";

// 基本反序列化
User user = JSON.parseObject(json, User.class);

// 带类型信息的反序列化
String jsonWithType = "{\"@type\":\"com.example.User\",\"id\":1,\"name\":\"张三\"}";
User user = JSON.parseObject(jsonWithType, User.class);

// 反序列化为 JSONObject
JSONObject obj = JSON.parseObject(json);
String name = obj.getString("name");
Long id = obj.getLong("id");

// 反序列化为 JSONArray
String jsonArray = "[{\"id\":1},{\"id\":2}]";
JSONArray array = JSON.parseArray(jsonArray);

// 泛型反序列化
List<User> users = JSON.parseObject(jsonArray, new TypeReference<List<User>>() {});

// 安全反序列化（推荐）
User user = JSON.parseObject(json, User.class, 
    JSONReader.Feature.SafeMode);
```

## 4. 注解详解

### 4.1 @JSONField

```java
public class User {
    
    // 字段名映射
    @JSONField(name = "user_name")
    private String userName;
    
    // 日期格式
    @JSONField(format = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createTime;
    
    // 序列化/反序列化控制
    @JSONField(serialize = false, deserialize = false)
    private String password;
    
    // 序列化顺序
    @JSONField(ordinal = 1)
    private Long id;
    
    @JSONField(ordinal = 2)
    private String name;
    
    // 自定义序列化器
    @JSONField(serializeUsing = MoneySerializer.class)
    private BigDecimal price;
    
    // 自定义反序列化器
    @JSONField(deserializeUsing = CustomDateDeserializer.class)
    private Date eventTime;
}
```

### 4.2 @JSONType

```java
// 类级别配置
@JSONType(
    typeKey = "@type",  // 类型信息字段名
    includes = {"id", "name", "email"},  // 包含字段
    orders = {"id", "name", "email"},    // 字段顺序
    serializer = UserSerializer.class,   // 自定义序列化器
    deserializer = UserDeserializer.class // 自定义反序列化器
)
public class User {
    private Long id;
    private String name;
    private String email;
}
```

### 4.3 多态处理

```java
// 方式一：@type 自动类型（需开启 autoType）
String json = "{\"@type\":\"com.example.Dog\",\"name\":\"旺财\"}";
Animal animal = JSON.parseObject(json, Animal.class);

// 方式二：@JSONType 指定子类型
@JSONType(
    typeKey = "animalType",
    subTypes = {
        @JSONType.Type(value = Dog.class, name = "dog"),
        @JSONType.Type(value = Cat.class, name = "cat")
    }
)
public abstract class Animal {
    private String name;
}
```

## 5. JSONPath 查询

```java
import com.alibaba.fastjson2.JSONPath;

String json = """
{
    "store": {
        "books": [
            {"category": "编程", "title": "Java编程思想", "price": 108},
            {"category": "编程", "title": "Effective Java", "price": 68},
            {"category": "小说", "title": "三体", "price": 56}
        ]
    }
}
""";

// 基本路径查询
Object result = JSONPath.extract(json, "$.store.books[0].title");
// "Java编程思想"

// 条件查询
Object result = JSONPath.extract(json, 
    "$.store.books[?(@.price > 80)].title");
// ["Java编程思想"]

// 聚合查询
Object result = JSONPath.extract(json, "$.store.books.length()");
// 3

Object result = JSONPath.extract(json, "$.store.books.sum(@.price)");
// 232

// 通配符
Object result = JSONPath.extract(json, "$.store.books[*].title");
// ["Java编程思想", "Effective Java", "三体"]

// 修改值
JSONPath.set(json, "$.store.books[0].price", 99);

// 删除元素
JSONPath.remove(json, "$.store.books[1]");
```

## 6. 流式 API

### 6.1 JSONReader

```java
String json = "{\"id\":1,\"name\":\"张三\",\"hobbies\":[\"reading\",\"coding\"]}";

try (JSONReader reader = JSONReader.of(json)) {
    // 方式一：自动绑定
    User user = reader.read(User.class);
    
    // 方式二：手动解析
    reader.startObject();
    while (reader.hasNext()) {
        String fieldName = reader.readFieldName();
        switch (fieldName) {
            case "id":
                long id = reader.readInt64Value();
                break;
            case "name":
                String name = reader.readString();
                break;
            case "hobbies":
                reader.startArray();
                while (reader.hasNext()) {
                    String hobby = reader.readString();
                }
                reader.endArray();
                break;
            default:
                reader.skipValue();
        }
    }
    reader.endObject();
}
```

### 6.2 JSONWriter

```java
try (JSONWriter writer = JSONWriter.of()) {
    writer.startObject();
    
    writer.writeFieldName("id");
    writer.writeInt64(1L);
    
    writer.writeFieldName("name");
    writer.writeString("张三");
    
    writer.writeFieldName("hobbies");
    writer.startArray();
    writer.writeString("reading");
    writer.writeString("coding");
    writer.endArray();
    
    writer.endObject();
    
    String json = writer.toString();
}
```

## 7. 安全配置

```java
// 安全模式（推荐）
String json = "...";
User user = JSON.parseObject(json, User.class, 
    JSONReader.Feature.SafeMode);

// autoType 白名单（如需使用）
// 方式一：JVM 参数
// -Dfastjson2.parser.autoTypeBeforeHandler=com.example.**

// 方式二：代码配置
JSONReader.autoTypeBeforeHandler(
    "com.example.",  // 包名前缀
    "com.example.model."  // 精确包名
);

// 方式三：类名白名单
AutoTypeBeforeHandler handler = AutoTypeBeforeHandler.of(
    "com.example.User",
    "com.example.Order"
);

// 禁用 autoType（默认）
String json = "...";
User user = JSON.parseObject(json, User.class);
// 不会自动识别 @type，更安全
```

## 8. JSONB 二进制格式

```java
import com.alibaba.fastjson2.JSONB;

User user = new User(1L, "张三", "zhangsan@example.com");

// 序列化为 JSONB 字节
byte[] bytes = JSONB.toBytes(user);

// 从 JSONB 字节反序列化
User user = JSONB.parseObject(bytes, User.class);

// JSONB 优势：
// 1. 更紧凑（比 JSON 小 30-50%）
// 2. 更快解析（比 JSON 快 2-3 倍）
// 3. 适合内部服务通信、缓存存储
```

## 9. Spring Boot 集成

```java
@Configuration
public class Fastjson2Config {
    
    @Bean
    public HttpMessageConverter<?> fastjson2Converter() {
        FastJsonHttpMessageConverter converter = new FastJsonHttpMessageConverter();
        
        FastJsonConfig config = new FastJsonConfig();
        config.setDateFormat("yyyy-MM-dd HH:mm:ss");
        config.setWriterFeatures(
            JSONWriter.Feature.PrettyFormat,
            JSONWriter.Feature.WriteDateUseDateFormat
        );
        config.setReaderFeatures(
            JSONReader.Feature.SafeMode
        );
        
        converter.setFastJsonConfig(config);
        return converter;
    }
}

// application.yml
spring:
  fastjson:
    date-format: yyyy-MM-dd HH:mm:ss
    pretty-print: true
    safe-mode: true
```

## 10. 性能优化

```java
// 1. 使用 ClassInfo 缓存
// Fastjson2 自动缓存类信息，无需额外配置

// 2. 使用 JSONB 格式（内部通信）
byte[] bytes = JSONB.toBytes(user);
User user = JSONB.parseObject(bytes, User.class);

// 3. 预编译 JSONPath
JSONPath path = JSONPath.of("$.store.books[0].title");
Object result = path.extract(json);

// 4. 使用流式 API（大数据量）
try (JSONReader reader = JSONReader.of(inputStream)) {
    // 流式处理
}

// 5. 关闭不需要的特性
JSON.toJSONString(user, 
    JSONWriter.Feature.IgnoreErrorGetter,  // 忽略 getter 异常
    JSONWriter.Feature.NotWriteRootClassName  // 不写根类名
);
```

## 11. 常见问题

### Q1: 如何从 Fastjson 1.x 迁移到 Fastjson2？

```java
// 旧代码 (Fastjson 1.x)
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;

// 新代码 (Fastjson2)
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.JSONObject;

// API 基本兼容，只需修改包名
// 或使用兼容包：
import com.alibaba.fastjson.JSON;  // 自动使用 fastjson2 实现
```

### Q2: autoType 安全风险如何处理？

```java
// 方案一：使用安全模式（推荐）
User user = JSON.parseObject(json, User.class, 
    JSONReader.Feature.SafeMode);

// 方案二：关闭 autoType（默认已关闭）
// 不使用 JSONWriter.Feature.WriteClassName

// 方案三：配置白名单
AutoTypeBeforeHandler handler = AutoTypeBeforeHandler.of("com.example.");
```

### Q3: 如何处理日期格式？

```java
// 方式一：全局配置
JSON.toJSONString(user, JSONWriter.Feature.WriteDateUseDateFormat);

// 方式二：注解
@JSONField(format = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime createTime;

// 方式三：自定义格式
FastJsonConfig config = new FastJsonConfig();
config.setDateFormat("yyyy-MM-dd HH:mm:ss");
```

### Q4: 如何美化输出？

```java
String prettyJson = JSON.toJSONString(user, JSONWriter.Feature.PrettyFormat);
```

## 12. 最佳实践

```
Fastjson2 使用 Checklist:

□ 安全配置:
  - 使用 SafeMode 或配置 autoType 白名单
  - 不要信任未经验证的 JSON 输入
  - 定期更新版本修复安全漏洞

□ 性能优化:
  - 复用 JSONPath 实例
  - 内部通信使用 JSONB 格式
  - 大数据量使用流式 API

□ 代码规范:
  - 使用 @JSONField 注解明确字段映射
  - 日期格式统一配置
  - null 值处理策略统一

□ 测试覆盖:
  - 边界值测试（null、空串、特殊字符）
  - 泛型反序列化测试
  - 性能基准测试
```
