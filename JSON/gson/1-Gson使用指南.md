# Gson 使用指南

## 1. 概述

```
Gson 特点:

┌─────────────────────────────────────────────────────────────┐
│                      Gson 核心特性                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✓ Google 开源，稳定可靠                                    │
│  ✓ 轻量级，无额外依赖                                      │
│  ✓ 简洁的 API 设计                                         │
│  ✓ 强大的泛型支持（TypeToken）                              │
│  ✓ 自定义序列化器（TypeAdapter）                            │
│  ✓ 安全的反序列化（默认不支持多态）                          │
│  ✓ 适合 Android 开发                                       │
│                                                             │
│  核心类:                                                    │
│  Gson          - 主要入口，线程安全                          │
│  GsonBuilder   - 配置构建器                                │
│  TypeToken     - 泛型类型信息保留                          │
│  TypeAdapter   - 自定义序列化/反序列化                      │
│  JsonElement   - 树模型节点                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 环境配置

```xml
<!-- Maven -->
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.11.0</version>
</dependency>

<!-- Gradle -->
implementation 'com.google.code.gson:gson:2.11.0'
```

## 3. 基础使用

### 3.1 创建 Gson 实例

```java
// 方式一：默认配置
Gson gson = new Gson();

// 方式二：Builder 配置
Gson gson = new GsonBuilder()
    .setPrettyPrinting()                    // 美化输出
    .setDateFormat("yyyy-MM-dd HH:mm:ss")  // 日期格式
    .serializeNulls()                       // 序列化 null 值
    .disableHtmlEscaping()                  // 禁用 HTML 转义
    .create();

// 方式三：单例（推荐，线程安全）
public class GsonHolder {
    public static final Gson GSON = new GsonBuilder()
        .setPrettyPrinting()
        .setDateFormat("yyyy-MM-dd HH:mm:ss")
        .serializeNulls()
        .create();
}
```

### 3.2 序列化

```java
User user = new User(1L, "张三", "zhangsan@example.com");

// 对象转 JSON 字符串
String json = gson.toJson(user);
// {"id":1,"name":"张三","email":"zhangsan@example.com"}

// 对象转 JSON（带类型信息）
String json = gson.toJson(user, User.class);

// 集合转 JSON
List<User> users = Arrays.asList(user1, user2);
String json = gson.toJson(users);

// Map 转 JSON
Map<String, Object> map = new HashMap<>();
map.put("name", "张三");
map.put("age", 25);
String json = gson.toJson(map);

// 对象转 JsonElement
JsonElement element = gson.toJsonTree(user);
```

### 3.3 反序列化

```java
String json = "{\"id\":1,\"name\":\"张三\",\"email\":\"zhangsan@example.com\"}";

// JSON 字符串转对象
User user = gson.fromJson(json, User.class);

// JSON 字符串转 Map
Map<String, Object> map = gson.fromJson(json, new TypeToken<Map<String, Object>>() {});

// JSON 字符串转 List
String jsonArray = "[{\"id\":1},{\"id\":2}]";
List<User> users = gson.fromJson(jsonArray, new TypeToken<List<User>>() {});

// JSON 字符串转 JsonElement
JsonElement element = JsonParser.parseString(json);
JsonObject obj = element.getAsJsonObject();
String name = obj.get("name").getAsString();
```

## 4. 注解详解

### 4.1 @SerializedName

```java
public class User {
    
    // 简单映射
    @SerializedName("user_name")
    private String userName;
    
    // 多名称兼容（反序列化时）
    @SerializedName(value = "email", alternate = {"mail", "e_mail"})
    private String email;
}
```

### 4.2 @Expose

```java
// 需要配合 GsonBuilder 使用
Gson gson = new GsonBuilder()
    .excludeFieldsWithoutExposeAnnotation()
    .create();

public class User {
    @Expose
    private Long id;           // 序列化和反序列化都参与
    
    @Expose(serialize = true, deserialize = false)
    private String name;       // 只序列化
    
    @Expose(serialize = false, deserialize = true)
    private String password;   // 只反序列化
    
    // 没有 @Expose 注解的字段完全忽略
    private String internal;
}
```

### 4.3 @Since / @Until

```java
Gson gson = new GsonBuilder()
    .setVersion(2.0)  // 只处理 @Since(<=2.0) 和 @Until(>2.0) 的字段
    .create();

public class User {
    @Since(1.0)
    private Long id;
    
    @Since(2.0)
    private String email;
    
    @Until(2.0)  // 2.0 版本之前有效
    private String oldField;
}
```

## 5. 自定义 TypeAdapter

### 5.1 自定义日期适配器

```java
public class LocalDateAdapter implements JsonSerializer<LocalDate>, 
                                          JsonDeserializer<LocalDate> {
    
    private static final DateTimeFormatter FORMATTER = DateTimeFormatter.ISO_LOCAL_DATE;
    
    @Override
    public JsonElement serialize(LocalDate date, Type typeOfSrc, 
                                  JsonSerializationContext context) {
        return new JsonPrimitive(date.format(FORMATTER));
    }
    
    @Override
    public LocalDate deserialize(JsonElement json, Type typeOfT, 
                                  JsonDeserializationContext context) 
            throws JsonParseException {
        return LocalDate.parse(json.getAsString(), FORMATTER);
    }
}

// 注册
Gson gson = new GsonBuilder()
    .registerTypeAdapter(LocalDate.class, new LocalDateAdapter())
    .registerTypeAdapter(LocalDateTime.class, new LocalDateTimeAdapter())
    .create();
```

### 5.2 自定义枚举适配器

```java
public class EnumAdapter<T extends Enum<T>> implements JsonSerializer<T>, 
                                                         JsonDeserializer<T> {
    
    private final Class<T> clazz;
    
    public EnumAdapter(Class<T> clazz) {
        this.clazz = clazz;
    }
    
    @Override
    public JsonElement serialize(T value, Type typeOfSrc, 
                                  JsonSerializationContext context) {
        return new JsonPrimitive(value.name().toLowerCase());
    }
    
    @Override
    public T deserialize(JsonElement json, Type typeOfT, 
                          JsonDeserializationContext context) 
            throws JsonParseException {
        String value = json.getAsString().toUpperCase();
        try {
            return Enum.valueOf(clazz, value);
        } catch (IllegalArgumentException e) {
            throw new JsonParseException("未知枚举值: " + value);
        }
    }
}

// 注册
Gson gson = new GsonBuilder()
    .registerTypeAdapter(Status.class, new EnumAdapter<>(Status.class))
    .create();
```

### 5.3 高性能 TypeAdapter

```java
// TypeAdapter 比 JsonSerializer/JsonDeserializer 更高效
public class UserAdapter extends TypeAdapter<User> {
    
    @Override
    public void write(JsonWriter out, User user) throws IOException {
        out.beginObject();
        out.name("id").value(user.getId());
        out.name("name").value(user.getName());
        out.name("email").value(user.getEmail());
        
        if (user.getAddress() != null) {
            out.name("address");
            out.beginObject();
            out.name("city").value(user.getAddress().getCity());
            out.name("street").value(user.getAddress().getStreet());
            out.endObject();
        }
        
        out.endObject();
    }
    
    @Override
    public User read(JsonReader in) throws IOException {
        User user = new User();
        
        in.beginObject();
        while (in.hasNext()) {
            String name = in.nextName();
            switch (name) {
                case "id":
                    user.setId(in.nextLong());
                    break;
                case "name":
                    user.setName(in.nextString());
                    break;
                case "email":
                    user.setEmail(in.nextString());
                    break;
                case "address":
                    user.setAddress(readAddress(in));
                    break;
                default:
                    in.skipValue();
                    break;
            }
        }
        in.endObject();
        
        return user;
    }
    
    private Address readAddress(JsonReader in) throws IOException {
        Address address = new Address();
        in.beginObject();
        while (in.hasNext()) {
            String name = in.nextName();
            if ("city".equals(name)) {
                address.setCity(in.nextString());
            } else if ("street".equals(name)) {
                address.setStreet(in.nextString());
            } else {
                in.skipValue();
            }
        }
        in.endObject();
        return address;
    }
}
```

## 6. 树模型操作

```java
// 解析 JSON
String json = "{\"name\":\"张三\",\"age\":25,\"hobbies\":[\"reading\",\"coding\"]}";
JsonObject root = JsonParser.parseString(json).getAsJsonObject();

// 读取基本类型
String name = root.get("name").getAsString();
int age = root.get("age").getAsInt();
boolean active = root.has("active") && root.get("active").getAsBoolean();

// 读取数组
JsonArray hobbies = root.getAsJsonArray("hobbies");
for (JsonElement hobby : hobbies) {
    System.out.println(hobby.getAsString());
}

// 读取嵌套对象
JsonObject address = root.getAsJsonObject("address");
String city = address.get("city").getAsString();

// 创建 JSON 对象
JsonObject newRoot = new JsonObject();
newRoot.addProperty("id", 1);
newRoot.addProperty("name", "张三");

JsonArray tags = new JsonArray();
tags.add("java");
tags.add("spring");
newRoot.add("tags", tags);

JsonObject address = new JsonObject();
address.addProperty("city", "北京");
newRoot.add("address", address);

String result = new Gson().toJson(newRoot);

// 修改 JSON
root.addProperty("name", "李四");
root.remove("email");

// 合并 JSON
JsonObject other = new JsonObject();
other.addProperty("email", "lisi@example.com");
for (Map.Entry<String, JsonElement> entry : other.entrySet()) {
    root.add(entry.getKey(), entry.getValue());
}
```

## 7. 泛型处理

```java
// TypeToken 保留泛型信息
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;
    // getters/setters
}

String json = "{\"code\":200,\"message\":\"success\",\"data\":{\"id\":1,\"name\":\"张三\"}}";

// 正确的泛型反序列化
Type type = new TypeToken<ApiResponse<User>>() {}.getType();
ApiResponse<User> response = gson.fromJson(json, type);

// List 泛型
Type listType = new TypeToken<List<User>>() {}.getType();
List<User> users = gson.fromJson(jsonArray, listType);

// Map 泛型
Type mapType = new TypeToken<Map<String, List<User>>>() {}.getType();
Map<String, List<User>> map = gson.fromJson(json, mapType);
```

## 8. 空值处理

```java
// 序列化 null 值
Gson gson = new GsonBuilder()
    .serializeNulls()  // 默认忽略 null
    .create();

User user = new User();
user.setName("张三");
user.setEmail(null);

String json = gson.toJson(user);
// {"name":"张三","email":null}  (serializeNulls=true)
// {"name":"张三"}  (serializeNulls=false, 默认)

// 自定义 null 处理
Gson gson = new GsonBuilder()
    .registerTypeAdapter(String.class, new JsonSerializer<String>() {
        @Override
        public JsonElement serialize(String src, Type typeOfSrc, 
                                      JsonSerializationContext context) {
            return src == null ? new JsonPrimitive("") : new JsonPrimitive(src);
        }
    })
    .create();
```

## 9. Spring Boot 集成

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

// application.yml
spring:
  gson:
    date-format: yyyy-MM-dd HH:mm:ss
    pretty-printing: true
    serialize-nulls: true
```

## 10. 常见问题

### Q1: 如何处理泛型擦除？

```java
// 使用 TypeToken
Type type = new TypeToken<List<User>>() {}.getType();
List<User> users = gson.fromJson(json, type);
```

### Q2: 如何忽略特定字段？

```java
// 方式一：transient 关键字
private transient String password;

// 方式二：@Expose 注解
@Expose(serialize = false, deserialize = false)
private String password;

// 方式三：自定义 ExclusionStrategy
Gson gson = new GsonBuilder()
    .addSerializationExclusionStrategy(new ExclusionStrategy() {
        @Override
        public boolean shouldSkipField(FieldAttributes f) {
            return f.getName().equals("password");
        }
        
        @Override
        public boolean shouldSkipClass(Class<?> clazz) {
            return false;
        }
    })
    .create();
```

### Q3: 如何处理日期格式？

```java
// 方式一：全局日期格式
Gson gson = new GsonBuilder()
    .setDateFormat("yyyy-MM-dd HH:mm:ss")
    .create();

// 方式二：自定义 TypeAdapter（推荐）
Gson gson = new GsonBuilder()
    .registerTypeAdapter(LocalDateTime.class, new LocalDateTimeAdapter())
    .create();
```

### Q4: 如何美化输出？

```java
Gson gson = new GsonBuilder()
    .setPrettyPrinting()
    .create();

String prettyJson = gson.toJson(user);
```
