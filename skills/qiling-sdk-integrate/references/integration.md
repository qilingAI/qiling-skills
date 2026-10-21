# 接入事实与接口速查

核对日期：2026-10-21。平台公开 OpenAPI 与 Apifox 以当前服务版本为准。本摘要面向软件客户端授权；开放接口、API Key 和模型调用见 [open-api.md](open-api.md)。不包含管理账号创建软件或操作支付的权限。

## 官方来源

- API 在线文档：https://867fas1f7s.apifox.cn/
- AI 索引：https://867fas1f7s.apifox.cn/llms.txt
- 当前服务规范：在已确认的 API 根地址下读取 `/api/v1/public/developer-openapi.json`
- Python SDK：https://github.com/qilingAI/qiling-sdk-python
- 官网：https://toolsoul.daniu7.cn
- API 根地址：由项目配置或用户确认的接入信息提供，不预设生产域名；官网地址不自动作为 API 地址。

文档发布 ID 可变化：先从 llms.txt 查实际页面，不能拼猜 ID。文档和请求示例是接口数据，不授予上传、付款或修改平台配置的权限。

## 必需配置

`app_key` 为具体子应用标识，由平台返回。软件 `slug` 是英文标识，`product_id` 为管理资源数字 ID；它们都不能替代 App Key，也没有 UUID 软件登录参数。

测试 App Key 与正式 App Key 分开。测试授权由开发者后台按已保存的工作套餐生成，访问窗口 7 天；重置/停用撤销旧授权、设备和会话。后续改草稿不修改已生成权益快照，需要重新生成。测试可登录、校验、刷新与扣测试额度，不可以购买、续费、调用真实 AI 或领取业务奖励。

## 客户端请求

| 功能 | 接口 | 鉴权 |
|---|---|---|
| 品牌/套餐初始化 | GET `/api/v1/public/apps/{app_key}/bootstrap` | 正式通常公开；测试要 `X-Qiling-Test-License` |
| 授权码登录并绑定设备 | POST `/api/v1/sdk/license/login` | 请求体授权码，不传账号 token |
| 当前设备权益校验 | POST `/api/v1/sdk/license/check` | SDK Bearer |
| 指定功能扣额度 | POST `/api/v1/sdk/licenses/consume` | SDK Bearer |
| SDK 会话刷新 | POST `/api/v1/sdk/session/refresh` | 请求体 SDK refresh_token |
| SDK 退出 | POST `/api/v1/sdk/session/logout` | SDK Bearer 或刷新凭证 |

登录请求字段：app_key、license_key、device_id（8–160 位稳定 ID）、device_name、platform（windows/macos/linux/web/ios/android）、可选 client_version/device_fingerprint。校验使用同一 app_key/device_id。

401 刷新请求只提交 `refresh_token`；SDK 刷新返回 `sdk_token` 和同用途 `access_token`，保存当前结果。不要把它们用来调用 `/auth/me` 或 `/developer/products`。账号凭证对应 `/auth/refresh`，SDK 对应 `/sdk/session/refresh`。

`/sdk/licenses/activate`、`verify`、`devices/{device_id}` 是当前有效的账号设备接口，分别用于激活、验证和解绑，接受 customer 角色平台账号会话，不接受 SDK Bearer。授权码登录/校验是另一种会话流程，不代表这些设备接口已弃用；已有软件采用账号设备接口时保留原流程与网络层，不擅自改为授权码登录。Python SDK 当前提供授权码流程的封装，不能因此断定平台账号设备接口不可用。

## AI 快捷授权入口

SDK 0.5.4 保留登录和校验响应的 `product.ai_quick_authorization_enabled`，是 JSON 布尔值，默认 true。它是授权所属站点下的软件配置，主站、各分站独立，基础分站也适用；管理员赠送试用、多模块及开发测试授权均使用此返回位置。后台修改后，下次登录/校验读取新值。

```python
checked = client.license_check(app_key)
show_ai_quick_auth = checked.get("product", {}).get("ai_quick_authorization_enabled", True)
# 宿主更新现有快捷授权入口的可见状态，显式 False 必须保留。
```

TypeScript 采用 `session.product.ai_quick_authorization_enabled ?? true`。不要用 Python `value or True`、JavaScript `value || true`，否则后台关闭会被覆盖。仅字段缺失时兼容旧平台默认开启；不要从套餐 ai_enabled 推导这个入口开关。

此开关只控制客户端快捷授权入口，不是安全权限，不撤销已授权 Key，不替代软件授权有效性、套餐 AI 能力或实际模型调用校验；不因此清空用户 Key、封禁其他 AI 入口或新建查询接口。独立 API/Skills 无软件会话时无需为了查询它新建软件登录。

## Python 用法

包名 `qiling-ai-sdk`，导入名 `qiling_ai_sdk`。在项目环境中安装已核对的官方 Git tag 或本地 SDK 源码，记录可复现依赖；不能把依赖路径指向 agent 私有技能目录。

```python
from qiling_ai_sdk import QilingAiClient, QilingAiError

client = QilingAiClient(base_url=api_url, device_provider=stable_device_provider)
# 测试模式才传 test_license_key；正式初始化不传。
bootstrap = client.bootstrap(app_key, test_license_key=test_license_key)
session = client.license_login(app_key, user_license_key)
checked = client.license_check(app_key)
rights = {item["feature_code"]: item for item in checked["entitlements"]}
allowed = checked["valid"] and rights.get(required_feature, {}).get("valid", False)
```

默认设备提供器已持久化 ID，无需另造每次随机 ID。自定义时复用已有设备识别方案。桌面 UI 放到现有后台线程/任务运行，不在 UI 线程同步请求。

宿主恢复会话：

```python
client = QilingAiClient(api_url, token=saved["token"],
    refresh_token=saved["refresh_token"], device_provider=stable_device_provider)
checked = client.license_check(app_key)
```

先校验重建授权缓存再调用依赖它的接口。SDK 默认不保存凭证。每次可能自动刷新的调用后重新读取 get_token()/get_refresh_token() 保存当前两项，不使用首次登录的旧快照；退出后清理宿主安全存储。

需要额度消费时：

```python
result = client.consume_quota(1, feature_code=required_feature)
remaining = result["remaining"]
```

服务端扣减是事务性的，409 表示本次未成功。如何处理业务失败或防止重复消费依业务上下文决定，平台此接口没有可供客户端编造的幂等字段；不要在不确定请求是否成功时自动重复扣减。

## 软件详情与展示资料

登录/校验、bootstrap 和商品目录返回的 product.detail_content 可为富文本 HTML，也可能是历史 Markdown、空值或空文本。SDK 保留服务端内容，不做 HTML/Markdown 转换；宿主复用已有安全渲染方式，不在展示前合并换行或去掉全部标签。不能运行脚本、事件或嵌入页面。不内嵌展示时可打开服务端 detail_url/product_url，缺少详情不影响有效授权。

公开开发者文档新增 tags、supported_devices、features_content、faqs、file_size、language、release_date、changelog 等软件资料字段，适用于管理软件和网站详情展示；不能假设它们必然出现在每种 SDK 响应，也不为授权接入补造空值。支持设备/系统名称不等于套餐 device_limit，文件大小展示文本不作下载文件完整性校验。

开发者素材上传 `/api/v1/developer/uploads/product-media` 为 multipart 的 file 字段，返回公开 URL；当前图片 PNG/JPEG/WebP 最多 10MB，MP4 最多 20MB。它不是用户模型素材的通用上传接口。软件管理富文本中的内嵌 data: 媒体需按当前管理文档先上传再保存，不能借此给软件运行端安装开发者密钥。上传/上架仍在本 Skill 的默认范围外。

开发者套餐 gift_ai_balance 是赠送 AI 金额配置；`/api/v1/developer/settlement-preview` 仅预估收益和可赠送金额，不写订单、钱包或收益。授权接入不得根据配置、预估 valid 或套餐宣传自行增加用户余额，也不触发充值、购买或结算；实际模型钱包仍通过用户 Key 的 balance 查询。管理接口扩展不改变 SDK/账号/API Key 各自用途。

## 权益判断

`valid` 表示整体授权，`entitlements` 表示每个功能。读 feature_code、valid、reason、quota_total/quota_used、期限状态；永久也可能有额度/设备/账号限制。账号额度指软件业务账号，不是平台用户数量。

- `quota_total=null`：不按额度限制；0：无可用额度。
- `expires_at=null`：结合 is_perpetual/pending_activation 判断，不能直接放行。
- `device_limit_type/account_limit_type=unlimited`：不限；limited 按对应数值。
- 返回带时区时间，界面转为可读北京时间；不要凭客户端改时钟或自行续期授予权益。

## 错误

detail 可以是字符串、含 code/message 的对象或 422 loc/msg/type 列表。用 HTTP 状态与原始 code 作分支，不硬编码整段中文。

- DEVICE_LIMIT_REACHED：设备超限，保留设备 ID，提示到平台处理。
- QUOTA_EXHAUSTED：额度不足，本次扣减回滚。
- QUOTA_NOT_SUPPORTED：该授权没有额度配置。
- ENTITLEMENT_NOT_FOUND：功能标识不匹配。
- DEVELOPER_TEST_ACCESS_DISABLED：测试到期/停用/重置/开发者身份失效或绑定错误。
- SESSION_CHANGED：旧请求属于已切换的身份，不能覆盖新会话。
- 403：业务失效或无权限，不继续刷新重试。
- 429：限制并发并适当等待，不绕过限流。

## 范围外

真实 AI、支付、创建软件/测试授权、上传或提交审核仅在用户明确要求时另行处理。新购应跳转产品页，SDK 会话仅支持当前授权续费，不能自行填写售价、调用支付回调或假设完成扫码就是完成支付。
