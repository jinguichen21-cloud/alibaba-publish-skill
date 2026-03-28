# 国际站发品 · Skill 部署包

## 快速开始

1. 在支持 SKILL.md 的平台（OpenClaw / 钉钉 AI 助手 / Coze / FastGPT / Dify）中选择"导入 Skill"
2. 上传本压缩包，平台会自动读取 `skill.json` 和 `SKILL.md`
3. 确认开启**联网搜索**和**网页访问**工具权限
4. 发布即可使用

## 文件说明

| 文件 | 说明 |
|------|------|
| `SKILL.md` | Skill 主体，包含完整对话逻辑和报告模板 |
| `skill.json` | Skill 配置清单，平台识别入口 |
| `README.md` | 本说明文件 |

## 必须开启的工具权限

- ✅ web_search（联网搜索，用于市场数据采集）
- ✅ web_fetch（网页读取，用于解析客户店铺链接）

## 手动部署（System Prompt 方式）

如平台不支持 Skill 包导入，将 `SKILL.md` 中从
`# 国际站发品` 开始的全部内容
复制粘贴至平台的 System Prompt / 角色设定框即可。
