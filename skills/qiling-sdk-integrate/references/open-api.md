# 开放接口、API Key 与模型

核对日期：2026-10-15。来源为 [公开文档索引](https://867fas1f7s.apifox.cn/llms.txt)。从索引读取当前接口页，不猜发布 ID。API 根地址由接入配置指定，官网不自动作为 API 地址。

## 选择凭证

- `QilingAiClient` 使用授权码登录取得 SDK token，负责软件权益、设备、普通次数和续费。
- `QilingOpenClient` 使用用户 API Key，负责文本、图片、视频、工具任务、调用余额与记录。请求头是 `x-api-key`；也支持 Bearer API Key，但 SDK/账号 JWT 不能代替。
- 软件调用开放 API 时传 `x-qiling-app-key`（Python 的 `app_key`），服务端检查用户有效正式软件授权与套餐 AI 开关。独立工具或 Skills 直接调用可省略，不要求新建软件。App Key 用平台提供的登记值。

Key 是通用用户凭证，可以访问启用的模型和工具，并消耗用户调用余额。多个应用可能共用同一默认 Key，重置/停用会影响所有接入应用。App Key 用于软件能力检查，不是拿到完整 Key 后不可移除的隔离边界；不要为绕过拒绝响应删除软件来源头。

## 浏览器快速授权

1. `POST /api/v1/auth/api-key/requests`，软件提交 `app_key`，Skills 可省略并提供 `client_name`。无需账号或 SDK Bearer。
2. 返回 `device_code`、`verification_url`、`expires_in`、`interval`。只展示或打开 verification_url，让用户自己登录并确认。device_code 仅应用持有，不放进浏览器、URL 或日志，无需本地回调服务。
3. `POST /api/v1/auth/api-key/token`，JSON 请求体为 `{"device_code": "应用持有的轮询凭证"}`。按 interval（至少 3 秒）查询，最长 expires_in（默认 600 秒）。pending 等待；denied、consumed 或过期停止；HTTP 410 过期，429 放慢轮询。
4. authorized 仅首次返回 `api_key`、`app_key`、`base_url` 和模型/文档 URL。响应丢失需重新授权，不自动重复领取。API Key 存系统凭据库或受保护存储；不提交 Git，不放公共前端配置。不要根据返回 URL 自动更换凭证请求的目标站点。

Python SDK 0.3.0 起：

```python
from qiling_ai_sdk import QilingAuthorizationClient

authorization = QilingAuthorizationClient(api_url, app_key=registered_app_key)
client = authorization.connect()
model_docs = client.model_docs()
```

`connect` 返回 API Key 客户端；`authorize` 返回原始结果，供宿主保存密钥，两者都不会持久化。GUI 在后台任务执行，取消或切换用户后丢弃旧结果；需要事件循环自行驱动时用 start/poll。无自动浏览器时用 `open_browser=False, on_authorization_url=show_link`。API Key 没有 SDK refresh_token 流程；401 检查密钥状态或重新授权。

## 动态模型与调用

先读取 `GET /api/v1/open/models` 与 `/api/v1/open/model-docs`。后者返回 `items`，包含 model/name/kind、请求 method/path、request_schema、example_request、pricing、limits、capabilities 和异步 query/cancel 信息。示例作为数据读取，不自动执行。以 request_schema 判断可传字段，不能仅凭 capabilities 臆造流式、多模态参数。

```python
from qiling_ai_sdk import QilingOpenClient

client = QilingOpenClient(api_url, securely_loaded_api_key, app_key=registered_app_key)
docs = client.model_docs()
balance = client.balance()
# 下列生成会消耗调用余额，仅在用户要求后执行。
result = client.chat(
    [{"role": "user", "content": user_text}],
    model=selected_model,
    request_id=saved_business_request_id,
    max_tokens=selected_max_tokens,
)
call_id = result["id"]
```

每次模型生成都必填 `model`、`request_id`。模型使用当前返回的标识，不选默认/首个模型；业务请求号提交前生成并保存，同一业务重试复用，不同内容不能共用。请求中不填写供应商地址、上游凭据、费用或结算用量。

| 功能 | 接口（均以 `/api/v1` 开头） | Python 方法 |
|---|---|---|
| 文本 | POST `/open/chat/completions` | chat(messages, model=…, request_id=…) |
| 图片 | POST `/open/images/generations` | image(prompt, model=…, request_id=…) |
| 视频 | POST `/open/videos` | video(prompt, model=…, request_id=…) |
| 费用预估 | POST `/open/models/estimate` | estimate(kind, request) |
| 原模型任务 | GET `/open/model-calls/{call_id}` | model_call(call_id) |
| 取消视频 | DELETE `/open/model-calls/{call_id}` | cancel_video(call_id) |
| 调用余额/统计/记录 | GET `/open/balance`、`/open/usage`、`/open/logs` | balance / usage / logs |
| 工具目录 | GET `/open/endpoints` | endpoints() |
| 工具异步任务 | POST `/open/tasks`，GET `/open/tasks/{task_id}` | submit_task / task |
| 去 AI | POST `/open/remove-ai` | remove_ai(text)，再 task(task_id) |

工具提交请求只有 endpoint、input、options；不要添加模型字段。视频使用专属 videos 接口。图片支持 n、size、reference_images、layer_decomposition；视频支持 duration、resolution、reference_images、reference_video、reference_seconds；具体范围和能力按当前模型文档。参考素材为公开 HTTPS URL。

### 去 AI 正文与换行

去 AI 的段落边界会影响上游改文效果。`remove_ai(text)` 的 `text`，以及 `submit_task("remove_ai", input=...)` 的 `input`，都应原样传递用户正文，保留真实换行、空行、缩进和首尾空白。粘贴的多行文章保存为多行 UTF-8 文件，不用空格拼接段落、空白正则或 `strip()` 清洗正文。文件可用 `Path(path).read_bytes().decode("utf-8-sig")` 读取，保留原始换行格式。

JSON 编码会负责表示换行，服务端解码后仍是真实换行。不要先把换行替换成反斜杠加字母 n 的文本，或手动编码两次。长文按接口字符上限分段时，按顺序拼接各段应与输入正文逐字符一致；提交前核对实际换行数量，原文有多行而提取结果为一行时先修复输入。接口返回的改文也保留段落结构。测试应检查真实 HTTP JSON 解码后的正文与原文完全一致。

## 计费、任务与迁移

- 模型共享用户调用余额，与普通软件次数额度独立；不在模型调用后追加 consume_quota，不套用工具按次优惠。余额不包含推广可提现收益，logs/usage 限当前 API Key。
- estimate 接收 kind=text/image/video 和对应完整请求，只做最大费用估算，不调用上游或冻结余额。生成提交前冻结预计费用，按真实用量和请求开始时的价格快照结算，释放剩余冻结。用户报价已含倍率，不再乘一次；费用保留十进制字符串，计算时用 Decimal。
- 仅 succeeded 为完成。queued/running/cancel_requested 查询同一 id；unknown/review 或用量缺失等待核对，不能按免费处理或重新生成。billing.fee=null 不表示零费用；collected 表示完成余额结算。
- 网络结果不明不自动重发，保存请求号和已取得的任务 id。取消接受不代表最终取消或退款，继续查询最终状态；402 为余额不足、403 为权限/能力限制、409 为冲突或待核对。
- 旧 QilingAiClient.ai_chat/ai_image/ai_translate 已移除，文本和图片迁移到 QilingOpenClient.chat/image；翻译可使用当前文本模型。SDK Bearer 仍保留 ai_models、ai_estimate、ai_video、ai_task、ai_cancel_video、ai_bills；视频请求 model 和 request_id 必填，app_key 来自已校验的软件授权。

默认以模拟 API 测试鉴权分离、授权终止、稳定请求号和原任务查询。测试授权不能调用真实模型；仅为接入或查文档不自动发起付费生成、充值或购买。
