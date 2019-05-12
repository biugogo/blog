---
title: Redis执行Lua脚本引擎Demo
date: 2018-7-27 22:22:16
tags:
 -DB
categories: DB
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# Redis执行Lua脚本引擎Demo
----
## Overview
最近看到老师谢了一个Lua脚本的简单执行引擎，感觉很不错。写成一个Demo，方便以后学习。

------
首先是Redis的Spring Boot下配置,直接上代码。
```java
@Bean(initMethod = "init", destroyMethod = "destroy")
@ConfigurationProperties(prefix = "app.redis", ignoreUnknownFields = false)
public RedisSentinelFactory redisSentinelFactory() {
    return new RedisSentinelFactory();
}
```
这里使用Spring的自动注入。yml中需要配置
```yml
app:
  redis:
    max-total: 300
    max-wait-millis: 8000
    max-idle: 100
    test-on-borrow: false
    test-on-return: false
    test-while-idle: true
    master-name: wolfkill_test_redis_001
    servers: group1003-bj-sentinel.yy.com:20060,group1003-sz-sentinel.yy.com:20060,group1003-wx-sentinel.yy.com:20060
```
这里的RedisSentinelFactory是公司包装的一个读写分离的工厂。

----
进入正题，这里开始介绍脚本引擎
```java
@Bean
public RedisLuaScriptEngine redisLuaScriptEngine(RedisSentinelFactory redisSentinelFactory) {
    return new RedisLuaScriptEngine("cp_room", "classpath:/script/cp_room.lua", new Supplier<Jedis>() {
        @Override
        public Jedis get() {
            return redisSentinelFactory.getMaster();
        }
    });
}
```
这里推荐一个Supplier接口，这是java.util.function包下的提供者接口，这个接口只有一个Get方法，从写它可以包装一个对象的提供方式。这里是一个简单引用。

让我们来看下这个类
构造函数和类变量
```java
@Slf4j
@Component
public class RedisLuaScriptEngine {
    /**
     * 脚本缓存到Redis的Key前缀
     */
    private static final String LUA_SCRIPT_KEY_PREFIX = "string:lua_script_sha:";

    /**
     * Spring的Resource加载器
     */
    private static final ResourceLoader RESOURCE_LOADER = new PathMatchingResourcePatternResolver();

    /**
     * 脚本后缀
     */
    private final String scriptName;
    /**
     * 脚本所在项目地址，这里我们一般设置为classpath:/script/cp_room.lua
     */
    private final String scriptPath;
    /**
     * Jedis提供者
     */
    private final Supplier<Jedis> jedisSupplier;

    /**
     * 脚本在Redis存储的Key，它等于前缀+脚本后缀
     */
    private final String luaScriptKey;
    /**
     *
     */
    private volatile String luaSHA;

    public RedisLuaScriptEngine(String scriptName, String scriptPath, Supplier<Jedis> jedisSupplier) {
    this.scriptName = scriptName;
    this.scriptPath = scriptPath;
    this.jedisSupplier = jedisSupplier;
    this.luaScriptKey = LUA_SCRIPT_KEY_PREFIX + scriptName;

    init();
}
   ...
}
```
再看init()方法
```java
private void init() {
    //加载脚本
    Resource resource = RESOURCE_LOADER.getResource(scriptPath);
    String scriptContent = null;
    try {
        //脚本写到一个String,指定编码格式
        scriptContent = IOUtils.toString(resource.getInputStream(), Charset.forName("utf-8"));
    } catch (IOException e) {
        log.warn("Load {} failed!", resource, e);
    }
    //判断是否加载到
    Assert.hasLength(scriptContent, "Lua script must not be empty!");

    //通过提供者拿到Jedis实例
    try (Jedis jedis = jedisSupplier.get()) {
        //缓存脚本。返回缓存脚本信息，我们可以根据luaSHA找到脚本
        luaSHA = jedis.scriptLoad(scriptContent);
        //把脚本信息放入Redis
        jedis.set(luaScriptKey, luaSHA);
    }

    log.info("Load script {} success! LuaSHA:{}", scriptName, luaSHA);
}
```
scriptLoad方法：Redis Script Load 命令用于将脚本 script 添加到脚本缓存中，但并不立即执行这个脚本。如果给定的脚本已经在缓存里面了，那么不执行任何操作。
在脚本被加入到缓存之后，通过 EVALSHA 命令，可以使用脚本的 SHA1 校验和来调用这个脚本。
脚本可以在缓存中保留无限长的时间，直到执行 SCRIPT FLUSH 为止。

我们加入一个定时器任务，5分钟更新脚本到代码。这里可以从redis动态修改脚本，再动态同步到代码的能力。
```java
@Scheduled(initialDelay = 10_1000, fixedDelay = 5 * 60 * 1000)
private void reload() {
    try (Jedis jedis = jedisSupplier.get()) {
        String value = jedis.get(luaScriptKey);
        if (StringUtils.isNotEmpty(value) && !Objects.equals(value, luaSHA)) {
            log.info("Reload luaSHA success! New:{}", luaSHA);
            luaSHA = value;
        }

    }
}
```

----

核心代码：

```java
/**
 * 执行Lua方法
 */
public LuaScriptResult execute(@NonNull String method, @NonNull List<Object> args) {
    List<String> list = new ArrayList(args.size() + 1);
    for (Object arg : args) {
        list.add(null == arg ? null : arg.toString());
    }

    list.add(method);

    try (Jedis jedis = jedisSupplier.get()) {
        //执行脚本，传入参数
        Object result = jedis.evalsha(luaSHA, 0, list.toArray(EMPTY_STRING_ARRAY));
        if (log.isDebugEnabled()) {
            log.debug("method:{} args:{} result:{}", method, args, result);
        }
        //返回执行器的包装结果，详见下文
        return new LuaScriptResult(result);
    }

}
```

这里有一个很有趣的代码：
```java
list.toArray(org.apache.commons.lang3.ArrayUtils.EMPTY_STRING_ARRAY)
```
为什么list的toArray能拷贝内容到一个不可变空数组？
让我们看看ArrayList的toArray方法
```java
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        // Make a new array of a's runtime type, but my contents:
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```
可以看到，如果数组长度低于size，那会直接拷贝到新建的相同类型的数组。传入的空数组只起getClass（）作用。

执行结果包装器,更加方便的拿到结果
```java
@ToString
public static class LuaScriptResult {
    public final Object object;

    public LuaScriptResult(Object object) {
        this.object = object;
    }

    public Integer asInt() {
        return null == object ? null : ((Number) object).intValue();
    }

    public List asList() {
        return (List) object;
    }

    public String asSting() {
        return null == object ? null : object.toString();
    }

    public boolean asBoolean() {
        return null == object ? false : true;
    }

}
```


-----------
再看看lua脚本，这里积累一下

```java

-- 程序入口
-- 常量定义
local rcall = redis.call
local rerror = redis.error_reply
local one_day_seconds = 86400

-- 连麦申请（uid->申请时间）
local key_mic_apply_list = "zset:cp:mic:apply:" -- + room_id
-- 当前麦上用户 uid->上麦时间
local key_on_mic_list = "zset:cp:mic:on:" -- + room_id
-- 麦序信息（{token:xxx,status:1开|2关}） 1
local key_mic_info = "hash:cp:mic:info:" -- + room_id + ":" +uid
-- 上麦通知确认
local key_mic_on_ack = "zset:cp:mic:on_ack:" -- + room_id
-- 下麦通知确认
local key_mic_off_ack = "zset:cp:mic:off_ack:" -- + room_id
-- 喜欢我的人（uid->喜欢时间）
local key_like_me = "zset:cp:likeme:" -- + room_id + ":" + uid
-- 我喜欢的人（uid->喜欢时间）
local key_my_like = "zset:cp:mylike:" -- + room_id + ":" + uid
-- 区内cp锁,防止出现cp重复（uid->cp_uid）
local key_cp_lock = "hash:cp:lock"

-- 连麦申请
-- 返回数组，定义如下
-- [1]-> 0：成功  1：失败
local function apply_mic(room_id, uid, time_now)
    local apply_key = key_mic_apply_list .. room_id;
    local apply_time = rcall("ZSCORE", apply_key, uid)

    -- 如果之前没有申请过
    if not apply_time then
        rcall("ZADD", apply_key, tonumber(time_now), uid)
        rcall("EXPIRE", apply_key, one_day_seconds) -- 防御性ttl
        return { 0 };
    end

    return { 1 }
end

-- 取消连麦
-- 返回数组，定义如下
-- [1]-> 0：连麦申请存在  1：连麦申请不存在
-- [2]-> 0：用户在麦上 1：用户没在麦上
local function cancel_mic(room_id, uid)
    local result = {}

    local apply_key = key_mic_apply_list .. room_id;
    local on_list_key = key_on_mic_list .. room_id

    local apply_count = rcall("ZREM", apply_key, uid)
    local on_list_count = rcall("ZREM", on_list_key, uid)

    result[1] = apply_count == 0 and 1 or 0;
    result[2] = on_list_count == 0 and 1 or 0

    rcall("DEL", key_mic_info .. room_id .. ":" .. uid)

    return result
end

local function off_none_owner_mic(room_id)
    local on_list_key = key_on_mic_list .. room_id
    local on_mic_uids = rcall("ZRANGE", on_list_key, 0, -1)

    local off_uid_list = {}
    for i, on_mic_uid in pairs(on_mic_uids) do
        local _mic_info_key = key_mic_info .. room_id .. ":" .. on_mic_uid
        -- 不是房主
        if not rcall("HGET", _mic_info_key, "owner") then
            table.insert(off_uid_list, on_mic_uid)
            rcall("ZREM", on_list_key, on_mic_uid)
            rcall("DEL", _mic_info_key)
        end
    end

    return off_uid_list
end

-- 上麦
-- 返回数组，定义如下
--  [1] -> 0：上麦成功   1：上麦失败
--  [2] -> 如果之前麦上有人则返回他们
local function on_mic(room_id, uid, token, time_now)
    local result = {};

    -- 没有申请过？
    if not rcall("ZSCORE", key_mic_apply_list .. room_id, uid) then
        result[1] = 1
    else
        local on_list_key = key_on_mic_list .. room_id
        local mic_on_ack_key = key_mic_on_ack .. room_id
        local mic_info_key = key_mic_info .. room_id .. ":" .. uid;
        local time_long = tonumber(time_now)

        if rcall("ZSCORE", on_list_key, uid) then -- 已经在麦上
            result[1] = 1
        else
            result[1] = 0
            -- 删掉麦上非主播用户
            result[2] = off_none_owner_mic(room_id)
            -- 上麦
            rcall("ZADD", on_list_key, time_long, uid)
            rcall("EXPIRE", on_list_key, one_day_seconds)
            -- 加入上麦确认
            rcall("ZADD", mic_on_ack_key, time_long, uid)
            rcall("EXPIRE", mic_on_ack_key, 60)
            -- 保存token
            rcall("HSET", mic_info_key, "token", token)
            rcall("EXPIRE", mic_info_key, one_day_seconds)
        end
    end

    return result
end


-- 主播上麦
local function on_owner_mic(room_id, uid, token, time_now)
    local on_list_key = key_on_mic_list .. room_id
    local mic_info_key = key_mic_info .. room_id .. ":" .. uid;

    -- 把自己抱上麦
    rcall("ZADD", on_list_key, tonumber(time_now), uid)
    rcall("EXPIRE", on_list_key, one_day_seconds)
    rcall("HMSET", mic_info_key, "owner", 1, "status", 1)
    rcall("EXPIRE", mic_info_key, one_day_seconds)
end

-- 下麦
-- 返回数组，定义如下
--  [1] -> 0：下麦成功 1：下麦失败
local function off_mic(room_id, uid, time_now)
    local result = {};

    if rcall("ZREM", key_on_mic_list .. room_id, uid) > 0 then -- 在麦上
        -- 删除连麦信息
        rcall("DEL", key_mic_info .. room_id .. ":" .. uid)

        -- 加入下麦确认
        local mic_off_ack_key = key_mic_off_ack .. room_id;
        rcall("ZADD", mic_off_ack_key, tonumber(time_now), uid)
        rcall("EXPIRE", mic_off_ack_key, 60)

        -- 删除连麦申请
        rcall("ZREM",key_mic_apply_list .. room_id,uid)
        result[1] = 0
    else
        result[1] = 1
    end

    return result
end

-- 用户开麦
-- 返回数组，定义如下
-- [1] -> 0 成功 1失败
local function open_mic(room_id, uid)
    local mic_info_key = key_mic_info .. room_id .. ":" .. uid

    if rcall("EXISTS", mic_info_key) == 1 then
        rcall("HSET", key_mic_info .. room_id .. ":" .. uid, "status", 1)
        return { 0 }
    end

    return { 1 }
end

-- 用户关麦
-- 返回数组，定义如下
-- [1] -> 0 成功 1失败
local function close_mic(room_id, uid)
    local mic_info_key = key_mic_info .. room_id .. ":" .. uid

    if rcall("EXISTS", mic_info_key) == 1 then
        rcall("HSET", key_mic_info .. room_id .. ":" .. uid, "status", 2)
        return { 0 }
    end

    return { 1 }
end

-- 喜欢某人
-- 返回数组，定义如下
-- [1] 0：to_uid喜欢from_uid 1：to_uid不喜欢from_uid
-- [2] 0：from_uid初次喜欢to_uid 1：from_uid已经喜欢过to_uid
-- [3] 0: 可以达成cp 1:不能达成cp

local function like_someone(room_id, from_uid, to_uid, can_make_cp, time_now)
    local result = {};
    local to_uid_like_me_key = key_like_me .. room_id .. ":" .. to_uid -- 喜欢to_uid的人
    local from_uid_like_me_key = key_like_me .. room_id .. ":" .. from_uid -- 喜欢from_uid的人

    local is_to_uid_like_from_uid = rcall("ZSCORE", from_uid_like_me_key, to_uid) -- to_uid是否已经喜欢from_uid
    local is_from_uid_like_to_uid = rcall("ZSCORE", to_uid_like_me_key, from_uid) -- from_uid是否已经喜欢to_uid

    -- to_uid是否喜欢from_uid
    if is_to_uid_like_from_uid then
        result[1] = 0
    else
        result[1] = 1
    end
    -- from_uid是否已经喜欢过to_uid
    if is_from_uid_like_to_uid then
        result[2] = 1
    else
        result[2] = 0
    end

    -- 检查是不是已经达成cp
    local cp_status = rcall("HMGET", key_cp_lock, from_uid, to_uid) -- 双方最近cp达成状态
    local is_from_uid_already_cp = cp_status[1]; -- from_uid的cp状态
    local is_to_uid_already_cp = cp_status[2]; -- to_uid的cp状态

    if not is_from_uid_already_cp and not is_to_uid_already_cp and "true" == can_make_cp then
        -- 双方都没有cp，可以达成cp
        result[3] = 0
    else
        result[3] = 1
    end

    -- 保存喜欢数据
    -- 如果from_uid之前没有喜欢过to_uid
    if not is_from_uid_like_to_uid then
        local time_long = tonumber(time_now)
        local from_uid_my_like_key = key_my_like .. room_id .. ":" .. from_uid -- from_uid喜欢的人

        rcall("ZADD", from_uid_my_like_key, time_long, to_uid)
        rcall("ZADD", to_uid_like_me_key, time_long, from_uid)

        -- 防御性ttl
        rcall("EXPIRE", from_uid_my_like_key, one_day_seconds)
        rcall("EXPIRE", to_uid_like_me_key, one_day_seconds)
    end

    if result[3] == 0 and is_to_uid_like_from_uid then
        -- 加个锁可以防止区内cp重复
        rcall("HMSET", key_cp_lock, from_uid, to_uid, to_uid, from_uid)
        rcall("EXPIRE", key_cp_lock, 10)
    end

    return result
end

-- 解除cp
local function cancel_cp(uid)
    -- 清空cp lock
    local cp_uid = rcall("HGET", key_cp_lock, uid)
    if cp_uid then
        rcall("HDEL", key_cp_lock, cp_uid, uid)
    else
        rcall("HDEL", key_cp_lock, uid)
    end
end

-- echo
local function echo(...)
    return ARGV
end

local fns = {}
fns["apply_mic"] = apply_mic
fns["cancel_mic"] = cancel_mic
fns["on_mic"] = on_mic
fns["off_none_owner_mic"] = off_none_owner_mic
fns["on_owner_mic"] = on_owner_mic
fns["off_mic"] = off_mic
fns["open_mic"] = open_mic
fns["close_mic"] = close_mic
fns["like_someone"] = like_someone
fns["cancel_cp"] = cancel_cp
fns["echo"] = echo

-- 程序入口
if #ARGV == 0 then
    rerror("invalid args!")
end

return fns[ARGV[#ARGV]](unpack(ARGV))

```

这个lua脚本有个地方写的很好
```
local fns = {}
fns["apply_mic"] = apply_mic
fns["cancel_mic"] = cancel_mic
fns["on_mic"] = on_mic
fns["off_none_owner_mic"] = off_none_owner_mic
fns["on_owner_mic"] = on_owner_mic
fns["off_mic"] = off_mic
fns["open_mic"] = open_mic
fns["close_mic"] = close_mic
fns["like_someone"] = like_someone
fns["cancel_cp"] = cancel_cp
fns["echo"] = echo

-- 程序入口
if #ARGV == 0 then
    rerror("invalid args!")
end

return fns[ARGV[#ARGV]](unpack(ARGV))
```
这是程序入口，用一个table来重定向函数，再用unpack制造一个可变参数函数。#是取参数长度。把函数名当成最后一个参数，就可以定位到方法。并执行。
