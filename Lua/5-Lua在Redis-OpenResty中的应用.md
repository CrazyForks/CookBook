# Lua 在 Redis/OpenResty 中的应用

## 1. Redis Lua 脚本

### 1.1 基础用法

```lua
-- Redis Lua 脚本基础
-- 在 Redis 中执行 Lua 脚本

-- EVAL 命令
-- EVAL script numkeys key [key ...] arg [arg ...]

-- 简单脚本
EVAL "return 'Hello Redis'" 0

-- 带参数的脚本
EVAL "return ARGV[1]" 0 "Hello"

-- 访问 Redis 键
EVAL "return redis.call('GET', KEYS[1])" 1 mykey

-- 示例: 原子性递增
EVAL "
    local current = redis.call('GET', KEYS[1]) or 0
    current = current + 1
    redis.call('SET', KEYS[1], current)
    return current
" 1 counter
```

### 1.2 常用模式

```lua
-- 模式1: 分布式锁
-- lock.lua
local key = KEYS[1]
local value = ARGV[1]
local ttl = tonumber(ARGV[2])

if redis.call('SET', key, value, 'NX', 'EX', ttl) then
    return 1
else
    return 0
end

-- 使用: EVAL script 1 lock_key unique_id 30

-- 模式2: 限流器
-- rate_limiter.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end

if current > limit then
    return 0  -- 超过限制
else
    return 1  -- 允许访问
end

-- 模式3: 购物车操作
-- cart.lua
local key = KEYS[1]
local item = ARGV[1]
local quantity = tonumber(ARGV[2])

if quantity > 0 then
    redis.call('HINCRBY', key, item, quantity)
else
    local current = redis.call('HGET', key, item) or 0
    if current + quantity <= 0 then
        redis.call('HDEL', key, item)
    else
        redis.call('HINCRBY', key, item, quantity)
    end
end

return redis.call('HGETALL', key)
```

### 1.3 Java 集成

```java
// Spring Redis Lua 脚本执行
@Service
public class RedisLuaService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    // 原子性递增
    public Long atomicIncrement(String key, long delta) {
        String script = """
            local current = redis.call('GET', KEYS[1]) or 0
            current = current + ARGV[1]
            redis.call('SET', KEYS[1], current)
            return current
            """;
        
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>(script, Long.class);
        return redisTemplate.execute(redisScript, List.of(key), String.valueOf(delta));
    }
    
    // 分布式锁
    public boolean tryLock(String lockKey, String requestId, int expireTime) {
        String script = """
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                redis.call('DEL', KEYS[1])
                return 1
            elseif redis.call('SET', KEYS[1], ARGV[2], 'NX', 'EX', ARGV[3]) then
                return 1
            else
                return 0
            end
            """;
        
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>(script, Long.class);
        Long result = redisTemplate.execute(redisScript, 
            List.of(lockKey), requestId, String.valueOf(expireTime));
        return result != null && result == 1;
    }
    
    // 限流器
    public boolean isAllowed(String key, int limit, int window) {
        String script = """
            local current = redis.call('INCR', KEYS[1])
            if current == 1 then
                redis.call('EXPIRE', KEYS[1], ARGV[2])
            end
            if current > tonumber(ARGV[1]) then
                return 0
            else
                return 1
            end
            """;
        
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>(script, Long.class);
        Long result = redisTemplate.execute(redisScript, 
            List.of(key), String.valueOf(limit), String.valueOf(window));
        return result != null && result == 1;
    }
}
```

## 2. OpenResty Lua 编程

### 2.1 OpenResty 概述

```
OpenResty 架构:

┌─────────────────────────────────────────────────────────────┐
│                    OpenResty                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Nginx                             │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │              LuaJIT VM                      │   │   │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │   │
│  │  │  │ init    │  │ rewrite │  │ access  │    │   │   │
│  │  │  │ 阶段    │  │ 阶段    │  │ 阶段    │    │   │   │
│  │  │  └─────────┘  └─────────┘  └─────────┘    │   │   │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │   │
│  │  │  │ content │  │ header  │  │ body    │    │   │   │
│  │  │  │ 阶段    │  │ 阶段    │  │ 阶段    │    │   │   │
│  │  │  └─────────┘  └─────────┘  └─────────┘    │   │   │
│  │  │  ┌─────────┐  ┌─────────┐                 │   │   │
│  │  │  │ log     │  │ balancer│                 │   │   │
│  │  │  │ 阶段    │  │ 阶段    │                 │   │   │
│  │  │  └─────────┘  └─────────┘                 │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  核心组件:                                                  │
│  ├── Nginx: Web 服务器                                     │
│  ├── LuaJIT: 高性能 Lua 解释器                             │
│  ├── lua-nginx-module: Nginx Lua 模块                      │
│  ├── lua-resty-*: 各种 Lua 库                              │
│  └── ngx_lua API: Nginx Lua API                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 基础配置

```nginx
# nginx.conf
http {
    # Lua 模块路径
    lua_package_path "/usr/local/openresty/lualib/?.lua;;";
    lua_package_cpath "/usr/local/openresty/lualib/?.so;;";
    
    # 共享内存
    lua_shared_dict my_cache 10m;
    lua_shared_dict rate_limit 10m;
    
    # 初始化
    init_by_lua_block {
        -- 全局初始化
        local cjson = require("cjson")
    }
    
    server {
        listen 80;
        
        # rewrite 阶段
        rewrite_by_lua_block {
            ngx.log(ngx.INFO, "Rewrite phase")
        }
        
        # access 阶段
        access_by_lua_block {
            -- 限流检查
            local limit = ngx.shared.rate_limit
            local key = ngx.var.remote_addr
            local current, err = limit:get(key)
            
            if current then
                if current >= 100 then
                    ngx.exit(429)
                end
                limit:incr(key, 1)
            else
                limit:set(key, 1, 60)  -- 60秒过期
            end
        }
        
        # content 阶段
        location /api {
            content_by_lua_block {
                ngx.say("Hello from Lua!")
            }
        }
        
        # 调用外部 Lua 文件
        location /users {
            content_by_lua_file lua/users.lua;
        }
    }
}
```

### 2.3 常用 API

```lua
-- 请求信息
local method = ngx.req.get_method()       -- GET/POST
local uri = ngx.var.uri                   -- 请求路径
local args = ngx.req.get_uri_args()       -- 查询参数
local headers = ngx.req.get_headers()     -- 请求头
local body = ngx.req.get_body_data()      -- 请求体

-- 响应操作
ngx.status = 200
ngx.header["Content-Type"] = "application/json"
ngx.say(cjson.encode({message = "success"}))
ngx.exit(ngx.HTTP_OK)

-- 内部重定向
ngx.exec("/internal/api")

-- 外部重定向
ngx.redirect("/new-location", 301)

-- 日志
ngx.log(ngx.ERR, "Error message")
ngx.log(ngx.INFO, "Info message")

-- 定时器
ngx.timer.at(0, function(premature)
    -- 异步任务
end)

-- 共享内存
local cache = ngx.shared.my_cache
cache:set("key", "value", 60)  -- 60秒过期
local value = cache:get("key")
cache:delete("key")

-- HTTP 客户端
local http = require("resty.http")
local httpc = http.new()
local res, err = httpc:request_uri("http://example.com/api", {
    method = "POST",
    body = '{"key":"value"}',
    headers = {
        ["Content-Type"] = "application/json"
    }
})

-- Redis 客户端
local redis = require("resty.redis")
local red = redis.new()
red:connect("127.0.0.1", 6379)
red:set("key", "value")
local value = red:get("key")
red:close()
```

### 2.4 实战案例

```lua
-- JWT 认证中间件
local jwt = require("resty.jwt")
local cjson = require("cjson")

local function verify_token()
    local auth_header = ngx.req.get_headers()["Authorization"]
    if not auth_header then
        ngx.status = 401
        ngx.say(cjson.encode({error = "Missing authorization header"}))
        return ngx.exit(401)
    end
    
    local token = string.match(auth_header, "Bearer%s+(.+)")
    if not token then
        ngx.status = 401
        ngx.say(cjson.encode({error = "Invalid authorization format"}))
        return ngx.exit(401)
    end
    
    local jwt_obj = jwt:verify("secret_key", token)
    if not jwt_obj.verified then
        ngx.status = 401
        ngx.say(cjson.encode({error = "Invalid token"}))
        return ngx.exit(401)
    end
    
    -- 将用户信息存入 ngx.var
    ngx.ctx.user_id = jwt_obj.payload.sub
    ngx.ctx.user_roles = jwt_obj.payload.roles
end

-- 在 access 阶段调用
access_by_lua_block {
    verify_token()
}
```

## 3. 最佳实践

```
Lua 开发 Checklist:

□ Redis Lua:
  - 脚本保持简短
  - 避免长时间运行
  - 使用 KEYS 和 ARGV 传参
  - 处理错误返回值

□ OpenResty:
  - 选择正确的执行阶段
  - 使用共享内存缓存
  - 异步非阻塞操作
  - 合理使用定时器

□ 性能优化:
  - 使用 LuaJIT
  - 避免全局变量
  - 复用连接对象
  - 使用连接池
```
