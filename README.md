# distributed-agent-on-aliyun

在阿里云上构建分布式 Agent Harness。

## 状态

**空骨架，无源码。** 方向设计未定，`src/` 暂空。

## 环境

| 项 | 值 | 说明 |
|---|---|---|
| Node | **24.14.0** | 见 `.nvmrc`。本机装了 nvm 多版本，**必须显式切换**：`export PATH="$HOME/.nvm/versions/node/v24.14.0/bin:$PATH"`，否则会被其他 Node 遮蔽 |
| TypeScript | 7.0.2 | 仅用于 `tsc --noEmit` 类型检查，不参与运行 |
| 运行时依赖 | 无 | 本地跑 `.ts` 靠 Node 原生 type stripping |

```sh
pnpm install          # 装 typescript + @types/node
pnpm typecheck        # tsc --noEmit
```

> `tsc` 目前会报 `No inputs were found` —— 因为 `src/` 还没有 `.ts` 文件，属预期。

## 目录

```
docs/research/        前期调研成品（可复用，勿删）
src/                  源码（空）
```

## 前期调研

`docs/research/` 下三份，都是从上一轮 DSH 复刻/阿里云方案里筛出的可复用成果：

| 文件 | 内容 |
|---|---|
| `dsh-to-aliyun-mapping.md` | DSH → 阿里云逐项映射表（654 行）：14 项前置决策、执行世界映射、会话数据平面、放弃清单、分阶段实施、10 个待验证项 |
| `dsh-0.2.2-architecture-diff.md` | DSH 0.2.2 相对 0.1.0-rc.8 的架构级重构实测 diff（事件模型、工具管线、投影接缝、Session 格式 v0→v4） |
| `decision-01-deploy-tool.md` | Serverless Devs vs aliyun CLI 选型调研：本机实测、能力矩阵、FC 平台硬约束（Node 运行时只到 `nodejs20`）、三条路径 |

**一条已被实测确认的硬约束**：FC 3.0 内置 Node.js 运行时最高 `nodejs20`，而 Node 20 无原生 TS 支持。本项目的「零构建跑 `.ts`」只在本地成立，上云必须编译或换运行时。详见 `decision-01-deploy-tool.md` §4.1。

## 待定

- 项目形态与模块划分
- 部署形态（FC Web 函数 / 事件函数 / 容器镜像）
- 执行世界（ECS+SSH / FC+NAS / 沙箱）
- 是否 `git init`（`.gitignore` 已就位）
