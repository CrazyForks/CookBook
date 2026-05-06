# Spring Boot JSON 集成最佳实践

## 1. 概述

```
Spring Boot JSON 支持:

┌─────────────────────────────────────────────────────────────┐
│              Spring Boot JSON 自动配置                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  默认框架: Jackson（spring-boot-starter-web 内置）          │
│                                                             │
│  自动配置:                                                  │
│  ├── JacksonAutoConfiguration                               │
│  │   └── ObjectMapper 自动配置                              │
│  ├── JacksonHttpMessageConvertersConfiguration              │
│  │   └── MappingJackson2HttpMessageConverter                │
│  └── 支持 application.yml 配置                              │
│                                                             │
│  可选框架:                                                  │
│  ├── Gson                                                   │
│  ├── Fastjson2                                              │
│  └── Jsonb (JSON-B)                                        │
│                                                             │
│  切换方式:                                                  │
│  1. 排除 Jackson 依赖                                       │
│  2. 引入其他框架依赖                                        │
│  3. 配置 HttpMessageConverter                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. Jackson 集成（推荐）

### 2.1 自动配置

```yaml
# application.yml
spring:
  jackson:
    # 日期格式
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
    
    # 序列化配置
    default-property-inclusion: non_null  # non_null, non_empty, non_default
    serialization:
      write-dates-as-timestamps: false
      write-enums-using-to-string: true
      indent-output: false  # 生产环境关闭美化
    
    # 反序列化配置
    deserialization:
      fail-on-unknown-properties: false
      accept-single-value-as-array: true
      adjust-dates-to-context-time-zone: true
    
    # Mapper 配置
    mapper:
      accept-case-insensitive-enums: true
      allow-implicit-creation-of-subtypes: true
    
    # Generator 配置
    generator:
      write-numbers-as-strings: false
```

### 2.2 自定义 ObjectMapper

```java
@Configuration
public class JacksonConfig {
    
    @Bean
    @Primary
    public ObjectMapper objectMapper(Jackson2ObjectMapperBuilder builder) {
        ObjectMapper mapper = builder.build();
        
        // 注册 Java 8 时间模块
        mapper.registerModule(new JavaTimeModule());
        
        // 自定义序列化器
        SimpleModule module = new SimpleModule("CustomModule");
        module.addSerializer(BigDecimal.class, new MoneySerializer());
        module.addDeserializer(LocalDateTime.class, new FlexibleLocalDateTimeDeserializer());
        mapper.registerModule(module);
        
        // 配置特性
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        
        // 自定义 Mixin
        mapper.addMixIn(User.class, UserMixin.class);
        
        return mapper;
    }
}
```

### 2.3 统一响应格式

```java
// 统一响应对象
@Data
public class ApiResponse<T> {
    
    @JsonProperty("code")
    private int code;
    
    @JsonProperty("message")
    private String message;
    
    @JsonProperty("data")
    @JsonInclude(JsonInclude.Include.NON_NULL)
    private T data;
    
    @JsonProperty("timestamp")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime timestamp;
    
    public static <T> ApiResponse<T> success(T data) {
        ApiResponse<T> response = new ApiResponse<>();
        response.setCode(200);
        response.setMessage("success");
        response.setData(data);
        response.setTimestamp(LocalDateTime.now());
        return response;
    }
    
    public static <T> ApiResponse<T> error(int code, String message) {
        ApiResponse<T> response = new ApiResponse<>();
        response.setCode(code);
        response.setMessage(message);
        response.setTimestamp(LocalDateTime.now());
        return response;
    }
}

// Controller 使用
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}")
    public ApiResponse<User> getUser(@PathVariable Long id) {
        User user = userService.getUser(id);
        return ApiResponse.success(user);
    }
}
```

## 3. Gson 集成

### 3.1 排除 Jackson，引入 Gson

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-json</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
</dependency>
```

### 3.2 配置 Gson

```java
@Configuration
public class GsonConfig {
    
    @Bean
    public Gson gson() {
        return new GsonBuilder()
            .setPrettyPrinting()
            .setDateFormat("yyyy-MM-dd HH:mm:ss")
            .serializeNulls()
            .registerTypeAdapter(LocalDateTime.class, new LocalDateTimeAdapter())
            .registerTypeAdapter(LocalDate.class, new LocalDateAdapter())
            .create();
    }
    
    @Bean
    public GsonHttpMessageConverter gsonHttpMessageConverter(Gson gson) {
        GsonHttpMessageConverter converter = new GsonHttpMessageConverter();
        converter.setGson(gson);
        return converter;
    }
}
```

## 4. Fastjson2 集成

### 4.1 添加依赖

```xml
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2-extension-spring6</artifactId>
    <version>2.0.50</version>
</dependency>
```

### 4.2 配置 Fastjson2

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
            JSONWriter.Feature.WriteDateUseDateFormat,
            JSONWriter.Feature.IgnoreErrorGetter
        );
        config.setReaderFeatures(
            JSONReader.Feature.SafeMode
        );
        
        // 配置支持的媒体类型
        List<MediaType> mediaTypes = new ArrayList<>();
        mediaTypes.add(MediaType.APPLICATION_JSON);
        converter.setSupportedMediaTypes(mediaTypes);
        converter.setFastJsonConfig(config);
        
        return converter;
    }
}
```

## 5. 自定义序列化器

### 5.1 金额序列化器

```java
// Jackson 版本
public class MoneySerializer extends JsonSerializer<BigDecimal> {
    @Override
    public void serialize(BigDecimal value, JsonGenerator gen, 
                          SerializerProvider provider) throws IOException {
        if (value != null) {
            gen.writeString("¥" + value.setScale(2, RoundingMode.HALF_UP).toPlainString());
        }
    }
}

// 使用
public class Product {
    @JsonSerialize(using = MoneySerializer.class)
    private BigDecimal price;
}

// 全局注册
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        SimpleModule module = new SimpleModule();
        module.addSerializer(BigDecimal.class, new MoneySerializer());
        mapper.registerModule(module);
        return mapper;
    }
}
```

### 5.2 枚举序列化器

```java
// 枚举基类
public interface EnumValue {
    int getCode();
    String getDesc();
}

// 枚举
public enum Status implements EnumValue {
    ACTIVE(1, "激活"),
    INACTIVE(0, "未激活");
    
    private final int code;
    private final String desc;
    
    // constructor, getters
}

// 枚举序列化器
public class EnumValueSerializer extends JsonSerializer<EnumValue> {
    @Override
    public void serialize(EnumValue value, JsonGenerator gen, 
                          SerializerProvider provider) throws IOException {
        gen.writeStartObject();
        gen.writeNumberField("code", value.getCode());
        gen.writeStringField("desc", value.getDesc());
        gen.writeEndObject();
    }
}

// 枚举反序列化器
public class EnumValueDeserializer<T extends Enum<T> & EnumValue> 
        extends JsonDeserializer<T> {
    
    private final Class<T> enumType;
    
    public EnumValueDeserializer(Class<T> enumType) {
        this.enumType = enumType;
    }
    
    @Override
    public T deserialize(JsonParser p, DeserializationContext ctxt) 
            throws IOException {
        int code = p.getIntValue();
        for (T enumConstant : enumType.getEnumConstants()) {
            if (enumConstant.getCode() == code) {
                return enumConstant;
            }
        }
        throw new JsonParseException(p, "未知枚举值: " + code);
    }
}
```

## 6. JSON 视图

```java
// 定义视图
public class Views {
    public interface Public {}
    public interface Internal extends Public {}
    public interface Admin extends Internal {}
}

// 实体类
public class User {
    @JsonView(Views.Public.class)
    private Long id;
    
    @JsonView(Views.Public.class)
    private String name;
    
    @JsonView(Views.Internal.class)
    private String email;
    
    @JsonView(Views.Admin.class)
    private String password;
    
    @JsonView(Views.Admin.class)
    private String internalNote;
}

// Controller
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @JsonView(Views.Public.class)
    @GetMapping("/public/{id}")
    public User getPublicUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
    
    @JsonView(Views.Internal.class)
    @GetMapping("/internal/{id}")
    public User getInternalUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
    
    @JsonView(Views.Admin.class)
    @GetMapping("/admin/{id}")
    public User getAdminUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
}
```

## 7. JSONP 支持

```java
// 配置 JSONP
@Configuration
public class WebConfig {
    
    @Bean
    public WebMvcConfigurer webMvcConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
                configurer.mediaType("js", MediaType.APPLICATION_JSON);
            }
        };
    }
}

// Controller
@RestController
public class JsonpController {
    
    @GetMapping(value = "/api/data", produces = "application/javascript")
    public MappingJacksonValue getData(@RequestParam String callback) {
        Data data = dataService.getData();
        
        MappingJacksonValue value = new MappingJacksonValue(data);
        value.setJsonpFunction(callback);
        
        return value;
    }
}
```

## 8. 错误处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ApiResponse<Void> handleJsonParseException(HttpMessageNotReadableException e) {
        String message = "请求参数格式错误";
        
        if (e.getCause() instanceof JsonParseException) {
            message = "JSON 格式错误";
        } else if (e.getCause() instanceof JsonMappingException) {
            JsonMappingException jme = (JsonMappingException) e.getCause();
            message = "字段映射错误: " + jme.getPath().stream()
                .map(JsonMappingException.Reference::getFieldName)
                .collect(Collectors.joining("."));
        }
        
        return ApiResponse.error(400, message);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ApiResponse<Void> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.joining(", "));
        
        return ApiResponse.error(400, message);
    }
}
```

## 9. 性能优化

```
性能优化建议:

1. ObjectMapper 单例:
   - ObjectMapper 是线程安全的，应该复用
   - Spring Boot 自动管理，通过 @Bean 注入

2. 避免重复创建:
   - 不要在循环中创建 ObjectMapper
   - 使用 ObjectReader/ObjectWriter 预配置

3. 流式 API:
   - 大数据量使用 @JsonRawValue 或流式处理
   - 避免将整个 JSON 加载到内存

4. 压缩传输:
   - 启用 Gzip 压缩
   - 使用 CBOR/Smile 二进制格式（内部通信）

5. 缓存:
   - 缓存序列化结果（如 Redis）
   - 使用 ETag/Last-Modified 减少重复序列化

6. 异步处理:
   - 使用 WebFlux 响应式编程
   - 异步序列化减少线程阻塞
```

## 10. 安全配置

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public ObjectMapper secureObjectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        
        // 禁用不安全的特性
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, true);
        mapper.configure(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES, true);
        
        // 设置安全限制
        mapper.getFactory().setStreamReadConstraints(
            StreamReadConstraints.builder()
                .maxStringLength(1000000)      // 最大字符串长度 1MB
                .maxNumberLength(1000)         // 最大数字长度
                .maxNestingDepth(100)          // 最大嵌套深度
                .build()
        );
        
        return mapper;
    }
}
```

## 11. 测试

```java
@SpringBootTest
@AutoConfigureMockMvc
public class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    public void testGetUser() throws Exception {
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(200))
            .andExpect(jsonPath("$.data.name").value("张三"))
            .andExpect(jsonPath("$.data.email").doesNotExist());  // 视图过滤
    }
    
    @Test
    public void testCreateUser() throws Exception {
        User user = new User();
        user.setName("李四");
        user.setEmail("lisi@example.com");
        
        mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(user)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(200));
    }
    
    @Test
    public void testInvalidJson() throws Exception {
        String invalidJson = "{\"name\":\"张三\",\"email\":}";  // 格式错误
        
        mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content(invalidJson))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(400))
            .andExpect(jsonPath("$.message").value("JSON 格式错误"));
    }
}
```

## 12. 最佳实践总结

```
JSON 集成 Checklist:

□ 框架选择:
  - 新项目首选 Jackson（Spring Boot 默认）
  - 性能敏感场景考虑 Fastjson2
  - Android 开发使用 Gson

□ 配置管理:
  - 统一日期格式（yyyy-MM-dd HH:mm:ss）
  - 统一 null 值处理策略
  - 配置错误处理策略

□ 安全防护:
  - 限制 JSON 解析深度和大小
  - 禁用不安全的反序列化特性
  - 输入验证和转义

□ 性能优化:
  - ObjectMapper/Gson 单例复用
  - 大数据量使用流式 API
  - 考虑二进制格式（CBOR/Smile/JSONB）

□ 代码规范:
  - 使用注解明确字段映射
  - 统一响应格式
  - 编写单元测试

□ 监控告警:
  - 记录 JSON 解析异常
  - 监控序列化耗时
  - 告警安全漏洞
```
