# better-life-suit

生活类 Skill 与工具的**默认仓库**。by Evan Song · [github.com/Songhonglei](https://github.com/Songhonglei)

> A suite of skills and tools which make our life better.

## 结构约定（multi-repo）

一个工具 / 一个 Skill 独占一个子目录，彼此独立运行：各自带 `README.md`、`package.json`、依赖与部署配置，不共享运行时依赖，也不在根目录放业务代码。根目录只放本说明与通用忽略规则。
新增一个工具：在根建子目录 → 放齐可运行代码、自己的 README 与验证方式 → 在下面表格登记一行 → 单独一次 commit。发布与部署仍按各子目录自己的流程走，不因为放进同一个仓库就合并。
跨子目录改动要单独说明；子目录内改动不要顺手改别的工具。

## 目录

| 子目录 | 内容 | 状态 |
| --- | --- | --- |
| [`upcycle/`](./upcycle/README.md) | 万物改造局：拍一张闲置物照片，得到实用款 / 好看款 / 脑洞款三种改造方案，含 OCR、视觉识图、步骤与清单导出 | 已上线，详见子目录 README |

## 通用规则

- 根 `.gitignore` 只管通用垃圾文件；各子目录保留自己的 `.gitignore`。
- **任何真实密钥都不进仓库**：只提交 `.env.example`，`.env`、云函数部署生成物、本机模型配置一律排除。
- 提交前自查 `git status` 与实际 diff；推送后用远端真实状态验真（`git ls-remote` + GitHub 目录/raw 文件），退出码 0 不代表内容真的到了远端。

## 署名

Evan Song · [github.com/Songhonglei](https://github.com/Songhonglei) · 提交邮箱 `Songhonglei@users.noreply.github.com`。
