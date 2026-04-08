#### 1. EVAL 执行 Lua 脚本

EVAL 是 Lua 脚本的核心执行命令，用于直接发送并执行 Lua 脚本。

- 语法：`EVAL script numkeys key [key ...] arg [arg ...]`
- 参数详解：
    
    - `script`：Lua 脚本内容，字符串格式；
    - `numkeys`：脚本中用到的 Redis Key 的数量，后续的 key 参数为对应的 Redis Key；
    - `key [key ...]`：脚本中用到的 Redis Key，在脚本内通过`KEYS[1]`、`KEYS[2]`... 访问，下标从 1 开始；
    - `arg [arg ...]`：脚本中用到的附加参数，在脚本内通过`ARGV[1]`、`ARGV[2]`... 访问，下标从 1 开始。
    
- 强制规范：脚本中用到的所有 Redis Key，必须通过`KEYS`数组传入，禁止在脚本内硬编码 Key，否则 Redis Cluster 集群模式下无法正确路由，执行报错。
- 典型示例（原子性库存扣减，防止超卖）：
    
    plaintext
    
    ```
    EVAL "local stock = tonumber(redis.call('GET', KEYS[1])); local num = tonumber(ARGV[1]); if stock >= num then redis.call('DECRBY', KEYS[1], num); return 1; else return 0; end" 1 product:stock:1001 10
    ```
    
    脚本逻辑：先获取库存，判断是否足够，足够则扣减并返回 1，否则返回 0，全程原子性，无并发竞态风险。

#### 2. SCRIPT LOAD 脚本预加载

将 Lua 脚本加载到 Redis 服务器的缓存中，生成并返回脚本的 SHA1 摘要，不会执行脚本，用于脚本复用。

- 语法：`SCRIPT LOAD script`
- 参数：`script`为 Lua 脚本内容
- 返回值：脚本的 40 位 SHA1 摘要字符串，后续可通过`EVALSHA`调用该脚本。
- 核心作用：将高频执行的固定脚本提前加载到 Redis 缓存，后续调用无需重复发送脚本内容，减少网络开销。

#### 3. EVALSHA 执行缓存脚本

通过脚本的 SHA1 摘要，执行 Redis 缓存中的 Lua 脚本，避免重复发送完整脚本内容，是生产环境高频脚本的首选执行方式。

- 语法：`EVALSHA sha1 numkeys key [key ...] arg [arg ...]`
- 参数详解：`sha1`为`SCRIPT LOAD`返回的脚本 SHA1 摘要，其余参数与`EVAL`完全一致。
- 优势：仅需发送 40 位 SHA1 摘要，无需发送完整脚本，极大减少网络传输量，提升性能；
- 注意事项：若 Redis 中无该 SHA1 对应的脚本缓存，会返回`NOSCRIPT`错误，需重新用`SCRIPT LOAD`加载脚本。

#### 辅助 SCRIPT 命令

- `SCRIPT EXISTS sha1 [sha1 ...]`：检查指定 SHA1 对应的脚本是否存在于 Redis 缓存中，返回 1/0 数组；
- `SCRIPT FLUSH`：清空 Redis 中所有缓存的 Lua 脚本，生产环境慎用；
- `SCRIPT KILL`：终止正在执行的 Lua 脚本，仅当脚本未执行写命令时有效，避免脚本死循环阻塞主线程。

#### 生产规范

1. 严格控制脚本执行时间，Redis 单线程模型下，脚本执行会阻塞所有其他命令，生产环境脚本执行时间建议控制在 1ms 以内，绝对不可超过 5ms；
2. 脚本内禁止使用死循环、耗时的复杂计算，所有非必要的复杂计算尽量在客户端完成，脚本内仅保留必要的 Redis 命令与逻辑判断；
3. 所有 Redis Key 必须通过`KEYS`数组传入，禁止硬编码，保证 Redis Cluster 集群兼容性；
4. 高频执行的固定脚本，必须使用`SCRIPT LOAD + EVALSHA`，减少网络开销；
5. Redis Cluster 集群模式下，脚本内的所有 Key 必须位于同一个哈希槽，可通过 Hash Tag 固定 Key 的哈希槽。