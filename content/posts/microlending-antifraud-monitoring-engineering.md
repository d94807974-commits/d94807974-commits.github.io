---
title: "小額信貸自動化風控的反詐欺監控：從環節攔截、Flink CEP 到 PSI 告警的工程實踐"
date: 2026-11-13T15:13:00+08:00
lastmod: 2026-11-13T14:12:00+08:00
draft: false
tags: ["風控", "反詐欺", "小額信貸", "Flink", "CEP", "PSI", "Prometheus", "Evidently", "開源授權", "模型監控"]
categories: ["工程實踐"]
description: "把小額信貸的反詐欺拆成兩條線：按業務環節攔截，以及持續監控規則、模型與資料是否失效。本文給出參考架構、Flink CEP 與 Flink SQL 的即時特徵範例、PSI 計算與 Prometheus 告警設定，並逐一核對各開源元件的授權。程式碼在本機實跑，法規條文逐條對照官方文本。"
summary: "反詐欺不是裝一套元件，而是把『環節攔截』和『持續監控』兩條線做成可驗證的工程資產：事件流、特徵、決策、標籤回流。本文附實跑的 Flink CEP / SQL 範例、PSI 程式與 promtool 驗證過的告警規則，以及元件授權查核表。資料核查：2026-11-13（Asia/Singapore）。"
ShowToc: true
---

> 📊 **圖解版**：[反詐欺全流程的攔截點、參考架構與監控分層](/visual/microlending-antifraud-monitoring-engineering.html)（單頁 HTML，由 answer-me-with-html 產生）

> [!VIEWPOINT] 核心觀點
> 小額信貸的反詐欺要拆成兩條線：一條在註冊、進件、審批、放款前、貸後、催收各環節攔截，一條持續監控規則、模型、資料來源與系統是否失效。工程上先把事件流、特徵、決策留痕和標籤回流做成閉環，再按授權條款選元件；PSI 只用來觸發調查，不能自動決定重訓或下線。

> **適用對象與邊界**：面向全流程線上、自動審批的網路小額貸款或助貸技術團隊。法規部分只引用本文已對照官方文本的條文；程式碼的實測範圍與未驗證項見文末〈本文邊界與未驗證項〉。查核日期 2026-11-13（Asia/Singapore）。本文不是法律意見。

很多團隊做反詐欺，是從「選一個規則引擎」開始的。但規則引擎只是決策鏈上的一個節點。上線後真正出問題的地方通常在別處：特徵在線上和離線算得不一樣，某條規則悄悄停止命中，三方資料介面逾時被預設值默默填充，模型分數分布慢慢漂移卻沒人發現。

本文從工程師視角把這條鏈走一遍：先看法規對工程有什麼硬約束，再看各環節在哪裡攔截，然後給出架構、即時特徵、PSI 與告警的可執行範例，最後比較元件和授權。

## 一、法規對工程的硬約束

先說明一個容易混淆的地方。網路小額貸款公司適用的是《小額貸款公司監督管理暫行辦法》（金規〔2024〕26號，金融監管總局 2024-12-31 印發，自印發之日起施行）。這份辦法**沒有**規定模型監控的細節，全文也沒有出現 PSI、人工複核或預警模型這些詞。模型有效性、穩定性、獨立評審和退出機制的具體要求，寫在《商業銀行互聯網貸款管理暫行辦法》（銀保監會令 2020 年第 9 號）裡。如果業務是與銀行聯合放款或助貸，銀行側的要求會傳導到你的系統。《國家金融監督管理總局關於加強商業銀行互聯網助貸業務管理提升金融服務質效的通知》（金規〔2025〕9號，自 2025-10-01 起施行）也明確：互聯網助貸屬於互聯網貸款，須遵守該辦法。

下表只列本文逐條對照過官方文本的條文，以及它們對工程的含義。

| 出處 | 條文要點 | 對工程的含義 |
|---|---|---|
| 26號 第九條 | 網路小貸的申請受理、風險審核、審批、發放和回收等核心環節應線上完成 | 每個環節都要有可程式化的決策點和完整留痕 |
| 26號 第十五條 | 不得將授信審查、風險控制等核心業務外包 | 決策服務、規則和特徵定義要自己掌握，三方只提供資料 |
| 26號 第二十三條 | 風險防控體系應包括風控模型、反詐欺系統、風險識別機制、風險監測手段、風險處置措施、客戶身份識別與登記系統等 | 監控與處置不是附加項，是體系的一部分 |
| 26號 第二十四條 | 逾期超過九十天的貸款劃分為不良貸款 | 貸後標籤要有一個與此對齊的定義，並與詐欺標籤分開 |
| 26號 第二十九條 | 互聯網業務系統能完整記錄並保存資料；等保定級不低於第三級 | 事件與決策日誌要可追溯，系統要通過等保評估 |
| 26號 第三十八條 | 收集、使用客戶資訊須取得授權；未經授權不得收集、使用、加工、傳輸 | 特徵採集前先檢查授權，授權範圍要能追溯到特徵 |
| 銀行辦法 第十八條 | 身份核驗不得全權委託合作機構 | 活體與聯網核查的結果要自己留存、自己校驗 |
| 銀行辦法 第二十二條 | 建立人工複核機制，作為模型自動審批的必要補充，並明確觸發條件 | 決策輸出要有「轉人工」分支和觸發原因 |
| 銀行辦法 第二十六條 | 授信與首筆放款間隔超過一個月，放款前須再評估信用狀況 | 放款前要重新跑一次決策，不能沿用授信結果 |
| 銀行辦法 第二十九條 | 建立風險監測預警模型，設定預警指標與觸發條件 | 預警指標要寫成可執行的告警規則 |
| 銀行辦法 第四十條 | 日常監測至少包括已上線模型的有效性與穩定性，以及模型審批貸款的實際違約情況 | PSI 之外要有 KS／AUC 與實際違約的對照 |
| 銀行辦法 第四十一條 | 無法繼續滿足要求的模型應立即停止使用 | 模型要有可一鍵停用的開關和備援策略 |

第二十三條有一句：「網路小額貸款公司建立的風險防控體系應當包括數據驅動的風控模型、反欺詐系統……」（引文由簡體轉為繁體，內容不變）。這句話的工程翻譯很直接：風控模型只是其中一項，監測和處置要一起建。

## 二、六個環節的攔截點

每個環節有各自的訊號，也有各自適合的計算方式。下表按「同步」「串流」「批次」區分落點，這決定了後面該用什麼元件。

| 環節 | 典型訊號 | 落點 | 說明 |
|---|---|---|---|
| 註冊／登入 | 設備指紋異常、同設備短時間多帳號註冊、撞庫式登入失敗 | 串流 | 視窗聚合，例如「同設備 10 分鐘內註冊帳號數」 |
| 進件申請 | 實名與活體、名單、多頭借貸、手機號與證件一致性 | 同步 | 申請時要即時回應，查徵信須先取得授權 |
| 審批授信 | 硬規則、反詐欺模型、關聯圖譜（共用設備、聯絡人、銀行卡） | 同步 | 輸出通過、拒絕或轉人工，並留存特徵快照 |
| 放款前 | 改卡、改手機號、換設備後立即提現 | 串流 | 事件序列，適合 CEP |
| 貸後 | 首期違約、按放款月份的 Vintage、集中逾期、失聯 | 批次 | 結果回流成標籤 |
| 催收 | 區分詐欺（冒用身份、團夥騙貸）與信用違約 | 批次 | 混用標籤會讓反詐欺模型學錯對象 |

這張表的重點在最後一欄：「同步」放在決策服務裡，「串流」放在 Flink，「批次」放在每日作業。把串流問題放進同步呼叫，會拖慢審批；把批次問題當即時問題做，成本高而且沒有必要。

## 三、參考架構與資料流

下圖是一條可以由小團隊落地的主鏈。進件系統同步呼叫決策服務，同時把事件寫進 Kafka。Flink 算視窗特徵和事件序列，特徵寫到線上特徵庫。決策服務讀特徵，串起規則引擎、模型和圖譜查詢，輸出決策並留痕。

```text
 [進件/登入/改卡/提現/還款 App]
      | sync request                         | events
      v                                      v
 [決策服務] <---- features ---- [特徵庫(online)] <---- [Flink: HOP window + CEP]
   |  rules / model / graph             ^                      ^
   |                                    |                      |
   |  decision + feature snapshot       |               [Kafka: risk.events]
   v                                    |
 [決策日誌 / 特徵快照] --> [貸後標籤(FPD, 逾期, 詐欺/信用違約)] --+--> 回流特徵與訓練庫
   |
   +--> [Prometheus: 耗時, 錯誤, 命中, 通過率] --> [告警]
   +--> [批次作業: PSI / KS / 缺失率] ----------> [告警]
```

資料流按四步走：

1. **事件進入**：App 與決策服務把註冊、登入、改卡、提現等事件寫進 `risk.events`，欄位用 `user_id`、`device_id`、`event_type`、`event_time`，不放明文證件號與手機號。
2. **即時特徵**：Flink 以事件時間算視窗特徵與序列命中，結果寫到另一個 Kafka 主題，再由消費端寫入線上特徵庫。
3. **決策與留痕**：決策服務讀取特徵，串行執行規則、模型和圖譜查詢，輸出通過、拒絕或轉人工，並把分數、命中規則和特徵快照落庫。
4. **標籤回流**：貸後與催收結果回流成標籤，區分詐欺與信用違約，供規則復盤和模型訓練使用。

特徵庫的角色是讓線上決策和離線訓練使用同一份特徵定義。Feast 官方有一份〈Fraud detection on GCP〉教程，涵蓋特徵回填、時間點正確的訓練資料集和低延遲線上預測。教程以 GCP 為例，自建環境要自行替換儲存。

## 四、即時特徵與事件序列：Flink 範例（實跑）

### 現象

放款前的詐欺常是一段行為序列：先改銀行卡，很快就提現。單看每個事件都正常，只有放在一起才可疑。同樣，批量註冊要看「同一設備在一段時間內出現幾個帳號」。

### 原因

這兩類訊號都依賴事件時間和狀態。如果在決策服務裡用資料庫即時查詢拼湊，延遲高、容易漏事件；用 Flink 的視窗與 CEP，狀態和逾時由引擎管理。

### 做法

**範例一：用 Flink CEP（DataStream API）判斷「改卡後 10 分鐘內提現」。** 以下程式在本機用 Flink 2.3.0 的 MiniCluster 實跑通過，使用 `flink-streaming-java`、`flink-clients`、`flink-cep` 三個依賴。

```java
Pattern<LoanEvent, ?> pattern = Pattern
    .<LoanEvent>begin("card", AfterMatchSkipStrategy.skipPastLastEvent())
    .where(SimpleCondition.of(e -> "CARD_CHANGED".equals(e.eventType)))
    .followedBy("withdraw")
    .where(SimpleCondition.of(e -> "WITHDRAW".equals(e.eventType)))
    .within(Duration.ofMinutes(10));

DataStream<String> alerts = CEP
    .pattern(events.keyBy(e -> e.userId), pattern)
    .select(new PatternSelectFunction<LoanEvent, String>() {
        @Override
        public String select(Map<String, List<LoanEvent>> match) {
            LoanEvent card = match.get("card").get(0);
            LoanEvent withdraw = match.get("withdraw").get(0);
            long gapSeconds = (withdraw.eventTime - card.eventTime) / 1000;
            return "ALERT user=" + card.userId + " gap_seconds=" + gapSeconds;
        }
    });
```

事件來源要先指定事件時間與水位線，否則視窗和 `within` 不會觸發：

```java
.assignTimestampsAndWatermarks(
    WatermarkStrategy.<LoanEvent>forBoundedOutOfOrderness(Duration.ofSeconds(5))
        .withTimestampAssigner((e, ts) -> e.eventTime));
```

測試資料有 4 個使用者：u1 改卡後 3 分鐘提現，u4 改卡後夾著一個其他事件、9 分鐘提現，u2 在 30 分鐘後才提現，u3 沒有改卡。實跑輸出只有 u1（180 秒）和 u4（540 秒）兩條告警。`followedBy` 允許中間夾其他事件；如果要求緊接著發生，要改用 `next`。Flink 官方文件也建議用 `within()` 或 `until()` 讓引擎清理狀態，否則未完成的部分匹配會一直佔用狀態。

**範例二：用 Flink SQL 算視窗特徵與序列。** 下面兩段 SQL 同樣在 Flink 2.3.0 MiniCluster 上跑過。我把資料以 DataStream 轉成臨時檢視 `risk_events`，事件時間欄位為 `rowtime`。

```sql
-- feature: distinct users registered per device in a 10-minute window, sliding every minute
SELECT deviceId, window_start, window_end, COUNT(DISTINCT userId) AS reg_users_10m
FROM TABLE(HOP(TABLE risk_events, DESCRIPTOR(rowtime),
               INTERVAL '1' MINUTES, INTERVAL '10' MINUTES))
WHERE eventType = 'REGISTER'
GROUP BY deviceId, window_start, window_end
HAVING COUNT(DISTINCT userId) >= 3;
```

測試資料中設備 `d1` 在 6 分鐘內有 4 個不同帳號註冊，輸出 8 個視窗，帳號數為 3 或 4；另一台只有 1 個帳號的設備沒有輸出。`HAVING` 的 3 是示範值，真實閾值要用自己的歷史資料校準。

```sql
-- sequence: card changed, then withdraw within 10 minutes (same user)
SELECT * FROM risk_events MATCH_RECOGNIZE (
  PARTITION BY userId ORDER BY rowtime
  MEASURES C.rowtime AS card_changed_at, W.rowtime AS withdraw_at
  ONE ROW PER MATCH
  AFTER MATCH SKIP PAST LAST ROW
  PATTERN (C X*? W) WITHIN INTERVAL '10' MINUTE
  DEFINE C AS C.eventType = 'CARD_CHANGED',
         W AS W.eventType = 'WITHDRAW'
);
```

這段只對 u1 輸出一筆（30 分鐘後才提現的 u6 沒有命中）。`X*?` 是懶惰量詞，作用是遇到第一個提現就結束匹配；寫成貪婪的 `X*` 會吞掉提現事件。

**正式環境的 Kafka 來源與輸出表**要寫成 DDL。我另建一個專案，用 Flink 2.2.0 加 `flink-sql-connector-kafka` 5.0.0-2.2 驗證 DDL 與 `INSERT INTO ... SELECT` 能通過規劃（`EXPLAIN`），但**沒有連真實 Kafka 叢集做端到端驗證**：

```sql
CREATE TABLE risk_events (
  user_id STRING, device_id STRING, event_type STRING, event_time TIMESTAMP_LTZ(3),
  WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
  'connector' = 'kafka', 'topic' = 'risk.events',
  'properties.bootstrap.servers' = '<kafka>:9092',
  'properties.group.id' = 'risk-features',
  'scan.startup.mode' = 'latest-offset', 'format' = 'json'
);

CREATE TABLE device_features_10m (
  device_id STRING, window_start TIMESTAMP(3), window_end TIMESTAMP(3), reg_users_10m BIGINT
) WITH (
  'connector' = 'kafka', 'topic' = 'risk.features.device_10m',
  'properties.bootstrap.servers' = '<kafka>:9092', 'format' = 'json'
);
```

這裡有兩個容易踩的坑：

- **連接器版本跟著 Flink 次版本走。** 查核日期，Maven Central 上 `flink-sql-connector-kafka` 最新版本是 5.0.0-2.1 與 5.0.0-2.2，沒有對應 Flink 2.3 的版本。升級 Flink 前先確認連接器有對應版本。
- **叢集上要有 `flink-connector-base`。** 我在本機 classpath 少了它，會出現 `ClassNotFoundException: DeliveryGuarantee`。標準 Flink 發行版已附帶，只有自己組 classpath 時才會遇到。

## 五、監控分五層

反詐欺上線後，監控要覆蓋五個層次。每層看什麼、怎麼告警，下表按「層次、關鍵指標、怎麼看、性質」列出。數值閾值除特別註明外，都是示範值，要用自己的歷史基線校準。

| 層次 | 關鍵指標 | 怎麼看 | 性質 |
|---|---|---|---|
| 業務 | 通過率、拒絕率、FPD、Vintage | 按通路、時段、地域拆開，與自身歷史基線比較 | 操作建議；本文所查官方文本沒有統一法定閾值 |
| 模型 | PSI、KS／AUC、實際違約 | PSI 看漂移，KS／AUC 看排序能力，兩者分開判讀 | 銀行辦法第四十條要求監測有效性、穩定性與實際違約 |
| 規則 | 每條規則的命中率與命中後表現 | 命中突然變多或降到零，都可能是失效 | 操作建議 |
| 資料來源 | 特徵缺失率、三方逾時率、欄位分布 | 按供應商與介面分別看，介面失敗被預設值填充要能被發現 | 操作建議 |
| 系統 | 決策耗時 p99、錯誤率、降級次數 | 降級要可追溯 | 操作建議；等保要求見第一節 |

「FPD」是首期違約，「Vintage」是按放款月份分組、隨帳齡觀察逾期率的曲線。FPD 高不等於詐欺，因為詐欺濃度只是通常更高，標籤仍要靠催收和調查區分。

## 六、PSI：計算、陷阱與工具對照

### 現象

分數分布漂移時，通過率和違約率要幾個月後才反映出來。PSI（Population Stability Index，人群穩定性指數）可以更早發現：它比較當前樣本與建模基準在各分箱上的佔比差異。

```text
PSI = sum over bins i of (A_i - E_i) * ln(A_i / E_i)
A_i = share of the current sample in bin i
E_i = share of the baseline sample in bin i
```

### 原因

PSI 的結果同時取決於分箱方式和空箱處理。同一組資料換個分箱，數字就不一樣；某個分箱在一邊為空，公式裡會出現 ln(0)。

### 做法

下面的 Python 用基準樣本的分位數切箱，兩端設為無窮大以接住超出範圍的值，空箱用一個下限值 `eps` 保護：

```python
import numpy as np

EPS = 1e-4  # floor for empty bins; without it ln(0) breaks the formula


def psi_numeric(expected, actual, bins=10, eps=EPS):
    """PSI of `actual` against `expected` (baseline). Bin edges come from baseline quantiles."""
    expected = np.asarray(expected, dtype=float)
    actual = np.asarray(actual, dtype=float)
    edges = np.unique(np.quantile(expected, np.linspace(0, 1, bins + 1)))
    edges[0], edges[-1] = -np.inf, np.inf  # catch values outside the baseline range
    e_cnt, _ = np.histogram(expected, bins=edges)
    a_cnt, _ = np.histogram(actual, bins=edges)
    e_pct = np.maximum(e_cnt / e_cnt.sum(), eps)
    a_pct = np.maximum(a_cnt / a_cnt.sum(), eps)
    return float(np.sum((a_pct - e_pct) * np.log(a_pct / e_pct)))
```

**實測（合成資料，固定亂數種子 42，numpy）**：基準為平均 600、標準差 50 的常態分布，各 5 萬筆。

| 情境 | PSI | 依經驗區間的讀法 |
|---|---|---|
| 同分布、不同樣本 | 0.0001 | 穩定 |
| 平均值 +10 | 0.0371 | 穩定 |
| 平均值 +40、標準差 60 | 0.5156 | 明顯偏移 |
| 當前樣本在最高十分位沒有值，`eps=1e-4` | 0.7572 | — |
| 同上，`eps=1e-8` | 1.6789 | — |

最後兩列是重點：**空箱時，PSI 的大小由你選的 `eps` 決定**，同一批資料可以得到 0.76 或 1.68。所以空箱要單獨告警，並在報告裡記錄使用的 `eps`，不要只看一個總數。

**和 Evidently 對照。** Evidently 0.7.23 可以用 `DataDriftPreset(method="psi")` 計算，預設閾值是 PSI ≥ 0.1 判為漂移（官方文件的預設值，不是統計結論）：

```python
from evidently import DataDefinition, Dataset, Report
from evidently.presets import DataDriftPreset

definition = DataDefinition(numerical_columns=["score"], categorical_columns=["channel"])
reference = Dataset.from_pandas(ref_df, data_definition=definition)
current = Dataset.from_pandas(cur_df, data_definition=definition)

snapshot = Report([DataDriftPreset(method="psi")]).run(current, reference)  # run(current, reference)
for m in snapshot.dict()["metrics"]:
    print(m["metric_name"], m["value"])
```

用同一組合成資料（各 5000 筆，基準平均 600、標準差 50；當前平均 640、標準差 60；類別特徵三個類別，佔比從 0.5／0.3／0.2 變成 0.3／0.3／0.4），結果如下：

| 特徵 | 本文程式 | Evidently |
|---|---|---|
| 類別特徵 `channel` | 0.2652 | 0.2652 |
| 數值特徵 `score` | 0.5177 | 0.6010 |

類別特徵完全一致；數值特徵差了約 0.08，因為兩邊的分箱方式不同。這說明：**跨工具比較 PSI 時，先固定分箱規則**，否則兩個數字不能直接比。另外，我的環境裡 Evidently 0.7.23 搭配 pandas 3.1.1 匯入就失敗（`ImportError: infer_dtype_from_object`），降到 pandas 2.3.3 後正常。安裝時要固定 pandas 版本。

**閾值怎麼用。** 常見的經驗區間是小於 0.10 穩定、0.10 到 0.25 需要調查、大於等於 0.25 為明顯變化。arXiv 上有一篇對 PSI 檢定程序的評述（見參考），指出這類固定臨界值沒有考慮樣本量和分箱數。所以把超過閾值當成「開始調查」的信號：先拆通路佔比、檢查三方欄位，再看 KS／AUC 與 Vintage，然後才決定是否重訓。不要讓告警自動觸發重訓或下線。

## 七、把 PSI 與業務指標接進 Prometheus

### 現象

PSI 是每日批次算出來的，決策耗時和通過率是即時產生的。如果兩套指標分在不同的地方，值班的人要開幾個頁面才能判斷問題出在哪。

### 做法

PSI 作業算完後，用 `prometheus_client` 把結果寫成文字檔，由 node_exporter 的 textfile collector 收集。核心是三個指標：每個特徵的 PSI、最後一次成功執行的時間、特徵缺失率。

```python
from prometheus_client import CollectorRegistry, Gauge, write_to_textfile

registry = CollectorRegistry()
psi_gauge = Gauge("risk_model_psi", "PSI against the training baseline",
                  ["model", "feature"], registry=registry)
last_run = Gauge("risk_psi_last_run_timestamp_seconds", "Unix time of the last successful PSI run",
                 ["model"], registry=registry)
missing_ratio = Gauge("risk_feature_missing_ratio", "Share of null or default-filled values",
                      ["feature"], registry=registry)

psi_gauge.labels(model="fraud_v1", feature="__score__").set(0.18)
last_run.labels(model="fraud_v1").set_to_current_time()
missing_ratio.labels(feature="vendor_a_score").set(0.02)
write_to_textfile("/var/lib/node_exporter/textfile/risk_psi.prom", registry)
```

實際輸出是標準的 Prometheus 文字格式，例如 `risk_model_psi{feature="__score__",model="fraud_v1"} 0.18`。決策服務自己另外暴露四個指標：`risk_decisions_total{decision,channel}`、`risk_rule_hits_total{rule_id}`、`risk_vendor_requests_total{vendor,result}` 與直方圖 `risk_decision_duration_seconds`。

告警規則如下。我用 `promtool check rules` 檢查語法，並用 `promtool test rules` 對 PSI 兩級告警與「作業過期」做了單元測試，三項都通過。

```yaml
groups:
  - name: risk-recording
    rules:
      - record: risk:approval_ratio:rate1h
        expr: |
          sum(rate(risk_decisions_total{decision="approve"}[1h]))
          / sum(rate(risk_decisions_total[1h]))

  - name: risk-model
    rules:
      - alert: RiskModelPsiInvestigate
        expr: risk_model_psi >= 0.10 and risk_model_psi < 0.25
        for: 1h
        labels: {severity: warning}
      - alert: RiskModelPsiShifted
        expr: risk_model_psi >= 0.25
        for: 1h
        labels: {severity: critical}
        annotations:
          description: "Large shift. Open a model review; do not retrain automatically."
      - alert: RiskPsiJobStale
        expr: time() - risk_psi_last_run_timestamp_seconds > 36 * 3600
        labels: {severity: warning}

  - name: risk-business
    rules:
      - alert: RiskApprovalRatioShift
        # illustrative threshold: calibrate on your own history
        expr: abs(risk:approval_ratio:rate1h / (risk:approval_ratio:rate1h offset 7d) - 1) > 0.3
        for: 30m
        labels: {severity: warning}
      - alert: RiskRuleStoppedHitting
        expr: |
          sum by (rule_id) (rate(risk_rule_hits_total[1h])) == 0
          and
          sum by (rule_id) (rate(risk_rule_hits_total[1h] offset 1d)) > 0
        for: 1h
        labels: {severity: warning}

  - name: risk-data-and-system
    rules:
      - alert: RiskFeatureMissingHigh
        expr: risk_feature_missing_ratio > 0.2
        for: 15m
      - alert: RiskVendorTimeouts
        expr: |
          sum by (vendor) (rate(risk_vendor_requests_total{result="timeout"}[10m]))
          / sum by (vendor) (rate(risk_vendor_requests_total[10m])) > 0.05
        for: 10m
        labels: {severity: critical}
      - alert: RiskDecisionLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(risk_decision_duration_seconds_bucket[5m]))) > 0.5
        for: 5m
        labels: {severity: critical}
```

幾個設計理由：

- **`RiskPsiJobStale` 不能少。** PSI 作業失敗時，指標會停在舊值，看起來「很穩定」。用最後成功時間加過期告警，才能發現作業本身壞了。
- **通過率用「同一時段上週」比較**（`offset 7d`），避開日內與週內的自然波動。0.3 只是示範值。
- **規則命中數降到零也要告警。** 特徵缺失或規則被繞過，都會讓命中悄悄歸零，而且不會報錯。
- 需要注意：這份規則裡 `RiskApprovalRatioShift`、`RiskRuleStoppedHitting` 與延遲、逾時等規則，我只做了語法檢查，**沒有用歷史資料驗證閾值**。

## 八、元件選型與授權

選型的第一個問題常常不是功能，而是授權。下表中的授權都是 2026-11-13 從官方倉庫的 LICENSE 檔、README 或官方網站讀取的結果。

| 元件 | 用途 | 授權（開源版） | 版本差異與注意 | 查核來源 |
|---|---|---|---|---|
| Apache Flink | 即時計算、CEP、SQL | Apache-2.0 | 連接器版本隨 Flink 次版本走（見第四節） | GitHub `apache/flink` 倉庫授權欄位 |
| Apache Kafka | 事件匯流排 | Apache-2.0 | — | GitHub `apache/kafka` 倉庫授權欄位 |
| Drools | 規則引擎、DMN、CEP | Apache-2.0 | 現屬 Apache KIE（孵化中）專案，倉庫為 `apache/incubator-kie` | kie.apache.org/drools；倉庫 LICENSE 與 DISCLAIMER-WIP |
| URule | 中文視覺化規則引擎 | 開源版 Apache-2.0；另有商用 Pro 版 | README 標示交叉決策表與複雜評分卡僅 Pro 版；倉庫最後推送時間為 2022-12-10 | github.com/youseries/urule README |
| Nussknacker | 即時決策視覺化平臺（跑在 Flink／Kafka 上） | 核心 Apache-2.0；另有付費方案 | 官方定價頁列出付費方案含 AI／ML 元件與標準支援；Designer 的 Integrations 介面僅 Cloud 與 Enterprise 提供，開源版用設定檔 | 倉庫 README、docs.nussknacker.io、nussknacker.io/pricing |
| Feast | 特徵庫 | Apache-2.0 | — | GitHub `feast-dev/feast` LICENSE；PyPI |
| NebulaGraph | 圖資料庫 | Apache-2.0（開源版） | Enterprise 需授權；Explorer 僅 Enterprise 提供 | 倉庫 README；官方手冊「About NebulaGraph licenses」 |
| Neo4j | 圖資料庫 | Community 為 GPL-3.0 | Enterprise 含閉源元件，需商業授權 | 倉庫 README 〈Licensing〉 |
| toad | Python 評分卡工具（分箱、WOE、PSI） | MIT | — | GitHub `amphibian-dev/toad` LICENSE |
| Evidently | 資料與模型漂移報告 | Apache-2.0 | PyPI 0.7.23；與 pandas 3.x 不相容（本文實測） | GitHub LICENSE；PyPI |
| Great Expectations（GX Core） | 資料品質校驗 | Apache-2.0 | 倉庫現在轉址到 `fivetran/great_expectations`（查核日期） | 倉庫 LICENSE；PyPI |
| Prometheus | 指標與告警 | Apache-2.0 | — | GitHub `prometheus/prometheus` 倉庫授權欄位 |
| Grafana | 看板 | AGPL-3.0-only | 官方說明部分元件有 Apache-2.0 例外，見其 LICENSING.md | github.com/grafana/grafana README；grafana.com/licensing |
| Apache Superset | 業務報表 | Apache-2.0 | 可作為不想引入 AGPL 時的替代 | GitHub `apache/superset` 倉庫授權欄位 |

三條選型取捨：

- **GPL 與 AGPL 要先交給法務。** Neo4j Community 是 GPL-3.0，Grafana 是 AGPL-3.0-only。是否影響你的閉源部署，取決於你怎麼使用、是否修改與分發，本文不下結論。想避開這個問題，圖資料庫可以先評估 NebulaGraph 開源版，報表可以先評估 Superset。
- **開源版和付費版的功能邊界要寫進選型文件。** URule 的複雜評分卡、Nussknacker 的 Integrations 介面與 AI／ML 元件、NebulaGraph 的 Explorer，都不在開源版裡。把付費功能當成預設可用，是常見的選型失誤。
- **維護狀態要看提交時間。** URule 開源倉庫最後一次推送是 2022 年 12 月。這不代表它不能用，但意味著安全補丁和新版本相容性要由你自己負責。

## 九、分階段落地

按能力推進，而不是按元件堆疊：

1. **合規底線**：身份核驗、授信審批和模型管理由自己掌握；特徵採集前檢查授權；事件與決策完整留痕。
2. **硬規則、名單與基礎看板**：用規則引擎配置黑名單、多頭、身份一致性、設備異常；用 Prometheus 看決策耗時與通過率、規則命中。這一階段先不上複雜模型。
3. **貸後指標閉環**：統一 FPD、Vintage 與失聯的定義；催收端把詐欺與信用違約分開打標。
4. **有足夠標籤再上模型**：用 toad 等工具做可解釋評分卡；建立獨立評審；每日跑 PSI，並同步看 KS／AUC 與實際違約。
5. **即時特徵、CEP 與圖譜**：量上來之後再上 Kafka、Flink 與特徵庫，再評估圖譜。

## 本文邊界與未驗證項

**已實測（本機）**

- Flink 2.3.0 MiniCluster：CEP 範例、HOP 視窗 SQL、`MATCH_RECOGNIZE`，資料為本文自造的小型測試集。
- Flink 2.2.0 加 `flink-sql-connector-kafka` 5.0.0-2.2：Kafka DDL 與 `INSERT INTO ... SELECT` 的 `EXPLAIN` 規劃通過。
- PSI 程式與 Evidently 0.7.23 的計算（合成資料，種子 42）。
- Prometheus 3.16.0 的 `promtool check rules`，以及 PSI 兩級告警與作業過期告警的 `promtool test rules`。

**文件依據**

- 法規條文：對照中國政府網上的 26號全文與銀行辦法全文。銀行辦法經 2021 年清理規章的決定修訂過第二十三條（「合同法」改「民法典」），與本文引用的條文無關。
- 授權：讀自各官方倉庫的 LICENSE、README 與官方網站，不構成法律意見。

**未驗證項**

- 沒有連真實 Kafka 叢集做端到端實測；沒有做效能、吞吐、狀態大小或 checkpoint 的測試。
- CEP 的亂序、遲到事件與部分匹配逾時處理（`TimedOutPartialMatchHandler`）沒有實測，正式使用前要補測。
- 沒有把文字檔交給真實的 node_exporter 與 Prometheus 伺服器抓取，只檢查了輸出格式與規則本身。
- 通過率偏移、規則命中歸零、缺失率、供應商逾時與延遲等告警規則只做了語法檢查，閾值都是示範值。
- PSI 的 0.10／0.25 經驗區間來自評分卡文獻的慣用說法，本文沒有查到原書頁碼；只引用了 arXiv 評述中「臨界值不考慮樣本量與分箱數」的結論。
- 金規〔2025〕9號只核對了標題、發文日期、施行日期與第一條「互聯網助貸屬於互聯網貸款」，沒有逐條對照。
- 沒有評估各元件的社群活躍度、漏洞公告與維運成本；圖譜、特徵庫沒有實測。
- 是否適用 26號、銀行辦法或其他規則，取決於你的牌照與業務模式，要由法務判斷。
- 授權與版本會變，落地前請再讀一次原文。

## 參考

法規與監管文件（中國政府網）：

- [小額貸款公司監督管理暫行辦法（金規〔2024〕26號，國務院公報 2025 年第 8 號）](https://www.gov.cn/gongbao/2025/issue_11926/202503/content_7013993.html)
- [商業銀行互聯網貸款管理暫行辦法（銀保監會令 2020 年第 9 號）](https://www.gov.cn/zhengce/2020-07/17/content_5718706.htm)
- [中國銀保監會關於清理規章規範性文件的決定（2021 年第 7 號令）](https://www.gov.cn/zhengce/zhengceku/2021-07/02/content_5622002.htm)
- [國家金融監督管理總局關於加強商業銀行互聯網助貸業務管理提升金融服務質效的通知（金規〔2025〕9號）](https://www.gov.cn/zhengce/zhengceku/202504/content_7017135.htm)

技術文件：

- [Apache Flink：Event-driven Applications](https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/event_driven/)
- [Apache Flink：FlinkCEP](https://nightlies.apache.org/flink/flink-docs-stable/docs/libs/cep/)
- [Apache Flink：Pattern Recognition（MATCH_RECOGNIZE）](https://nightlies.apache.org/flink/flink-docs-stable/docs/sql/reference/queries/match_recognize/)
- [Apache Flink：Windowing TVF](https://nightlies.apache.org/flink/flink-docs-stable/docs/sql/reference/queries/window-tvf/)
- [Apache Flink：Kafka SQL Connector](https://nightlies.apache.org/flink/flink-docs-stable/docs/connectors/table/kafka/)
- [Feast：Fraud detection on GCP](https://docs.feast.dev/tutorials/tutorials-overview/fraud-detection)
- [Evidently：Customize Data Drift（含 PSI 預設閾值）](https://docs.evidentlyai.com/metrics/customize_data_drift)
- [Prometheus：Unit testing for rules](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/)
- [A critical review of existing and new population stability testing procedures in credit risk scoring（arXiv:2303.01227）](https://arxiv.org/abs/2303.01227)

元件授權來源：

- [Apache KIE Drools](https://kie.apache.org/drools/)、[URule](https://github.com/youseries/urule)、[Nussknacker](https://github.com/TouK/nussknacker)、[Nussknacker 定價與版本比較](https://nussknacker.io/pricing/)、[Nussknacker 開源版文件](https://docs.nussknacker.io/OSS_docs/)
- [NebulaGraph](https://github.com/vesoft-inc/nebula)、[NebulaGraph 授權說明](https://docs.nebula-graph.io/3.4.4/20.appendix/8.about-license/)、[Neo4j](https://github.com/neo4j/neo4j)
- [toad](https://github.com/amphibian-dev/toad)、[Great Expectations（fivetran/great_expectations）](https://github.com/fivetran/great_expectations)、[Grafana 授權說明](https://grafana.com/licensing/)、[Grafana LICENSING.md](https://github.com/grafana/grafana/blob/HEAD/LICENSING.md)
