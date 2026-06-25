# CC 检测逻辑重构 Plan

## 核心逻辑（极简一条件）

**同一 URL path 在 15 秒内被请求 ≥ N 次 → 判定为 CC 攻击 → 立即 mitigate 该域名**

### 双模式阈值

| 模式 | 阈值（15 秒内同一 URL path） | 说明 |
|------|------|------|
| **Normal（日常）** | ≥ 500 次 | 宽松，不误杀正常流量 |
| **War（战时）** | ≥ 200 次 | 收紧，攻击活跃时更容易拦截 |

### 模式切换

```
初始状态 → Normal Mode

任一受保护域名被判定攻击 → 切 War Mode
War Mode 持续 15 分钟无新攻击 → 自动切回 Normal
```

### 为什么一个条件就够了？

CC 攻击的本质：**大量请求集中在同一 URL → 后端被打爆**

不管是一个 IP 猛刷还是两万个 IP 同时打，对下游后端的压力是一样的。
只要同一 URL 在 15 秒内突然暴增（远超正常 QPS 5-10 倍），切走 DNS 就能保护下游。

### 验证：上次误判不会触发

`pwacn.lcpj2901.xyz` 在 22:10-22:18（8 分钟）3510 个请求：
- 最高频 URL `/api/parse/x_video_get2` 只有 397 次（8 分钟）
- 换算 15 秒 ≈ 10 次 → 远低于 500 阈值
- **不触发 ✓**

## 改动范围

### 只改一个文件
`/mnt/code/vj/reverse_proxy_analyzer/utils/openrestyanalyzer.js`

### 具体改动

1. **新增配置** — `CC_ROLLING_WINDOW_MS`, `CC_NORMAL_THRESHOLD`, `CC_WAR_THRESHOLD`, `CC_WAR_COOLDOWN_MS`
2. **新增方法** — `detectCCByRollingWindow()` — 基于 15 秒 Rolling Window 检测
3. **新增状态** — `ccMode`（normal/war）、`ccWarStartTime`
4. **移除** — 旧的 `detectCCAttack()` 方法（~200 行）
5. **移除** — `HIGH_FREQ_TO_CC` 联动逻辑（`checkHighFreqToCC()`, `_recordHighFreqDomain()`, `highFreqDomainMap`）
6. **移除** — `CC_RATE_THRESHOLD`, `CC_RATE_MIN_SAMPLES`, `CC_DETECTION_COOLDOWN_MS` 配置
7. **修改** — `_runCCDetection()` 调用新方法
8. **修改** — `mitigateCCAttack()` 接收新检测结果的格式

### 不改

- `mitigator.js` — mitigate 逻辑不变
- 日志格式不变
- 健康检查接口不变（`/cc-status` 返回字段微调）

## 测试

- Unit Test: 模拟正常流量 + CC 攻击日志
- E2E: 用 `pwacn.lcpj2901.xyz` 真实日志回放验证不误判
