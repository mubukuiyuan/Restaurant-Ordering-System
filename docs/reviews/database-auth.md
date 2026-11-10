# 数据库与权限验收

- 执行人：刘畅。验收日期：2026-11-10。
- 验收基线：远程 `main` 的 `59fe124bb0ce80d1fc248a214ab7a17692534a6a`。
- 环境：Windows 11 x64、Python 3.11.9、MySQL Community Server 8.0.31、PyMySQL 1.1.1。按项目 SQL 初始化隔离的本机数据库，以 `restaurant_app` 连接。业务请求通过项目 TCP/TLS 客户端发送，SQL 核对使用独立测试库。
- 结论：模板七项基线检查均通过。补充验收发现两项权限问题，整体待修复后复验。

## 基线检查

| 检查项 | 实际操作与结果 | 结论 |
| --- | --- | --- |
| 7 张业务表 | 查询 `information_schema`，得到 `employee`、`dish_category`、`dish`、`dining_table`、`orders`、`order_item`、`bill`，均为 InnoDB，共 8 个外键。核对应用账号仅有该库的查询和增删改权限，执行 `CREATE TABLE` 被 MySQL 以 1142 拒绝。 | 通过 |
| 员工响应省略密码 | 用四种角色登录，响应只有员工 ID、姓名和角色。管理员新增、修改和列表响应均只有公开字段。另查测试员工的数据库记录，`password` 保存的是 scrypt 散列。 | 通过 |
| 角色越权拒绝 | 对接口文档中的 18 个受保护操作发送请求。未登录的 18 个请求均返回 401，已登录的 53 个不允许的角色与操作组合均返回 403。请求体中附带管理员 ID 和角色也无法改变结果。 | 通过 |
| 停用账号后访问拒绝 | 临时服务员登录后可查询桌台。管理员停用账号后，同一 TLS 连接再次查询返回 401，新连接登录也返回 401。重新启用时的会话行为见 AUTH-01。 | 通过 |
| 被引用记录不能删除 | 下单、出菜并结账后，管理员删除在用分类、已点菜品、下单服务员和结账收银员，四次均返回 409。服务端记录 MySQL 外键错误 1217，查询确认四条记录均保留。 | 通过 |
| 历史订单单价保持 | 菜品单价 28.00 元时完成一笔一份菜品的订单与结账，随后调价至 30.00 元。历史订单的明细单价、订单金额和账单金额仍为 28.00 元，数据库明细一致。新订单以 30.00 元计价。 | 通过 |
| 管理员不能自行停用降权 | 当前管理员通过接口尝试删除自己、停用自己、将自己降为服务员，三次均返回 409。数据库确认其仍为启用的管理员，该会话仍可管理员工。多管理员并发操作见 AUTH-02。 | 通过 |

## 问题记录

### AUTH-01（P2）密码重置后旧会话继续有效，停用的旧会话可随账号恢复

1. 在 TLS 连接 A 用测试员工的原密码登录并查询桌台。管理员在另一连接停用该员工，A 查询返回 401。
2. 管理员重新启用该员工并重置密码。A 保持原连接且没有重新登录，再次查询桌台返回 `code=0`。新连接用原密码登录返回 401，用重置后的密码登录成功。
3. 保持新连接登录，再单独重置一次密码。该连接继续查询桌台仍返回 `code=0`，另开连接用刚作废的密码登录返回 401。

预期在停用或密码重置后，之前建立的会话需要重新登录。实际只在账号处于停用状态时拦截请求，曾被拒绝的连接也能在重新启用后恢复访问。密码重置无法撤销已登录连接的访问权。

登录会话只保存 `employee_id`，请求检查没有比较凭据或会话版本。复测应验证停用、重新启用及密码重置后，旧连接均返回 401，重新登录后才恢复访问。当前状态为已复现，待修复复测。 [登录与请求授权](https://github.com/mubukuiyuan/Restaurant-Ordering-System/blob/59fe124bb0ce80d1fc248a214ab7a17692534a6a/restaurant/router.py#L64)、[员工当前状态检查](https://github.com/mubukuiyuan/Restaurant-Ordering-System/blob/59fe124bb0ce80d1fc248a214ab7a17692534a6a/restaurant/services.py#L28)

### AUTH-02（P2）两名管理员并发互相降权可导致管理入口全部失效

1. 准备两名启用的管理员 A、B，各用独立 TLS 连接登录。
2. A 和 B 各发送一次 `manage_employee` 的 `update` 请求，将对方的 `role` 改为 `服务员`。实测在两条请求各自通过角色检查后用线程屏障同步，然后放行原有业务逻辑和 MySQL 事务。
3. 两条请求均返回 `code=0`。查询 `SELECT COUNT(*) FROM employee WHERE role='管理员' AND enabled=1` 得到 0。随后两个连接查询员工列表均返回 403。

预期并发操作结束后至少还有一名启用的管理员。实际两次更新均提交，系统内已没有可用于管理员工的账号。

自行降权检查只比较操作者与目标 ID，两个事务可以在角色检查通过后分别修改对方的记录。复测应再次同步发送互相降权请求，验证系统拒绝会移除最后一名可用管理员的变更。当前状态为已复现，待修复复测。 [授权后执行业务的事务](https://github.com/mubukuiyuan/Restaurant-Ordering-System/blob/59fe124bb0ce80d1fc248a214ab7a17692534a6a/restaurant/router.py#L74)、[员工变更及目标行加锁](https://github.com/mubukuiyuan/Restaurant-Ordering-System/blob/59fe124bb0ce80d1fc248a214ab7a17692534a6a/restaurant/services.py#L296)

## 自动测试与本机证据

- 使用 `RESTAURANT_TEST_HOME` 指向本次新建目录，执行 `python -m pytest tests/test_domain.py tests/test_integration.py -q`。33 项单元测试、37 项真实 MySQL 集成测试全部通过，合计 `70 passed in 23.76s`。其中覆盖订单及结账写入回滚、并发下单和结账、密码散列、角色拒绝以及 TLS 断线后重新登录。
- 本机专项检查输出保存在项目原目录的 `.runtime/database-auth-review/manual.log`，自动测试的输出和 JUnit 结果分别为同目录的 `pytest.log` 和 `pytest-results.xml`。
