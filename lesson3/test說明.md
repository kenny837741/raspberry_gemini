# test.py 程式說明

## 概述

這是一個專為 **Open-WebUI** 平台設計的「過濾器（Filter）」插件。  
它的主要功能是：**限制使用者在一次對話中最多能發送幾輪訊息**，超過上限就會拋出錯誤，阻止繼續對話。

| 欄位 | 內容 |
|------|------|
| 標題 | Example Filter |
| 作者 | open-webui |
| 版本 | 0.1 |

---

## 程式架構

```
Filter（主類別）
├── Valves（系統層級設定）
├── UserValves（使用者層級設定）
├── __init__()（初始化）
├── inlet()（請求前處理）
└── outlet()（回應後處理）
```

---

## 各部分詳細說明

### 1. 引入套件

```python
from pydantic import BaseModel, Field
from typing import Optional
```

- `pydantic`：用來定義「有型別驗證」的設定資料模型，確保傳入的值符合預期格式。
- `Optional`：表示某個參數可以是指定型別，也可以是 `None`（不傳也沒關係）。

---

### 2. `Valves`（系統管理員設定）

```python
class Valves(BaseModel):
    priority: int = Field(default=0, ...)
    max_turns: int = Field(default=8, ...)
```

這是給**系統管理員**設定的參數，控制整個過濾器的行為。

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `priority` | `0` | 過濾器的執行優先順序，數字越小越先執行 |
| `max_turns` | `8` | 全域對話輪數上限，所有使用者都受此限制 |

---

### 3. `UserValves`（使用者個人設定）

```python
class UserValves(BaseModel):
    max_turns: int = Field(default=4, ...)
```

這是給**個別使用者**自訂的參數。

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `max_turns` | `4` | 該使用者自己的對話輪數上限 |

> 注意：最終生效的上限是取 `UserValves.max_turns` 和 `Valves.max_turns` 兩者中**較小的值**，避免使用者自己設定超過系統允許的上限。

---

### 4. `__init__`（初始化）

```python
def __init__(self):
    self.valves = self.Valves()
```

當這個 Filter 被載入時，會自動建立一個 `Valves` 實例，套用預設的系統設定。  
（`self.file_handler = True` 那行被註解掉了，代表目前不啟用自訂檔案處理邏輯。）

---

### 5. `inlet`（請求進入時的前處理）

```python
def inlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
```

這個方法在**使用者送出訊息、API 處理之前**被呼叫，相當於「入口守衛」。

執行流程：

1. 印出目前模組名稱、請求內容、使用者資訊（方便除錯）。
2. 檢查使用者角色是否為 `"user"` 或 `"admin"`。
3. 取出對話歷史 `messages`，計算目前已有幾輪。
4. 用 `min()` 取使用者設定與系統設定中較嚴格的上限。
5. 如果訊息數量超過上限，**拋出例外錯誤**，阻止這次請求繼續。
6. 若未超過，原封不動回傳 `body`，讓請求繼續送往 AI。

```python
max_turns = min(__user__["valves"].max_turns, self.valves.max_turns)
# 例如：使用者設定 4，系統設定 8 → 取 4，以較嚴格的為準

if len(messages) > max_turns:
    raise Exception(f"Conversation turn limit exceeded. Max turns: {max_turns}")
```

---

### 6. `outlet`（回應送出時的後處理）

```python
def outlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
```

這個方法在 **AI 回應產生後、送回給使用者之前**被呼叫，相當於「出口檢查」。

目前這個範例只做了：
- 印出模組名稱、回應內容、使用者資訊（方便除錯）。
- 原封不動回傳 `body`，不做任何修改。

實際應用中，你可以在這裡加入：記錄 log、過濾敏感詞、修改回應格式等邏輯。

---

## 整體運作流程圖

```
使用者送出訊息
      ↓
  inlet() 執行
  ├─ 檢查對話輪數
  ├─ 超過上限 → 拋出錯誤，對話中止
  └─ 未超過   → 繼續送給 AI
      ↓
   AI 產生回應
      ↓
  outlet() 執行
  └─ （目前）直接回傳給使用者
      ↓
使用者收到回應
```

---

## 重點觀念整理

- **Valves vs UserValves**：前者是管理員控制的全域設定，後者是使用者自己的設定，兩者取較小值確保安全。
- **inlet / outlet**：Open-WebUI 的 Filter 固定有這兩個鉤子（hook），分別在請求前後被呼叫。
- **`raise Exception`**：在 `inlet` 中拋出例外可以直接中斷請求，WebUI 會把錯誤訊息顯示給使用者。
