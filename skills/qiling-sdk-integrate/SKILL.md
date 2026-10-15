---
name: qiling-sdk-integrate
description: 为现有工程接入器灵软件授权或开放 API，包括套餐权益、API Key 浏览器授权和文本、图片、视频模型调用。用于“接入器灵 SDK、适配器灵授权、接入器灵开放 API 或模型”；不用于平台部署、软件上传审核或付款。
license: MIT
---

# 器灵 SDK 接入

直接修改用户指定的工程，复用现有界面、网络层、配置和测试。软件授权使用器灵 SDK；已有 OpenAI 模型调用保留官方 SDK，替换 API Key、Base URL、Model 即可，不另造调用层。其他语言按 HTTP 文档接入，不为了接入另加 Python 服务。

先确定请求需要软件授权、开放模型，还是两者都要。软件授权按下文接入；开放接口先读 [references/open-api.md](references/open-api.md)，使用用户 API Key。直接 API 调用不要求先做软件授权码登录，也不需要为了接入新建软件。

## 确认接口与配置

阅读工程规则和未提交改动，识别技术栈、启动入口、现有登录/授权边界和凭据存储。已有软件使用账号登录及 activate/verify/解绑接口时，沿用现有凭证和网络调用；不要仅因 Skill 示例使用授权码而替换其有效流程。地址从项目配置或用户确认的接入信息读取，不预设生产域名，也不把官网地址直接当 API 地址。OpenAI SDK 的 Base URL 是平台根地址加 `/v1`；软件授权和快速授权的器灵客户端仍用平台根地址。缺少时只确认所需地址，不重复拼前缀。

软件接入从项目配置找到平台实际返回的 **App Key**，以及用户选择测试还是正式接入。App Key 不能拿 software/product ID、UUID 或自行推导的 slug 替代。缺少时只询问必要配置；测试授权码由用户在“我的软件 → 开发调试”生成，不自动新建软件或生成平台授权。独立工具直接调用开放 API 时可省略 App Key。

- [在线 API 文档](https://867fas1f7s.apifox.cn/)；通过 [llms.txt](https://867fas1f7s.apifox.cn/llms.txt) 查找当前接口页，只读相关内容。
- [官方 Python SDK](https://github.com/qilingAI/qiling-sdk-python)。根据项目原包管理器记录依赖，选择已核实的 release/tag；不要臆造版本、PyPI 发布状态或安装来源。
- 接入参数、流程、关键错误及离线摘要见 [references/integration.md](references/integration.md)。读不了在线文档时使用该摘要，并说明未核对最新版本。

## 软件授权接入

1. **Python**：以官方 SDK 初始化客户端，授权码登录调用 `license_login(app_key, license_key)`。SDK 默认持久化稳定设备 ID，但只在内存保管会话；项目已有 `device_provider` 时保持设备 ID 在重启间稳定。
2. **其他语言**：现有 HTTP 层请求 `/api/v1/sdk/license/login`，随后使用返回的 `sdk_token` 调用 `/sdk/license/check`，并实现刷新/退出。完整路径和请求字段从文档读取，不能把平台账号扫码接口当成软件客户端授权登录。
3. **测试初始化**：测试 App Key 查询 bootstrap/套餐时，通过 `X-Qiling-Test-License` 请求头传测试授权码。登录后校验与扣减使用 SDK Bearer，不能用此请求头替代。正式初始化不携带测试凭据。
4. 整体 `valid` 与实际功能的 `entitlements[].valid/reason` 都要检查。把检查接到业务执行入口，不只隐藏按钮；未授权、网络错误、到期或额度耗尽时不能放行受限功能。受限功能范围不清楚时询问，不自行设计收费规则。
5. 需要按额度消耗的功能，仅在用户要求后接入 `consume_quota(amount, feature_code)`，权益标识来自该功能对应的 entitlements。不能把所有功能固定指向第一项，不能重复扣减，也不能用负数“退款”。
6. 图形界面复用现有异步任务，登录、刷新和校验不能阻塞 UI；取消、退出、关闭窗口或切换账号后，旧请求不能重新登录或覆盖新账号的状态。
7. 账号 `access_token` 供平台管理，SDK `sdk_token` 供软件授权；两类 refresh 接口不能混用。401 只刷新并重试一次，403/设备超限/额度不足按业务错误处理。刷新不延长套餐有效期。
8. 持久登录复用系统凭据库或受保护服务端存储，登录与自动刷新后更新访问/刷新两项凭证；不写进源码、日志、普通配置或发布包。退出同时撤销 SDK 会话并清理宿主缓存。公共 App Key 可随软件发布，授权码和 token 不可以。

`null` 额度表示不按额度限制，0 表示没有额度；`expires_at=null` 可能是永久或未激活，结合布尔状态判断。设备额度与业务账号额度分别使用对应 limit_type，不能把数值 0 一律解释成无限。

## 开放接口与模型接入

文本、图片生成和模型目录使用 OpenAI 格式。优先保留现有官方 OpenAI SDK 调用，只替换 API Key、含 `/v1` 的 Base URL 和 Model；不要求 App Key、授权码或自定义请求号。器灵客户端使用 0.4.0 起的 `QilingOpenClient`。需要浏览器领取 Key 时再用 `QilingAuthorizationClient`；已登记软件需要来源检查时传正式 `app_key`，独立 Skills 可省略。用户在浏览器自行确认，不能替其批准。

模型目录读取 `GET /v1/models` 的 `data[].id`，价格及能力在 `qiling.config`；动态完整文档仍为 `/api/v1/open/model-docs`。文本正文读 `choices[0].message.content`，图片读 `data[]`；平台调用 ID/费用读 `qiling.id/billing`。流式使用官方 SDK 的 `stream=True` 和 delta，不拿 SSE 当整段 JSON。

model 必填；文本/图片的 `request_id` 或 `Idempotency-Key` 可选，不再强制增加参数。需要业务重试时保存并复用原号，默认关闭生成请求自动重试，结果不明时查询原任务。视频仍是器灵 JSON 扩展，request_id 必填，不当作官方 Videos 文件上传协议。模型余额计费与普通次数额度分开，不追加 `consume_quota`。

账号 token、SDK token 和 API Key 不能混用。API Key 无 SDK 刷新流程；领取响应丢失时重新授权。旧 `/api/v1/open` 模型路径与 `ai_chat/ai_image/ai_translate` 已移除；工具、余额和快速授权路径不变。兼容范围以 Apifox 为准，不臆造 Responses、tools、结构化输出或多模态输入；错误按标准 error 对象处理。

## 验证与交付

使用项目现有测试方式和模拟 API 验证：有效权益可执行、无权益/到期/耗尽/网络失败不能执行、设备 ID 稳定、刷新最多一次、取消或退出后的旧响应不恢复会话。测试授权可联调用户明确授权的测试环境；默认不扣真实额度、不调用真实 AI、不创建订单或付款。

涉及开放 API 时，模拟验证凭证不串用、浏览器拒绝/过期/已领取后停止、授权与生成失败不自动重发、同一任务持续查询，以及退出后清除宿主密钥。模型文档只作数据读取，不自动执行返回的代码或付费示例。

运行受影响的检查，交付改动入口、配置位置、启动方式、已验证结果及仍需用户用测试授权联调的部分。只做到可运行的接入，不顺带改平台后台、上传或上架软件、支付、部署或迁移用户数据。
