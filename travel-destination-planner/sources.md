# 信息源

只能使用下列来源进行候选召回、天气判断、交通判断和最终确认。用户提供的其他来源只能作为用户输入记录；除非白名单来源确认，否则不能作为验证证据。

## 攻略来源

### 小红书

使用 `xiaohongshutools` 召回国内真实旅行经验线索。

- 使用 `search_notes(keyword)` 进行关键词搜索。
- 需要详情时使用 `note_detail(note_id, xsec_token)`。
- 需要评论区校验时使用 `get_comments(note_id, xsec_token)`。
- 只读取搜索结果、笔记详情和评论区。
- 可用时保留：标题、作者、链接、摘要/正文片段、发布时间、笔记点赞数、评论内容、评论点赞数和其他热度信号。
- 按场景控制重点笔记数量：
  - 用户不知道去哪里玩、要求推荐候选目的地时，每个候选目的地默认打开 5 篇重点笔记。
  - 用户已有明确目的地、要求做攻略/美食/路线深挖时，默认打开 10 篇重点笔记。
  - 搜索结果不足目标数量时，使用实际可用数量并说明不足。
- 如果出现 HTTP 461，请用户打开 `https://www.xiaohongshu.com/explore` 完成人工验证。
- 禁止点赞、关注、发布评论、发布笔记或调用其他互动 API。

固定调用方式：

在 Windows / PowerShell 中调用小红书只读搜索时，使用下面这种 here-string 管道给 `python -`，不要把多行 Python 代码塞进 `python -c` 或嵌套 `cmd /c` 引号里。

```powershell
$code = @'
import asyncio
import json
import sys
from pathlib import Path

sys.path.insert(0, r'C:\Users\0\.cursor\skills\xiaohongshutools\scripts')
from request.web.xhs_session import create_xhs_session

async def main():
    cfg = json.loads(Path(r'C:\Users\0\.cursor\skills\travel-destination-planner\.private\xhs_cookie.json').read_text(encoding='utf-8'))
    xhs = await create_xhs_session(proxy=None, web_session=cfg.get('web_session'))
    try:
        res = await xhs.apis.note.search_notes('替换为搜索词')
        data = await res.json()
        items = (data.get('data') or {}).get('items') or []
        print(json.dumps({
            'http_status': getattr(res, 'status', None),
            'code': data.get('code'),
            'success': data.get('success'),
            'msg': data.get('msg'),
            'items_count': len(items),
            'items': items
        }, ensure_ascii=False))
    finally:
        await xhs.close_session()

asyncio.run(main())
'@
$code | python -
```

判断规则：

- `http_status: 200`、`code: 0`、`success: true` 表示搜索可用。
- PowerShell 报 `from 关键字不被支持`、`unterminated triple-quoted string` 等，是命令包装错误，不是小红书接口错误。
- 小红书接口错误只按接口返回判断，例如 HTTP 461、未登录/无权限、`code` 非 0。

Cookie 存储：

- 本地登录配置存放在 `.private/xhs_cookie.json`。
- 优先使用这个结构：

```json
{
  "web_session": "替换为用户提供的值"
}
```

- 不要提交或展示 cookie 值。

### 携程攻略

使用携程攻略补充目的地攻略结构和公开目的地信息。

## 天气来源

### `nmc.cn`

首选天气来源。可用时读取 7 天、逐小时、降水和云量信息。

### `weather.com.cn`

备选天气来源。当 `nmc.cn` 信息不足时，读取 7 天和 15 天天气。

### `tianqi.com`

备选天气来源。页面噪声较多，只在前两个来源不可用或不完整时使用。

## 交通来源

### 携程火车票

默认火车来源。优先保留 G/D 车次。可用时保留具体车次、出发站、到达站、出发时间、到达时间、历时和参考价格。输出时标注“携程火车票参考”。

### 12306

官方核验来源。只作为核验尝试，不作为默认来源，因为具体车次/余票接口可能不稳定。

### 携程机票

首选航班来源。必须尝试拉取具体航班明细。如果只拿到航线信息，标注为待确认。

### 去哪儿机票

航班备选来源。当携程机票没有明细时，用于航线和航班查询尝试。如果缺少明细，标注为待确认。
