# 代码走读检查清单 (Code Review Checklist)

- 适用范围：Java 后端代码走读 | 审查 | 审核
- AI 使用时逐条对照，☒ 为反例，☑ 为推荐写法。

---

## 一、SQL 类检查

### S1 分页查询必须有稳定排序（强制）
**问题：** 缺少主键作为最后一级排序，分页会重复/遗漏数据。

```sql
-- ☒ 排序不稳定，分页可能返回重复数据
ORDER BY CREATE_TIME DESC

-- ☑ 主键兜底保证稳定排序
ORDER BY CREATE_TIME DESC, id
```

---

### S2 所有查询必须加软删除过滤（强制）
**问题：** 遗漏 `DEL_FLAG = '0'` 会查出已删除的脏数据

```sql
-- ☒ 缺少 DEL_FLAG 过滤
SELECT * FROM RPMS PROJECT ACCEPT_PLAN WHERE id IN (...,)

-- ☑ 显式加软删除条件
SELECT * FROM RPMS PROJECT ACCEPT_PLAN WHERE DEL_FLAG = '0' AND id IN (...,)
```

---

### S3 LEFT|RIGHT JOIN 主表条件必须放在 WHERE 后，从表条件放在 ON（强制）
**问题：** 主表条件放在 `ON` 中不过过滤主表行

```sql
-- ☒ t1.DEL_FLAG = '0' 在 ON 中，对 LEFT JOIN 的主表过滤无效
FROM table1 t1
LEFT JOIN table2 t2 ON t2.table1_id = t1.id AND t1.DEL_FLAG = '0'

-- ☑ 主表条件放在 WHERE 后
FROM table1 t1
LEFT JOIN table2 t2 ON t2.table1_id = t1.id
WHERE t1.DEL_FLAG = '0'
```

---

### S4 LEFT|RIGHT JOIN 数据膨胀需重视（建议）
**问题：** 一对多关联产生多条中间结果，后续聚合或 IN 查询参数重复

```sql
-- ☒ 未去重，SQL LEFT JOIN 一对多导致 NAME 数据重复
SELECT T1.NAME FROM TABLE_1 T1 LEFT JOIN TABLE_2 T2 ON T2.TABLE_1_ID = T1.ID

-- ☑ 对查询字段 group by 去重
group by t1.name

-- ☑ 先查询 id，再用 id 反查
SELECT T1.id FROM TABLE_1 T1 LEFT JOIN TABLE_2 T2 ON T2.TABLE_1_ID = T1.ID;
SELECT T1.name from TABLE_1 T1 WHERE T1.id IN (?)
```

---

## 二、Java 代码类检查

### J1 Stream toMap key和value为null会报错（建议）
**问题：** 集合元素为 null 时，'map' 取字段 NPE

```java
// ☒ 如果列表中有 position|username 有null值，会NPE
list.stream().collect(Collectors.toMap(CloudConfig::getPosition, CloudConfig::getUsername))

// ☑ 先 filter null 元素再 toMap
list.stream()
    .filter(it -> StrUtil.isNotEmpty(it.getPosition()))
    .filter(it -> StrUtil.isNotEmpty(it.getUsername()))
    .collect(Collectors.toMap(CloudConfig::getPosition, CloudConfig::getUsername));
```

---

### J2 集合传参给 SQL IN 前需去重（建议）
**问题：** 集合元素存在重复时，SQL `IN` 参数重复，增加查询开销

```java
// ☒ 未去重，IN 参数可能重复
List<String> ids = list.stream().map(XXX::getId).filter(StrUtil::isNotBlank)
    .collect(Collectors.toList());

// ☑ 去重后再传给 SQL IN 查询
List<String> ids = list.stream().map(XXX::getId).filter(StrUtil::isNotBlank).distinct()
    .collect(Collectors.toList());
// ☑ 或者使用Set
Set<String> ids = list.stream().map(XXX::getId).filter(StrUtil::isNotBlank)
    .collect(Collectors.toSet());
```

---

### J3 魔法字符串字段名用常量替代（强制）
**问题：** `"projectId"` 是魔法字符串，重构时 IDE 无法感知

```java
// ☒ 魔法字符串
example.createCriteria().andIn("projectId", projectIds);
```

- tkmybatis
```java
// ☑ 实体类加 @FieldNameConstants 注解，使用 {类}.Fields.{字段名}
// 编译期自动生成，重构安全
example.createCriteria().andIn(ProjectCfgsTagsets.Fields.projectIds, projectIds);
```

- mybatis-plus
```java
// ☑ 使用 Lambda 获取字段名，重构安全
QueryWrapper<CloudConfig> wrapper = new QueryWrapper<>();
wrapper.in(CloudConfig::getProjectIds, projectIds);
```

---

### J4 全量查询改为分批查询（强制）
**问题：** 数据量大时有 OOM 风险

```java
// ☒ 全量查询，数据量大时可能 OOM
list = mapper.page();

// ☑ 使用 BatchUtil 分批拉取
BatchUtil.batch((pn, ps) -> {
  Page<PageProjectPlanItemEntity> page = PageHelper.startPage(pn, ps);
  // 执行业务逻辑
  ...
  // 安全返回1个元素保持继续，返回空则结束
  return CollUtil.get(mapper.page(), 0);
});
```

---

### J5 批量的IO操作使用分批调用（强制）
**问题：** 数据量大时有 timeout 或`IN`超限

```java
// ☒ 一次查询，数据量大时可能 timeout 或`IN`超限
list = userClient.list(userIds)

// ☑ 使用 BatchUtil 分批拉取
list = BatchUtil.batch(userIds, it -> userClient.list(it));
```

---

### J6 stream 中先 filter 再 trim（建议）
**问题：** 先 trim 需要 Optional 包装 null，冗余

```java
// ☒ 先 trim 需要 Optional 包装 null，冗余
map(str -> Optional.ofNullable(str).map(String::trim).orElse(null)).filter(StringUtils::isNotBlank)

// ☑ 先 filter 非空，trim 无需 Optional
.filter(StringUtils::isNotBlank).map(String::trim);
```

---

### J7 并发场景禁止共享可变状态（强制）
**问题：** 静态或全局共享的非线程安全对象（如 `SimpleDateFormat`、普通 `Map`/`List` 缓存）在多线程下并发读写，产生脏数据或抛 `ConcurrentModificationException`

```java
// ☒ 共享非线程安全对象，多线程下时间解析错乱
private static final SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd");

// ☑ 使用线程安全的 DateTimeFormatter
private static final DateTimeFormatter DTF = DateTimeFormatter.ofPattern("yyyy-MM-dd");
```

---

### J8 @Transactional 事务注意失效场景与边界（强制）
**问题：** 同类内部自调用、异常被 `catch` 吞掉、长事务内做耗时 IO，会导致事务失效或长时间占用数据库连接

```java
// ☒ 同类内部自调用，@Transactional 不生效
public void outer() {
    this.inner(); // 代理不拦截自调用
}
@Transactional
public void inner() { ... }

// ☑ 拆到其他 Spring Bean，或注入自身代理
@Resource
private OuterService self;
public void outer() {
    self.inner();
}
```

```java
// ☒ catch 吞掉异常，事务不回滚仍提交
@Transactional
public void doBiz() {
    try {
        dao.insert(record);
    } catch (Exception e) {
        log.error("insert fail", e); // 异常被吞，数据落库
    }
}

// ☑ 抛出或显式标记回滚
@Transactional
public void doBiz() {
    try {
        dao.insert(record);
    } catch (Exception e) {
        log.error("insert fail", e);
        throw new BizException("insert fail", e);
    }
}
```

---

### J9 异常处理与日志规范（建议）
**问题：** `catch` 后空处理（吞异常）导致故障无痕难排查；敏感信息明文打日志导致泄露

```java
// ☒ catch 空处理，异常被完全吞掉
try {
    int i = doSomething();
} catch (Exception e) {
    // 什么都不做
}

// ☑ 记录异常并转业务异常抛出
try {
    int i = doSomething();
} catch (Exception e) {
    log.error("doSomething fail, param={}", param, e);
    throw new BizException("doSomething fail", e);
}
```

```java
// ☒ 明文打印手机号/密码等敏感字段
log.info("login user={}, password={}", user.getUsername(), user.getPassword());

// ☑ 敏感信息脱敏后打印
log.info("login user={}, phone={}", user.getUsername(), DesensitizedUtil.phone(user.getPhone()));
```

---

### J10 服务端参数校验兜底（建议）
**问题：** 前端校验不可信，服务端缺失校验导致非法数据入库、空指针、越界

```java
// ☒ 直接使用前端入参，未做服务端校验
public void createOrder(OrderReq req) {
    orderService.insert(req);
}

// ☑ 使用 @Validated + 校验注解
public void createOrder(@Validated @RequestBody OrderReq req) {
    orderService.insert(req);
}
```

```java
public class OrderReq {
    @NotBlank(message = "订单号不能为空")
    private String orderNo;

    @NotNull(message = "金额不能为空")
    @DecimalMin(value = "0.01", message = "金额必须大于0")
    private BigDecimal amount;
}
```

---

### J11 禁止字符串拼接 SQL（强制）
**问题：** 字符串拼接 SQL 存在注入风险，恶意入参可拼接条件读取或删除任意数据

```java
// ☒ 字符串拼接 SQL，存在注入风险
String sql = "SELECT * FROM user WHERE name = '" + name + "'";
List<User> list = mapper.selectBySql(sql);
```

```xml
<!-- ☒ MyBatis ${} 直接替换参数，存在注入风险 -->
<select id="selectByName" resultType="User">
    SELECT * FROM user WHERE name = '${name}'
</select>
```

```xml
<!-- ☑ 统一使用 #{} 预编译传参，防注入 -->
<select id="selectByName" resultType="User">
    SELECT * FROM user WHERE name = #{name}
</select>
```

---

### J12 校验数据归属，防止越权（强制）
**问题：** 未校验数据归属，用户可越权访问他人数据（水平越权）

```java
// ☒ 未校验数据归属，水平越权可访问他人数据
public DetailVo getDetail(Long id) {
    return mapper.selectById(id);
}

// ☑ 校验数据归属当前用户
public DetailVo getDetail(Long id) {
    Detail detail = mapper.selectById(id);
    if (detail == null || !StrUtil.equals(detail.getUserId(), UserContext.getUserId())) {
        throw new BizException("无权访问");
    }
    return detail;
}
```

---

### J13 避免循环内查询（N+1）（建议）
**问题：** 循环内逐个查库或调远程接口，N 条数据产生 N+1 次查询，数据量大时性能劣化

```java
// ☒ 循环内逐个查库，N+1 次查询
for (Order o : orders) {
    o.setUserName(userMapper.selectById(o.getUserId()).getName());
}

// ☑ 先批量查询一次，再内存关联，避免 N+1
List<Long> userIds = orders.stream().map(Order::getUserId).collect(Collectors.toList());
Map<Long, User> userMap = userMapper.selectBatchIds(userIds).stream()
    .collect(Collectors.toMap(User::getId, u -> u));
for (Order o : orders) {
    User user = userMap.get(o.getUserId());
    if (user != null) {
        o.setUserName(user.getName());
    }
}
```

---

### J14 禁止 newFixedThreadPool 快捷创建线程池（强制）
**问题：** `newFixedThreadPool` 等 `Executors` 快捷创建使用无界队列和默认线程工厂，任务积压可导致 OOM，线程名难排查

```java
// ☒ newFixedThreadPool 快捷创建，无界队列任务积压可能 OOM
ExecutorService pool = Executors.newFixedThreadPool(10);

// ☑ 手动创建，指定线程工厂与拒绝策略
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    10, 20, 60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),
    new NamedThreadFactory("order-task"),
    new ThreadPoolExecutor.CallerRunsPolicy());
```

---

## 三、内存与资源管理

### M1 大文件/大附件禁止全量加载到内存（强制）
**问题：** 文件内容全量禁止加载到内存，避免触发 OOM 或增加 GC 压力

```java
// ☒ 整个 Base64 图片加载到 byte[]
byte[] bytes = FileUtil.readBytes(new File("/tmp/abc.txt"));

// ☑ 使用流式处理，不持有完整 byte[]
try (InputStream in = Files.newInputStream(new File("/tmp/abc.txt").toPath())) {
    // 直接流式处理
}
```

---

### M2 连接/流/锁等资源必须关闭（建议）
**问题：** 数据库连接、IO 流、锁未关闭导致连接/句柄泄漏，随请求累积耗尽资源

```java
// ☒ 资源未关闭，连接/句柄泄漏
InputStream in = Files.newInputStream(path);
byte[] bytes = in.readAllBytes();
// 未调用 in.close()

// ☑ 使用 try-with-resources 自动关闭
try (InputStream in = Files.newInputStream(path)) {
    ...
}
```

```java
// ☑ 手动关闭时判空，避免 null 指针
InputStream in = null;
try {
    in = Files.newInputStream(path);
    ...
} finally {
    if (in != null) {
        in.close();
    }
}
```

---

## 四、第三方接口与可用性

### T1 三方接口调用禁止同步阻塞等待（强制）
**问题：** 同步轮询等待三方接口结果时，线程持有请求上下文。如果三方接口慢（网络延迟、队列拥堵），会快速耗尽 Tomcat 线程池，导致服务不可用。

```java
// ☒ 同步轮询等待 OCR 结果
return executeWithStopWatch(() -> {
    BaseResponse response = ocrClient.submit(imageBytes);
    return pollParseResult(response.getTaskId()); // 阻塞等待
});

// ☑ 异步模式：submit 后返回 taskId，前端轮询查询结果
@PostMapping("/ocr/submit")
public Long submitOcr(@RequestBody Body OcrRequest req) {
    Long taskId = ocrClient.submitAsync(req.getImageBase64());
    return taskId;
}

@GetMapping("/ocr/result")
public OcrResultVo getOcrResult(@RequestBody Long taskId) {
    return ocrClient.queryResult(taskId);
}
```

---

### T2 外部接口调用必须设置超时（强制）
**问题：** 远程/三方调用未设置超时，对端一直不返回时线程长时间挂起，耗尽线程池导致服务不可用

```java
// ☒ 外部调用未设置超时，对端不返回时线程挂起
HttpResponse resp = HttpRequest.post(url).body(json).execute();

// ☑ 显式设置连接/读取超时
HttpResponse resp = HttpRequest.post(url)
    .setConnectionTimeout(3000)
    .setReadTimeout(5000)
    .body(json)
    .execute();
```

---

## 五、部署与运维

### D1 新功能上线必须同步补充多环境配置（强制）
**问题：** 新增 / 新依赖上线时只配置了开发/测试环境，缺少生产配置。导致上线遗漏、临时补配置出错。

```
// ☒ 只有开发环境配置
配置只在 profiles 中有一个文件存在，如 application-dev.yml

// ☑ 多环境配置齐全
配置必须在所有 profiles 文件中同时存在，如：application-dev.yml / application-uat.yml / application-prd.yml
```

---
