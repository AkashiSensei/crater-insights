# Go LDAP 库介绍

本文档介绍 LDAP 协议以及 Go 语言中常用的 LDAP 客户端库，帮助理解 Crater 项目中 LDAP 认证的实现基础。

## 更新记录

> **2025-12-24** | [chore: format translation in pre-commit hook (#315)](https://github.com/raids-lab/crater/commit/518c88705ec7462f6c5d06d24a7aa70d7607f8b0) | `518c887`  
> 介绍 LDAP 协议基础以及 Go 语言中两个主要的 LDAP 客户端库：go-ldap/ldap（底层库）和 jtblin/go-ldap-client（封装库）。详细说明 DN、OU、DC 等 LDAP 核心概念，以及在 Crater 项目中的实际使用情况。

## LDAP 协议概述

LDAP（Lightweight Directory Access Protocol，轻量级目录访问协议）是一种用于访问和管理分布式目录信息服务的应用层协议。它基于 TCP/IP 协议栈，通常使用 389 端口（LDAP）或 636 端口（LDAPS）。

## LDAP 核心概念

### 目录信息树（DIT）

LDAP 数据以树状结构组织，类似于文件系统。每个节点（条目）都有唯一的标识符（DN），用于在目录树中定位。目录树从根节点开始，向下延伸到叶子节点（通常是用户或资源条目）。

### 属性类型（Attribute Type）与属性（Attribute）

属性类型和属性是两个相关但不同的概念：

| 概念 | 类比 | 说明 | 示例 |
|------|------|------|------|
| **属性类型（Attribute Type）** | 键（Key） | 属性的"名称"，定义这是什么信息 | `cn`、`mail`、`uid` |
| **属性值（Attribute Value）** | 值（Value） | 属性的"内容"，具体的数据 | `John Doe`、`john@example.com` |
| **属性（Attribute）** | 键值对（Key-Value Pair） | 属性类型和属性值的组合 | `cn=John Doe`、`mail=john@example.com` |

**属性类型（Attribute Type）**：
- 类似于数据库中的"列名"或编程中的"字段名"
- 定义了"这是什么信息"（比如是姓名、邮箱、电话等）
- 是固定的、预定义的标识符
- **大小写不敏感**：`CN` 和 `cn` 是等价的，`MAIL` 和 `mail` 也是等价的
- **约定俗成的写法**：
  - 在 DN 和 RDN 中，通常使用大写（如 `CN=John Doe`、`OU=Users`、`DC=example`）
  - 在搜索请求和属性访问中，通常使用小写（如 `cn`、`mail`、`uid`）
- 示例：`cn`（或 `CN`）、`mail`（或 `MAIL`）、`uid`（或 `UID`）、`sAMAccountName`、`displayName`

**属性值（Attribute Value）**：
- 类似于数据库中的"单元格内容"或编程中的"字段值"
- 定义了"具体内容是什么"（比如姓名是"John Doe"，邮箱是"john@example.com"）
- 是实际存储的数据
- 示例：`John Doe`、`john@example.com`、`john`

**属性（Attribute）**：
- 是属性类型和属性值的组合
- 格式：`属性类型=属性值`
- 一个条目可以有多个属性
- 示例：`cn=John Doe`、`mail=john@example.com`、`uid=john`

**条目中属性的特性**：

条目中的属性是**无序的**，属性的顺序不影响条目的含义。一个条目本质上是一个属性的集合（Set），而不是有序列表。

**示例**：

假设有一个用户条目，包含以下属性：

```
cn=John Doe                    ← 属性：cn（属性类型）= John Doe（属性值）
mail=john@example.com          ← 属性：mail（属性类型）= john@example.com（属性值）
uid=john                        ← 属性：uid（属性类型）= john（属性值）
displayName=John Doe            ← 属性：displayName（属性类型）= John Doe（属性值）
department=Engineering          ← 属性：department（属性类型）= Engineering（属性值）
```

在这个例子中：
- **属性类型**有：`cn`、`mail`、`uid`、`displayName`、`department`（5个不同的类型）
- **属性**有：5个键值对（每个属性类型对应一个属性值）
- **顺序无关**：这些属性的顺序可以任意调换，不影响条目的含义

**常见属性类型**：

| 属性类型 | 全称 | 用途 | 示例属性值 |
|---------|------|------|-----------|
| **CN** | Common Name | 通用名称，通常用于人名或对象名称 | `John Doe`、`it@lab` |
| **OU** | Organizational Unit | 组织单元，用于组织目录结构 | `Users`、`Lab`、`ACT` |
| **DC** | Domain Component | 域组件，用于表示域名 | `example`、`com`、`lab` |
| **UID** | User ID | 用户ID，唯一标识用户 | `john` |
| **sAMAccountName** | SAM Account Name | Windows Active Directory 中的用户账号名 | `john` |
| **mail** | Mail | 邮箱地址 | `john@example.com` |
| **displayName** | Display Name | 显示名称 | `John Doe` |
| **department** | Department | 部门 | `Engineering` |

**属性类型大小写规则**：

LDAP 属性类型不区分大小写，但有不同的使用约定：

| 使用场景 | 约定写法 | 示例 | 说明 |
|---------|---------|------|------|
| **DN/RDN** | 大写 | `CN=John Doe`、`OU=Users`、`DC=example` | 在 DN 中通常使用大写，使 DN 更易读 |
| **搜索过滤器** | 小写或大写 | `(cn=John)` 或 `(CN=John)` | 两者等价，通常使用小写 |
| **返回属性列表** | 小写 | `[]string{"cn", "mail", "uid"}` | 在代码中通常使用小写 |
| **属性访问** | 小写 | `entry.GetAttributeValue("cn")` | 在代码中通常使用小写 |

**代码示例**：

```go
// 在 DN 中使用大写（约定俗成）
searchRequest := ldap.NewSearchRequest(
    "OU=Users,DC=example,DC=com",  // Base DN 使用大写
    // ...
    "(cn=John Doe)",               // 搜索过滤器中可以使用小写
    []string{"dn", "cn", "mail"},  // 返回属性列表使用小写
    nil,
)

// 在代码中访问属性使用小写
entry.GetAttributeValue("cn")     // ✅ 正确
entry.GetAttributeValue("CN")     // ✅ 也正确，但通常使用小写
```

### 条目（Entry）

条目是 LDAP 目录树中的一个节点，类似于文件系统中的一个文件或目录。每个条目都有一个唯一的 DN（Distinguished Name），并包含一组属性。

**条目的组成**：
- **DN（Distinguished Name）**：条目的唯一标识符
- **属性集合**：条目包含的多个属性（键值对）
- **对象类（Object Classes）**：定义条目可以包含哪些属性（详见对象类部分）

**示例**：

假设有一个用户条目：

```
DN: CN=john,OU=Lab,DC=lab
对象类: person, organizationalPerson, user
属性:
  cn: john
  sn: Doe
  mail: john@example.com
  uid: john
```

在这个例子中：
- **DN**：`CN=john,OU=Lab,DC=lab` 是条目的唯一标识
- **对象类**：`person`、`organizationalPerson`、`user` 定义了条目可以包含的属性类型
- **属性**：`cn`、`sn`、`mail`、`uid` 是条目的实际数据

### 对象类（Object Classes）

对象类类似于编程中的**接口（Interface）**，定义了条目必须和可以包含哪些属性。一个条目可以"实现"多个对象类。

**对象类的关键特性**：
- **MUST 属性**：条目必须包含的属性（类似于接口的必需方法）
- **MAY 属性**：条目可以选择包含的属性（类似于接口的可选方法）
- **继承关系**：对象类可以有父对象类（SUP），继承父类的属性定义（类似于接口继承）
- **多对象类**：一个条目可以同时实现多个对象类（类似于实现多个接口）

**条目与对象类的关系**：

- **对象类定义结构**：对象类定义了条目可以包含哪些属性类型（类似于接口定义方法）
- **条目实现对象类**：条目必须实现对象类中定义的必需属性（MUST），可以选择实现可选属性（MAY）
- **一个条目可以有多个对象类**：条目可以"实现"多个对象类（类似于实现多个接口）
- **对象类继承**：对象类可以有继承关系（SUP），子对象类继承父对象类的属性定义（类似于接口继承）

**条目与对象的类比**：

| 概念 | 类比 | 说明 |
|------|------|------|
| **对象类（Object Class）** | 接口（Interface） | 定义条目必须和可以包含的属性，类似于编程中的接口定义 |
| **条目（Entry）** | 实现接口的对象（Object Implementing Interface） | 实际的数据对象，必须实现对象类中定义的必需属性 |
| **MUST 属性** | 接口的必需方法（Required Methods） | 条目必须包含的属性 |
| **MAY 属性** | 接口的可选方法（Optional Methods） | 条目可以选择包含的属性 |
| **属性类型（Attribute Type）** | 方法签名（Method Signature） | 定义属性的类型和约束 |
| **属性（Attribute）** | 方法实现/字段值（Method Implementation/Field Value） | 实际的属性数据 |

**常见对象类**：
- `person`：人员对象，包含 `cn`、`sn`（Surname）等属性
- `organizationalUnit`：组织单元对象，用于组织目录结构
- `computer`：计算机对象
- `user`：用户对象（Active Directory）
- `group`：组对象

### Schema（模式）与自定义属性类型

LDAP 不是简单的键值对存储系统，而是一个强类型的目录服务。所有属性类型都必须先定义 Schema，不能像普通键值对那样随意使用。

**LDAP 与普通键值对的对比**：

| 特性 | 普通键值对（如 JSON、Redis） | LDAP |
|------|---------------------------|------|
| **属性定义** | 可以随意添加任意键 | 必须先定义 Schema |
| **类型检查** | 无类型约束 | 有严格的类型和语法定义 |
| **注册要求** | 无需注册 | 必须在服务器中注册 |
| **唯一标识** | 不需要 | 需要 OID（Object Identifier） |

**对比示例**：

```json
// JSON（普通键值对）- 可以随意添加
{
  "name": "John",
  "email": "john@example.com",
  "myCustomField": "任意值"  ← 可以直接添加，无需定义
}
```

```
// LDAP - 不能随意添加
cn: John
mail: john@example.com
myCustomField: 任意值  ← ❌ 错误！必须先定义 Schema
```

**自定义属性类型的完整流程**：

1. **分配 OID（Object Identifier）**
   - OID 是一个全局唯一的数字标识符，格式如：`1.3.6.1.4.1.99999.1.1.1`
   - 用于唯一标识自定义属性类型
   - 可以从 IANA（Internet Assigned Numbers Authority）申请，或使用私有 OID 范围

2. **定义 Schema**
   - 使用 LDIF 格式定义属性类型
   - 指定属性名称、语法、匹配规则等
   - 示例：
     ```ldif
     attributetype ( 1.3.6.1.4.1.99999.1.1.1
       NAME 'customEmployeeId'
       DESC 'Custom employee identifier'
       EQUALITY caseIgnoreMatch
       SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
       SINGLE-VALUE )
     ```

3. **在对象类中声明**
   - 定义或修改对象类，声明可以使用这个自定义属性
   - 示例：
     ```ldif
     objectclass ( 1.3.6.1.4.1.99999.2.1.1
       NAME 'customPerson'
       SUP person
       MAY ( customEmployeeId ) )
     ```

4. **在 LDAP 服务器中注册**
   - 将 Schema 定义加载到 LDAP 服务器
   - 服务器验证并注册新的属性类型
   - 之后才能使用这个自定义属性

**OID（Object Identifier）**：

OID 是全局唯一的数字标识符，用于唯一标识属性类型。

**OID 的作用**：
- **全局唯一性**：确保不同组织定义的属性类型不会冲突
- **标准化**：LDAP 协议要求使用 OID 来标识属性类型
- **可扩展性**：支持不同组织定义自己的属性类型

**OID 格式解析**：
```
1.3.6.1.4.1.99999.1.1.1
│ │ │ │ │ │     │ │ │ │
│ │ │ │ │ │     │ │ │ └─ 属性编号
│ │ │ │ │ │     │ │ └─── 对象类编号
│ │ │ │ │ │     │ └───── 企业编号
│ │ │ │ │ └───── 私有企业编号（IANA 分配）
│ │ │ │ └─────── 私有企业（private enterprise）
│ │ │ └───────── Internet
│ │ └─────────── ISO
│ └───────────── 组织
└─────────────── ISO 标识
```

**标准属性类型 vs 自定义属性类型**：

| 特性 | 标准属性类型 | 自定义属性类型 |
|------|------------|--------------|
| **定义** | LDAP 规范中预定义 | 用户自己定义 |
| **OID** | 已分配（如 `cn` 的 OID 是 `2.5.4.3`） | 必须分配 |
| **Schema** | 已在标准 Schema 中定义 | 必须定义 |
| **注册** | 无需额外配置 | 必须在服务器中注册 |
| **使用** | 可直接使用 | 注册后才能使用 |

**实际使用场景对比**：

**场景1：使用标准属性类型（推荐）**
```
cn: John Doe                    ← 标准属性类型，已定义，可直接使用
mail: john@example.com          ← 标准属性类型，已定义，可直接使用
```
✅ **优点**：无需注册，兼容性好  
❌ **缺点**：受限于标准属性的定义

**场景2：使用自定义属性类型**
```
必须先完成：
1. 分配 OID：1.3.6.1.4.1.99999.1.1.1
2. 定义 Schema：attributetype ( 1.3.6.1.4.1.99999.1.1.1 NAME 'customField' ... )
3. 注册到服务器：加载 Schema 文件

然后才能使用：
customField: 任意值              ← 自定义属性类型，已注册，可以使用
```
✅ **优点**：完全符合业务需求  
❌ **缺点**：需要完整的 Schema 定义和服务器配置

### DN（Distinguished Name，可分辨名称）

DN 是 LDAP 中每个条目的唯一标识符，类似于文件系统中的绝对路径。

**关键特性**：
- **RDN 是有序的**：DN 由多个 RDN（Relative Distinguished Name）组成，RDN 必须按照从叶子节点到根节点的顺序排列
- **格式**：从叶子节点到根节点，用逗号分隔
- **示例**：`CN=John Doe,OU=Users,DC=example,DC=com`
- **在 Crater 项目中**：`CN=it@lab,OU=Lab,OU=ACT,DC=lab,DC=act,DC=buaa,DC=edu,DC=cn`

**DN 与文件系统路径的对比**：

DN 的结构类似于文件系统的绝对路径，但顺序相反：

| 文件系统路径 | LDAP DN | 说明 |
|------------|---------|------|
| `/root/home/user/file.txt` | `CN=file.txt,CN=user,OU=home,DC=root` | 文件系统从根到叶子，LDAP 从叶子到根 |
| 路径顺序：根 → ... → 叶子 | DN 顺序：叶子 → ... → 根 | 顺序相反 |

**示例对比**：

文件系统路径：
```
/usr/local/bin/python
```
- 顺序：根目录 `/` → `usr` → `local` → `bin` → `python`（文件）
- 从根到叶子，用 `/` 分隔

LDAP DN：
```
CN=python,OU=bin,OU=local,OU=usr,DC=example,DC=com
```
- 顺序：`CN=python`（条目）→ `OU=bin` → `OU=local` → `OU=usr` → `DC=example` → `DC=com`（根）
- 从叶子到根，用 `,` 分隔

**DN 顺序的重要性**：

1. **唯一性**：DN 的顺序决定了条目的唯一位置
   - `CN=John,OU=Users,DC=com` ≠ `CN=John,OU=Admins,DC=com`
   - 即使 RDN 相同，顺序不同或位置不同，DN 也不同

2. **层级关系**：RDN 的顺序反映了目录树的层级结构
   - 第一个 RDN（最左边）是条目本身
   - 后续 RDN 依次是父容器、祖父容器...直到根

3. **搜索基准**：搜索时必须指定正确的 Base DN（基准 DN）
   - Base DN 必须是 DN 中从某个 RDN 开始到根的部分
   - 示例：如果条目 DN 是 `CN=John,OU=Users,DC=example,DC=com`
     - 可以以 `OU=Users,DC=example,DC=com` 作为 Base DN 搜索
     - 不能以 `CN=John,OU=Users` 作为 Base DN（缺少根部分）

### RDN（Relative Distinguished Name，相对可分辨名称）

RDN 是 DN 中的一个组成部分，标识条目在其父容器中的唯一性。

**关键特性**：
- **格式**：`属性类型=属性值`
- **示例**：`CN=John Doe`、`OU=Users`、`DC=example`
- **位置固定**：RDN 在 DN 中的位置是固定的，不能随意调换顺序
- **需要显式指定**：创建条目时，必须明确指定使用哪个属性作为 RDN

**RDN 的本质**：

RDN 是一个特定的属性键值对（`属性类型=属性值`），这个键值对必须在父容器中唯一。RDN 的唯一性检索是以整个键值对进行的，只要保证这个键值对在父容器中唯一，就可以作为 RDN。

**RDN 的唯一性规则**：

1. **唯一性要求**：RDN 值（`属性类型=属性值`的组合）必须在父容器中唯一
2. **不同属性类型**：同一个父容器中的不同条目可以使用不同的属性类型作为 RDN
3. **属性值不受限制**：其他条目的属性值不受 RDN 限制，可以相同（只要 RDN 不同）

**示例**：

```
父容器: OU=Lab,DC=lab

条目1: CN=john,OU=Lab,DC=lab        ← RDN: CN=john（唯一）
条目2: UID=mary,OU=Lab,DC=lab       ← RDN: UID=mary（唯一，✅ 允许使用不同属性类型）
条目3: CN=bob,OU=Lab,DC=lab         ← RDN: CN=bob（唯一，✅ 允许）
```

**重要说明**：
- **创建时指定**：RDN 是在创建条目时通过 DN 显式指定的，不是自动选择的
- **作为 RDN 的属性必须存在**：如果使用 `CN=john` 作为 RDN，该条目必须有 `cn=john` 属性
- **其他条目不受限制**：同一个父容器中的其他条目可以没有作为 RDN 的属性类型
- **创建后固定**：一旦条目创建，RDN 就固定了，不能随意更改（修改 RDN 需要重命名条目）
- **常见做法**：通常使用 `CN`、`UID`、`sAMAccountName` 等唯一标识属性作为 RDN

## DN 解析示例

以 Crater 项目中的搜索基准 DN 为例：
```
OU=Lab,OU=ACT,DC=lab,DC=act,DC=buaa,DC=edu,DC=cn
```

**DN 结构解析**（从叶子到根，从左到右）：

| RDN | 位置 | 说明 | 层级 |
|-----|------|------|------|
| `OU=Lab` | 最左边（叶子） | 实验室组织单元 | 第1层（搜索基准） |
| `OU=ACT` | 第2个 | ACT 组织单元 | 第2层（Lab 的父容器） |
| `DC=lab` | 第3个 | 实验室域名组件 | 第3层 |
| `DC=act` | 第4个 | ACT 组织域名组件 | 第4层 |
| `DC=buaa` | 第5个 | 北京航空航天大学域名组件 | 第5层 |
| `DC=edu` | 第6个 | 教育机构域名组件 | 第6层 |
| `DC=cn` | 最右边（根） | 顶级域名组件（中国） | 第7层（根） |

**目录树结构示意**（从根到叶子）：
```
DC=cn（根）
 └── DC=edu
      └── DC=buaa
           └── DC=act
                └── DC=lab
                     └── OU=ACT
                          └── OU=Lab（搜索基准，这里可以包含用户条目）
```

**重要理解**：
- **DN 的顺序是固定的**：`OU=Lab,OU=ACT,DC=lab,...` 不能写成 `DC=cn,DC=edu,...,OU=Lab`
- **类似于文件系统**：就像 `/usr/local/bin` 不能写成 `/bin/local/usr` 一样
- **搜索基准**：`OU=Lab,OU=ACT,DC=lab,DC=act,DC=buaa,DC=edu,DC=cn` 表示从这个位置开始搜索
- **用户条目的完整 DN**：如果用户 `john` 在这个基准下，他的完整 DN 可能是：
  ```
  CN=john,OU=Lab,OU=ACT,DC=lab,DC=act,DC=buaa,DC=edu,DC=cn
  ```
  - `CN=john` 是最左边的 RDN（叶子节点，用户条目本身）
  - 后面的 RDN 依次是父容器路径

**搜索基准 DN（Base DN）**：
- 搜索操作的起始点，类似于文件系统中的工作目录
- 搜索范围可以是：
  - `ScopeBaseObject`：仅搜索基准 DN 本身
  - `ScopeSingleLevel`：搜索基准 DN 的直接子节点
  - `ScopeWholeSubtree`：搜索基准 DN 及其所有子节点（包括子节点的子节点）

**DN 设计的灵活性**：

DN 的结构不需要严格遵循某种固定模式，可以根据实际需求灵活设计。对于实验室内部使用的 LDAP，可以简化 DN 结构，例如省略 `edu`、`buaa`、`act`、`cn` 等部分，使用更简洁的结构如 `OU=Lab,DC=lab`。只要保证 DN 的唯一性和层级关系的一致性即可。

## LDAP 基本操作

1. **Bind（绑定）**：认证操作，验证用户身份
2. **Search（搜索）**：查询目录中的条目
3. **Add（添加）**：添加新条目
4. **Modify（修改）**：修改现有条目
5. **Delete（删除）**：删除条目
6. **Unbind（解绑）**：关闭连接

### Search（搜索）操作详解

LDAP 搜索使用**搜索过滤器（Search Filter）**来匹配条目，而不是基于完整的键值对进行精确匹配。

**搜索请求的组成**：

1. **Base DN（基准 DN）**：搜索的起始点，类似于文件系统中的工作目录
2. **Scope（搜索范围）**：
   - `ScopeBaseObject`：仅搜索基准 DN 本身
   - `ScopeSingleLevel`：搜索基准 DN 的直接子节点
   - `ScopeWholeSubtree`：搜索基准 DN 及其所有子节点（包括子节点的子节点）
3. **Filter（搜索过滤器）**：用于匹配条目的条件表达式
4. **Attributes（返回属性列表）**：指定要返回的属性类型（如 `[]string{"dn", "cn", "mail"}`）

**搜索过滤器格式**：

搜索过滤器使用 `(属性类型=匹配规则)` 的格式，支持多种匹配方式：

| 匹配类型 | 格式 | 说明 | 示例 |
|---------|------|------|------|
| **精确匹配** | `(属性类型=值)` | 匹配属性值完全等于指定值的条目 | `(cn=John Doe)` |
| **存在性检查** | `(属性类型=*)` | 检查是否存在该属性（不关心值） | `(cn=*)` |
| **前缀匹配** | `(属性类型=前缀*)` | 匹配属性值以指定前缀开头的条目 | `(cn=John*)` |
| **逻辑 AND** | `(&(条件1)(条件2))` | 同时满足多个条件 | `(&(cn=John)(mail=john@example.com))` |
| **逻辑 OR** | `(\|(条件1)(条件2))` | 满足任一条件 | `(\|(cn=John)(cn=Mary))` |
| **逻辑 NOT** | `(!(条件))` | 不满足条件 | `(!(cn=John))` |

**搜索示例**：

```go
// 搜索用户名为 "john" 的用户
searchRequest := ldap.NewSearchRequest(
    "OU=Lab,OU=ACT,DC=lab,DC=act,DC=buaa,DC=edu,DC=cn", // Base DN
    ldap.ScopeWholeSubtree,                              // 搜索整个子树
    ldap.NeverDerefAliases, 0, 0, false,
    "(sAMAccountName=john)",                             // 搜索过滤器：精确匹配
    []string{"dn", "cn", "mail"},                       // 返回的属性列表
    nil,
)
```

**搜索 vs RDN 唯一性检索的区别**：

| 特性 | RDN 唯一性检索 | LDAP 搜索 |
|------|---------------|----------|
| **用途** | 在父容器中唯一标识条目 | 在目录树中查找匹配的条目 |
| **匹配方式** | 必须精确匹配完整的键值对 | 可以使用多种匹配规则（精确、前缀、存在性等） |
| **范围** | 仅在父容器内 | 可以在整个子树中搜索 |
| **灵活性** | 固定，必须完全匹配 | 灵活，支持多种匹配模式 |

## Go LDAP 库

### go-ldap/ldap

**仓库地址**：[https://github.com/go-ldap/ldap](https://github.com/go-ldap/ldap)

**定位**：Go 语言的 LDAP v3 基础库，提供完整的协议实现

**特点**：
- **底层库**：提供完整的 LDAP v3 协议实现
- **功能全面**：支持所有 LDAP 基本操作和扩展操作
- **精细控制**：需要手动处理连接、搜索、解析等细节
- **标准实现**：遵循多个 RFC 规范（RFC 4511、RFC 3062、RFC 4514 等）

**主要功能**：
- 连接 LDAP 服务器（非TLS、TLS、STARTTLS、自定义拨号器）
- 多种绑定方式（Simple Bind、GSSAPI、SASL）
- 搜索操作（普通搜索、分页搜索、异步搜索）
- 修改操作（Add、Modify、Delete、Modify DN）
- 密码修改操作
- 内容同步操作
- LDAPv3 过滤器编译/反编译
- 服务器端排序
- LDAPv3 扩展操作和控制支持

**使用场景**：
- 需要精细控制 LDAP 操作的场景
- 需要实现自定义 LDAP 功能的场景
- 需要完整 LDAP 协议支持的场景

**安装方式**：
```bash
go get github.com/go-ldap/ldap/v3
```

**示例代码**：
```go
import "github.com/go-ldap/ldap/v3"

// 连接LDAP服务器
l, err := ldap.DialURL("ldap://ldap.example.com:389")
if err != nil {
    log.Fatal(err)
}
defer l.Close()

// 管理员绑定
err = l.Bind("cn=admin,dc=example,dc=com", "password")
if err != nil {
    log.Fatal(err)
}

// 搜索用户
searchRequest := ldap.NewSearchRequest(
    "dc=example,dc=com",
    ldap.ScopeWholeSubtree, ldap.NeverDerefAliases, 0, 0, false,
    "(uid=john)",
    []string{"dn", "cn", "mail"},
    nil,
)

result, err := l.Search(searchRequest)
if err != nil {
    log.Fatal(err)
}

// 处理搜索结果
for _, entry := range result.Entries {
    fmt.Printf("DN: %s\n", entry.DN)
    fmt.Printf("CN: %s\n", entry.GetAttributeValue("cn"))
    fmt.Printf("Mail: %s\n", entry.GetAttributeValue("mail"))
}
```

### jtblin/go-ldap-client

**仓库地址**：[https://github.com/jtblin/go-ldap-client](https://github.com/jtblin/go-ldap-client)

**定位**：简单的 LDAP 客户端封装库，专注于认证和用户信息获取

**特点**：
- **高级封装**：提供简化的 API，开箱即用
- **专注场景**：主要面向用户认证和信息获取场景
- **依赖底层库**：基于 `gopkg.in/ldap.v2` 或 `go-ldap/ldap` 实现
- **易于使用**：减少样板代码，快速实现 LDAP 认证

**主要功能**：
- **Authenticate**：用户认证（验证用户名密码）
- **GetGroupsOfUser**：获取用户所属的组
- **GetUserAttributes**：获取用户属性（姓名、邮箱、UID 等）
- 自动处理连接管理和错误处理

**使用场景**：
- 快速实现 LDAP 用户认证
- 需要获取用户基本信息和组信息
- 不需要精细控制 LDAP 操作的场景

**安装方式**：
```bash
go get github.com/jtblin/go-ldap-client
```

**示例代码**：
```go
import "github.com/jtblin/go-ldap-client"

client := &ldapclient.LDAPClient{
    Base:         "dc=example,dc=com",
    Host:         "ldap.example.com",
    Port:         389,
    UseSSL:       false,
    SkipTLS:      true,
    BindDN:       "cn=admin,dc=example,dc=com",
    BindPassword: "password",
    UserFilter:   "(uid=%s)",
    GroupFilter:  "(memberUid=%s)",
    Attributes:   []string{"givenName", "sn", "mail", "uid"},
}

defer client.Close()

// 用户认证
ok, user, err := client.Authenticate("john", "password")
if err != nil {
    log.Fatal(err)
}
if !ok {
    log.Fatal("Authentication failed")
}

// 获取用户信息
fmt.Printf("User: %+v\n", user)

// 获取用户组
groups, err := client.GetGroupsOfUser("john")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Groups: %+v\n", groups)
```

## 在 Crater 项目中的使用

### 当前实现

Crater 项目使用的是 **`go-ldap/ldap`**（底层库），而不是 `go-ldap-client`（封装库）。

**代码位置**：`backend/internal/handler/auth.go` 的 `actLDAPAuth` 函数

**使用方式**：
```go
import "github.com/go-ldap/ldap/v3"

// 连接LDAP服务器
l, err := ldap.DialURL(authConfig.RaidsLab.LDAP.Address)

// 管理员绑定
err = l.Bind(authConfig.RaidsLab.LDAP.UserName, authConfig.RaidsLab.LDAP.Password)

// 搜索用户DN
searchRequest := ldap.NewSearchRequest(
    authConfig.RaidsLab.LDAP.SearchDN,
    ldap.ScopeWholeSubtree, ldap.NeverDerefAliases, 0, 0, false,
    fmt.Sprintf("(sAMAccountName=%s)", username),
    []string{"dn"}, // 仅返回DN
    nil,
)

searchResult, err := l.Search(searchRequest)

// 用户密码验证
userDN := searchResult.Entries[0].DN
err = l.Bind(userDN, password)
```

### 选择 go-ldap/ldap 的原因

1. **精细控制**：需要实现两步绑定机制（管理员绑定 + 用户绑定）
2. **最小化查询**：只需要获取用户DN，不需要其他属性
3. **灵活性**：未来可能需要扩展功能（如直接从LDAP获取用户详细信息）

### 解耦方案中的考虑

如果需要在解耦方案中直接从 LDAP 获取用户详细信息，可以继续使用 `go-ldap/ldap`，只需：

1. **扩展搜索请求**：在搜索时添加更多返回属性
   ```go
   []string{"dn", "cn", "mail", "displayName", "department"} // 根据实际LDAP属性调整
   ```

2. **配置化属性映射**：通过配置文件定义 LDAP 属性到 `UserAttribute` 的映射关系

3. **保持灵活性**：`go-ldap/ldap` 提供了足够的灵活性来适应不同的 LDAP 服务器配置

## 参考资料

- **go-ldap/ldap 官方仓库**：[https://github.com/go-ldap/ldap](https://github.com/go-ldap/ldap)
- **go-ldap-client 官方仓库**：[https://github.com/jtblin/go-ldap-client](https://github.com/jtblin/go-ldap-client)
- **LDAP RFC 4511**：<https://datatracker.ietf.org/doc/html/rfc4511>
- **LDAP RFC 4514**：<https://datatracker.ietf.org/doc/html/rfc4514>