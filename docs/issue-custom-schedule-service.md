# Issue: 複雜週期排程外部服務設計

## 需求描述

為通用任務排程系統設計一個外部服務，專門處理複雡的週期性排程邏輯，如隔週、隔月、每季等無法用標準 cron 表達式處理的複雜週期。

## 專案目標

開發一個獨立的排程計算服務，能夠：

- 處理複雜的週期計算邏輯（隔週、隔月、每季等）
- 預先生成未來排程時間點
- 與主要任務排程系統整合
- 支援自訂業務邏輯的週期規則

## 功能需求

### 1. 複雜週期類型支援

- **隔週排程** (bi-weekly)：每兩週執行一次
- **隔月排程** (bi-monthly)：每兩個月執行一次
- **季度排程** (quarterly)：每三個月執行一次
- **半年排程** (semi-annually)：每六個月執行一次
- **年度排程** (annually)：每年執行一次
- **自訂間隔排程**：可指定任意天數、週數、月數間隔
- **業務日排程**：跳過週末和假日的工作日排程
- **月末排程**：每月最後一個工作日
- **相對日期排程**：如每月第二個星期二

### 2. 服務架構設計

#### 排程計算 API

```http
POST /calculate-schedule
Content-Type: application/json

{
  "taskId": "task_12345",
  "scheduleType": "bi-weekly",
  "baseDate": "2025-10-01T00:00:00Z",
  "scheduleConfig": {
    "interval": 2,
    "dayOfWeek": 1,
    "time": "09:00",
    "startDate": "2025-10-01",
    "endDate": "2025-12-31"
  },
  "requestCount": 10
}
```

#### 排程驗證 API

```http
GET /validate-schedule/{taskId}?checkDate=2025-10-15T09:00:00Z
```

#### 預生成排程 API

```http
POST /generate-future-schedules
Content-Type: application/json

{
  "taskId": "task_12345",
  "fromDate": "2025-10-01T00:00:00Z",
  "count": 10
}
```

### 3. 資料庫設計

#### 排程規則表 (schedule_rules)

```sql
CREATE TABLE schedule_rules (
    rule_id SERIAL PRIMARY KEY,
    task_id INTEGER NOT NULL,
    rule_type VARCHAR(50) NOT NULL,        -- bi-weekly, bi-monthly, quarterly, etc.
    rule_config JSONB NOT NULL,            -- 規則配置參數
    base_date TIMESTAMP NOT NULL,          -- 基準日期
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 預生成排程表 (pre_generated_schedules)

```sql
CREATE TABLE pre_generated_schedules (
    schedule_id SERIAL PRIMARY KEY,
    task_id INTEGER NOT NULL,
    rule_id INTEGER REFERENCES schedule_rules(rule_id),
    scheduled_time TIMESTAMP NOT NULL,
    is_consumed BOOLEAN DEFAULT FALSE,      -- 是否已被主系統消費
    generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 4. 整合流程設計

#### 主系統整合點

```
主系統 Scheduler Lambda
    ↓
檢查 schedule_type = 'custom' 的任務
    ↓
調用外部排程服務 API
    ↓
外部服務回傳：
├── 當前是否需要執行 (boolean)
├── 下次執行時間 (timestamp)
└── 未來排程列表 (array)
    ↓
主系統根據回應：
├── 執行當前任務（如果需要）
├── 更新 next_execution_time
└── 檢查是否需要補充未來排程
```

#### 預生成機制

- 當未來排程數量 < 5 個時，自動生成接下來 10 個排程
- 避免重複生成相同時間點的排程
- 支援排程規則變更時重新計算

### 5. 技術架構

#### 服務技術棧

- **Runtime**: Node.js 22.x + TypeScript
- **Framework**: Express.js 或 AWS Lambda + API Gateway
- **Database**: PostgreSQL 或 AWS RDS
- **Deployment**: AWS SAM CLI 或 Docker + ECS
- **Cache**: Redis（可選，用於提升計算效能）

#### 演算法設計

- 實作各種週期計算演算法
- 處理閏年、月份天數差異
- 支援時區轉換
- 業務日曆整合（假日、週末處理）

## 實作階段

### Phase 1: 基礎架構建立

- [ ] 建立專案架構和資料庫設計
- [ ] 實作基本的 API 框架
- [ ] 設定部署管道

### Phase 2: 核心演算法實作

- [ ] 實作隔週、隔月排程演算法
- [ ] 實作季度、年度排程演算法
- [ ] 加入業務日曆邏輯

### Phase 3: API 整合

- [ ] 與主排程系統整合測試
- [ ] 實作預生成機制
- [ ] 加入錯誤處理和降級策略

### Phase 4: 進階功能

- [ ] 支援更複雜的相對日期邏輯
- [ ] 加入排程預覽和驗證功能
- [ ] 效能優化和快取策略

## 優先級和依賴

- **優先級**: 中等（等主系統基本功能完成後再開始）
- **前置條件**: 主任務排程系統的基礎功能完成
- **預估開發時間**: 10-15 天
- **風險評估**: 中等（主要風險在演算法複雜度和整合測試）

## 預期效益

1. **靈活性**: 支援各種複雜的業務排程需求
2. **擴展性**: 可以不影響主系統持續加入新的週期類型
3. **維護性**: 排程邏輯獨立，便於測試和維護
4. **效能**: 預生成機制減少即時計算負擔

---

**建立日期**: 2025 年 10 月 2 日  
**狀態**: 待主系統完成後開始  
**優先級**: 中等  
**依賴**: 主任務排程系統基礎功能
