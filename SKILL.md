---
name: lark-safe-write
version: 1.0.0
description: "Lark/Feishu wiki 安全写入流程：强制先备份、再写入、后验证，防止覆盖丢失"
metadata:
  requires:
    bins: ["lark-cli"]
  relatedSkills:
    - "../lark-wiki/SKILL.md"
    - "../lark-shared/SKILL.md"
---

# Lark Safe Write

> **触发时机**：任何需要向 Lark wiki 创建、更新或覆盖文档内容的任务。
> **核心原则**：先读后写，凡覆盖必备份，写入必验证。

## 预检（必须执行，不可跳过）

1. **身份确认**：所有命令必须携带 `--bot` 标志，以 bot 身份执行
2. **连通性测试**：执行 `lark-cli auth test` 确认 bot token 有效
3. **目标解析**：
   - 若用户给的是 wiki URL，提取 wiki_token（`.../wiki/<token>` 中的 `<token>`）
   - 若用户给的是 node_token，确认该节点类型为 `wiki` 而非普通 `doc`
4. **权限确认**：若首次操作该 wiki 空间，先执行 `lark-cli wiki spaces get --params '{"token":"<wiki_token>"}'` 确认 bot 有访问权限

## 备份（更新/覆盖场景必须执行）

> 仅当文档**已存在**时执行。创建新文档可跳过此步。

1. **读取现有内容**：
   ```bash
   lark-cli wiki get --wiki-token <wiki_token> --bot
   ```
   或等效命令读取当前文档完整内容。
2. **保存备份**：将现有内容写入本地文件：
   ```bash
   mkdir -p .backups
   # 内容写入 .backups/YYYYMMDD-HHMMSS-{wiki_token}.md
   ```
3. **记录备份路径**：向用户报告备份文件位置，作为可恢复点。

## 写入

### 创建新 wiki 文档

1. 确定父节点：`parent_node_token`（非 `parent_wiki_token`）
2. 执行创建：优先使用 Shortcut `lark-cli wiki +node-create`，或调用 `lark-cli wiki nodes create`
3. 记录返回的 `node_token` 和文档 URL

### 更新/覆盖现有 wiki 文档

1. **准备内容**：确认内容已保存为 markdown 文件
2. **使用 stdin 管道写入**（禁止 `--markdown /path/to/file` 方式，该方式会把文件路径当作文本内容写入）：
   ```bash
   cat <content-file>.md | lark-cli docs +update --markdown - --document-token <document_token> --bot
   ```
   > 若 wiki 文档的内容更新需通过 `wiki` 子命令完成，则使用对应 API，但同样遵循 stdin 管道原则。
3. **写入后等待 2 秒**：飞书文档有异步索引，立即读取可能拿到旧内容。

## 验证（必须执行，不可跳过）

1. **重新读取文档**：
   ```bash
   lark-cli wiki get --wiki-token <wiki_token> --bot
   ```
2. **关键内容比对**：检查标题、首段、关键标记（如用户指定的特定文本）是否与新内容一致
3. **输出验证结果**：
   - ✅ 验证通过：返回文档 URL 和 `doc_id`
   - ❌ 验证失败：立即停止，向用户报告失败详情，提供备份路径，**禁止**在未查明原因前再次覆盖

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| 预检时 `auth test` 失败 | 暂停执行，提示用户检查 `LARK_APP_ID` / `LARK_APP_SECRET` 和 bot 权限 |
| 备份时读取失败（文档不存在） | 按"创建新文档"流程继续，跳过备份 |
| 备份时读取失败（权限不足） | 停止执行，提示用户在 Feishu 控制台为 bot 添加 `wiki:node:read` 和 `doc:wiki` 权限 |
| 写入失败（4xx/5xx） | 停止执行，保留备份，不执行验证，向用户报告错误详情 |
| 验证失败（内容不匹配） | 停止执行，保留备份，分析原因（异步延迟？token 错误？），不自动重试覆盖 |

## 输出格式

执行完毕后返回：

```
✅ Lark Safe Write 完成
- 操作类型：创建 / 更新 / 覆盖
- Wiki Token: <token>
- 文档 URL: <url>
- Doc ID: <id>
- 备份路径：.backups/YYYYMMDD-HHMMSS-<token>.md（如适用）
- 验证状态：通过 / 失败（详情）
```

## 禁止行为

- ❌ 跳过备份直接覆盖现有文档
- ❌ 跳过验证直接结束任务
- ❌ 验证失败后自动重试覆盖
- ❌ 使用 `--markdown /path/to/file` 方式传递文件路径
- ❌ 使用 `doc` 子命令操作 wiki 页面（wiki 节点操作必须用 `wiki` 子命令）
