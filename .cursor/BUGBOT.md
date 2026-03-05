# Bugbot Review 规则

请遵循以下指令对 Pull Request 进行评审。

任何评审意见必须使用**简体中文**编写。

## 🌲 README 目录树同步
如果 PR 新增、删除、移动了文档，或者文档内容有较大变动（不包括 AI 指令或提示词），则：
- 检查 `README.md` 中的目录树是否与实际文件系统同步。
- 如果存在不一致，添加一个 Bug，标题为 "README directory tree out of sync"，并提供具体的修改建议。
- 添加标签 `consistency`。

## 🛡 安全与合规

### 1. 凭据泄露
如果任何修改的文件包含硬编码的密码 (`password`)、API 密钥 (`API Key`)、私钥 (`Private Key`)、访问令牌 (`Access Token`) 或任何类似的凭据，则：
- 添加一个阻塞 Bug，标题为 "Sensitive credentials detected"。
- 内容中明确指出：检测到硬编码的敏感凭据。请将其移至环境变量或加密存储，并要求用户执行 Git 历史清洗以彻底删除敏感信息。
- 添加标签 `security`。

### 2. 个人隐私
如果任何修改的文件包含疑似真实的手机号码（如中国 11 位数字模式）或非测试用途的真实邮箱地址，则：
- 添加一个阻塞 Bug，标题为 "PII leakage detected"。
- 内容中明确指出：检测到真实的联系方式泄露。请进行脱敏处理并清洗 Git 历史。
- 添加标签 `security`。

### 3. 安全检查豁免
在执行上述安全检查时，如果检测到的内容符合以下情况，则：
- **内网 IP**：允许出现符合内网范围的 IP 地址（如 `10.x.x.x`, `172.16.x.x`-`172.31.x.x`, `192.168.x.x`），不计为 Bug。
- **测试占位符**：对于明显的测试占位符（如 `your_password_here`, `example@test.com`），忽略或仅给出非阻塞建议。

## 📝 文档更新记录检查
如果 PR 新增了文档，或者对已有文档进行了较大规模的修改（新增章节、大幅重写流程等），则：
- 检查该文档是否包含“更新记录”章节，且是否已添加针对本次 PR 改动的最新记录。
- 记录格式应遵循：`> **YYYY-MM-DD** | [commit title](commit_link) | short_hash` 紧跟一段深度总结。
- 如果缺失，添加一个阻塞 Bug，标题为 "Missing or incomplete update records"，提醒用户补齐更新记录以维持文档的可追溯性。
- 添加标签 `completeness`。
