# Issue: 擴充多元狀態查詢邏輯實作

## 需求描述

擴充通用任務排程系統的狀態查詢處理邏輯，從目前僅實作 HTTP 查詢邏輯，擴展為支援多種查詢方式的處理邏輯，包括 Lambda 函數、DynamoDB、SQS、S3 檔案等，以實際執行不同 `query_config` 類型的查詢操作。

## 背景說明

### 當前狀態

- 資料庫已使用 `query_config JSONB` 欄位，支援多種查詢方式的配置格式
- 目前系統只實作了 `type: "http"` 的查詢處理邏輯
- 對於其他類型（Lambda、DynamoDB 等）會回傳「暫不支援」錯誤

### 實作需求

需要實作各種 `query_config` 類型的實際查詢處理邏輯：

- **Lambda 函數**：調用指定 Lambda 函數並處理回應
- **DynamoDB**：直接查詢 DynamoDB 表格獲取狀態
- **S3 檔案**：從 S3 讀取狀態檔案並解析
- **SQS 訊息**：查詢 SQS 訊息獲取處理狀態
- **EventBridge 事件**：透過事件溯源查詢狀態

## 專案目標

### 核心功能擴充

**需要實作的查詢邏輯：**

1. **HTTP API 查詢**（已實作）

   ```json
   {
     "type": "http",
     "config": {
       "url": "https://api.service.com/task/123/status",
       "method": "GET",
       "headers": { "Authorization": "Bearer token" }
     }
   }
   ```

2. **Lambda 函數查詢**（待實作）

   ```json
   {
     "type": "lambda",
     "config": {
       "functionArn": "arn:aws:lambda:region:account:function:get-task-status",
       "payload": { "taskId": "123" },
       "timeout": 30
     }
   }
   ```

3. **DynamoDB 直接查詢**（待實作）

   ```json
   {
     "type": "dynamodb",
     "config": {
       "tableName": "task-status",
       "key": { "taskId": "123" },
       "projection": ["status", "progress", "result"]
     }
   }
   ```

4. **S3 檔案查詢**（待實作）

   ```json
   {
     "type": "s3",
     "config": {
       "bucket": "task-results",
       "key": "executions/task-123/status.json"
     }
   }
   ```

5. **SQS 訊息查詢**（待實作）

   ```json
   {
     "type": "sqs",
     "config": {
       "queueUrl": "https://sqs.region.amazonaws.com/account/status-queue",
       "messageGroupId": "task-123"
     }
   }
   ```

6. **EventBridge 事件查詢**（待實作）
   ```json
   {
     "type": "eventbridge",
     "config": {
       "eventBusArn": "arn:aws:events:region:account:event-bus/task-events",
       "source": "task.service",
       "detailType": "Task Status Update",
       "correlationId": "task-123"
     }
   }
   ```

## 技術設計

### 當前架構狀態

**資料庫已完成調整：**

```sql
-- query_config JSONB 欄位已存在
-- GIN 索引已建立
CREATE INDEX idx_task_execution_history_query_type ON task_execution_history USING GIN ((query_config->>'type'));
```

**當前查詢處理邏輯：**

```typescript
// 目前只處理 HTTP 類型
if (queryConfig.type === "http") {
  return await this.httpQueryProvider.query(queryConfig.config);
} else {
  throw new Error(`查詢類型 '${queryConfig.type}' 暫不支援`);
}
```

### 查詢抽象層設計

**統一查詢介面：**

```typescript
interface IQueryProvider {
  type: string;
  query(config: QueryConfig): Promise<TaskStatus>;
  validate(config: QueryConfig): boolean;
}

interface QueryConfig {
  type: "http" | "lambda" | "dynamodb" | "s3" | "sqs" | "eventbridge";
  [key: string]: any; // 各種查詢方式的特定參數
}

interface TaskStatus {
  status: "pending" | "processing" | "success" | "failed";
  progress?: number;
  progressMessage?: string;
  details?: any;
  lastUpdated: string;
}
```

### Lambda 函數調整

**新增查詢管理器：**

```typescript
class QueryManager {
  private providers: Map<string, IQueryProvider>;

  constructor() {
    this.providers = new Map([
      ["http", new HttpQueryProvider()],
      ["lambda", new LambdaQueryProvider()],
      ["dynamodb", new DynamoDBQueryProvider()],
      ["s3", new S3QueryProvider()],
      ["sqs", new SQSQueryProvider()],
      ["eventbridge", new EventBridgeQueryProvider()],
    ]);
  }

  async queryTaskStatus(queryConfig: QueryConfig): Promise<TaskStatus> {
    const provider = this.providers.get(queryConfig.type);
    if (!provider) {
      throw new Error(`Unsupported query type: ${queryConfig.type}`);
    }

    if (!provider.validate(queryConfig)) {
      throw new Error(`Invalid config for query type: ${queryConfig.type}`);
    }

    return await provider.query(queryConfig);
  }
}
```

## 實作階段

### Phase 1: 查詢架構完善（1-2 天）

- [ ] 完善查詢管理器架構，支援多種查詢類型
- [ ] 建立查詢抽象層介面定義
- [ ] 實作查詢類型驗證機制

### Phase 2: HTTP 查詢邏輯優化（1 天）

- [ ] 優化現有 HTTP 查詢邏輯
- [ ] 完善錯誤處理和重試機制
- [ ] 新增查詢超時和快取機制

### Phase 3: Lambda 查詢支援（2-3 天）

- [ ] 實作 LambdaQueryProvider
- [ ] 加入 IAM 權限管理
- [ ] 錯誤處理和重試機制

### Phase 4: DynamoDB 查詢支援（2 天）

- [ ] 實作 DynamoDBQueryProvider
- [ ] 支援複合鍵和 GSI 查詢
- [ ] 資料格式標準化

### Phase 5: S3 查詢支援（1-2 天）

- [ ] 實作 S3QueryProvider
- [ ] 支援 JSON/CSV 檔案格式
- [ ] 快取機制優化

### Phase 6: SQS 查詢支援（2 天）

- [ ] 實作 SQSQueryProvider
- [ ] 訊息過濾和查詢邏輯
- [ ] 處理 FIFO 和標準佇列

### Phase 7: EventBridge 查詢支援（2-3 天）

- [ ] 實作 EventBridgeQueryProvider
- [ ] 事件溯源和查詢邏輯
- [ ] 時間範圍和過濾器支援

### Phase 8: 整合測試和優化（2-3 天）

- [ ] 端到端測試各種查詢方式
- [ ] 效能優化和監控
- [ ] 文件更新

## API 設計調整

### 建立排程任務 API 擴充

**新的 targetConfig 格式：**

```json
{
  "taskType": "data-sync",
  "scheduleConfig": {
    "scheduleType": "interval",
    "intervalMinutes": 60
  },
  "taskConfig": {
    "taskType": "sync",
    "targetConfig": {
      "triggerType": "lambda",
      "functionName": "data-sync-service",
      "triggerTimeoutSeconds": 30
    },
    "queryConfig": {
      "type": "dynamodb",
      "tableName": "sync-job-status",
      "key": { "jobId": "{{execution_id}}" },
      "projection": ["status", "progress", "completedAt", "errorMessage"]
    },
    "taskPayload": {
      "syncScope": "daily_orders",
      "batchSize": 100
    }
  }
}
```

### 狀態查詢 API

**新增通用查詢端點：**

```http
GET /tasks/{taskId}/executions/{executionId}/status
```

**回應格式：**

```json
{
  "executionId": "exec-123",
  "systemStatus": "pending",
  "queryConfig": {
    "type": "lambda",
    "functionArn": "arn:aws:lambda:...",
    "payload": { "taskId": "123" }
  },
  "targetStatus": {
    "status": "processing",
    "progress": 65,
    "progressMessage": "Processing batch 7 of 10",
    "details": {
      "processedItems": 650,
      "totalItems": 1000,
      "errors": []
    },
    "lastUpdated": "2025-01-15T14:25:30Z"
  }
}
```

## 效能考量

### 查詢最佳化

- **快取機制**：對頻繁查詢的狀態進行快取
- **批次查詢**：支援一次查詢多個任務狀態
- **非同步查詢**：長時間查詢使用非同步機制

### 監控和告警

- **查詢效能監控**：追蹤各種查詢方式的回應時間
- **錯誤率監控**：監控查詢失敗率和重試機制
- **資源使用監控**：監控 Lambda 函數和資料庫資源使用

## 安全性考量

### 權限管理

- **最小權限原則**：每種查詢方式僅授予必要權限
- **跨服務認證**：安全的服務間認證機制
- **資料加密**：敏感查詢配置資料加密儲存

### 資料隱私

- **查詢日誌**：記錄查詢活動但保護敏感資料
- **存取控制**：限制查詢配置的讀取和修改權限

## 相容性考量

### 現有查詢處理

1. **統一格式**：所有查詢配置都使用 `query_config JSONB` 格式
2. **類型檢查**：根據 `query_config.type` 路由到對應的處理邏輯
3. **漸進式實作**：先實作高優先級的查詢類型（Lambda、DynamoDB）
4. **錯誤處理**：對於未實作的查詢類型提供明確的錯誤訊息

## 測試策略

### 單元測試

- [ ] 各 QueryProvider 的單元測試
- [ ] QueryManager 的整合測試
- [ ] 配置驗證邏輯測試

### 整合測試

- [ ] 各種查詢方式的端到端測試
- [ ] 錯誤處理和重試機制測試
- [ ] 效能壓力測試

### 相容性測試

- [ ] 向後相容性測試
- [ ] 資料遷移測試
- [ ] 版本升級測試

## 文件更新

### 開發者文件

- [ ] 新查詢方式使用指南
- [ ] 各 QueryProvider 的設定範例
- [ ] 最佳實踐指南

### API 文件

- [ ] 更新 API 規格說明
- [ ] 新增查詢配置範例
- [ ] 錯誤代碼和處理說明

## 預期效益

### 技術效益

- **靈活性提升**：支援多種 AWS 原生服務查詢
- **效能優化**：針對不同資料源的最佳化查詢
- **擴展性增強**：易於新增新的查詢方式

### 業務效益

- **更好的監控**：更精確的任務狀態追蹤
- **降低延遲**：直接查詢減少中間層延遲
- **成本優化**：避免不必要的 HTTP 呼叫和資源消耗

## 風險評估

### 技術風險

- **複雜度增加**：多種查詢方式增加系統複雜度
- **效能影響**：不當的查詢配置可能影響效能
- **依賴增加**：對更多 AWS 服務的依賴

### 緩解措施

- **完整測試**：全面的測試覆蓋各種情境
- **監控告警**：完善的監控和告警機制
- **漸進部署**：分階段部署降低風險

## 預期時程

- **Phase 1（架構調整）**: 2-3 天
- **Phase 2（HTTP 重構）**: 1 天
- **Phase 3（Lambda 支援）**: 2-3 天
- **Phase 4（DynamoDB 支援）**: 2 天
- **Phase 5（S3 支援）**: 1-2 天
- **Phase 6（SQS 支援）**: 2 天
- **Phase 7（EventBridge 支援）**: 2-3 天
- **Phase 8（整合測試）**: 2-3 天

**總計預估時程：12-17 天**

---

**建立日期：** 2025 年 10 月 4 日  
**狀態：** 待開始  
**優先級：** 中  
**依賴：** 需要完成 `issue-universal-task-scheduler-system.md` Phase 1 開發

**相關 Issue：**

- [通用任務排程系統](./issue-universal-task-scheduler-system.md)
