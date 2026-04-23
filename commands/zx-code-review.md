# zx-code-review — AI 改动上线前安全检查

对当前 git 工作区的改动文件做全面静态分析，确保编译安全、调用链无新 bug、启动不报错，再决定是否需要跑 Maven 验证。

## 触发方式

```
/zx-code-review
/zx-code-review <文件路径>   # 只检查指定文件
```

---

## 执行流程

### Step 0：确定检查范围

```bash
git diff --name-only HEAD       # 未提交改动
git diff --name-only HEAD~1     # 最近一次提交
```

仅检查 `.java` 和 `.xml` 文件（MyBatis mapper）。若用户指定了文件路径，只检查该文件。

---

### Step 1：Import 合法性检查（静态，秒级）

对每个改动的 `.java` 文件：

1. 提取所有 `import com.dc.it.*` 语句
2. 用 `find` 确认对应 `.java` 文件存在于工程中
3. **红线**：import 存在但找不到源文件 → 报错，列出缺失类

```bash
# 示例：检查某个 import
grep "^import" <file> | grep "com.dc.it" | while read line; do
  cls=$(echo $line | sed 's/import //;s/;//' | tr '.' '/')
  find /home/devtrump/zx-projects/DC-API-2018 -path "*${cls}.java" | grep -q . || echo "❌ MISSING: $line"
done
```

---

### Step 2：新增方法调用合法性检查（静态）

对每个改动文件，提取新增行（`git diff` 中 `+` 开头）里的方法调用：

1. 找到形如 `xxxDao.yyyMethod(` / `xxxService.yyyMethod(` 的调用
2. 在对应 DAO/Service 接口文件中确认方法签名存在
3. 若是 MyBatis DAO 调用，同步检查 `*Dao.xml` 中有没有对应 `<select id="yyyMethod">`
4. **红线**：调用了不存在的方法 → 报错

---

### Step 3：Spring Bean 注入检查（静态）

1. 新增的 `@Autowired` 字段 → 搜索对应类是否有 `@Service` / `@Component` / `@Repository` 注解
2. 新增的 `@Service` / `@Component` 类 → 检查包路径是否在 `component-scan` 范围内（看 Spring XML 配置）
3. **红线**：
   - `@Autowired` 字段找不到 Bean → 启动时 `NoSuchBeanDefinitionException`
   - 新 `@Service` 在扫描范围外 → 注入永远为 null

```bash
# 找 component-scan 配置
grep -r "component-scan" /home/devtrump/zx-projects/DC-API-2018 --include="*.xml" | grep "base-package"
```

---

### Step 4：MyBatis XML 检查（静态）

对改动的 `*Dao.xml`：

1. `resultType` / `parameterType` 中的类名 → 确认类存在
2. `#{param}` 占位符 → 确认对应 Java 方法参数/VO 字段存在（grep VO 类的 getter）
3. 新增 SQL 语句的 `id` → 确认 DAO 接口中有同名方法
4. **红线**：resultType 类不存在、id 无对应接口方法

---

### Step 4.5：提前 return 阻断检查（静态）

针对 Service 方法被 Controller 调用的场景：

1. 找到改动的 Service 方法里新增或保留的 `return` 语句
2. 检查 Controller 中调用该 Service 方法后，是否还有赋值/字段操作
3. **红线**：Service 在某分支提前 return，导致 Controller 后续对返回 Map 的操作（如 `result.put("experts", ...)`）被跳过，字段静默丢失

典型案例：`findWorldCupTopicHome` Service 原来在 `show=false` 时直接 `return result`，导致 Controller 里 `result.put("experts", experts)` 永远不执行。

### Step 5：接口实现完整性检查（静态）

若改动的类 `implements` 某接口：

1. 读接口文件中所有方法签名
2. 确认改动类中每个方法都有实现
3. **红线**：接口方法未实现 → 编译失败

---

### Step 6：PageHelper 线程安全检查（静态）

`PageHelper.startPage()` 必须紧接着 DAO 查询，中间不能插入其他 DAO 查询。

扫描新增的 `PageHelper.startPage` 调用，检查从 `startPage` 到第一个 DAO 调用之间，有没有**另一个 DAO 查询**插入（变量声明/空行可以有，别的 DAO 调用不行）。

**红线**：`PageHelper.startPage()` 和目标 DAO 调用之间夹了其他 DAO 查询 → 分页参数被第一个查询消费掉，目标查询不分页。

---

### Step 7：Maven 编译（按需，30-60s）

前6步若全部通过，**跳过 Maven**，直接出报告。

若任意一步发现疑点（非明确报错，但有不确定性），则运行：

```bash
cd /home/devtrump/zx-projects/DC-API-2018

# 只编译改动所在模块，-am 编译依赖模块，--no-transfer-progress 去掉进度条
# 识别改动文件所在模块（dc-service / dc-api / dc-dao 等）
MODULE=$(git diff --name-only HEAD | head -1 | cut -d'/' -f2)
mvn compile -pl $MODULE -am --no-transfer-progress -q 2>&1 | tail -20
```

若编译失败，把错误信息完整输出，逐条定位到具体文件+行号。

---

## 输出格式

```
=== zx-code-review 检查报告 ===

📋 检查范围：3 个文件
  - dc-service/.../MasterPushOrderContentManagementImpl.java
  - dc-dao/.../FUserDao.xml
  - dc-domain/.../PersonalInfoVo.java

✅ Step 1 Import 检查       — 通过
✅ Step 2 方法调用检查      — 通过（新增 userDao.findRedOrderRateList() 已确认存在）
✅ Step 3 Spring Bean 检查  — 通过
✅ Step 4 MyBatis XML 检查  — 通过
✅ Step 5 接口实现检查      — 通过
✅ Step 6 PageHelper 检查   — 通过（startPage 紧接 DAO 查询，无风险）
⏭️ Step 7 Maven 编译        — 已跳过（静态分析全部通过）

🟢 结论：可以发版，未发现编译错误和启动风险。
```

若有问题：

```
❌ Step 2 方法调用检查 — 失败
  文件：MasterPushOrderContentManagementImpl.java:2443
  问题：调用 userDao.findRedOrderRateList() 但 FUserDao.java 中未找到该方法
  建议：检查方法名拼写，或确认 FUserDao.java 已保存
```

---

## 红线一览（任意一条触发即阻止发版）

| # | 红线 | 后果 |
|---|------|------|
| R1 | import 的类不存在 | 编译失败 |
| R2 | 调用不存在的方法 | 编译失败 |
| R3 | @Autowired 找不到 Bean | 启动 NoSuchBeanDefinitionException |
| R4 | MyBatis XML resultType 类不存在 | 启动时 ClassNotFoundException |
| R5 | XML id 无对应 DAO 方法 | 运行时 BindingException |
| R6 | 接口方法未实现 | 编译失败 |
| R7 | PageHelper.startPage 后未紧接查询 | 分页参数泄漏，影响其他请求 |
| R8 | Service 方法提前 return 阻断 Controller 后续逻辑 | Controller 在 Service 调用后的赋值/字段全部跳过，静默丢数据 |

---

## 工程路径参考

- 主工程：`/home/devtrump/zx-projects/DC-API-2018`
- 模块：`dc-api`（Controller）/ `dc-service`（Service）/ `dc-dao`（DAO+XML）/ `dc-domain`（VO/POJO）
- Spring 扫描配置：`dc-api/src/main/resources/spring/spring-context.xml`
