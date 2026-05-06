# JSON 处理框架概览与对比

## 1. JSON 简介

```
JSON (JavaScript Object Notation) 发展历程:

1999: Douglas Crockford 在 RFC 4627 中定义 JSON
2001: JSON.org 官网上线
2006: JSON 被广泛应用于 Ajax Web 应用
2013: ECMA-404 标准化
2014: RFC 7159 (JSON 数据交换格式)
2017: RFC 8259 (更新规范)

JSON 基本类型:
┌─────────────────────────────────────────────────────────────┐
│  类型       │  示例                                          │
├─────────────┼──────────────────────────────────────────────┤
│  字符串     │  "hello"                                      │
│  数字       │  42, 3.14, -1.5e10                            │
│  布尔值     │  true, false                                  │
│  null      │  null                                          │
│  对象       │  {"key": "value", "count": 10}                │
│  数组       │  [1, 2, 3], ["a", "b"]                        │
└─────────────────────────────────────────────────────────────┘
```

## 2. Java JSON 框架发展历史

```
Java JSON 框架演进:

2001: org.json - 最早的 Java JSON 库
2002: JSON-lib - 基于 JSONObject/JSONArray
2006: Jackson - 诞生，高性能流式 API
2008: Gson - Google 开源，面向对象设计
2011: Fastjson - 阿里开源，极致性能
2012: javax.json - Java EE 7 标准 API (JSR 353)
2017: JSON-B - Java EE 8 绑定 API (JSR 367)
2022: Fastjson2 - Fastjson 重写版，安全+性能

主流框架现状:
┌──────────────┬─────────────┬─────────────┬─────────────┐
│ 框架         │ 开发商      │ 最新版本    │ 活跃度      │
├──────────────┼─────────────┼─────────────┼─────────────┤
│ Jackson      │ FasterXML   │ 2.17.x     │ 极高        │
│ Gson         │ Google      │ 2.11.x     │ 高          │
│ Fastjson2    │ Alibaba     │ 2.0.x      │ 高          │
│ Moshi        │ Square      │ 1.15.x     │ 中          │
│ Json-lib     │ Apache      │ 2.4        │ 低(已停更)  │
└──────────────┴─────────────┴─────────────┴─────────────┘
```

## 3. 框架对比

### 3.1 功能特性对比

| 特性 | Jackson | Gson | Fastjson2 | Json-lib |
|------|---------|------|-----------|----------|
| **流式 API** | ✅ JsonParser/Generator | ✅ JsonReader/Writer | ✅ JSONReader/Writer | ❌ |
| **树模型** | ✅ JsonNode | ✅ JsonElement | ✅ JSONObject | ✅ JSONObject |
| **数据绑定** | ✅ ObjectMapper | ✅ Gson | ✅ JSON | ✅ JSONObject.toBean |
| **注解支持** | ✅ 丰富 | ✅ @SerializedName | ✅ 丰富 | ❌ 有限 |
| **自定义序列化** | ✅ Serializer/Deserializer | ✅ TypeAdapter | ✅ ObjectWriter | ✅ JsonConfig |
| **日期处理** | ✅ @JsonFormat | ✅ 自定义 | ✅ 自动 | ✅ JsonConfig |
| **泛型支持** | ✅ TypeReference | ✅ TypeToken | ✅ TypeReference | ❌ 有限 |
| **JSONPath** | ✅ jackson-jsonpath | ❌ | ✅ 内置 | ❌ |
| **JSON Schema** | ✅ jackson-jsonschema | ❌ | ✅ 内置 | ❌ |
| **二进制格式** | ✅ CBOR/Smile/Protobuf | ❌ | ✅ JSONB | ❌ |

### 3.2 性能对比

```
性能测试结果（10万次序列化/反序列化）:

序列化耗时（ms）:
┌──────────────┬────────────┬────────────┬────────────┐
│ 框架         │ 小对象     │ 中等对象   │ 大对象     │
├──────────────┼────────────┼────────────┼────────────┤
│ Jackson      │ 45         │ 120        │ 580        │
│ Gson         │ 68         │ 180        │ 850        │
│ Fastjson2    │ 38         │ 95         │ 460        │
│ Json-lib     │ 120        │ 350        │ 1800       │
└──────────────┴────────────┴────────────┴────────────┘

反序列化耗时（ms）:
┌──────────────┬────────────┬────────────┬────────────┐
│ 框架         │ 小对象     │ 中等对象   │ 大对象     │
├──────────────┼────────────┼────────────┼────────────┤
│ Jackson      │ 52         │ 145        │ 720        │
│ Gson         │ 75         │ 210        │ 1050       │
│ Fastjson2    │ 42         │ 110        │ 540        │
│ Json-lib     │ 150        │ 420        │ 2200       │
└──────────────┴────────────┴────────────┴────────────┘

性能排名: Fastjson2 > Jackson > Gson > Json-lib
```

### 3.3 安全性对比

```
安全漏洞历史:

Fastjson 1.x:
- 2017-2022: 多次反序列化漏洞 (CVE-2017-18349 等)
- 原因: autoType 功能允许任意类实例化
- 解决: 升级 Fastjson2 或关闭 autoType

Jackson:
- 2017: CVE-2017-7525 (PolymorphicDeserialization)
- 原因: 多态反序列化风险
- 解决: 启用 FAIL_ON_UNKNOWN_PROPERTIES, 禁用多态

Gson:
- 安全记录良好，设计上避免了反序列化漏洞
- 默认不支持多态反序列化

推荐:
- 新项目首选 Jackson 或 Gson
- Fastjson2 安全性已大幅改进，但仍需谨慎配置
- 避免使用 Json-lib（已停止维护）
```

### 3.4 框架选择建议

```
选型决策树:

需求分析
    │
    ├─ Spring Boot 项目？
    │   └─ 推荐 Jackson（官方默认，深度集成）
    │
    ├─ 追求极致性能？
    │   └─ 推荐 Fastjson2（性能最优）
    │
    ├─ Android 移动端？
    │   └─ 推荐 Gson（轻量、Google 官方）
    │
    ├─ 需要安全稳定？
    │   └─ 推荐 Jackson 或 Gson（安全记录好）
    │
    ├─ 需要二进制格式？
    │   └─ 推荐 Jackson（支持 CBOR/Smile）
    │
    └─ 遗留系统维护？
        └─ 根据现有选择，逐步迁移到 Jackson

综合推荐:
1. Jackson - 功能最全、生态最好、Spring 默认
2. Fastjson2 - 性能最优、API 简洁
3. Gson - 轻量级、安全稳定
```

## 4. 快速入门对比

### 4.1 基础用法对比

```java
// 定义测试类
public class User {
    private Long id;
    private String name;
    private String email;
    private LocalDateTime createTime;
    // getters/setters
}

User user = new User(1L, "张三", "zhangsan@example.com", LocalDateTime.now());
```

**Jackson:**
```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(new JavaTimeModule());

// 序列化
String json = mapper.writeValueAsString(user);

// 反序列化
User parsed = mapper.readValue(json, User.class);
```

**Gson:**
```java
import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

Gson gson = new GsonBuilder()
    .setDateFormat("yyyy-MM-dd HH:mm:ss")
    .create();

// 序列化
String json = gson.toJson(user);

// 反序列化
User parsed = gson.fromJson(json, User.class);
```

**Fastjson2:**
```java
import com.alibaba.fastjson2.JSON;

// 序列化
String json = JSON.toJSONString(user);

// 反序列化
User parsed = JSON.parseObject(json, User.class);
```

### 4.2 泛型处理对比

```java
// 场景：反序列化 List<User>
String json = "[{\"id\":1,\"name\":\"张三\"},{\"id\":2,\"name\":\"李四\"}]";

// Jackson
List<User> users = mapper.readValue(json, new TypeReference<List<User>>() {});

// Gson
List<User> users = gson.fromJson(json, new TypeToken<List<User>>() {}.getType());

// Fastjson2
List<User> users = JSON.parseObject(json, new TypeReference<List<User>>() {});
```

### 4.3 复杂对象处理

```java
// 场景：嵌套对象
public class Order {
    private Long orderId;
    private User buyer;
    private List<OrderItem> items;
    private Map<String, Object> extra;
}

// Jackson - 日期格式处理
ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(new JavaTimeModule());
mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

// Gson - 自定义 TypeAdapter
Gson gson = new GsonBuilder()
    .registerTypeAdapter(LocalDateTime.class, new LocalDateTimeAdapter())
    .serializeNulls()  // 序列化 null 值
    .create();

// Fastjson2 - 自动处理日期
String json = JSON.toJSONString(user, 
    JSONWriter.Feature.WriteClassName,  // 写入类型信息
    JSONWriter.Feature.WriteNulls       // 序列化 null 值
);
```

## 5. 注解对比

### 5.1 字段命名

```java
// Jackson
public class User {
    @JsonProperty("user_name")
    private String userName;
}

// Gson
public class User {
    @SerializedName("user_name")
    private String userName;
}

// Fastjson2
public class User {
    @JSONField(name = "user_name")
    private String userName;
}
```

### 5.2 忽略字段

```java
// Jackson
public class User {
    @JsonIgnore
    private String password;
    
    @JsonIgnoreProperties({"password", "salt"})
    private UserDetail detail;
}

// Gson
public class User {
    @Expose(serialize = false, deserialize = false)
    private String password;
}

// Fastjson2
public class User {
    @JSONField(serialize = false, deserialize = false)
    private String password;
}
```

### 5.3 日期格式

```java
// Jackson
public class User {
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Shanghai")
    private LocalDateTime createTime;
}

// Gson
Gson gson = new GsonBuilder()
    .setDateFormat("yyyy-MM-dd HH:mm:ss")
    .create();

// Fastjson2
public class User {
    @JSONField(format = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createTime;
}
```

## 6. 高级特性对比

### 6.1 自定义序列化器

```java
// Jackson
public class MoneySerializer extends JsonSerializer<BigDecimal> {
    @Override
    public void serialize(BigDecimal value, JsonGenerator gen, 
                          SerializerProvider provider) throws IOException {
        gen.writeString(value.setScale(2, RoundingMode.HALF_UP).toString());
    }
}

public class Product {
    @JsonSerialize(using = MoneySerializer.class)
    private BigDecimal price;
}

// Gson
public class MoneyAdapter implements JsonSerializer<BigDecimal>, 
                                      JsonDeserializer<BigDecimal> {
    @Override
    public JsonElement serialize(BigDecimal src, Type typeOfSrc, 
                                  JsonSerializationContext context) {
        return new JsonPrimitive(src.setScale(2, RoundingMode.HALF_UP));
    }
    
    @Override
    public BigDecimal deserialize(JsonElement json, Type typeOfT, 
                                   JsonDeserializationContext context) {
        return new BigDecimal(json.getAsJsonPrimitive().getAsString());
    }
}

Gson gson = new GsonBuilder()
    .registerTypeAdapter(BigDecimal.class, new MoneyAdapter())
    .create();

// Fastjson2
public class MoneyFilter implements ObjectWriter {
    @Override
    public void write(JSONWriter writer, Object object, String fieldName, 
                      Type fieldType, long features) {
        BigDecimal value = (BigDecimal) object;
        writer.writeString(value.setScale(2, RoundingMode.HALF_UP).toString());
    }
}
```

### 6.2 多态处理

```java
// Jackson - @JsonTypeInfo
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS, property = "@class")
@JsonSubTypes({
    @JsonSubTypes.Type(value = Dog.class, name = "dog"),
    @JsonSubTypes.Type(value = Cat.class, name = "cat")
})
public abstract class Animal {
    private String name;
}

// Gson - RuntimeTypeAdapterFactory
RuntimeTypeAdapterFactory<Animal> adapter = RuntimeTypeAdapterFactory
    .of(Animal.class, "type")
    .registerSubtype(Dog.class, "dog")
    .registerSubtype(Cat.class, "cat");

Gson gson = new GsonBuilder()
    .registerTypeAdapterFactory(adapter)
    .create();

// Fastjson2 - @type
String json = "{\"@type\":\"dog\",\"name\":\"旺财\"}";
Animal animal = JSON.parseObject(json, Animal.class);
```

### 6.3 JSON 树操作

```java
// Jackson - JsonNode
JsonNode root = mapper.readTree(json);
String name = root.get("name").asText();
int age = root.get("age").asInt();
JsonNode address = root.get("address");
String city = address.get("city").asText();

// 修改节点
((ObjectNode) root).put("name", "新名字");
String modified = mapper.writeValueAsString(root);

// Gson - JsonElement
JsonElement root = JsonParser.parseString(json);
String name = root.getAsJsonObject().get("name").getAsString();

// Fastjson2 - JSONObject
JSONObject root = JSON.parseObject(json);
String name = root.getString("name");
int age = root.getIntValue("age");

// 路径查询
Object value = root.extract("/address/city");
```

## 7. 生产环境注意事项

```
安全配置 Checklist:

□ 禁用不安全的反序列化:
  - Jackson: 禁用 ACCEPT_SINGLE_VALUE_AS_ARRAY (可选)
  - Fastjson: 关闭 autoType 或配置白名单
  - Gson: 默认安全，无需特殊配置

□ 配置未知属性处理:
  - Jackson: FAIL_ON_UNKNOWN_PROPERTIES = true (推荐)
  - 防止意外字段导致的安全问题

□ 日期格式统一:
  - 明确指定日期格式，避免依赖默认
  - 使用 ISO 8601 或自定义格式

□ null 值处理:
  - 序列化: 根据需求决定是否包含 null
  - 反序列化: 使用 Optional 或默认值

□ 字符编码:
  - 统一使用 UTF-8
  - 特殊字符转义处理

□ 性能优化:
  - ObjectMapper 单例复用
  - 大数据量使用流式 API
  - 考虑使用二进制格式（CBOR/Smile）
```

## 8. 框架迁移指南

### 8.1 Fastjson 迁移到 Fastjson2

```java
// 旧代码 (Fastjson 1.x)
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;

// 新代码 (Fastjson2)
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.JSONObject;

// API 基本兼容，主要变化:
// 1. 包名从 com.alibaba.fastjson 改为 com.alibaba.fastjson2
// 2. 自动类型处理更安全（需显式配置）
// 3. 性能提升 30-100%
```

### 8.2 Json-lib 迁移到 Jackson

```java
// 旧代码 (Json-lib)
import net.sf.json.JSONObject;
import net.sf.json.JSONArray;

JSONObject obj = JSONObject.fromObject(jsonStr);
String name = obj.getString("name");

// 新代码 (Jackson)
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.JsonNode;

ObjectMapper mapper = new ObjectMapper();
JsonNode obj = mapper.readTree(jsonStr);
String name = obj.get("name").asText();
```

## 9. 依赖配置

```xml
<!-- Jackson (推荐) -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.17.0</version>
</dependency>
<!-- Java 8 日期支持 -->
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
    <version>2.17.0</version>
</dependency>

<!-- Gson -->
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.11.0</version>
</dependency>

<!-- Fastjson2 -->
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2</artifactId>
    <version>2.0.50</version>
</dependency>

<!-- 不推荐：Json-lib (已停更) -->
<dependency>
    <groupId>net.sf.json-lib</groupId>
    <artifactId>json-lib</artifactId>
    <version>2.4</version>
    <classifier>jdk15</classifier>
</dependency>
```
