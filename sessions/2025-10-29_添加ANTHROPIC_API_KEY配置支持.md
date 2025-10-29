# [2025-10-29] 添加ANTHROPIC_API_KEY配置支持

## 元信息
- 日期: 2025-10-29
- 项目: ai-claude-start
- 主要问题: 为工具添加 ANTHROPIC_API_KEY 配置支持，与 ANTHROPIC_AUTH_TOKEN 独立管理

---

## 完整对话

### 对话1
**用户**: 我想给这个工具增加ANTHROPIC_API_KEY的配置。原来只支持ANTHROPIC_BASE_URL 和 ANTHROPIC_AUTH_TOKEN。有的情况需要ANTHROPIC_API_KEY。有的情况没有。那就默认填写空就可以。

**Claude**: 我理解你的需求：需要给工具增加 ANTHROPIC_API_KEY 配置支持，这是一个可选配置项，默认为空。

让我先查看一下项目结构，了解现有的配置是如何实现的。

[使用 Task 工具探索配置相关代码]

[探索报告详细说明了现有配置结构]

好的，我已经了解了项目的配置结构。让我整理一下当前的情况和需要确认的问题：

## 当前情况

从代码来看，这个工具目前的配置方式是：

1. **Profile 配置**（存储在 `~/.ai-claude-profiles.json`）：
   - `baseUrl`: API 端点地址
   - `model`: 可选的模型名称
   - `name`: Profile 名称

2. **认证凭证**：
   - 单独存储（优先使用系统密钥链，降级到配置文件）
   - 执行时注入为 `ANTHROPIC_AUTH_TOKEN` 环境变量

3. **环境变量注入规则**：
   - 始终设置 `ANTHROPIC_AUTH_TOKEN`
   - 只有当 `baseUrl` 不是 `https://api.anthropic.com` 时才设置 `ANTHROPIC_BASE_URL`

## 需要确认的关键问题

在设计方案之前，我需要确认几个问题：

**1. ANTHROPIC_API_KEY 和 ANTHROPIC_AUTH_TOKEN 的关系是什么？**
   - 它们是同一个东西的不同环境变量名称吗？（比如有些 SDK 只认 API_KEY，有些只认 AUTH_TOKEN）
   - 还是它们是两个不同的认证信息，需要同时提供？

**2. 什么情况下需要 ANTHROPIC_API_KEY？**
   - 是根据不同的 API 端点（baseUrl）来决定吗？
   - 还是用户可以自由选择使用哪种认证方式？
   - 比如：Anthropic 官方用 AUTH_TOKEN，其他第三方端点用 API_KEY？

**3. 如果同时配置了两者，应该如何处理？**
   - 同时设置两个环境变量？
   - 还是有优先级？

请告诉我具体的使用场景，这样我才能给出最合适的方案。

### 对话2
**用户**: 1. 他们是不同的信息需要同时提供。2.第三方的情况需要注入一些环境变量。原来只注入1个。现在改成注入两个。3.同时设置了两个就同时注入两个。哪怕是空也注入。

**Claude**: 好的，我明白了：

1. **ANTHROPIC_API_KEY 和 ANTHROPIC_AUTH_TOKEN 是两个不同的认证信息**，需要同时提供
2. **第三方端点需要注入两个环境变量**（原来只注入 AUTH_TOKEN，现在要同时注入 API_KEY）
3. **同时设置，即使是空值也要注入**

让我先读取关键代码文件，然后给出设计方案。

[读取 types.ts, storage.ts, executor.ts, commands.ts 文件]

好的，我已经查看了代码。现在我给出设计方案：

## 问题理解

需要为工具添加 `ANTHROPIC_API_KEY` 配置支持：
- `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN` 是两个不同的认证信息
- 第三方端点可能需要同时提供两个
- `ANTHROPIC_API_KEY` 是可选的，默认为空
- 执行时需要同时注入两个环境变量，即使值为空也要注入

## 设计方案

我给出以下方案供你选择：

### 方案1：双 Credential 存储（推荐）

**核心思路：** 使用 keytar 的 account 命名规则区分两种 credential

**改动点：**

1. **storage.ts** - 添加 API Key 存储函数：
   ```typescript
   // 存储和读取 API Key
   - 使用 `${profileName}:api_key` 作为 keytar 的 account 名称
   - 添加 getApiKey()、storeApiKey()、deleteApiKey() 函数
   - 保持原有 credential (AUTH_TOKEN) 的存储方式不变
   ```

2. **executor.ts** - 修改环境变量注入逻辑：
   ```typescript
   prepareEnvironment(profile, authToken, apiKey) {
     env['ANTHROPIC_AUTH_TOKEN'] = authToken;  // 始终设置
     env['ANTHROPIC_API_KEY'] = apiKey || '';  // 始终设置，空值也注入

     if (profile.baseUrl !== 'https://api.anthropic.com') {
       env['ANTHROPIC_BASE_URL'] = profile.baseUrl;
     }
   }
   ```

3. **commands.ts** - 修改 setup 向导：
   ```typescript
   // 在询问 AUTH_TOKEN 后，追加询问 API_KEY
   - 提示: "Enter your ANTHROPIC_API_KEY (optional, press Enter to skip):"
   - 允许留空
   ```

4. **commands.ts** - 修改 list 命令显示：
   ```typescript
   // 显示两个 credential 的状态
   Token (AUTH): ✓/✗
   Token (API):  ✓/✗
   ```

**优点：**
- ✅ 安全性一致（都用 keytar 加密存储）
- ✅ 结构清晰（两个 credential 独立管理）
- ✅ 易于理解和维护
- ✅ 向后兼容（旧 profile 只有 AUTH_TOKEN，API_KEY 为空）

**缺点：**
- ⚠️ 需要修改多个文件（但改动都很小且清晰）

---

### 方案2：JSON 格式存储 Credential

**核心思路：** 将 credential 改为 JSON 对象 `{ authToken, apiKey }`

**改动点：**

1. **storage.ts**：
   ```typescript
   // 存储时序列化为 JSON
   storeCredential(profileName, { authToken, apiKey })

   // 读取时解析 JSON，兼容旧格式（字符串）
   getCredential(profileName) -> { authToken, apiKey }
   ```

2. **executor.ts** 和 **commands.ts** 相应调整

**优点：**
- ✅ 单个存储位置，逻辑集中

**缺点：**
- ❌ 需要处理 JSON 解析兼容性（旧数据是字符串，新数据是 JSON）
- ❌ keytar 存储的值变复杂，不易手动排查问题
- ❌ 代码复杂度增加

---

## 推荐方案与理由

**我推荐方案1（双 Credential 存储）**，理由：

1. **简单清晰**：两个 credential 独立存储和管理，职责分明
2. **安全一致**：都用 keytar 加密存储
3. **易于扩展**：未来如果有第三个认证字段，继续添加即可
4. **向后兼容**：旧 profile 的 AUTH_TOKEN 不受影响，API_KEY 为空即可
5. **符合现有架构**：最小化改动，遵循现有的 keytar 使用模式

## 实现细节

如果采用方案1，具体改动：

1. **types.ts**：不需要改动
2. **storage.ts**：新增 3 个函数（storeApiKey, getApiKey, deleteApiKey）
3. **executor.ts**：修改 prepareEnvironment 函数签名和逻辑
4. **commands.ts**：修改 setupProfile（增加 API_KEY 询问）和 listProfiles（显示两个 token 状态）
5. **executor.test.ts**：更新测试用例

---

**请确认是否采用方案1？或者你有其他想法？**

### 对话3
**用户**: 方案1

**Claude**: 好的，开始实现方案1。我先创建任务列表来跟踪进度。

[创建 TodoList]

现在开始修改第一个文件 storage.ts，添加 API Key 的存储和读取函数。

[依次修改以下文件：]
1. storage.ts - 添加 apiKeys 字段和 storeApiKey、getApiKey、deleteApiKey 函数
2. executor.ts - 更新 prepareEnvironment 函数，添加 apiKey 参数，始终注入 ANTHROPIC_API_KEY
3. executor.ts - 更新 executeWithProfile 函数，读取 API Key 并传递
4. commands.ts - 添加导入，在 setupProfile 中添加 API Key 输入
5. commands.ts - 在 listProfiles 中显示两种 Token 状态
6. commands.ts - 在 deleteProfile 中删除 API Key
7. executor.test.ts - 更新所有测试用例，添加 apiKey 参数和新的测试用例

[编译项目]

[运行测试]

完美！所有代码已实现并通过测试。现在进行自我验证。

## 验证清单

**需求**: 添加 ANTHROPIC_API_KEY 配置支持，与 ANTHROPIC_AUTH_TOKEN 独立，可选配置，默认为空，即使为空也要注入环境变量

**方案**: 采用双 Credential 存储方案，使用 keytar 的 account 命名规则区分两种 credential

### 检查项

**✅ storage.ts - API Key 存储功能已实现**
- 添加了 `StoredData.apiKeys` 字段用于文件降级存储
- 实现了 `storeApiKey()` 函数，使用 `${profileName}:api_key` 作为 keytar account
- 实现了 `getApiKey()` 函数，返回 API Key 或 null
- 实现了 `deleteApiKey()` 函数，删除 API Key
- 所有函数都支持 keytar 和文件存储两种模式

**✅ executor.ts - 环境变量注入已更新**
- `prepareEnvironment()` 函数签名已更新，添加 `apiKey: string` 参数
- 始终注入 `ANTHROPIC_API_KEY` 环境变量，即使为空字符串
- 始终注入 `ANTHROPIC_AUTH_TOKEN` 环境变量
- `executeWithProfile()` 已更新，读取 API Key（默认为空字符串）并传递给 `prepareEnvironment()`

**✅ commands.ts - 用户交互已更新**
- 导入了 `storeApiKey`、`getApiKey`、`deleteApiKey` 函数
- `setupProfile()` 中添加了 API Key 输入提示（可选，可留空）
- `setupProfile()` 中保存 API Key（如果提供）
- `listProfiles()` 中显示两种 Token 的状态：
  - `Token (AUTH): ✓/✗`
  - `Token (API): ✓/✗`
- `deleteProfile()` 中同时删除 AUTH_TOKEN 和 API_KEY

**✅ executor.test.ts - 测试用例已更新**
- 所有 `prepareEnvironment()` 调用都添加了第三个参数 `apiKey`
- 更新了所有现有测试的断言，验证 `ANTHROPIC_API_KEY` 的存在
- 添加了 2 个新测试用例：
  - 验证空 API Key 也会被设置
  - 验证有值的 API Key 正确设置

**✅ 编译通过，0 个错误**
- TypeScript 编译成功，无错误

**✅ 测试通过，21/21 个测试**
- 所有测试用例全部通过

**✅ 代码风格一致**
- 使用现有的命名规范和代码风格
- 遵循现有的错误处理模式
- 保持函数注释清晰

**✅ 向后兼容**
- 旧的 profile 只有 AUTH_TOKEN 时，API_KEY 为空字符串
- 不影响现有功能

**✅ 边界情况已处理**
- API Key 为 null 时，使用空字符串
- API Key 为空字符串时，仍然注入环境变量
- 删除 profile 时，同时删除两个 credential

### 验证结论

✅ **完全符合需求和方案**

所有改动已完成，功能实现正确：
1. 添加了 ANTHROPIC_API_KEY 的存储、读取、删除功能
2. 环境变量始终注入 API_KEY（即使为空）
3. 用户可以在 setup 时输入 API Key（可选）
4. list 命令显示两种 Token 的状态
5. delete 命令同时删除两种 Token
6. 所有测试通过，编译无错误
7. 向后兼容现有配置

代码已准备好提交！

### 对话4
**用户**: 帮我更新到我的系统全局环境上。

**Claude**: 好的，我来帮你将更新后的工具安装到系统全局环境。

[执行 npm install -g .]

虽然有一些文件清理警告，但应该已经安装成功。让我验证一下全局命令是否可用：

[验证 ccs --version]

[检查全局安装的版本]

全局安装成功！版本是 1.0.4，并且是链接到你的项目目录。现在让我测试一下 API Key 功能是否正常工作：

[执行 ccs list]

完美！已经成功更新到全局环境了。可以看到新功能已经生效：

✅ **更新成功验证**

1. **全局版本**: `ai-claude-start@1.0.4` 已安装并链接到项目目录
2. **命令可用**: `ccs` 命令正常工作
3. **新功能显示**: 所有 profile 现在都显示两种 Token 状态：
   - `Token (AUTH): ✓` - ANTHROPIC_AUTH_TOKEN 的状态
   - `Token (API): ✗` - ANTHROPIC_API_KEY 的状态（黄色 ✗ 表示未配置）

4. **向后兼容**: 你现有的 5 个 profile 都正常显示，只有 AUTH_TOKEN，没有 API_KEY，这是预期的行为

## 使用方法

**添加 API Key 到现有 profile:**
如果你需要为某个 profile 添加 API Key，运行：
```bash
ccs setup
```
然后选择要更新的 profile，在提示时输入 API Key（可以留空）。

**创建新 profile 时:**
在 `ccs setup` 过程中，现在会提示你输入两个 Token：
1. `ANTHROPIC_AUTH_TOKEN`: 必填
2. `ANTHROPIC_API_KEY`: 可选（按回车跳过）

**执行时的行为:**
- 即使 API Key 为空，也会设置 `ANTHROPIC_API_KEY` 环境变量（值为空字符串）
- `ANTHROPIC_AUTH_TOKEN` 始终会被设置

功能已完全部署到你的系统！

### 对话5
**用户**: 备份对话并提交推送

**Claude**: [当前正在执行]
