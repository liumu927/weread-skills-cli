---
name: weread-skills-cli
description: 微信读书 CLI 助手，内置 weread-skills API 文档，可搜索书籍、查看书架/笔记/阅读数据，并将微信读书划线、想法/点评导出为适配 Obsidian 的 Markdown。
version: 1.1.0
---

# 微信读书 CLI 助手

提供两类能力：

1. 查询微信读书数据：搜索、书籍信息、书架、阅读统计、笔记划线、书评、推荐。
2. 导出单本书笔记：把划线和想法/点评渲染为 Obsidian Markdown。

本 skill 已内置官方 `weread-skills` Markdown 文档，不再要求用户额外安装官方 skill。

## 内置 API 文档

调用接口或解释字段前，先按用户意图读取对应文档：

| 能力 | 文档 |
|------|------|
| 总览、鉴权、通用规则、深度链接 | `references/weread-api/official-skill.md` |
| 搜索书籍 | `references/weread-api/search.md` |
| 书籍信息、章节、阅读进度 | `references/weread-api/book.md` |
| 书架管理 | `references/weread-api/shelf.md` |
| 阅读统计 | `references/weread-api/readdata.md` |
| 笔记、划线、想法/点评 | `references/weread-api/notes.md` |
| 书籍点评 | `references/weread-api/review.md` |
| 推荐发现 | `references/weread-api/discover.md` |
| 用户资料 | `references/weread-api/profile.md` |

## 鉴权

微信读书 API 使用 Agent Gateway：

```text
POST https://i.weread.qq.com/api/agent/gateway
Authorization: Bearer $WEREAD_API_KEY
Content-Type: application/json
```

`WEREAD_API_KEY` 从环境变量读取，格式为 `wrk-xxxxxxxx`。如果未设置，提示用户先配置该环境变量。

请求体必须平铺参数，并带 `skill_version`：

```json
{"api_name":"/store/search","keyword":"三体","count":10,"skill_version":"1.0.3"}
```

## 导出命令

### `/export [书名]`

导出指定书籍的笔记为 Markdown 文件。

工作流：

1. 检查 `WEREAD_API_KEY`。
2. 若用户提供书名，先通过书架或搜索解析 `bookId`；若未提供，展示书架列表供用户选择。
3. 获取导出所需数据：
   - `/book/info`：书名、作者、分类等基本信息。
   - `/book/chapterinfo`：章节目录，用于分组。
   - `/book/bookmarklist`：划线内容。
   - `/review/list/mine`：个人想法/点评。
   - `/book/getprogress`：阅读进度、阅读时长、开始/完成时间。
4. 运行 `scripts/weread_export.py` 渲染 `assets/template.md`。
5. 输出文件名使用 `阅读结束日期《书名》.md`；未读完时使用导出日期。

## 导出模板

`assets/template.md` 必须以用户提供的 Obsidian 模板为准。模板中的示例划线颜色只作为版式约束，脚本会按章节替换为真实划线和想法内容。

## 数据处理规则

- 时间戳统一展示为 `YYYY-MM-DD`。
- 阅读时长由秒转换为 `X小时Y分钟`。
- 划线使用 `<span style="color:#HEX">文本</span>`，不额外添加下划线。
- `colorStyle` 映射：

| 值 | 颜色 | Hex |
|----|------|-----|
| 1 | 红色 | `#FF909C` |
| 2 | 紫色 | `#B89FFF` |
| 3 | 蓝色 | `#74B4FF` |
| 4 | 绿色 | `#70D382` |
| 5 | 黄色 | `#FFCB7E` |

- 当前 `/book/bookmarklist` 只导出划线内容，不导出书签正文；书签区域无可用内容时输出 `无书签`。
- 深度链接只在「深度链接」章节输出总阅读入口，按官方 URL Schema 构造为 `weread://reading?bId={bookId}`；不要默认给每条划线或想法添加定位链接，除非用户明确要求。
- 若接口返回 `upgrade_info`，立即暂停当前操作并按提示升级，不能忽略。
