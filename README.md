# 器灵 AI Skills

帮助 AI 编程助手为现有软件接入器灵 AI 平台。每个 Skill 是可读取的工作说明与必要参考资料，按 Agent Skills 的 `SKILL.md` 结构组织。

[官网](https://ql.xiazq.com) · [在线开发文档](https://867fas1f7s.apifox.cn/) · [Python SDK](https://github.com/qilingAI/qiling-sdk-python)

## 当前 Skill

| Skill | 用途 |
|---|---|
| [qiling-sdk-integrate](skills/qiling-sdk-integrate/SKILL.md) | 识别软件技术栈，接入授权码登录、设备与套餐权益检查、退出和相关测试；需要时接入额度扣减 |

适用于 Python 桌面工具、Web 和其他语言软件。Python 使用官方 SDK，其他语言按在线 HTTP 文档接入。不要求把原工程重写，也不自动上传、上架、付款或部署。

## 使用

支持 Skills CLI 的编程助手，可以安装指定 Skill：

```sh
npx skills add qilingAI/qiling-skills --skill qiling-sdk-integrate
```

安装时选择需要的助手与项目范围；本命令不是上传、付款或部署授权。也可以手动安装：克隆仓库，将 `skills/qiling-sdk-integrate` 整个文件夹放入你的编程助手支持的 Skills 目录。支持显式调用的助手可以这样请求：

```text
使用 $qiling-sdk-integrate 为这个工程接入器灵 SDK。
先做测试授权登录和套餐权益校验，复用现有界面与配置。
```

没有 Skill 自动加载功能时，将 [SKILL.md](skills/qiling-sdk-integrate/SKILL.md) 提供给助手，并让它按其中引用读取接入说明。

## 准备配置

在器灵开发者后台保存软件和套餐后，到“我的软件 → 开发调试”生成测试 App Key 和测试授权码。测试 App Key 不是软件英文标识或软件 ID。授权码不要提交到仓库或发送到公共对话；把它放在本地安全输入/存储中。

实际接口和字段以 [在线文档](https://867fas1f7s.apifox.cn/) 为准，AI 可读取 [llms.txt](https://867fas1f7s.apifox.cn/llms.txt)。附带参考摘要用于快速开始和离线场景，不宣称替代平台的最新版本。

测试授权只做登录、设备、权益与测试额度调试，不能调用真实 AI 或付款。Skill 不包含 SDK 二进制、用户凭据、后台源码或生产配置。

## 维护

修改 Skill 时保持它与当前平台 API 一致，核对参考链接与安装来源。使用模拟服务验证未授权拒绝、设备 ID、会话刷新和退出，不为验证创建真实订单或消耗正式权益。

许可证为 MIT，见 [LICENSE](LICENSE)。
