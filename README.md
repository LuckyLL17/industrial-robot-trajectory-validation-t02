# 工业机器人离线轨迹规划与碰撞验证服务

纯后端 Node.js + TypeScript + Fastify 服务，默认无需 Docker、数据库或消息中间件即可运行。

## 启动

```bash
npm install
npm run dev
```

默认端口：`6303`。健康检查：`GET /health`。

## 业务闭环

- 领域状态模型与事件输入
- 可回放的时间序列/仿真流程
- 约束校验与风险门禁
- 方案版本、审计轨迹与异常处置
- 所有状态暂存于进程内，便于本机演示；可扩展为数据库、事件总线和实时流处理架构

## 增强业务闭环

- TCP 工具标定误差和标定有效期管理
- 工件基准坐标系登记
- 动态障碍物上报与清除
- 轨迹发布前的碰撞、标定、动态障碍物多重安全门禁
- 执行中关节跟踪误差和 TCP 力阈值保护停机
- 轨迹 release、执行遥测和安全审计

扩展接口：`/api/calibrations`、`/api/frames`、`/api/dynamic-obstacles`、`/api/paths/:id/release`、`/api/executions/:id/telemetry`。

## 优化候选选择—新版本—规划闭环

优化接口保持只读，候选选择形成可追溯版本，重新校验通过后才能规划：

1. `POST /api/paths/:id/optimize`（只读）：登记一次优化会话（run），返回多个速度/blend 候选及 `runId`，不修改原路径、不建版、不规划。
2. `GET /api/paths/:id/candidates` / `GET /api/optimize-runs/:runId`：查看会话候选、可行性、校验指标与采用状态。
3. `POST /api/optimize-runs/:runId/adopt`：选择候选。原路径保留为同 family 的基线（baseline），被选候选生成版本号递增的新版本（adopted-candidate），记录 runId/candidateId/来源版本等追溯信息。选择时会重新校验候选快照；新版本处于 draft，需再次 `validate` 通过后才能 `plan`。
4. `POST /api/paths/:newVersionId/validate` → `POST /api/paths/:newVersionId/plan`：仅对已校验版本规划，不影响基线。
5. `GET /api/paths/:id/compare`：区分 `baseline`、`candidate`（候选，含 adopted 标记）与 `adoptedVersions`（采用版本，含校验与规划情况）。

失败处置：无可行候选（`best=null`、run 状态 `no-feasible-candidate`）、候选重新校验失效（`candidate-stale`）或重复选择（`409 alreadyAdopted`）均不覆盖基线、不重复建版、不生成规划；重复选择返回已采用版本信息以便恢复。
