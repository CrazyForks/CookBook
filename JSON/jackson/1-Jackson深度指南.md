# Jackson 编程指南（深度版）

## 1. 概述

```
Jackson 核心模块:

┌─────────────────────────────────────────────────────────────┐
│                      Jackson 生态                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  │
│  │ jackson-core  │  │jackson-annot. │  │jackson-databind│  │
│  │  (流式 API)   │  │  (注解处理)   │  │  (数据绑定)    │  │
│  │              │  │              │  │              │  │
│  │ JsonParser   │  │ @JsonProperty │  │ ObjectMapper │  │
│  │ JsonGenerator│  │ @JsonIgnore  │  │ ObjectReader │  │
│  │ JsonFactory  │  │ @JsonFormat  │  │ ObjectWriter │  │
│  └───────────────┘  └───────────────┘  └───────────────┘  │
│                                                             │
│  扩展模块:                                                  │
│  ├── jackson-datatype-jsr310    # Java 8 日期时间          │
│  ├── jackson-datatype-joda      # Joda-Time               │
│  ├── jackson-module-kotlin      # Kotlin                  │
│  ├── jackson-module-scala_2.13  # Scala                   │
│  ├── jackson-dataformat-xml     # XML                     │
│  ├── jackson-dataformat-yaml    # YAML                    │
│  ├── jackson-dataformat-cbor    # CBOR 二进制             │
│  └── jackson-dataformat-smile   # Smile 二进制            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 环境配置

```xml
<!-- pom.xml -->
<properties>
    <jackson.version>2.17.0</jackson.version>
</properties>

<dependencies>
    <!-- 核心依赖 -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>${jackson.version}</version>
    </dependency>
    
    <!-- Java 8 日期时间支持 -->
    <dependency>
        <groupId>com.fasterxml.jackson.datatype</groupId>
        <artifactId>jackson-datatype-jsr310</artifactId>
        <version>${jackson.version}</version>
    </dependency>
    
    <!-- 可选：其他格式支持 -->
    <dependency>
        <groupId>com.fasterxml.jackson.dataformat</groupId>
        <artifactId>jackson-dataformat-yaml</artifactId>
        <version>${jackson.version}</version>
    </dependency>
    
    <!-- 可选：注解处理器 (编译时) -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-annotations</artifactId>
        <version>${jackson.version}</version>
    </dependency>
</dependencies>
```

## 3. ObjectMapper 配置

```java
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.annotation.JsonInclude;

@Configuration
public class JacksonConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        
        // 注册模块
        mapper.registerModule(new JavaTimeModule());
        
        // 序列化配置
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);  // 忽略 null
        mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);  // 日期格式化
        mapper.configure(SerializationFeature.INDENT_OUTPUT, true);  // 美化输出
        
        // 反序列化配置
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);  // 忽略未知字段
        mapper.configure(DeserializationFeature.ACCEPT_SINGLE_VALUE_AS_ARRAY, true);  // 单值数组
        mapper.configure(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES, true);  // 原始类型 null
        
        // 日期格式
        mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
        mapper.setTimeZone(TimeZone.getTimeZone("Asia/Shanghai"));
        
        // 自定义序列化器
        SimpleModule module = new SimpleModule();
        module.addSerializer(BigDecimal.class, new MoneySerializer());
        module.addDeserializer(LocalDateTime.class, new CustomLocalDateTimeDeserializer());
        mapper.registerModule(module);
        
        return mapper;
    }
}
```

## 4. 基础操作

### 4.1 序列化

```java
@Autowired
private ObjectMapper objectMapper;

// 1. 对象转 JSON 字符串
User user = new User(1L, "张三", "zhangsan@example.com");
String json = objectMapper.writeValueAsString(user);
// {"id":1,"name":"张三","email":"zhangsan@example.com"}

// 2. 对象转 JSON 字节
byte[] bytes = objectMapper.writeValueAsBytes(user);

// 3. 对象写入文件
objectMapper.writeValue(new File("user.json"), user);

// 4. 对象写入输出流
objectMapper.writeValue(response.getOutputStream(), user);

// 5. 美化输出
String prettyJson = objectMapper.writerWithDefaultPrettyPrinter()
    .writeValueAsString(user);

// 6. 指定特征输出
String json = objectMapper.writer()
    .without(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
    .writeValueAsString(user);
```

### 4.2 反序列化

```java
// 1. JSON 字符串转对象
String json = "{\"id\":1,\"name\":\"张三\"}";
User user = objectMapper.readValue(json, User.class);

// 2. JSON 字节转对象
byte[] bytes = json.getBytes();
User user = objectMapper.readValue(bytes, User.class);

// 3. 从文件读取
User user = objectMapper.readValue(new File("user.json"), User.class);

// 4. 从输入流读取
User user = objectMapper.readValue(inputStream, User.class);

// 5. 泛型处理
List<User> users = objectMapper.readValue(jsonArray, new TypeReference<List<User>>() {});

Map<String, Object> map = objectMapper.readValue(json, new TypeReference<Map<String, Object>>() {});

// 6. 配置读取特征
User user = objectMapper.reader()
    .without(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
    .readValue(json, User.class);
```

## 5. 注解详解

### 5.1 属性控制

```java
public class User {
    
    // 指定 JSON 字段名
    @JsonProperty("user_id")
    private Long id;
    
    // 忽略字段
    @JsonIgnore
    private String password;
    
    // 序列化时忽略，反序列化时正常
    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    private String secret;
    
    // 设置默认值
    @JsonProperty(defaultValue = "0")
    private Integer age;
    
    // 必填字段
    @JsonProperty(required = true)
    private String name;
    
    // 忽略多个字段（类级别）
    @JsonIgnoreProperties({"password", "salt", "token"})
    private UserDetail detail;
}
```

### 5.2 日期格式

```java
public class Order {
    
    // 标准日期格式
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Shanghai")
    private LocalDateTime createTime;
    
    // 日期类型
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate birthday;
    
    // 时间类型
    @JsonFormat(pattern = "HH:mm:ss")
    private LocalTime updateTime;
    
    // 时间戳格式
    @JsonFormat(shape = JsonFormat.Shape.NUMBER)
    private Instant timestamp;
}
```

### 5.3 空值处理

```java
public class User {
    
    // null 时序列化为空字符串
    @JsonSerialize(nullsUsing = NullStringSerializer.class)
    private String name;
    
    // null 时序列化为 0
    @JsonSerialize(nullsUsing = NullIntegerSerializer.class)
    private Integer age;
    
    // null 时序列化为空数组
    @JsonSerialize(nullsUsing = NullListSerializer.class)
    private List<String> tags;
}

public class NullStringSerializer extends JsonSerializer<Object> {
    @Override
    public void serialize(Object value, JsonGenerator gen, 
                          SerializerProvider provider) throws IOException {
        gen.writeString("");
    }
}
```

### 5.4 多态处理

```java
// 方式一：CLASS 方式（完整类名）
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS, property = "@class")
@JsonSubTypes({
    @JsonSubTypes.Type(value = Dog.class, name = "com.example.Dog"),
    @JsonSubTypes.Type(value = Cat.class, name = "com.example.Cat")
})
public abstract class Animal {
    private String name;
}

// 方式二：NAME 方式（简洁名称）
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = Dog.class, name = "dog"),
    @JsonSubTypes.Type(value = Cat.class, name = "cat")
})
public abstract class Animal {
    private String name;
}

// 方式三：MINIMAL_CLASS 方式（最短类名）
@JsonTypeInfo(use = JsonTypeInfo.Id.MINIMAL_CLASS, property = "@c")
public abstract class Animal {
    private String name;
}
```

## 6. 树模型操作

```java
// 创建节点
ObjectMapper mapper = new ObjectMapper();
ObjectNode root = mapper.createObjectNode();
root.put("id", 1);
root.put("name", "张三");

// 嵌套对象
ObjectNode address = mapper.createObjectNode();
address.put("city", "北京");
address.put("street", "朝阳区");
root.set("address", address);

// 数组
ArrayNode hobbies = mapper.createArrayNode();
hobbies.add("reading");
hobbies.add("coding");
root.set("hobbies", hobbies);

// 读取节点
JsonNode node = mapper.readTree(jsonString);
String name = node.get("name").asText();
int age = node.get("age").asInt(0);  // 默认值 0
String city = node.at("/address/city").asText();  // JSON Pointer

// 修改节点
((ObjectNode) node).put("name", "新名字");
((ObjectNode) node).remove("email");

// 遍历节点
JsonNode arrayNode = node.get("hobbies");
for (JsonNode item : arrayNode) {
    System.out.println(item.asText());
}

// 转换为对象
User user = mapper.treeToValue(node, User.class);

// 对象转换为节点
JsonNode node = mapper.valueToTree(user);
```

## 7. 流式 API

```java
// 低级 API：JsonParser（解析器）
String json = "{\"name\":\"张三\",\"age\":25}";

JsonFactory factory = new JsonFactory();
try (JsonParser parser = factory.createParser(json)) {
    while (parser.nextToken() != null) {
        JsonToken token = parser.currentToken();
        if (token == JsonToken.FIELD_NAME) {
            String fieldName = parser.getCurrentName();
            parser.nextToken();  // 移动到值
            if ("name".equals(fieldName)) {
                String value = parser.getText();
            } else if ("age".equals(fieldName)) {
                int value = parser.getIntValue();
            }
        }
    }
}

// 低级 API：JsonGenerator（生成器）
StringWriter writer = new StringWriter();
try (JsonGenerator generator = factory.createGenerator(writer)) {
    generator.writeStartObject();  // {
    generator.writeStringField("name", "张三");  // "name":"张三"
    generator.writeNumberField("age", 25);  // "age":25
    generator.writeFieldName("hobbies");  // "hobbies":
    generator.writeStartArray();  // [
    generator.writeString("reading");
    generator.writeString("coding");
    generator.writeEndArray();  // ]
    generator.writeEndObject();  // }
}
String json = writer.toString();
```

## 8. 高级特性

### 8.1 自定义序列化器

```java
// 金额序列化器
public class MoneySerializer extends JsonSerializer<BigDecimal> {
    @Override
    public void serialize(BigDecimal value, JsonGenerator gen, 
                          SerializerProvider provider) throws IOException {
        if (value == null) {
            gen.writeNull();
        } else {
            gen.writeString("¥" + value.setScale(2, RoundingMode.HALF_UP));
        }
    }
}

// 使用
public class Product {
    @JsonSerialize(using = MoneySerializer.class)
    private BigDecimal price;
}

// 全局注册
SimpleModule module = new SimpleModule("MoneyModule");
module.addSerializer(BigDecimal.class, new MoneySerializer());
objectMapper.registerModule(module);
```

### 8.2 自定义反序列化器

```java
// 日期反序列化器
public class CustomDateDeserializer extends JsonDeserializer<Date> {
    
    private static final String[] DATE_FORMATS = {
        "yyyy-MM-dd",
        "yyyy-MM-dd HH:mm:ss",
        "yyyy/MM/dd"
    };
    
    @Override
    public Date deserialize(JsonParser p, DeserializationContext ctxt) 
            throws IOException {
        String dateStr = p.getText();
        
        for (String format : DATE_FORMATS) {
            try {
                SimpleDateFormat sdf = new SimpleDateFormat(format);
                sdf.setLenient(false);
                return sdf.parse(dateStr);
            } catch (ParseException ignored) {
            }
        }
        
        throw new JsonParseException(p, "无法解析日期: " + dateStr);
    }
}

// 使用
public class Event {
    @JsonDeserialize(using = CustomDateDeserializer.class)
    private Date eventTime;
}
```

### 8.3 混入（Mixin）

```java
// 不修改原始类，通过 Mixin 添加注解
public abstract class UserMixin {
    @JsonIgnore
    abstract String getPassword();
    
    @JsonProperty("user_name")
    abstract String getUserName();
}

// 注册 Mixin
objectMapper.addMixIn(User.class, UserMixin.class);
```

### 8.4 JSON 视图

```java
// 定义视图
public class Views {
    public static class Public {}
    public static class Internal extends Public {}
}

public class User {
    @JsonView(Views.Public.class)
    private Long id;
    
    @JsonView(Views.Public.class)
    private String name;
    
    @JsonView(Views.Internal.class)
    private String email;
    
    @JsonView(Views.Internal.class)
    private String password;
}

// 序列化指定视图
String publicJson = objectMapper.writerWithView(Views.Public.class)
    .writeValueAsString(user);
// {"id":1,"name":"张三"}

String internalJson = objectMapper.writerWithView(Views.Internal.class)
    .writeValueAsString(user);
// {"id":1,"name":"张三","email":"zhangsan@example.com","password":"123456"}
```

## 9. Spring Boot 集成

```yaml
# application.yml
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
    default-property-inclusion: non_null
    serialization:
      write-dates-as-timestamps: false
      indent-output: true
    deserialization:
      fail-on-unknown-properties: false
      accept-single-value-as-array: true
    mapper:
      accept-case-insensitive-enums: true
```

```java
// 全局配置
@Configuration
public class JacksonConfig {
    
    @Bean
    public ObjectMapper objectMapper(Jackson2ObjectMapperBuilder builder) {
        ObjectMapper mapper = builder.build();
        mapper.registerModule(new JavaTimeModule());
        return mapper;
    }
    
    // 自定义 HttpMessageConverter
    @Bean
    public MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter() {
        MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
        converter.setObjectMapper(objectMapper());
        return converter;
    }
}
```

## 10. 性能优化

```
性能优化建议:

1. ObjectMapper 单例:
   - ObjectMapper 是线程安全的，应该复用
   - Spring Boot 自动管理，直接注入即可

2. 使用 ObjectReader/ObjectWriter:
   - 可以预配置，减少重复配置开销
   - 适合特定场景的定制化处理

3. 流式 API:
   - 大数据量时使用 JsonParser/JsonGenerator
   - 避免将整个 JSON 加载到内存

4. 二进制格式:
   - 内部通信使用 CBOR/Smile 格式
   - 比 JSON 更紧凑、更快

5. 缓存配置:
   - 使用 TypeFactory 缓存类型信息
   - 避免重复创建 JavaType

6. 异步处理:
   - 使用 Jackson 2.10+ 的异步 API
   - 非阻塞读写提升吞吐量
```

```java
// 性能测试示例
@Test
public void performanceTest() throws Exception {
    ObjectMapper mapper = new ObjectMapper();
    User user = new User(1L, "张三", "zhangsan@example.com");
    
    // 预热
    for (int i = 0; i < 1000; i++) {
        mapper.writeValueAsString(user);
    }
    
    // 测试序列化
    long start = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
        mapper.writeValueAsString(user);
    }
    long end = System.currentTimeMillis();
    System.out.println("序列化 10万次: " + (end - start) + "ms");
    
    // 测试反序列化
    String json = mapper.writeValueAsString(user);
    start = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
        mapper.readValue(json, User.class);
    }
    end = System.currentTimeMillis();
    System.out.println("反序列化 10万次: " + (end - start) + "ms");
}
```

## 11. 常见问题

### Q1: 日期序列化为时间戳怎么办？

```java
// 方案一：全局配置
mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);

// 方案二：注解
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime createTime;

// 方案三：自定义序列化器
```

### Q2: 如何处理 null 值？

```java
// 方案一：全局忽略 null
mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);

// 方案二：注解控制
@JsonInclude(JsonInclude.Include.NON_NULL)
private String name;

// 方案三：自定义 null 序列化器
```

### Q3: 如何忽略未知字段？

```java
// 方案一：全局配置
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

// 方案二：注解
@JsonIgnoreProperties(ignoreUnknown = true)
public class User { }
```

### Q4: 泛型擦除怎么办？

```java
// 使用 TypeReference
List<User> users = mapper.readValue(json, new TypeReference<List<User>>() {});

// 使用 JavaType
JavaType type = mapper.getTypeFactory()
    .constructCollectionType(List.class, User.class);
List<User> users = mapper.readValue(json, type);
```
