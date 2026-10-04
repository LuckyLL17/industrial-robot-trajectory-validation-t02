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

## 优化候选「选择—新版本—规划」闭环

原始路径自动成为家族基线（`kind=baseline`），其 waypoints 在整个闭环中只读。

1. `POST /api/paths/:id/optimize`（只读，不写审计）返回多个可行候选，每个候选带确定性 `candidateId`（由速度/blend 指纹生成）与校验结果。
2. `POST /api/paths/:id/candidates/select`，body `{"candidateId": "..."}`：
   - 选择时对候选**重新校验**；无可行候选、候选失效（不在当前可行集/重新校验不过）、重复选择均返回 `409`，不覆盖基线、不生成新版本或规划，响应中带 `recover` 恢复提示；缺 `candidateId` 返回 `400`。
   - 成功则在同一家族生成 `kind=adopted` 的可追溯版本（`version` 递增，`source` 记录 candidateId/scale/来源路径/操作员，审计 `candidate.select`），初始 `status=draft`。
3. 新版本须显式 `POST /api/paths/:id/validate` 通过后，`POST /api/paths/:id/plan` 才放行（未重新校验规划返回 `409`）；规划记录带 `familyId/version/kind` 溯源。
4. `GET /api/paths/:id/compare` 按家族区分三类：`baseline`（基线版本及其校验/规划）、`candidates`（候选实时校验结果 + `adopted`/`adoptedBy` 标记）、`adoptedVersions`/`latestAdopted`（采用版本及来源）。

失败恢复：选择失败后基线与既有版本均不变，可重新 `optimize` 取最新候选后再选，也可直接对基线或已采用版本继续校验/规划。
