# Issue: 排程系統重試機制與狀態一致性優化

## 問題描述

當前的通用任務排程系統在處理 timeout 重試和狀態管理時存在潛在的業務邏輯重複執行風險和狀態一致性問題。需要設計更完善的重試機制、回調 Token 管理和狀態同步策略，以確保系統的可靠性和業務邏輯的正確性。

## 問題背景

### 當前系統機制

1. 系統觸發目標服務（攜帶 callback_url 和 callback_token）
2. 30 秒超時機制，超時後進行重試（最多 5 次）
3. 目標服務可透過 callback_token 寫回狀態更新
4. 目標服務回傳 query_config 供終端應用查詢狀態

### 發現的問題場景

**問題 1：Timeout 重試導致重複執行**

```
時間軸範例：
T0:  觸發執行 A (callback_token_A) → 創建訂單開始處理
T30: 系統判定 A timeout → 觸發重試 B (callback_token_B) → 重複創建訂單
T60: 系統判定 B timeout → 觸發重試 C (callback_token_C) → 再次重複創建訂單
T90: 執行 A 實際完成，嘗試回調 callback_token_A
```

**問題結果**：

- 同一個業務需求被執行了 3 次（重複下單、重複發送通知等）
- 系統狀態顯示失敗，但實際業務已成功處理
- 延遲回調可能更新錯誤的執行記錄

**問題 2：回調 Token 生命週期混亂**

```
執行記錄狀態：
execution_123: callback_token_A, status=timeout
execution_124: callback_token_B, status=timeout
execution_125: callback_token_C, status=pending

實際回調：
- callback_token_A 延遲回調 → 找不到對應的 pending 記錄
- callback_token_B 延遲回調 → 覆蓋已 timeout 的記錄
```

**問題 3：狀態查詢與重試決策脫節**

- 系統有 query_config 可查詢目標服務狀態
- 但重試決策沒有利用查詢機制
- 可能在目標服務仍在處理時就重試

## 複雜性分析

### 業務影響風險

**電商場景**：

- 重複下單：同一筆購買產生多個訂單
- 重複扣款：財務對帳困難
- 庫存混亂：商品庫存計算錯誤

**通知服務**：

- 重複通知：用戶收到多封相同郵件/簡訊
- 成本浪費：第三方通知服務重複計費

**資料同步**：

- 資料重複：同一批資料被多次匯入
- 關聯性錯誤：依賴資料的業務邏輯異常

### 技術複雜度

**冪等性設計**：

- 需要業務層面的唯一執行標識
- 目標服務需要實作去重邏輯
- 跨系統的冪等性約定

**狀態一致性**：

- 多個執行記錄的狀態衝突處理
- 延遲回調的時序問題
- 系統狀態與業務狀態的同步

**重試決策智能化**：

- 結合查詢機制的重試判斷
- 動態調整重試間隔和策略
- 不同業務場景的重試策略差異化

## 潛在解決方案框架

### 方案 1：執行唯一標識機制

**設計思路**：

```json
{
  "businessExecutionId": "task_123_cycle_1_v1",  // 業務執行唯一ID
  "systemCallbackToken": "token_456",            // 系統回調Token
  "idempotencyKey": "idem_789",                  // 冪等性鍵值
  "taskPayload": {...}
}
```

**核心概念**：

- 同一週期的重試使用相同的 businessExecutionId
- 目標服務根據 businessExecutionId 實作冪等性檢查
- 系統內部用 systemCallbackToken 管理回調路由

### 方案 2：智能重試決策引擎

**決策流程**：

```javascript
async function shouldRetryExecution(lastExecution) {
  // 1. 檢查是否有查詢配置
  if (lastExecution.query_config) {
    const currentStatus = await queryTargetServiceStatus(
      lastExecution.query_config
    );
    if (currentStatus.isProcessing || currentStatus.isCompleted) {
      return { shouldRetry: false, reason: "still_processing_or_completed" };
    }
  }

  // 2. 檢查最近回調時間
  const timeSinceLastCallback = now() - lastExecution.last_callback_time;
  if (timeSinceLastCallback < MINIMUM_RETRY_INTERVAL) {
    return { shouldRetry: false, reason: "too_soon_since_callback" };
  }

  // 3. 檢查重試次數和策略
  return evaluateRetryPolicy(lastExecution);
}
```

### 方案 3：回調 Token 版本化管理

**Token 結構**：

```javascript
const callbackToken = {
  taskId: "task_123",
  cycleNumber: 1,
  executionSequence: 2, // 該週期內的第幾次執行
  version: "v1",
  expiresAt: timestamp,
};
```

**回調處理邏輯**：

```javascript
async function handleCallback(token, statusUpdate) {
  const execution = await findExecutionByToken(token);

  // 檢查 Token 有效性
  if (!isValidToken(token, execution)) {
    return { error: "invalid_or_expired_token" };
  }

  // 狀態優先級檢查
  if (
    !canTransitionToNewStatus(execution.current_status, statusUpdate.status)
  ) {
    return { error: "invalid_status_transition" };
  }

  // 更新狀態
  await updateExecutionStatus(execution.id, statusUpdate);
}
```

### 方案 4：狀態同步與查詢整合

**雙軌狀態管理**：

```javascript
// 系統內部狀態：用於重試決策
const systemStatus = {
  execution_status: "pending|processing|success|failed|timeout",
  last_system_update: timestamp,
  retry_count: number,
};

// 業務狀態：從目標服務查詢
const businessStatus = {
  business_status: "any_custom_status",
  last_query_time: timestamp,
  query_source: "callback|query_api",
};
```

## 實作優先級與階段規劃

### Phase 1: 問題分析與監控（2-3 天）

- [ ] 建立重試行為的詳細監控
- [ ] 統計 timeout 重試的實際業務影響
- [ ] 分析不同業務場景的重試需求差異
- [ ] 建立問題重現的測試環境

### Phase 2: 基礎冪等性支援（3-4 天）

- [ ] 設計業務執行唯一標識機制
- [ ] 修改觸發請求格式，加入冪等性參數
- [ ] 建立目標服務冪等性實作指南
- [ ] 測試冪等性機制的有效性

### Phase 3: 智能重試決策（4-5 天）

- [ ] 整合查詢機制到重試決策中
- [ ] 實作基於狀態查詢的重試判斷
- [ ] 設計可配置的重試策略引擎
- [ ] 不同業務類型的重試策略差異化

### Phase 4: 回調管理優化（3-4 天）

- [ ] 實作回調 Token 版本化管理
- [ ] 建立回調狀態衝突處理機制
- [ ] 優化延遲回調的處理邏輯
- [ ] 回調有效期和清理機制

### Phase 5: 狀態一致性保證（4-5 天）

- [ ] 設計多執行記錄的狀態同步策略
- [ ] 實作狀態優先級和轉換規則
- [ ] 建立狀態異常的自動修復機制
- [ ] 完善狀態查詢與回調的整合

### Phase 6: 全面測試與優化（3-4 天）

- [ ] 複雜場景的端到端測試
- [ ] 效能和穩定性驗證
- [ ] 向後相容性確保
- [ ] 監控和告警完善

## 風險評估

### 實作風險

- **複雜度激增**：多個機制的交互可能帶來新的 bug
- **效能影響**：額外的查詢和檢查邏輯影響執行效率
- **相容性問題**：現有目標服務需要配合調整

### 業務風險

- **過度設計**：可能解決了不存在的問題
- **開發週期**：複雜功能延遲系統上線時間
- **學習成本**：開發團隊需要理解複雜的狀態管理邏輯

### 緩解策略

- **漸進式實作**：分階段推出，每階段都可獨立運作
- **可選機制**：新機制作為可選功能，不影響現有流程
- **充分測試**：建立完整的測試場景覆蓋

## 成功指標

### 可靠性指標

- 重複業務邏輯執行事件 = 0
- 狀態不一致事件 < 0.1%
- 延遲回調處理成功率 > 99%

### 效能指標

- 重試決策時間 < 500ms
- 狀態查詢回應時間 < 2s
- 系統整體執行時間增加 < 10%

### 業務指標

- 業務流程正確性 = 100%
- 重複操作投訴 = 0
- 系統可用性 > 99.9%

## 相關文件

### 依賴項目

- [通用任務排程系統](./issue-universal-task-scheduler-system.md)
- [Lambda 並發處理效能優化](./issue-lambda-performance-optimization.md)
- [多元狀態查詢方式擴充](./issue-query-methods-extension.md)

### 技術參考

- 分散式系統冪等性設計模式
- 事件驅動架構的狀態管理
- 微服務間的一致性保證機制

---

**建立日期：** 2025 年 10 月 4 日  
**狀態：** 待評估  
**優先級：** 中高（影響業務正確性）  
**觸發條件：** 當發現重複業務邏輯執行或狀態不一致問題時

**預期時程：** 19-25 天（完整實作所有階段）  
**最小可行版本：** Phase 1-2 (基礎冪等性支援，5-7 天)
