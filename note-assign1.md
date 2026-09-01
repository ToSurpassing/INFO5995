# 模块一：一个现代应用到底由哪些部分组成？

先给你一张全局图。绝大多数现实中的手机 App、网页和桌面客户端，都可以放进下面这个模型：

```text
┌──────────────── 用户设备 ────────────────┐
│                                        │
│  用户                                  │
│   ↓ 点击、输入                         │
│  App 界面层                            │
│   ↓ 调用                               │
│  App 业务逻辑层                        │
│   ├── 本地存储                         │
│   └── 网络客户端                       │
│             ↓ HTTPS                    │
└─────────────┼──────────────────────────┘
              │ 互联网 / 局域网
              ↓
┌──────────────── 服务器端 ────────────────┐
│                                        │
│  API 入口                              │
│   ↓                                    │
│  身份认证与权限检查                    │
│   ↓                                    │
│  服务器业务逻辑                        │
│   ├── 数据库                           │
│   ├── 文件存储                         │
│   ├── 日志系统                         │
│   └── 第三方服务                       │
│                                        │
└────────────────────────────────────────┘
```

最重要的一条原则是：

> 手机 App 通常只是整个系统的客户端。真正可信的身份验证、权限判断和核心数据操作，通常应该在服务器端完成。

SafeCampus 是一个教学模型。它模仿了这套结构，但很多服务器行为实际上被本地代码、虚构响应或 `.invalid` 地址替代。理解真实架构和教学 APK 的差别，是判断漏洞影响的基础。

---

## 1. 首先区分“应用”和“系统”

日常说“这个 App”，可能指的是手机上的图标和界面。但从安全角度看，一个应用服务通常是完整系统：

```text
手机上的客户端
+
传输网络
+
后端 API
+
身份认证服务
+
数据库
+
运维和日志系统
```

例如银行 App：

- 手机 App 负责显示余额和收集转账信息；
- HTTPS 负责把请求安全地送到银行服务器；
- 身份系统确认用户身份；
- 权限系统判断这个用户能否操作这个账户；
- 后端业务逻辑执行转账规则；
- 数据库存储余额和交易记录；
- 日志系统记录安全事件。

所以“Android App 的安全”不只等于“Java 代码有没有 bug”，而是要看整个系统中：

- 数据在哪里产生；
- 数据经过哪里；
- 谁作出安全决策；
- 什么部分可以被攻击者控制；
- 每一层依赖什么安全假设。

---

# 2. 用户和界面层

界面层也叫 UI，User Interface。

它包括：

- 文本框；
- 按钮；
- 页面；
- 列表；
- 提示信息；
- 登录窗口；
- 错误消息。

在 SafeCampus 中，例如：

```text
LoginActivity
SupportRequestActivity
AccountRecoveryActivity
DashboardActivity
ServiceStatusActivity
```

这些 Activity 大体上对应用户看到的页面。

例如用户提交 Support Request：

```text
用户输入 “Badge access test”
        ↓
点击 Submit Request
        ↓
SupportRequestActivity 读取文本框内容
```

界面的职责通常应该是：

1. 接收用户输入；
2. 做基本格式检查；
3. 调用业务逻辑；
4. 显示处理结果。

界面层不应该单独承担关键安全决策。例如：

```java
if (username.equals("admin")) {
    showAdminPage();
}
```

如果这就是全部权限控制，那么问题很严重。因为攻击者控制自己的客户端，可以：

- 修改 APK；
- 跳过界面；
- 直接调用内部方法；
- 构造网络请求；
- 修改变量；
- 使用其他客户端访问服务器。

因此：

> “按钮没显示给用户”不等于“用户没有权限”。

安全权限必须由可信的一方强制执行，通常是服务器。

---

# 3. 业务逻辑层

业务逻辑是“这个应用具体应该做什么”的代码。

例如：

- 登录时怎样处理用户输入；
- 恢复码如何生成；
- 支持请求怎样提交；
- 哪些情况下进入 Service Status；
- 校园警报怎样显示；
- 同步结果怎样保存。

SafeCampus 中的业务逻辑分布在很多类中，例如：

```text
AppStateResolver
AccountWorkflow
ReferenceCalculator
SupportTicketRepository
CampusAlertRepository
CampusSyncService
SessionState
```

“Repository”“Service”“Workflow”“Resolver”不是 Android 强制规定的组件，而是开发者为了组织代码使用的名字。

大致可以这样理解：

| 名称 | 常见职责 |
|---|---|
| Activity | 页面和用户交互 |
| Repository | 负责取得或提交数据 |
| Service | 执行某类后台或业务操作 |
| Workflow | 组织多步业务流程 |
| Resolver | 根据输入或状态作出判断 |
| Model | 表示数据，例如用户、警报、设备 |
| Client | 与外部服务通信 |
| Parser | 解析外部数据 |
| Preferences/Session | 保存本地状态 |

这些只是常见习惯，不是可靠的安全保证。一个叫 `SecureLoginManager` 的类完全可能一点也不安全。

必须读实际代码。

---

# 4. 本地存储层

App 经常需要在设备上保存少量数据，例如：

- 用户设置；
- 是否已经登录；
- 用户显示名称；
- 页面状态；
- 缓存；
- 最近一次同步时间；
- 临时 token。

Android 提供多种本地存储方式：

```text
SharedPreferences    小型 key/value 配置
SQLite               结构化数据库
普通文件             图片、文档、缓存
Room                 SQLite 的高级封装
Keystore             密钥和密码学材料
缓存目录             可删除的临时数据
```

SafeCampus 主要使用的是 `SharedPreferences`。

你可以把它暂时理解为应用自己的小型配置文件：

```xml
<string name="profile_name">Maya Chen</string>
<string name="profile_ref">SCU-2026-1842</string>
<boolean name="workspace_state" value="true" />
```

正常情况下，每个 Android App 都运行在独立的 Linux 用户身份下：

```text
App A → Linux UID A
App B → Linux UID B
App C → Linux UID C
```

App A 通常不能直接读取 App B 的私有目录。这就是 Android 应用沙箱。

SafeCampus 的私有目录类似：

```text
/data/user/0/au.edu.safecampus.connect/
```

其中包名是：

```text
au.edu.safecampus.connect
```

但是 SafeCampus 被构建成：

```xml
android:debuggable="true"
```

因此，在拥有该模拟器 ADB 访问权限的情况下，我们成功使用 `run-as` 进入 SafeCampus 的应用身份，读取了它的私有 SharedPreferences。

这不是说“Android 沙箱彻底失效”，而是说：

> 开发者主动发布了可调试构建，使本地调试接口能够以该应用身份访问其私有状态。

---

# 5. 网络客户端层

手机 App 不会自动懂得怎样与 SafeCampus 服务器通信。开发者需要编写网络客户端代码。

SafeCampus 中主要是：

```text
ApiClient
ConnectionPolicy
ResponseParser
```

它们的职责大致是：

```text
ApiClient
├── 选择请求地址
├── 创建 HTTPS 连接
├── 设置请求方法
├── 添加请求头
├── 写入请求内容
└── 读取服务器响应

ConnectionPolicy
├── 配置 TLS
├── 配置证书验证
└── 配置主机名验证

ResponseParser
├── 读取响应字节
├── 转换成字符串
└── 转换成应用内部对象
```

例如支持请求路径：

```text
SupportRequestActivity
        ↓ 用户输入 issue
SupportTicketRepository
        ↓
ApiClient.submitSupportRequest(issue)
        ↓
HTTPS POST
        ↓
服务器 API
```

在 SafeCampus 中，目标地址是：

```text
https://api.safecampus.invalid/
```

`.invalid` 是保留给无效或示例域名使用的后缀，因此它不是一个真实课程服务器。

这意味着 APK 中的网络代码是在模拟真实 App 的写法，但不连接真实 SafeCampus 后端。连接失败后，部分功能会使用本地 fallback，也就是备用结果。

---

# 6. API 是什么？

API 是 Application Programming Interface。

在这个语境下，它通常指客户端与服务器之间约定的通信接口。

可以把 API 想象成服务器公开的一组“远程函数”：

```text
POST /api/v1/auth/login
GET  /campus/alerts
POST /support/request
GET  /access/sync
```

客户端不能直接进入服务器内存调用 Java 方法，所以它需要发送网络消息：

```http
POST /support/request HTTP/1.1
Host: api.safecampus.example
Content-Type: application/x-www-form-urlencoded

issue=Badge%20access%20test
```

服务器收到请求后：

1. 识别路径；
2. 解析参数；
3. 检查身份；
4. 检查权限；
5. 执行业务操作；
6. 返回结果。

例如：

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "reference": "SC-78755",
  "status": "recorded"
}
```

API 其实是一份协议约定：

```text
客户端发送什么
服务器如何解释
服务器返回什么
客户端如何解释
```

如果双方对格式理解不同，就会出现普通 bug；如果攻击者可以利用这种理解差异影响安全决策，就可能成为漏洞。

SafeCampus 的 raw form body 问题就是一个例子。

代码大致相当于：

```java
String payload = "issue=" + body;
```

如果用户输入：

```text
topic=badge&urgent=true
```

程序发送：

```text
issue=topic=badge&urgent=true
```

按表单协议，`&` 通常表示下一个参数，所以服务器可能理解为：

```text
issue = topic=badge
urgent = true
```

而客户端原意是：

```text
issue = topic=badge&urgent=true
```

正确编码后应该类似：

```text
issue=topic%3Dbadge%26urgent%3Dtrue
```

这说明 API 安全不仅涉及加密，还涉及消息格式、解析和验证。

---

# 7. 后端服务器是什么？

后端是运行在服务提供方控制环境中的程序，而不是用户手机上的代码。

现实中的 SafeCampus 后端可能包括：

```text
API Gateway
    ↓
Authentication Service
    ↓
Application Server
    ├── Student Profile Service
    ├── Campus Alert Service
    ├── Support Ticket Service
    ├── Account Recovery Service
    └── Access Sync Service
```

后端具有客户端没有的特点：

- 用户通常不能读取服务器源代码；
- 用户通常不能直接修改服务器程序；
- 服务器控制数据库；
- 服务器可以集中执行权限检查；
- 服务器可以保存不可公开的秘密；
- 服务器可以记录所有用户的安全事件。

因此，真正的身份认证通常应该这样：

```text
用户输入用户名和密码
        ↓ HTTPS
服务器验证密码
        ↓
服务器创建受保护的 session/token
        ↓
客户端保存短期凭证
        ↓
之后每个请求都携带凭证
        ↓
服务器再次检查权限
```

而不是：

```text
App 内部保存正确用户名和密码
        ↓
App 自己做字符串比较
        ↓
匹配就显示 Dashboard
```

后者只能控制这份未经修改的 APK 的界面，不能建立服务器端安全边界。

---

# 8. 数据库是什么位置？

数据库通常位于服务器端：

```text
Android App
    ↓ HTTPS
Backend API
    ↓ 经过认证、授权、验证的查询
Database
```

正常情况下，手机 App 不应该直接连接生产数据库。

这是因为如果 App 直接掌握数据库地址和数据库凭据：

- 凭据会被反编译出来；
- 用户可能绕过业务规则；
- 数据库暴露到网络；
- 很难执行细粒度权限控制；
- 数据库结构会直接暴露。

所以 App 通常只知道 API，例如：

```text
POST /api/v1/auth/login
```

API 服务器才知道如何访问：

```text
campus_identity database
```

SafeCampus 登录失败时显示：

```text
database=campus_identity
internalServer=10.0.2.15
serviceAccount=auth_api_user
```

这些内容看起来像服务器内部实现细节。

问题不在于用户能直接访问那个数据库——我们没有证明这一点。问题是客户端不应向普通用户暴露这些内部信息，因为它能帮助攻击者建立服务器结构地图。

---

# 9. 身份认证和授权有什么区别？

这是整个应用安全中最重要的区分之一。

## Authentication：身份认证

回答：

> 你是谁？

例如：

```text
用户名 + 密码
指纹
硬件密钥
验证码
登录 token
```

用户登录后，系统可能确认：

```text
你是学生 Maya Chen。
```

## Authorization：授权

回答：

> 你被允许做什么？

例如：

```text
普通学生可以查看自己的资料
支持人员可以查看工单
管理员可以修改校园警报
学生不能查看其他学生资料
```

认证正确不等于可以做所有事情。

正常请求应该经历：

```text
请求到达服务器
    ↓
认证：这个 token 属于谁？
    ↓
授权：这个用户能否执行该操作？
    ↓
输入验证
    ↓
业务逻辑
```

SafeCampus 的 `profile_bundle` 测试并没有绕过认证。你仍然完成了正常课堂登录。

它证明的是：

> 一个登录前由外部提供的值，被带到登录后，并影响了内部导航和持久状态。

因此它更接近“信任外部输入影响授权后状态”的问题，而不是“无需密码登录”。

这个措辞差别对报告非常重要。

---



# 10. 操作系统在架构中的位置

Android 操作系统位于 App 和硬件之间，并提供：

- 进程管理；
- 内存隔离；
- 文件权限；
- 应用沙箱；
- 网络接口；
- 相机、麦克风、定位等权限；
- Activity 启动和生命周期；
- Intent 消息传递；
- 密钥存储；
- 包安装和签名验证。

简化关系是：

```text
用户
 ↓
SafeCampus App
 ↓ 调用 Android API
Android Framework
 ↓
Linux 内核
 ↓
设备硬件
```

Android 不只是“运行 Java 的界面”。它本身就是安全架构的一部分。

例如：

- Manifest 告诉 Android 哪些组件存在；
- `exported=true` 告诉 Android 外部是否能启动组件；
- Linux UID 隔离每个 App 的私有文件；
- `debuggable=true` 改变调试访问能力；
- Intent 由 Android 负责在组件之间传递；
- 权限由 Android 系统执行。

因此，Android 漏洞分析经常同时涉及：

```text
应用代码
+
Manifest 配置
+
Android 操作系统规则
```

只看 Java 文件可能不够。

---

# 11. Trust Boundary：信任边界

信任边界是：

> 数据从一个信任等级进入另一个信任等级的位置。

常见边界包括：

```text
用户输入 → App
外部 App → exported Activity
App → 后端 API
互联网 → App
服务器 → 数据库
ADB 操作者 → App 私有目录
反编译者 → APK 内部逻辑
```

穿越信任边界的数据应该被当作不可信输入。

例如：

```text
外部 profile_bundle
        ↓
MainActivity
```

这是一个信任边界，因为数据可能由 SafeCampus 之外的调用者提供。

再例如：

```text
网络响应
        ↓
ResponseParser
```

也是信任边界。服务器响应不应仅因为“来自网络连接”就被无条件相信：

- 连接可能被中间人影响；
- 服务器可能出错；
- 响应可能损坏；
- 内容可能超长；
- 格式可能与预期不符。

因此通常需要：

- 验证服务器身份；
- 检查 HTTP 状态码；
- 检查 Content-Type；
- 限制响应大小；
- 按 schema 解析；
- 验证字段类型和范围；
- 失败时进入明确的失败状态。

SafeCampus 的 `ResponseParser` 基本把响应当作任意文本接受，这就是我们发现的一个较弱、但有依据的额外问题。

---

# 12. 真实系统中的一次完整登录

假设这是一个真实、安全设计的 SafeCampus。

用户点击登录后的完整流程可能是：

```text
1. LoginActivity 收集用户名和密码
                ↓
2. 检查输入是否为空
                ↓
3. ApiClient 创建 HTTPS 请求
                ↓
4. TLS 验证服务器证书和域名
                ↓
5. 登录请求发送到服务器
                ↓
6. 服务器查找账号
                ↓
7. 服务器验证密码 hash
                ↓
8. 服务器检查账号状态
                ↓
9. 服务器签发 session/token
                ↓
10. App 安全保存短期 token
                ↓
11. App 请求当前用户资料
                ↓
12. 服务器根据 token 返回该用户的资料
                ↓
13. Dashboard 显示数据
```

其中安全责任是分散的：

| 阶段 | 主要安全责任 |
|---|---|
| App UI | 不泄露密码，合理处理输入 |
| HTTPS/TLS | 保护传输并验证服务器身份 |
| 登录 API | 防暴力尝试、统一错误响应 |
| 密码系统 | 加盐并使用慢速密码哈希 |
| Session 系统 | 生成不可预测、可过期的凭证 |
| 授权系统 | 每个敏感操作重新检查权限 |
| App 存储 | 安全保存必要的短期凭证 |
| 日志系统 | 记录安全事件但避免敏感信息泄露 |

---

# 13. SafeCampus 实际上做了什么？

SafeCampus 是教学 APK，不是真实生产系统。它保留了一个真实 App 的外形和部分结构，但主动加入了安全问题。

它的登录流程更接近：

```text
用户输入用户名和密码
        ↓
LoginActivity
        ↓
与 APK 内硬编码的 demo 字符串比较
        ↓
匹配
        ↓
在本地创建 SessionState
        ↓
载入本地虚构资料
        ↓
进入 Dashboard
```

它同时还包含看起来像网络客户端的结构：

```text
ApiClient
ConnectionPolicy
ResponseParser
```

但目标域名是：

```text
api.safecampus.invalid
```

因此没有真实 SafeCampus 后端。

部分功能的逻辑是：

```text
尝试虚构网络请求
        ↓ 失败
捕获异常
        ↓
显示本地 mock/fallback 数据
```

这解释了为什么 Support Request 页面可以显示：

```text
Mock success: classroom request recorded
```

它不代表请求真的到达某个服务器。相反，它表示网络操作失败后，应用生成了一个本地成功式结果。

---

# 14. SafeCampus 的实际分层地图

```text
┌────────────────────────────────────────────┐
│ UI / Android Activity                      │
│                                            │
│ LoginActivity                              │
│ DashboardActivity                          │
│ SupportRequestActivity                     │
│ AccountRecoveryActivity                    │
│ ServiceStatusActivity                      │
└───────────────────┬────────────────────────┘
                    ↓
┌────────────────────────────────────────────┐
│ Application / Business Logic               │
│                                            │
│ AppStateResolver                           │
│ AccountWorkflow                            │
│ ReferenceCalculator                        │
│ SupportTicketRepository                    │
│ CampusAlertRepository                      │
│ CampusSyncService                          │
└───────────────┬─────────────────┬──────────┘
                ↓                 ↓
┌──────────────────────┐  ┌──────────────────┐
│ Local State          │  │ Network Layer    │
│                      │  │                  │
│ SessionState         │  │ ApiClient        │
│ AppPreferences       │  │ ConnectionPolicy │
│ SharedPreferences    │  │ ResponseParser   │
└──────────────────────┘  └─────────┬────────┘
                                    ↓
                          api.safecampus.invalid
                          不存在的教学域名
```

这里没有我们能够观察到的真实：

- SafeCampus 服务器；
- 真实学生数据库；
- 真实认证服务；
- 真实工单系统；
- 真实校园门禁系统。

所以我们审计的是：

> APK 中实现出来的客户端安全模型，以及这些代码如果作为真实设计模式使用时会有什么风险。

---

# 15. 每个现有 finding 属于架构的哪一层？

| Finding | 所属位置 | 被破坏的基本规则 |
|---|---|---|
| TLS 验证被关闭 | 网络传输层 | 客户端没有可靠验证服务器身份 |
| `profile_bundle` 影响状态 | Android 组件与业务逻辑 | 外部输入被用于内部安全相关决策 |
| `debuggable=true` | 构建配置与本地存储 | 生产式应用向本地调试者开放私有状态 |
| 登录失败泄露调试信息 | UI、错误处理 | 内部诊断信息暴露给普通用户 |
| 客户端生成恢复码 | 恢复业务逻辑 | 安全凭证由不可信客户端确定性生成 |
| 硬编码 demo 凭据 | 客户端认证 | 身份验证依赖可反编译的客户端秘密 |
| raw form body | API 编码边界 | 用户输入未经协议编码便进入请求 |
| 任意文本响应 | 网络响应处理 | 外部数据未经结构验证便进入 UI/状态 |
| 失败后显示成功 | Repository 与错误处理 | 系统无法区分操作成功和失败 |

你可以看到，这些漏洞并非互不相关的随机错误。它们集中反映了几个更大的架构问题：

```text
不可信客户端承担了太多安全责任
外部输入没有经过充分验证
网络身份和响应没有被可靠验证
错误状态与成功状态没有清楚区分
调试配置和内部信息进入了发布版本
```

---

# 16. 本模块最重要的原则

### 1. 客户端不可信

用户拥有 APK、设备和输入，因此可以检查或修改客户端。

### 2. 服务器负责最终安全决策

认证、授权、敏感状态改变和核心业务规则不能只靠客户端执行。

### 3. 所有跨越信任边界的数据都不可信

包括：

- 用户输入；
- Intent extra；
- 深度链接；
- 网络响应；
- 本地可修改状态；
- 导入文件。



---

# 模块二：Android 应用结构——Manifest、Activity、Intent、沙箱、SharedPreferences 与 ADB

这一模块的核心目标是让你理解：

> SafeCampus 安装到模拟器以后，Android 如何识别它、启动它、隔离它、在页面之间传递数据，以及为什么 `profile_bundle` 和 `debuggable=true` 会成为安全问题。

先看整体关系：

```text
APK 安装
  ↓
Android 读取 Manifest
  ↓
为应用分配包名、Linux UID 和私有目录
  ↓
用户或外部程序启动 Activity
  ↓
Android 通过 Intent 传递启动请求和参数
  ↓
Activity 执行业务逻辑并打开其他 Activity
  ↓
应用通过 SharedPreferences 保存本地状态
```

---

## 1. Android 在整个计算机系统中的位置

Android 并不是单一程序，而是一整套软件栈：

```text
┌─────────────────────────────┐
│ SafeCampus 等应用           │
├─────────────────────────────┤
│ Android Framework           │
│ Activity、Intent、权限、UI   │
├─────────────────────────────┤
│ Android Runtime             │
│ 执行 DEX 字节码             │
├─────────────────────────────┤
│ Native Libraries            │
│ TLS、SQLite、图像等          │
├─────────────────────────────┤
│ Linux Kernel                │
│ 进程、UID、文件权限、网络    │
├─────────────────────────────┤
│ 设备硬件                    │
└─────────────────────────────┘
```

从你熟悉的理论视角看：

- Linux 内核提供基本隔离机制；
- Android Framework 把这些机制包装成移动应用模型；
- App 使用 Android 提供的接口；
- 每个 App 默认运行在自己的进程和 Linux 身份下。

Android 的安全并不是单靠 Java 语言实现，而是由：

```text
Linux UID
+ 文件权限
+ 应用签名
+ Manifest 权限
+ 组件导出规则
+ Android Runtime
```

共同构成。

---

# 2. Package：应用的系统身份

SafeCampus 的包名是：

```text
au.edu.safecampus.connect
```

包名可以理解为 Android 系统中应用的唯一标识符。

Android 使用它来区分：

```text
com.example.bank
com.example.browser
au.edu.safecampus.connect
```

包名参与：

- 安装和卸载；
- 应用私有目录命名；
- ADB 操作；
- 组件地址；
- 权限声明；
- 应用升级识别。

例如 SafeCampus 的私有目录是：

```text
/data/user/0/au.edu.safecampus.connect/
```

启动它的某个页面时，可以使用：

```text
au.edu.safecampus.connect/.MainActivity
```

这里：

```text
包名：au.edu.safecampus.connect
组件：MainActivity
```

不过包名本身不是身份认证秘密，攻击者很容易从 APK 或设备中找到它。

---

# 3. APK 安装时发生了什么？

当 Android 安装一个 APK 时，大致会做这些事：

1. 解析 APK；
2. 检查文件格式；
3. 检查数字签名；
4. 读取 `AndroidManifest.xml`；
5. 确认包名；
6. 注册 Activity 等组件；
7. 为应用分配 Linux UID；
8. 创建私有数据目录；
9. 准备 DEX 代码运行；
10. 把应用加入系统的包管理数据库。

因此，APK 不只是“代码文件”。安装行为还会在 Android 系统中注册一系列信息。

---

# 4. Android 应用签名是什么？

每个正常 APK 都需要数字签名。

这个签名主要用于证明：

> 新安装或升级的 APK，是否由与旧版本相同的签名密钥签署。

假设手机中已经有：

```text
SafeCampus v1
```

后来要升级到：

```text
SafeCampus v2
```

Android 会检查 v2 是否使用和 v1 兼容的签名身份。否则，通常不能直接覆盖原应用。

这可以防止攻击者随便制作一个同包名恶意 APK，然后作为正式升级覆盖原应用。

但 APK 签名不意味着：

- APK 代码是安全的；
- 开发者没有写漏洞；
- APK 内容不可反编译；
- App 与服务器通信一定安全；
- 安装者一定知道真实开发者是谁。

它主要保护软件包来源连续性和更新关系，不负责检查业务逻辑。

---

# 5. AndroidManifest.xml 是什么？

Manifest 是 Android 应用的总注册表和配置文件。

它告诉 Android：

- 应用包是什么；
- 有哪些组件；
- 哪个 Activity 是入口；
- 哪些组件允许外部启动；
- 应用需要哪些权限；
- 是否允许明文网络；
- 是否可调试；
- 使用什么主题；
- 支持哪些 Android 版本。

可以把它类比为：

```text
程序的目录
+
对操作系统提交的组件清单
+
部分安全策略
```

一个简化 Manifest 可能是：

```xml
<manifest ...>

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:debuggable="false"
        android:networkSecurityConfig="@xml/network_security_config">

        <activity
            android:name=".MainActivity"
            android:exported="true">

            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

        </activity>

        <activity
            android:name=".DashboardActivity"
            android:exported="false" />

    </application>
</manifest>
```

安装时，Android 根据这份文件决定系统如何与应用交互。

SafeCampus 的关键配置包括：

- `MainActivity` 是启动入口；
- `MainActivity` 可以被外部启动；
- `ServiceStatusActivity` 不直接对外导出；
- 应用设置了 `android:debuggable="true"`；
- 应用引用了网络安全配置。

---

# 6. Android 组件是什么？

Android 应用通常由四类主要组件构成：

| 组件 | 典型用途 |
|---|---|
| Activity | 一个可交互页面 |
| Service | 没有页面的后台任务 |
| BroadcastReceiver | 接收系统或应用广播 |
| ContentProvider | 向其他组件提供结构化数据 |

这次 SafeCampus 作业最重要的是 Activity。

注意：Android 的 `Service` 组件和普通 Java 类名中的 `CampusSyncService` 不一定是一回事。

一个类叫：

```java
CampusSyncService
```

并不自动意味着它是 Android Framework 的 `Service` 组件。必须看它是否继承：

```java
android.app.Service
```

以及是否在 Manifest 中声明。

开发者可以随意给普通业务类加上 `Service` 后缀。

---

# 7. Activity 是什么？

Activity 可以暂时理解为一个页面控制器。

例如：

```text
LoginActivity           登录页面
DashboardActivity       主菜单页面
SupportRequestActivity  支持请求页面
ServiceStatusActivity   服务状态页面
```

Activity 不只是界面布局本身，它还负责页面的生命周期和交互逻辑。

一个简化 Activity：

```java
public class LoginActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);
    }
}
```

其中：

- `onCreate()`：Activity 被创建时执行；
- `setContentView()`：指定界面布局；
- `findViewById()`：找到按钮或文本框；
- `setOnClickListener()`：设置点击事件；
- `startActivity()`：打开另一个 Activity。

---

# 8. Activity 生命周期

Android 手机资源有限，而且用户经常：

- 切换应用；
- 旋转屏幕；
- 返回上一页；
- 把 App 放到后台；
- 被系统回收进程。

因此 Activity 有一套生命周期：

```text
onCreate()
    ↓
onStart()
    ↓
onResume()
    ↓
用户正在操作
    ↓
onPause()
    ↓
onStop()
    ↓
onDestroy()
```

最重要的几个：

| 方法 | 含义 |
|---|---|
| `onCreate()` | Activity 首次创建，通常初始化页面 |
| `onResume()` | 页面回到前台，可与用户交互 |
| `onPause()` | 页面即将失去前台位置 |
| `onStop()` | 页面已经不可见 |
| `onDestroy()` | Activity 被销毁 |

为什么安全分析关心生命周期？

因为开发者可能：

- 只在 `onCreate()` 检查登录；
- 页面恢复时没有重新检查权限；
- 把敏感数据留在界面；
- 在生命周期变化中错误恢复状态；
- 接受旧 Intent 或缓存状态；
- 把临时安全状态永久保存。

SafeCampus 的状态持久化问题就和页面重新启动后的状态恢复有关。

---

# 9. Intent 是什么？

Intent 是 Android 用来表达“我想做什么”的消息对象。

例如：

```text
打开另一个页面
启动后台操作
打开网页
拍照
分享文本
把一些参数传给其他组件
```

你可以把它理解为 Android 组件之间传递的结构化消息。

一个 Intent 通常包含：

```text
目标组件
动作 action
数据 URI
类别 category
附加参数 extras
Flags
```

例如在 App 内部打开 Dashboard：

```java
Intent intent = new Intent(this, DashboardActivity.class);
startActivity(intent);
```

这里明确指定了目标：

```text
DashboardActivity
```

---

# 10. Explicit Intent 与 Implicit Intent

## Explicit Intent：显式 Intent

明确指定接收组件：

```java
new Intent(this, DashboardActivity.class)
```

含义是：

> Android，请启动当前应用中的 DashboardActivity。

这种方式通常用于应用内部导航。

## Implicit Intent：隐式 Intent

只描述要执行的动作，不指定具体应用：

```java
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setData(Uri.parse("https://example.com"));
startActivity(intent);
```

含义是：

> Android，请寻找一个能打开网页的应用。

系统可能让浏览器处理。

隐式 Intent 更依赖 Android 的组件匹配机制，因此需要注意：

- 哪个应用会接收；
- 是否可能被恶意应用拦截；
- 接收方是否可信；
- 发送的数据是否敏感。

SafeCampus 的主要问题更接近外部调用者显式启动一个 exported Activity，并附带 extra。

---

# 11. Intent Extra 是什么？

Extra 是附加在 Intent 上的 key/value 数据。

例如：

```java
Intent intent = new Intent(this, DashboardActivity.class);
intent.putExtra("username", "Maya Chen");
startActivity(intent);
```

接收方读取：

```java
String username =
    getIntent().getStringExtra("username");
```

可以把它想象成函数参数：

```text
openDashboard(username="Maya Chen")
```

但它不是普通的进程内函数调用参数。Intent 可能来自：

- 应用自己的组件；
- Android 系统；
- 另一个 App；
- 浏览器链接；
- 通知；
- ADB；
- 自动化测试工具。

所以安全规则是：

> 不能因为某个值来自 Intent，就默认它由自己应用生成。

必须先判断接收组件是否可能被外部访问。

---

# 12. `android:exported` 是什么？

`exported` 决定一个 Android 组件是否可以被当前 App 之外的调用者启动。

简化理解：

```xml
android:exported="false"
```

表示：

> 主要供本应用内部使用，外部应用通常不能直接启动。

而：

```xml
android:exported="true"
```

表示：

> 该组件对应用边界之外开放。

`exported=true` 不一定是漏洞。启动器 Activity 通常就需要导出，否则 Android 桌面无法启动它。

问题在于：

> 一个 exported 组件接收的输入，应当被视为外部不可信输入。

---

# 13. 为什么 MainActivity 必须 exported？

SafeCampus 的 `MainActivity` 是应用入口。

它会声明类似：

```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
</intent-filter>
```

含义是：

- `MAIN`：这是应用主入口；
- `LAUNCHER`：桌面启动器可以显示并启动它。

Android 桌面本身不是 SafeCampus 进程的一部分，所以入口 Activity 需要允许外部系统启动。

因此：

```text
MainActivity exported=true
```

本身很正常。

漏洞来自它之后做了什么：

```text
外部可以启动 MainActivity
        ↓
MainActivity 接受 profile_bundle
        ↓
没有建立可信来源
        ↓
把这个值转发进登录后流程
```

---

# 14. SafeCampus 的 `profile_bundle` 数据流

这是理解 Intent 安全最好的实际例子。

整个路径是：

```text
外部调用者 / ADB
    │
    │ 启动 MainActivity
    │ extra:
    │ profile_bundle=c3ZjOmxhbnRlcm46NDI=
    ↓
MainActivity
    │
    │ 读取 profile_bundle
    │ 转发给 LoginActivity
    ↓
LoginActivity
    │
    │ 用户仍需完成正常登录
    │ 登录后继续转发
    ↓
DashboardActivity
    │
    │ 调用 AppStateResolver
    ↓
AppStateResolver
    │
    │ Base64 解码
    │ 与预期值比较
    ↓
匹配 svc:lantern:42
    │
    ├── 写入 workspace_state=true
    └── 返回“打开服务状态”
                 ↓
DashboardActivity 启动 ServiceStatusActivity
```

这里需要注意两个结论。

### 它没有绕过登录

用户仍然输入了课程提供的正常 demo 凭据。

因此不应称作：

```text
Authentication bypass
```

更准确的是：

```text
Externally supplied Intent data influences
post-login navigation and persistent state.
```

### ServiceStatusActivity 本身不需要 exported

即使：

```xml
ServiceStatusActivity exported=false
```

外部仍然可以通过一条间接路径影响它被打开：

```text
外部
→ exported MainActivity
→ 内部 LoginActivity
→ 内部 DashboardActivity
→ 内部 ServiceStatusActivity
```

所以：

> 单独把敏感页面设置为 `exported=false` 并不能解决所有问题。

还必须确保所有能间接到达它的入口都正确验证外部输入和权限。

---



---

# 15. SharedPreferences 是什么？

SharedPreferences 是 Android 提供的轻量 key/value 存储。

代码可能类似：

```java
SharedPreferences prefs =
    getSharedPreferences("campus_session_cache", MODE_PRIVATE);

prefs.edit()
    .putString("profile_name", "Maya Chen")
    .putBoolean("workspace_state", true)
    .apply();
```

磁盘上通常会形成类似 XML：

```xml
<map>
    <string name="profile_name">Maya Chen</string>
    <boolean name="workspace_state" value="true" />
</map>
```

适合保存：

- 显示主题；
- 用户偏好；
- 简单状态；
- 非敏感配置。

不应未经额外考虑直接保存：

- 明文密码；
- 长期身份凭据；
- 私钥；
- 高价值访问 token；
- 可直接重置账号的秘密；
- 大量敏感个人资料。

`MODE_PRIVATE` 的含义主要是文件只能由该应用 UID 正常访问。它不代表：

- 数据经过加密；
- root 无法读取；
- 可调试应用无法通过 `run-as` 读取；
- 设备备份永远无法获得；
- 应用自身漏洞无法泄露。

“私有”和“加密”是两件不同的事。

---

# 16. `SessionState` 在 SafeCampus 中做什么？

SafeCampus 使用 `SessionState` 和相关 preferences 保存本地应用状态。

我们动态观察到的虚构值包括：

```text
profile_name = Maya Chen
profile_ref = SCU-2026-1842
device_label = Android 16 classroom device
view_anchor = 42
workspace_state = true
```

这些值的性质不同：

| 值 | 作用 |
|---|---|
| `profile_name` | 显示虚构用户名称 |
| `profile_ref` | 显示虚构学生编号 |
| `device_label` | 显示本地设备标签 |
| `view_anchor` | 参与构造教学状态值 |
| `workspace_state` | 记录是否进入特殊页面状态 |

其中 `workspace_state=true` 很重要，因为它解释了为什么：

1. 使用 `profile_bundle` 启动；
2. 登录后进入 Service Status；
3. 之后不带 extra 正常启动；
4. 仍然再次进入 Service Status。

这说明外部输入不仅产生了一次临时页面跳转，还触发了本地持久状态。

---

# 17. ADB 是什么？

ADB 是 Android Debug Bridge。

它是开发者电脑与 Android 设备或模拟器之间的调试工具。

关系是：

```text
你的电脑
    ↓ ADB client
ADB server
    ↓ USB 或模拟器连接
Android 设备上的 ADB daemon
```

ADB 可以用于合法开发工作，例如：

- 查看连接设备；
- 安装 APK；
- 卸载 APK；
- 查看日志；
- 启动 Activity；
- 传入 Intent extra；
- 打开系统 shell；
- 调试应用；
- 复制测试文件；
- 自动化测试。

我们使用 ADB 的目的不是攻击真实设备，而是在课程授权模拟器中控制测试条件。

---

# 18. Android Emulator 是什么？

Android Emulator 是在电脑上模拟 Android 设备的软件。

你创建的 Pixel 6 虚拟设备包含：

- 模拟 CPU；
- Android 系统镜像；
- 虚拟磁盘；
- 虚拟屏幕；
- 虚拟网络；
- Android Framework；
- ADB 接口。

它和 Android Studio 的关系是：

```text
Android Studio
├── Device Manager：管理虚拟设备
├── Emulator：实际运行 Android
├── SDK Tools：ADB 等工具
└── 编辑和构建环境
```

SafeCampus 被安装在：

```text
emulator-5554
```

这只是该模拟器当前在 ADB 中的设备标识。

所有动态测试都限制在这个本地课程模拟器和指定 APK 中。

---

# 19. `run-as` 是什么？

`run-as` 是 Android 提供的调试工具，可以尝试以某个应用的 Linux UID 执行命令。

概念上：

```text
当前 ADB shell 身份
        ↓ run-as package
目标应用 UID
        ↓
访问目标应用私有目录
```

我们对 SafeCampus 做的测试相当于：

```text
请求以 au.edu.safecampus.connect 身份运行
        ↓
Android 允许
        ↓
当前命令成为 u0_a213
        ↓
读取该应用的 SharedPreferences
```

关键点：

- 我们不是 root；
- 我们没有绕过其他真实设备的权限；
- 我们具有本地模拟器 ADB 权限；
- 目标 APK 标记为 debuggable；
- Android 因此允许针对该包进行调试式访问。

这就是 `android:debuggable="true"` finding 的动态证据。

---

# 20. `android:debuggable=true` 为什么危险？

开发阶段，调试能力很有用：

- 连接 debugger；
- 检查变量；
- 查看应用状态；
- 分析性能；
- 使用 `run-as`；
- 调试数据库和文件。

但发布版本一般应该：

```xml
android:debuggable="false"
```

因为持有设备调试访问能力的人可能更容易：

- 检查应用私有文件；
- 读取本地数据库；
- 检查 SharedPreferences；
- 调试应用控制流；
- 检查运行时状态；
- 对应用进行动态分析。

这并不自动等于远程漏洞。攻击者通常至少需要：

- 接触设备或模拟器；
- 获得 ADB 连接能力；
- 设备允许相关调试操作。

因此它的威胁模型是“本地调试访问者”，不是“互联网上的任意攻击者”。

---



# 21. Android Task 和返回栈

Activity 通常按栈组织：

```text
MainActivity
    ↓
LoginActivity
    ↓
DashboardActivity
    ↓
SupportRequestActivity
```

按返回键时，通常反方向退出：

```text
SupportRequestActivity
    ↓ Back
DashboardActivity
    ↓ Back
LoginActivity
```

这个结构叫 back stack。

安全上需要注意：

- 登出后，返回键能否回到受保护页面；
- 敏感 Activity 是否仍留在栈中；
- 登录前 Intent 是否被登录后 Activity 继续使用；
- 应用重启时是否恢复了不应该恢复的页面；
- token 失效后页面是否重新检查权限。

SafeCampus 的 `profile_bundle` 被穿过登录流程继续转发，正是一个跨 Activity 流程的状态传播问题。

---

# 22. 修复 `profile_bundle` 问题应考虑什么？

不能只说“删除 Base64”。

Base64 不是根本原因。根本原因是：

> 不可信的外部输入被用于登录后的内部状态和导航决策。

合理修复方向包括：

1. `MainActivity` 不接收无业务必要的 `profile_bundle`；
2. 登录前外部参数不自动转发到登录后区域；
3. 登录后重新从可信状态计算目标页面；
4. 导航决定依赖服务器授权或内部可信 session；
5. 对外部 Intent 做严格格式和语义验证；
6. 敏感组件增加明确权限检查；
7. 不让外部输入直接写入 `workspace_state`；
8. 退出或重新登录时清理不适用的状态。

如果确实需要某种外部深度链接，正常设计可能是：

```text
外部链接请求某个页面
        ↓
App 将其作为“请求”，而不是命令
        ↓
完成身份认证
        ↓
检查当前用户是否有权限
        ↓
服务器或可信内部逻辑确认
        ↓
满足条件才导航
```

外部输入只能表达意图，不能直接授予权限。

---

# 23. 修复 `debuggable=true` 问题应考虑什么？

最直接的修复是发布构建中使用：

```xml
android:debuggable="false"
```

更完整的工程措施包括：

- 使用不同的 debug 和 release 构建配置；
- CI/CD 阻止 debuggable release 发布；
- 发布前自动检查 Manifest；
- release 版本删除测试日志；
- 不在本地保存不必要的敏感状态；
- 必要时使用 Android Keystore；
- token 采用短期有效和可撤销设计；
- 即使数据泄露，也限制其实际价值。

需要注意：

> 关闭 debuggable 只是减少本地调试暴露，不等于 SharedPreferences 自动变成加密存储。

这是不同层次的防御措施。

---

## 本模块的最终心智模型

现在可以把 SafeCampus 的 Android 运行方式理解为：

```text
APK
 ↓ 安装
Android 读取 Manifest
 ↓
注册 au.edu.safecampus.connect
 ↓
分配应用 UID 和私有目录
 ↓
桌面通过 Intent 启动 exported MainActivity
 ↓
MainActivity 读取 Intent extras
 ↓
Activity 之间继续用 Intent 导航和传参
 ↓
业务类根据参数作出决定
 ↓
SessionState 把部分结果写入 SharedPreferences
```

两个核心漏洞正好出现在这条链的不同位置：

```text
profile_bundle：
外部 Intent 输入跨越信任边界
→ 影响登录后导航
→ 写入持久状态

debuggable=true：
发布配置开放调试能力
→ 本地 ADB 使用 run-as
→ 读取应用私有状态
```


> SafeCampus 发起一次 HTTPS 请求时，从应用代码到 Android 系统再到网络，究竟经过哪些组件？每个组件是谁提供的？SafeCampus 又修改了什么？

# 模块三：Android 中一次 HTTPS 请求是怎样组织起来的？

先看总图：

```text
SafeCampus 功能代码
    │  决定请求什么数据
    ↓
SafeCampus 的 ApiClient
    │  创建并配置 HTTPS 请求
    ↓
Android/Java 的 HttpsURLConnection
    │  实现 HTTP，并调用 TLS 组件
    ↓
Android TLS 实现
    ├── TrustManager：验证证书链
    ├── HostnameVerifier：检查域名
    ├── 系统 CA 信任库
    └── 加密、握手、会话密钥
    ↓
Android 网络系统
    ├── DNS：域名解析
    ├── TCP：建立可靠连接
    └── IP：把数据包送往目标地址
    ↓
Wi-Fi / 虚拟网络 / 互联网
    ↓
后端服务器
```

在真实应用里，最下面应该有一个真实服务器。SafeCampus 使用的是：

```text
https://api.safecampus.invalid/
```

这个地址被故意设计成不存在，所以课程 APK 没有真实后端。网络连接失败后，部分功能会改用本地 mock/fallback 结果。

---

## 1. 最上层：功能页面决定“为什么要联网”

最上层不是网络协议，而是 App 功能。

例如 SafeCampus 有三个会尝试访问网络的功能：

```text
Campus Alerts
Support Request
Sync Campus Access
```

对应的调用方向大致是：

```text
页面 Activity
    ↓
Repository 或 Service
    ↓
ApiClient
```

具体来说：

```text
Campus Alerts 页面
    ↓
CampusAlertRepository
    ↓
ApiClient.fetchCampusAlerts()
```

```text
Support Request 页面
    ↓
SupportTicketRepository
    ↓
ApiClient.submitSupportRequest(issue)
```

```text
Sync Campus Access 页面
    ↓
CampusSyncService
    ↓
ApiClient.fetchSyncEnvelope()
```

这一层负责回答：

- 用户想执行什么操作？
- 应该调用哪个 API？
- 请求中放什么业务数据？
- 网络失败后界面显示什么？
- 返回结果怎样进入应用状态？

这一层通常由 App 开发者编写。

它不负责亲自实现 TCP、IP 或 AES 加密。

---

## 2. `ApiClient`：应用自己的网络入口

SafeCampus 把网络请求集中在：

```text
ApiClient.java
```

它是课程 App 自己写的类，不是 Android 系统组件。

它负责：

- 保存服务器基础地址；
- 拼接 API 路径；
- 选择 GET 或 POST；
- 设置请求头；
- 设置超时时间；
- 写入请求体；
- 读取服务器响应；
- 为连接应用 TLS 配置。

例如服务器基础地址是：

```java
private static final String BASE_URL =
    "https://api.safecampus.invalid/";
```

支持请求大致按下面的方式构造：

```java
HttpsURLConnection connection =
    createConnection("support/request");

connection.setRequestMethod("POST");
connection.setDoOutput(true);

byte[] payload =
    ("issue=" + body).getBytes("UTF-8");

connection.setRequestProperty(
    "Content-Type",
    "application/x-www-form-urlencoded"
);

connection.getOutputStream().write(payload);
```

这里 `ApiClient` 决定了应用层内容：

```text
目标路径：support/request
方法：POST
Content-Type：表单格式
正文：issue=<用户输入>
```

但是 `ApiClient` 没有自己从头实现 HTTPS。它调用的是 Android/Java 提供的：

```java
HttpsURLConnection
```

---

## 3. `HttpsURLConnection`：Android 提供的高级网络接口

`HttpsURLConnection` 是平台网络库提供的类。

开发者通常只需要说：

```java
URL url = new URL("https://example.com/path");

HttpsURLConnection connection =
    (HttpsURLConnection) url.openConnection();
```

然后设置：

```java
connection.setRequestMethod("POST");
connection.setRequestProperty(...);
connection.getOutputStream().write(...);
connection.getInputStream();
```

平台会在背后完成：

- 解析 URL；
- 查找域名；
- 建立 TCP 连接；
- 发起 TLS 握手；
- 验证服务器证书；
- 验证证书域名；
- 加密 HTTP 请求；
- 接收和解密响应；
- 向 App 提供响应数据流。

所以正常 Android 开发者不需要自己编写：

- TCP 重传；
- IP 路由；
- AES-GCM；
- ECDHE；
- X.509 证书解析；
- TLS 报文格式。

这些由 Android 系统和其网络/密码学库完成。

---

## 4. 实际连接并不是在 `openConnection()` 时立刻发生

这点有助于读代码。

下面这句：

```java
url.openConnection()
```

通常只是创建一个连接对象。此时可能还没有真正访问网络。

之后代码设置：

```java
请求方法
请求头
超时
TLS 配置
HostnameVerifier
请求正文
```

当程序执行以下操作时，才通常真正触发网络连接：

```java
connection.connect();
connection.getOutputStream();
connection.getInputStream();
connection.getResponseCode();
```

因此实际顺序大致是：

```text
创建连接对象
    ↓
配置 HTTP 和 TLS
    ↓
第一次真正读写连接
    ↓
DNS
    ↓
TCP
    ↓
TLS 握手
    ↓
发送 HTTP
    ↓
接收 HTTP 响应
```

SafeCampus 的 `ConnectionPolicy` 就是在真正连接发生前，修改了连接对象的 TLS 验证方式。

---

# 5. Android TLS 部分由谁完成？

TLS 不是由 `ApiClient` 自己实现的。

它由 Android 平台内的 TLS/密码学实现完成。应用通过 Java 安全接口调用这些实现，例如：

```text
SSLContext
SSLSocketFactory
X509TrustManager
HostnameVerifier
```

可以把职责拆成：

| 组件 | 谁提供 | 职责 |
|---|---|---|
| `SSLContext` | Android/Java 安全框架 | 表示一套 TLS 配置 |
| `SSLSocketFactory` | TLS 框架 | 创建执行 TLS 的安全 socket |
| `X509TrustManager` | 默认由平台提供，也可被 App 替换 | 验证证书链是否可信 |
| `HostnameVerifier` | 默认由平台提供，也可被 App 替换 | 验证证书是否属于目标域名 |
| 系统 CA 信任库 | Android 系统 | 保存受信任根 CA |
| 密码学算法实现 | Android 密码学提供者 | 执行密钥交换、签名验证和认证加密 |

正常情况下，开发者不需要替换这些组件。

平台默认完成：

```text
服务器发送证书
    ↓
默认 TrustManager 验证证书链
    ↓
默认 HostnameVerifier 检查域名
    ↓
都通过
    ↓
继续建立 TLS 通道
```

SafeCampus 的问题是：它主动替换了平台默认行为。

---

# 6. `TrustManager` 在系统中的具体位置

服务器建立 TLS 连接时会发送证书链。

例如：

```text
服务器证书
    ↓
中间 CA
    ↓
系统信任的根 CA
```

`X509TrustManager` 的工作是判断这条链是否值得信任。

它会处理的问题包括：

- 签名是否有效；
- 证书是否过期；
- 是否能连接到可信根 CA；
- 证书是否允许用于服务器认证；
- 证书链结构是否有效。

正常情况下，Android 使用自己的默认 TrustManager，并参考系统 CA 信任库。

逻辑相当于：

```text
服务器证书链
    ↓
Android 默认 TrustManager
    ↓
系统 CA 信任库
    ↓
可信 → 继续
不可信 → 终止连接
```

SafeCampus 自己创建了一个 `X509TrustManager`，其中：

```java
public void checkServerTrusted(
    X509Certificate[] chain,
    String authType
) {
}
```

方法体为空。

这相当于平台问：

> 这张服务器证书可信吗？

SafeCampus 自定义的验证器回答：

> 我不检查，也不拒绝。

因为方法没有抛出验证异常，证书会被当作可接受。

于是系统默认的证书链判断被替换成：

```text
任何证书 → 接受
```

---

# 7. `HostnameVerifier` 在系统中的具体位置

即使证书链有效，还要检查证书是否属于当前访问的域名。

假设 App 访问：

```text
api.safecampus.example
```

服务器却提供一张属于：

```text
attacker.example
```

的证书。

这张证书可能：

- 没过期；
- 签名正确；
- 由可信 CA 签发；

但它不属于 SafeCampus 的服务器。

因此系统还会调用 `HostnameVerifier`：

```text
URL 中的 hostname
        +
证书声明的域名
        ↓
是否匹配？
```

正常结果应该是：

```text
api.safecampus.example
与证书匹配
→ true

api.safecampus.example
与 attacker.example 不匹配
→ false
```

SafeCampus 的自定义 verifier 无条件返回：

```java
true
```

相当于：

```text
无论证书属于哪个域名
→ 都接受
```

---

# 8. 为什么要分别检查证书链和域名？

因为它们回答不同问题。

```text
TrustManager：
这张证书是否属于一个可信证书体系？

HostnameVerifier：
这张可信证书是否属于我正在访问的服务器？
```

举个现实类比。

一个人出示了真实有效的悉尼大学学生证：

```text
TrustManager：
这确实是大学签发的有效学生证。
```

但你约的是 Maya Chen，而证件属于另一个学生：

```text
HostnameVerifier：
证件有效，但不是我要找的人。
```

只有两项都通过，身份才确认完整。

SafeCampus 同时关闭了两项：

```text
不检查证件真伪
+
不检查证件姓名
```

---

# 9. SafeCampus 的 `ConnectionPolicy` 做了什么？

SafeCampus 自己定义了：

```text
ConnectionPolicy.java
```

它不是 Android 强制要求的类，而是课程 App 开发者写的封装。

它大致完成：

```text
创建一个自定义 SSLContext
        ↓
放入 trust-all X509TrustManager
        ↓
生成对应的 SSLSocketFactory
        ↓
安装到 HttpsURLConnection
        ↓
安装永远返回 true 的 HostnameVerifier
```

于是原本的平台默认结构：

```text
HttpsURLConnection
    ├── Android 默认 TrustManager
    ├── Android 默认 HostnameVerifier
    └── Android 系统 CA
```

被修改成：

```text
HttpsURLConnection
    ├── SafeCampus 自定义 TrustManager
    │      └── 不检查证书
    └── SafeCampus 自定义 HostnameVerifier
           └── 不检查域名
```

底层 TLS 加密算法仍然由 Android 完成，但身份判断已经被 App 改坏了。

这是理解 finding 的关键：

> SafeCampus 没有自己实现整个 TLS，它只是把 Android TLS 系统中的两个验证钩子替换成了“全部允许”。

---

# 10. `network_security_config.xml` 位于哪里？

Android 允许 App 通过 XML 配置网络安全策略，例如：

```text
是否允许 HTTP 明文通信
信任哪些 CA
是否信任用户安装的 CA
是否使用特定域名规则
是否配置证书 pinning
```

该配置通常通过 Manifest 连接到应用：

```xml
<application
    android:networkSecurityConfig=
        "@xml/network_security_config">
```

正常组织关系是：

```text
AndroidManifest.xml
    ↓ 指向
network_security_config.xml
    ↓ 告诉 Android
默认网络连接应该信任什么
```

SafeCampus 的配置中禁止明文通信，并声明系统 CA。这表面上是合理的。

但它不能修复 Java 代码中的自定义 trust-all 逻辑。

因为实际连接又被设置为：

```text
使用这个自定义 SSLSocketFactory
使用这个自定义 HostnameVerifier
```

所以两套东西的关系是：

```text
network_security_config
→ 应用级默认政策

ConnectionPolicy
→ 针对实际连接对象主动安装的自定义政策
```

SafeCampus 的实际连接使用了后者的不安全设置。

---

# 11. DNS 由谁完成？

假设目标 URL 是：

```text
https://api.example.com/
```

应用需要把：

```text
api.example.com
```

转换成 IP 地址。

这一工作通常由 Android 系统的网络解析器完成，应用开发者一般不自己编写 DNS 客户端。

组织关系是：

```text
HttpsURLConnection
    ↓ 请求连接 hostname
Android 网络库
    ↓
系统 DNS resolver
    ↓
配置的 DNS 服务
    ↓
返回 IP 地址
```

SafeCampus 使用 `.invalid` 域名，所以正常情况下无法解析成真实服务地址。

这一步发生在 TLS 之前，因为客户端首先需要知道应该连接哪个 IP。

但是：

> DNS 只帮助定位地址，不负责最终证明服务器身份。

即使 DNS 返回了错误 IP，正确 TLS 验证仍应阻止假服务器。

---

# 12. TCP 和 IP 由谁完成？

域名解析得到 IP 后，Android 会建立 TCP 连接。

这些主要由 Android 的网络库和 Linux 内核完成。

```text
App
 ↓
Java/Android socket API
 ↓
Linux TCP/IP stack
 ↓
网络设备
```

## IP 的职责

IP 主要负责：

- 源地址和目标地址；
- 数据包跨网络路由；
- 尽力把包送到目的地。

IP 本身通常不保证：

- 数据一定到达；
- 顺序正确；
- 不重复；
- 内容加密。

## TCP 的职责

TCP 在 IP 之上提供：

- 可靠字节流；
- 顺序控制；
- 丢包重传；
- 流量控制；
- 拥塞控制；
- 端口。

对于经典 HTTPS：

```text
HTTP
↓
TLS
↓
TCP
↓
IP
```

App 开发者一般不需要亲自实现 TCP 重传。这由操作系统内核完成。

---

# 13. TLS 在什么时候发生？

建立 TCP 连接后，TLS 客户端和服务器执行握手。

Android TLS 库大致负责：

1. 与服务器协商 TLS 版本；
2. 协商密码套件；
3. 执行密钥交换；
4. 接收服务器证书；
5. 调用 TrustManager；
6. 检查服务器签名；
7. 调用主机名验证；
8. 派生会话密钥；
9. 建立加密通道。

应用开发者通常只看到：

```java
connection.getInputStream()
```

但背后已经发生：

```text
DNS
→ TCP
→ TLS 握手
→ 证书验证
→ 主机名验证
→ 加密 HTTP
```

如果任何默认安全检查失败，平台通常会抛出异常，应用不能继续读取正常响应。

---

# 14. HTTP 由谁完成？

TLS 安全通道建立后，`HttpsURLConnection` 会通过它发送 HTTP 消息。

App 负责指定：

```text
GET 或 POST
URL path
HTTP headers
request body
timeout
```

平台负责把这些组织成实际 HTTP 数据，并通过 TLS 发送。

返回后：

```text
服务器发送 HTTP status + headers + body
        ↓
TLS 解密和完整性验证
        ↓
HttpsURLConnection 暴露 response
        ↓
App 读取 status、headers、body
```

SafeCampus 的另一个问题在这里：

```text
ResponseParser.readText()
```

主要直接读取响应正文，并把它当作普通文本。

它没有明确、严格地处理：

- HTTP status code；
- Content-Type；
- 字符编码；
- 响应大小；
- JSON schema；
- 字段类型；
- 业务字段合法性。

所以 TLS finding 和 response validation finding 位于两个不同位置：

```text
TLS finding：
还没确认“响应是谁发来的”

Response validation finding：
即使收到响应，也没确认“内容是否符合预期”
```

---

# 15. 真实服务器端由谁完成？

真实情况下，HTTPS 连接的另一端应该有：

```text
Web server / API gateway
        ↓
后端应用
        ↓
认证与授权
        ↓
数据库或其他服务
```

服务器端负责：

- 持有服务器私钥；
- 提供 TLS 证书；
- 接收 HTTP 请求；
- 验证用户 token；
- 验证输入；
- 执行业务逻辑；
- 访问数据库；
- 返回结构化响应。

但是这个 assignment 没有给我们真实后端。

SafeCampus 使用：

```text
api.safecampus.invalid
```

因此实际执行经常是：

```text
App 尝试连接
    ↓
DNS 或连接失败
    ↓
代码抛出异常
    ↓
Repository 捕获异常
    ↓
使用本地 fallback
```

例如 Support Request：

```text
用户提交请求
    ↓
ApiClient 尝试访问无效域名
    ↓
网络失败
    ↓
SupportTicketRepository 捕获 Exception
    ↓
返回 “Mock success: classroom request recorded”
    ↓
界面显示成功式结果
```

这里并没有服务器参与。

---

# 16. 一次 SafeCampus Support Request 的完整实际流程

把所有层连起来：

```text
1. 用户在 Support Request 页面输入
   “Badge access test”

2. SupportRequestActivity 读取文本

3. SupportTicketRepository.submit(issue)
   调用 ApiClient

4. ApiClient 创建 URL：
   https://api.safecampus.invalid/support/request

5. HttpsURLConnection 对象被创建

6. ApiClient 设置：
   POST
   Content-Type
   request body
   timeout

7. ConnectionPolicy 修改连接：
   trust-all TrustManager
   always-true HostnameVerifier

8. 平台尝试解析 api.safecampus.invalid

9. 因为没有真实服务，网络请求失败

10. 异常返回 SupportTicketRepository

11. Repository 捕获异常，但没有显示失败

12. Repository 本地生成：
    Mock success: classroom request recorded

13. SupportRequestActivity 把该结果显示在页面
```

在这个流程中，有三个不同问题：

```text
问题一：TLS 验证被关闭
位置：步骤 7

问题二：用户输入没有正确 form encoding
位置：步骤 6

问题三：失败后显示成功式 fallback
位置：步骤 11–13
```

它们在同一功能中出现，但根因不同，不能混成一个 finding。

---

# 17. 一次真实、安全版本的流程应该怎样？

```text
1. Activity 收集用户输入

2. 输入通过标准表单编码器或 JSON serializer 编码

3. ApiClient 使用平台默认 HttpsURLConnection

4. 不安装 trust-all TrustManager

5. 不安装 always-true HostnameVerifier

6. Android DNS 解析域名

7. Linux 内核建立 TCP 连接

8. Android TLS：
   - 验证证书链
   - 验证目标域名
   - 建立加密通道

9. HTTP 请求发送到真实 API

10. API 验证身份和权限

11. API 返回明确的 HTTP status 和结构化数据

12. 客户端检查：
    - status code
    - Content-Type
    - response schema
    - 字段范围

13. 成功：
    显示服务器返回的工单编号

14. 失败：
    明确显示未提交成功或状态未知
```

---

# 18. 最重要的“由谁负责”总表

| 层次 | 具体组件 | 默认由谁提供 | App 开发者负责什么 |
|---|---|---|---|
| 功能页面 | Activity | Android 提供基类 | 开发者写页面逻辑 |
| 业务逻辑 | Repository/Service | 无强制实现 | 开发者自己设计 |
| API 调用 | `ApiClient` | SafeCampus 自己写 | URL、方法、请求体、响应处理 |
| HTTP/HTTPS 接口 | `HttpsURLConnection` | Android/Java | 正确使用，不破坏默认安全性 |
| TLS 配置 | `SSLContext` 等 | Android/Java | 通常使用安全默认值 |
| 证书链验证 | `TrustManager` | Android 有默认实现 | 不要替换成 trust-all |
| 域名验证 | `HostnameVerifier` | Android 有默认实现 | 不要无条件返回 true |
| 根 CA | 系统 trust store | Android 系统 | 必要时合理配置 |
| DNS | 系统 resolver | Android/操作系统 | 提供正确 hostname |
| TCP/IP | Linux 内核 | Android/Linux | App 通常无需实现 |
| 网络传输 | Wi-Fi/虚拟网络 | 系统与硬件 | 视为不可信路径 |
| 后端 API | 服务器程序 | 服务提供方 | 服务器团队实现 |
| 数据库 | 服务器基础设施 | 服务提供方 | 后端访问，不应由 App 直连 |
| 响应解析 | `ResponseParser` | SafeCampus 自己写 | 检查状态、格式和内容 |

---

# 19. 这个 assignment 特殊在哪里？

一般 Android 应用网络架构中：

```text
Android App
    ↓ HTTPS
真实服务器
    ↓
真实数据库
```

这个课程 APK 中：

```text
SafeCampus App
    ↓ HTTPS 尝试
api.safecampus.invalid
    ↓
没有真实服务器
    ↓
连接失败
    ↓
本地 mock/fallback
```

所以课程希望我们分析的是：

1. 如果这些客户端设计用于真实环境，会造成什么风险；
2. APK 当前代码是否错误建立了信任；
3. 哪些行为能在本地模拟器里动态证明；
4. 哪些影响只能作为条件性分析；
5. 不能把教学模型描述成真实系统已被攻击。

TLS 问题可以通过静态代码充分确认：

```text
验证逻辑确实被关闭
```

但没有真实服务，所以真实 MITM 影响是条件性的：

```text
如果相同代码连接真实 API，
并且攻击者控制网络路径，
则攻击者可能冒充服务器。
```

---

## 最终只需要记住这条实际链

```text
SafeCampus Activity
    ↓
Repository / Service
    ↓
ApiClient
    ↓
HttpsURLConnection
    ↓
ConnectionPolicy 修改 TLS 验证
    ↓
Android TLS 库
    ├── TrustManager
    └── HostnameVerifier
    ↓
Android DNS
    ↓
Linux TCP/IP
    ↓
模拟器虚拟网络
    ↓
不存在的 .invalid 服务器
    ↓ 失败
本地 fallback
```

其中：

- Android 已经提供了安全的 TLS 默认实现；
- SafeCampus 自己替换了证书验证和域名验证；
- 密码算法本身仍由 Android 执行；
- 错误不在 AES、ECDHE 或签名算法；
- 错误在于应用告诉平台“不要验证对方身份”；
- 课程 APK 没有真实服务器，因此没有发生真实网络攻击。