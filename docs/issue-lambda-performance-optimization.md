# Issue: Lambda 並發處理效能優化與資源管理

## 問題描述

當通用任務排程系統需要處理大量排程任務時，可能面臨 Lambda 函數的效能瓶頸和資源限制問題。隨著系統規模擴大，單次執行可能需要觸發數百甚至數千個外部服務，這將對 Lambda 的執行時間、記憶體使用和並發處理能力帶來挑戰。

## 問題背景

### 當前系統設計

- EventBridge 每分鐘觸發 Scheduler Lambda
- 查詢到期的排程任務並批次處理
- 對每個任務發送 HTTP 觸發請求（30 秒超時）
- 更新任務狀態和計算下次執行時間

### 潛在問題場景

1. **大量同時到期任務**：尖峰時段可能有 100-1000+ 個任務需要觸發
2. **外部服務回應緩慢**：部分服務可能接近 30 秒超時
3. **記憶體使用激增**：大量並發 HTTP 請求佔用記憶體
4. **Lambda 執行時間過長**：接近或超過 15 分鐘限制

## 具體風險分析

### 🚨 執行時間風險

**序列執行風險**：

- 100 個任務 × 30 秒超時 = 50 分鐘 > 15 分鐘限制 ❌
- Lambda 會被強制終止，任務處理不完整

**並發執行考量**：

- 100 個任務並發 ≈ 30 秒執行時間 ✅
- 但記憶體使用會大幅增加

### 💾 記憶體使用風險

**並發請求記憶體估算**：

```
每個 HTTP 請求：12-103KB
100個並發請求：1.2-10.3MB
500個並發請求：6-51.5MB
1000個並發請求：12-103MB
```

**Lambda 記憶體配置影響**：

- 128MB：可能不足以處理 100+ 並發
- 256MB：安全處理 100-200 並發
- 512MB：處理 500+ 並發
- 1GB+：處理 1000+ 並發

### 📊 資料庫查詢效能

**大量任務查詢**：

```sql
SELECT * FROM scheduled_tasks
WHERE next_execution_time <= NOW()
AND is_active = true AND is_finished = false;
```

**潛在問題**：

- 查詢結果集過大
- 資料庫連線超時
- 索引效能下降

### 💰 成本影響

**記憶體配置成本**（每月 100 萬次執行）：

- 128MB → 512MB：成本增加 4 倍
- 執行時間延長：額外的計算成本
- 資料庫連線時間：RDS 連線成本

## 監控指標

### 需要追蹤的關鍵指標

**Lambda 效能指標**：

- 執行時間分佈
- 記憶體使用峰值
- 超時錯誤率
- 並發執行數量

**任務處理指標**：

- 每分鐘處理任務數量
- 任務積壓數量
- 觸發成功率
- 平均觸發回應時間

**資源使用指標**：

- 資料庫查詢回應時間
- 資料庫連線池使用率
- Lambda 冷啟動頻率

## 優化策略方案

### 方案 1：批次並發控制

**實作策略**：

```javascript
async function processTasksInBatches(tasks, batchSize = 50, concurrency = 20) {
  const batches = createBatches(tasks, batchSize);
  const results = [];

  for (const batch of batches) {
    const batchResults = await processWithConcurrency(batch, concurrency);
    results.push(...batchResults);

    // 記憶體回收間隔
    await sleep(100);
  }

  return results;
}
```

**優勢**：

- 控制記憶體使用峰值
- 可調整的並發參數
- 保持在 Lambda 時間限制內

**適用場景**：

- 中等規模任務量（100-500 個）
- 記憶體預算有限
- 需要穩定可控的處理

### 方案 2：動態資源調整

**實作策略**：

```javascript
function calculateOptimalConfig(taskCount) {
  if (taskCount <= 50) {
    return { memory: "256MB", concurrency: 20, batchSize: 50 };
  } else if (taskCount <= 200) {
    return { memory: "512MB", concurrency: 50, batchSize: 100 };
  } else {
    return { memory: "1GB", concurrency: 100, batchSize: 200 };
  }
}
```

**優勢**：

- 根據負載自動調整
- 成本效益最佳化
- 避免資源浪費

**適用場景**：

- 任務量變化較大
- 需要自動化管理
- 成本敏感應用

### 方案 3：分散式處理

**實作策略**：

```javascript
// 主 Lambda 只負責任務分發
async function distributeTasks(tasks) {
  const chunks = createChunks(tasks, 50);

  const promises = chunks.map((chunk) =>
    invokeLambda("task-processor", { tasks: chunk })
  );

  return await Promise.allSettled(promises);
}
```

**優勢**：

- 突破單一 Lambda 限制
- 水平擴展能力
- 故障隔離

**適用場景**：

- 超大規模任務量（1000+個）
- 需要高可用性
- 複雜的錯誤恢復需求

### 方案 4：佇列化處理

**實作策略**：

```javascript
// 將任務推送到 SQS，由多個 Lambda 消費
async function enqueueTasksForProcessing(tasks) {
  const messages = tasks.map((task) => ({
    MessageBody: JSON.stringify(task),
    DelaySeconds: 0,
  }));

  await sqs
    .sendMessageBatch({
      QueueUrl: TASK_QUEUE_URL,
      Entries: messages,
    })
    .promise();
}
```

**優勢**：

- 解耦任務發現與執行
- 自動負載均衡
- 內建重試機制

**適用場景**：

- 超高頻率任務
- 需要持久化佇列
- 複雜的重試邏輯

## 實作優先級

### Phase 1: 監控與基線建立（1-2 天）

- [ ] 實作關鍵效能指標監控
- [ ] 建立不同負載下的效能基線
- [ ] 設定告警閾值

### Phase 2: 批次並發控制（2-3 天）

- [ ] 實作可配置的批次處理邏輯
- [ ] 新增記憶體使用監控
- [ ] 調整並發參數和批次大小

### Phase 3: 動態資源調整（3-4 天）

- [ ] 實作負載評估邏輯
- [ ] 建立資源配置決策引擎
- [ ] 測試不同負載場景

### Phase 4: 高級優化（視需求）

- [ ] 分散式處理架構
- [ ] SQS 佇列化處理
- [ ] 複雜的故障恢復機制

## 測試計畫

### 效能測試場景

**基準測試**：

- 10, 50, 100, 500, 1000 個任務
- 不同的外部服務回應時間
- 各種記憶體配置下的表現

**壓力測試**：

- 極端負載情況
- 外部服務全部超時
- 記憶體不足場景

**長期穩定性測試**：

- 連續運行 24 小時
- 模擬真實流量模式
- 記憶體洩漏檢測

## 成功指標

### 效能目標

- Lambda 執行時間 < 5 分鐘（預留緩衝）
- 記憶體使用率 < 80%
- 任務觸發成功率 > 99%
- 平均觸發延遲 < 2 秒

### 可用性目標

- 系統正常運行時間 > 99.9%
- 故障恢復時間 < 5 分鐘
- 任務丟失率 = 0%

### 成本目標

- Lambda 成本增加 < 50%（相對於優化前）
- 整體系統 TCO 保持合理範圍

## 風險評估

### 技術風險

- **記憶體不足**：導致 Lambda 執行失敗
- **執行超時**：未完成的任務狀態不一致
- **並發限制**：AWS 帳戶層級限制

### 業務風險

- **任務延遲**：影響依賴排程的業務流程
- **成本超支**：資源配置過度導致成本激增
- **服務中斷**：優化過程中的系統不穩定

### 緩解措施

- **漸進式部署**：小批量測試後再全面推廣
- **回滾機制**：保留優化前的配置作為備案
- **監控告警**：即時發現並處理異常情況

## 相關文件

### 依賴項目

- [通用任務排程系統](./issue-universal-task-scheduler-system.md)
- [多元狀態查詢方式擴充](./issue-query-methods-extension.md)

### 參考資料

- [AWS Lambda 限制和配額](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
- [Node.js 記憶體管理最佳實踐](https://nodejs.org/en/docs/guides/simple-profiling/)
- [AWS Lambda 效能優化指南](https://aws.amazon.com/lambda/performance/)

---

**建立日期：** 2025 年 10 月 4 日  
**狀態：** 待觸發  
**優先級：** 中（當系統負載增加時提升為高）  
**觸發條件：** 單次處理任務數量 > 100 個 或 Lambda 執行時間 > 3 分鐘

**預期時程：** 8-12 天（視選擇的優化方案而定）
