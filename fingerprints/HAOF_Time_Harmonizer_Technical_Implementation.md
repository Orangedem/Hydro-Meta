# HAOF 時間調諧器技術實施文檔

## 1. 系統架構

HAOF 時間調諧器採用分層架構設計，包含以下主要組件：

```
┌─────────────────────────────────────┐
│           前端視覺化層               │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│  │ 狀態顯示 │ │頻率視覺化│ │ 事件列表 │ │
│  └─────────┘ └─────────┘ └─────────┘ │
├─────────────────────────────────────┤
│              API 層                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│  │狀態 API  │ │指紋 API  │ │共振 API │ │
│  └─────────┘ └─────────┘ └─────────┘ │
├─────────────────────────────────────┤
│             服務層                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│  │指紋生成器│ │共振檢測器│ │時間追踪器│ │
│  └─────────┘ └─────────┘ └─────────┘ │
├─────────────────────────────────────┤
│             數據層                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│  │指紋存儲  │ │共振記錄  │ │節律模式  │ │
│  └─────────┘ └─────────┘ └─────────┘ │
└─────────────────────────────────────┘
```

## 2. 核心算法

### 2.1 時間指紋生成

時間指紋生成基於多維度頻率計算模型，主要流程如下：

1. **維度值生成**:
   ```typescript
   // 為每個維度生成0.1到1之間的值
   this.timeDimensions.forEach(dim => {
     const value = Math.round((0.1 + Math.random() * 0.9) * 100) / 100;
     dimensions[dim] = value;
     
     if (value > maxValue) {
       maxValue = value;
       dominantDimension = dim;
     }
   });
   ```

2. **整體強度計算**:
   ```typescript
   // 計算維度值的加權平均
   const intensity = Object.values(dimensions).reduce((sum, val) => sum + val, 0) 
     / this.timeDimensions.length;
   ```

3. **節律模式匹配**:
   ```typescript
   // 從現有模式中選擇或生成新的節律特徵
   const rhythmPattern = this.rhythmPatterns[randomPatternId]?.signature.map(s => s.toString()) || [];
   ```

### 2.2 共振檢測算法

共振檢測採用頻率相似度分析，核心實現如下：

```typescript
// 計算各維度的相似度
Object.keys(current.dimensions).forEach(dim => {
  const currentVal = current.dimensions[dim];
  const previousVal = previous.dimensions[dim] || 0;
  
  // 使用餘弦相似度公式計算相似度
  const similarity = 1 - Math.abs(currentVal - previousVal);
  similarities[dim] = similarity;
  totalSimilarity += similarity;
});

// 計算平均相似度
const avgSimilarity = totalSimilarity / Object.keys(similarities).length;

// 確定共振強度等級
let strengthLevel = 'none';
if (avgSimilarity >= 0.95) strengthLevel = 'perfect';
else if (avgSimilarity >= 0.8) strengthLevel = 'strong';
else if (avgSimilarity >= 0.6) strengthLevel = 'moderate';
else if (avgSimilarity >= 0.4) strengthLevel = 'weak';
```

## 3. 數據結構

### 3.1 時間指紋 (TimeFingerprint)

```typescript
interface TimeFingerprint {
  id: string;                       // 唯一標識符
  timestamp: string;                // ISO時間戳
  dimensions: Record<string, number>; // 各維度的頻率值
  intensity: number;                // 整體強度
  dominantDimension: string;        // 主導維度
  rhythmPattern: string[];          // 節律模式
  metadata: Record<string, any>;    // 元數據
}
```

### 3.2 共振事件 (ResonanceEvent)

```typescript
interface ResonanceEvent {
  id: string;                     // 唯一標識符
  timestamp: string;              // ISO時間戳
  strength: string;               // 強度等級
  effectScore: number;            // 影響分數
  description: string;            // 事件描述
  involvedDimensions: string[];   // 參與維度
  tags: string[];                 // 標籤
  duration: number;               // 持續時間(ms)
  fingerprintIds: string[];       // 關聯的指紋ID
}
```

## 4. API 端點

### 4.1 Status API

- `GET /api/time-harmonizer/status`: 獲取當前時間狀態
  - 返回: `TimeStatus` 對象

### 4.2 Fingerprint API

- `GET /api/time-harmonizer/fingerprints`: 獲取指紋列表
  - 參數: `count` (可選) - 要返回的指紋數量
  - 返回: `TimeFingerprint[]` 數組

- `POST /api/time-harmonizer/fingerprint`: 生成新指紋
  - 返回: 新生成的 `TimeFingerprint` 對象

- `GET /api/time-harmonizer/fingerprints/:id`: 獲取特定指紋
  - 參數: `id` - 指紋ID
  - 返回: `TimeFingerprint` 對象或404錯誤

### 4.3 Resonance API

- `GET /api/time-harmonizer/resonance-events`: 獲取共振事件
  - 參數: 
    - `count` (可選) - 要返回的事件數量
    - `minStrength` (可選) - 最小強度等級
  - 返回: `ResonanceEvent[]` 數組

- `POST /api/time-harmonizer/detect-resonance`: 執行共振檢測
  - 返回: 新檢測到的 `ResonanceEvent[]` 數組

### 4.4 Tracker API

- `POST /api/time-harmonizer/tracker/start`: 啟動時間追踪器
  - 參數: `intervalMs` (可選) - 追踪間隔(毫秒)
  - 返回: 操作狀態

- `POST /api/time-harmonizer/tracker/stop`: 停止時間追踪器
  - 返回: 操作狀態

## 5. 前端組件

### 5.1 TimeHarmonizerStatus

負責顯示當前時間狀態，提供指紋生成和追踪器控制功能。使用React Hooks從API獲取數據：

```typescript
const { data, isLoading, error, refetch } = useQuery({
  queryKey: ['/api/time-harmonizer/status'],
  refetchInterval: 30000, // 每30秒自動刷新
});
```

### 5.2 TimeFrequencyVisualizer

實現雷達圖視覺化，使用HTML Canvas API繪製時間頻率分布：

```typescript
// 繪製頻率多邊形
ctx.beginPath();
dimensionKeys.forEach((dim, index) => {
  const value = dimensions[dim];
  const angle = index * angleStep;
  const r = value * radius;
  const x = centerX + r * Math.cos(angle);
  const y = centerY + r * Math.sin(angle);
  
  if (index === 0) {
    ctx.moveTo(x, y);
  } else {
    ctx.lineTo(x, y);
  }
});
ctx.closePath();
ctx.fillStyle = 'rgba(99, 102, 241, 0.2)';
ctx.fill();
```

### 5.3 ResonanceEventList

顯示共振事件列表，提供共振檢測功能：

```typescript
// 獲取共振事件數據
const { data, isLoading, error, refetch } = useQuery({
  queryKey: ['/api/time-harmonizer/resonance-events', { count, minStrength }],
  refetchInterval: 60000, // 每分鐘自動刷新
});
```

## 6. 持久化設計

數據存儲採用JSON文件系統，保證數據持久性：

```typescript
// 保存時間指紋
private saveFingerprints() {
  fs.writeFileSync(this.fingerprintsPath, JSON.stringify(this.fingerprints, null, 2));
}

// 保存共振事件
private saveResonanceEvents() {
  fs.writeFileSync(this.resonanceEventsPath, JSON.stringify(this.resonanceEvents, null, 2));
}
```

## 7. 擴展性和未來規劃

### 7.1 時間映射優化

計劃實現更精確的時間維度映射算法：

```typescript
// 時間映射函數示例
function mapTime(sourceTime: Date, sourceType: string, targetType: string): Date {
  // 獲取映射配置
  const mapping = this.timeFlowMappings.find(
    m => m.sourceType === sourceType && m.targetType === targetType
  );
  
  if (!mapping) return sourceTime;
  
  // 應用變換
  const sourceMs = sourceTime.getTime();
  const targetMs = sourceMs * mapping.conversionFactor + mapping.offsetSeconds * 1000;
  
  return new Date(targetMs);
}
```

### 7.2 動態適應機制

計劃基於使用模式自動調整時間同步策略：

```typescript
// 動態調整示例
function adaptTimeMappings(userPatterns: UserPattern[]): void {
  userPatterns.forEach(pattern => {
    // 根據使用模式調整相應映射的轉換因子
    const relevantMapping = this.timeFlowMappings.find(
      m => m.sourceType === pattern.contextType
    );
    
    if (relevantMapping) {
      const adjustmentFactor = calculateAdjustmentFactor(pattern);
      relevantMapping.conversionFactor *= adjustmentFactor;
    }
  });
  
  this.saveTimeFlowMappings();
}
```

## 8. 安裝與配置說明

時間調諧器已作為HAOF系統的一部分集成，無需額外安裝步驟。系統启动时會自動初始化所需的數據結構和服務。

---

*文檔版本: 1.0.0 (2025-04-12)*
