---
title: "開源即時 CDC 同步選型：帶 Web 設定、MySQL/SQL Server→Kafka/Doris/ClickHouse、全量+增量（2026-11 核實）"
url: https://d94807974-commits.github.io/posts/realtime-cdc-sync-opensource-comparison/
markdown_url: https://d94807974-commits.github.io/posts/realtime-cdc-sync-opensource-comparison/index.md
date: 2026-11-05T13:52:00+08:00
lastmod: 2026-11-05T14:45:00+08:00
author: "Donald"
language: zh-TW
summary: "按「Web 可配 + MySQL/SQL Server 源 + Kafka/Doris/ClickHouse 目標 + 全量+增量 + star\u003e1k + 開源」條件核實候選；對比 SeaTunnel(+Web)、InLong、Flink CDC(+Dinky/StreamPark)、TIS，並特別原始碼走讀 DBSyncer 2.1.6（star 954，功能匹配但未過 1k 門檻）及衍生倉 Zhh9126/DBSYNC。資料核實：2026-11-05（Asia/Singapore）。"
tags: ["CDC","資料同步","SeaTunnel","Flink CDC","InLong","Doris","ClickHouse","Kafka","DBSyncer","開源選型","資料整合"]
---

# 開源即時 CDC 同步選型：帶 Web 設定、MySQL/SQL Server→Kafka/Doris/ClickHouse、全量+增量（2026-11 核實）


> 📊 **圖解版**：[目標鏈路、全量切增量流程與橫向對比](https://d94807974-commits.github.io/visual/realtime-cdc-sync-opensource-comparison.html)（單頁 HTML，由 answer-me-with-html 產生）

> [!VIEWPOINT] 核心觀點
> 要用一條帶 Web 設定的鏈路把 MySQL 與 SQL Server 全量加增量同步到 Kafka、Doris、ClickHouse，優先選 SeaTunnel 加 SeaTunnel Web；已有 Flink 平臺就選 Flink CDC 搭配 Dinky 或 StreamPark。DBSyncer 功能吻合但 star 未過 1k，只適合能接受自行加固的單機場景。

> **調研邊界（請先讀）**：本文只討論同時滿足下列條件的開源專案——① 有 Web 頁面可配置任務；② 源端至少覆蓋 **MySQL 與 SQL Server**；③ 目標端覆蓋 **Kafka、Apache Doris、ClickHouse**（或可明確組合達成）；④ 支援 **全量 + 增量**（CDC / binlog / CDC 日誌類，而非僅定時 JDBC 批抽）；⑤ GitHub **stargazers_count > 1000**；⑥ OSI/ASF 友好開源許可（Apache-2.0 等）。Star、license、最近 push 一律以 `gh api repos/<owner>/<repo>` 在 **2026-11-05** 拉取為準；功能以官方文件頁面為準，**不憑記憶斷言**。未能開啟或無法交叉驗證的點，會在文末「未能核實」列出。

<figure>
<img src="https://d94807974-commits.github.io/images/realtime-cdc-sync-opensource-comparison/seatunnel-website-logo.jpg" alt="Apache SeaTunnel 官網 Logo" loading="lazy" width="720">
<figcaption>圖片：Apache SeaTunnel 官網 <a href="https://seatunnel.apache.org/image/logo.png">logo.png</a>；專案許可證 <a href="https://github.com/apache/seatunnel/blob/dev/LICENSE">Apache-2.0</a>。已壓縮並本地託管於本站 <code>static/images/…</code>。</figcaption>
</figure>

## 一、問題怎麼拆

業務側常見訴求可以寫成一條鏈路：

```
[OLTP] MySQL / SQL Server
        |  (snapshot 全量 + binlog/CDC 增量)
        v
   同步引擎 / 平臺 (+ Web 配置)
        |
        +--> Kafka（匯流排 / 解耦）
        +--> Apache Doris（分析型 OLAP）
        +--> ClickHouse（列存分析）
```

工程上真正卡人的通常不是「能不能連上」，而是：

1. **有沒有 Web**：維運/資料同學是否必須手寫 YAML / Flink SQL。
2. **SQL Server 是否一等公民**：很多「國產 CDC」其實只做深了 MySQL。
3. **三目標是否原生 sink**：Kafka / Doris / ClickHouse 缺一就會被迫疊兩套棧。
4. **全量切增量是否平滑**：`startup.mode=initial`、checkpoint、exactly-once、DDL。
5. **許可證與 star**：商業分發、二次開發、社群活躍度。

跨地域補充一句：若同步鏈路兩端跨大洲（例如控制面/作業在新加坡、Doris 在墨西哥），RTT 與跨洋頻寬會放大 checkpoint 與 Stream Load 延遲；選型時優先選**可把 sink 寫路徑貼近目標叢集**、或**中間以同區域 Kafka 解耦**的架構，而不是只看功能 checklist。

## 二、入選專案（滿足或「組合滿足」）

下表 star / license / 最近 push 均為 **2026-11-05** 經 GitHub REST API 核實（UTC 時間已換算備註）。

**表 1｜專案概況：倉庫、star、授權、活躍度與 Web 介面**

| 方案 | 倉庫 | Stars | License | 最近 push (UTC) | Web |
| ------ | ------ | ------: | --------- | ----------------- | ----- |
| SeaTunnel + SeaTunnel Web | [apache/seatunnel](https://github.com/apache/seatunnel) + [apache/seatunnel-web](https://github.com/apache/seatunnel-web) | **9788** / **888** | Apache-2.0 | 2026-11-04 / 2026-01-22 | ✅ Web（獨立倉） |
| Apache InLong | [apache/inlong](https://github.com/apache/inlong) | **1507** | Apache-2.0 | 2026-09-16 | ✅ Dashboard |
| Flink CDC + Dinky | [apache/flink-cdc](https://github.com/apache/flink-cdc) + [DataLinkDC/dinky](https://github.com/DataLinkDC/dinky) | **6492** / **3779** | Apache-2.0 | 2026-11-05 / 2026-10-22 | ⚠️ CDC 本身無 Web；Dinky 提供 |
| Flink CDC + StreamPark | 同上 + [apache/streampark](https://github.com/apache/streampark) | **6492** / **4346** | Apache-2.0 | 同上 / 2026-10-31 | ⚠️ 同上，StreamPark 管 Flink 作業 |
| TIS | [datavane/tis](https://github.com/datavane/tis) | **1440** | Apache-2.0 | 2026-11-04 | ✅ Console |

**表 2｜能力對比：來源端、目標端與全量+增量支援**

| 方案 | MySQL | SQL Server | Kafka | Doris | ClickHouse | 全量+增量 |
| ------ | ------- | ------------ | ------- | ------- | ------------ | ----------- |
| SeaTunnel + SeaTunnel Web | ✅ MySQL-CDC | ✅ SqlServer-CDC | ✅ | ✅ | ✅ | ✅ `startup.mode=initial` |
| Apache InLong | ✅ Extract | ✅ Extract | ✅ Load | ✅ Load | ✅ Load | ✅ 同步場景 |
| Flink CDC + Dinky | ✅ | ✅ Pipeline 3.7 | ✅ Pipeline sink | ✅ Pipeline sink | ⚠️ Pipeline 無官方 CH sink，需 Flink SQL/其他 | ✅ |
| Flink CDC + StreamPark | ✅ | ✅ | ✅ | ✅ | ⚠️ 同上 | ✅ |
| TIS | ✅ Flink-CDC + DataX | ✅ Flink-CDC + DataX | ✅ | ✅ | ✅ | ✅ 批次 DataX + 即時 Flink-CDC |

> **關於 SeaTunnel Web 888★**：單獨未過 1k，但它是 ASF 官方 Web 控制台，與引擎同屬 SeaTunnel 生態；本文按「引擎 star>1k + 官方 Web」入選，並在對比裡標明 Web 倉活躍度（最近 push 停在 2026-01，需關注版本繫結）。

### 1. Apache SeaTunnel（+ SeaTunnel Web）

**能力依據（文件）**：

- MySQL CDC：支援 snapshot + 增量，`startup.mode` 含 `initial`（先歷史後增量），可選 exactly-once、schema-changes。[MySQL-CDC 文件](https://seatunnel.apache.org/docs/2.3.11/connector-v2/source/MySQL-CDC/)
- SQL Server CDC：[SqlServer-CDC](https://seatunnel.apache.org/docs/2.3.11/connector-v2/source/SqlServer-CDC/)
- Sink： [Kafka](https://seatunnel.apache.org/docs/2.3.11/connector-v2/sink/Kafka/)、[Doris](https://seatunnel.apache.org/docs/2.3.11/connector-v2/sink/Doris/)、[Clickhouse](https://seatunnel.apache.org/docs/2.3.11/connector-v2/sink/Clickhouse/) 均有獨立文件頁（HTTP 301→正式路徑，已確認存在）。
- Web： [seatunnel-web README](https://github.com/apache/seatunnel-web/blob/main/README.md) 描述為管理指令碼、資料來源、任務並提交到 Zeta Engine；預設 UI `http://127.0.0.1:8801/ui/`（admin/admin）。**Web 與引擎版本強繫結**，部署要求客戶端與 Server 同 OS/同目錄結構。

**架構示意**：

```
Browser ──► SeaTunnel Web (8801)
                │  submit job
                v
         SeaTunnel Zeta Cluster
      (MySQL-CDC / SqlServer-CDC)
                │
        ┌───────┼────────┐
        v       v        v
     Kafka    Doris   ClickHouse
```

**適合**：希望「一個引擎覆蓋三目標」、團隊接受 Zeta/Flink 引擎維運、中文文件友好。  
**坑**：Web 倉 star 與活躍度弱於引擎；外掛 jar / `plugin-mapping.properties` / `SEATUNNEL_HOME` 配置易踩坑；CDC 權限（REPLICATION）與 GTID/binlog 格式要按文件開。

### 2. Apache InLong

<figure>
<img src="https://d94807974-commits.github.io/images/realtime-cdc-sync-opensource-comparison/inlong-architecture.jpg" alt="Apache InLong 標準架構圖" loading="lazy" width="900">
<figcaption>圖片：Apache InLong 官方文件架構圖 <a href="https://inlong.apache.org/img/inlong-structure-en.png">inlong-structure-en.png</a>（見 <a href="https://inlong.apache.org/docs/introduction/">Introduction</a>）；Apache-2.0。本地壓縮託管。</figcaption>
</figure>

**能力依據**：[Introduction - Supported Data Nodes](https://inlong.apache.org/docs/introduction/) 明確列出：

- Extract：MySQL、**SQLServer**（2012–2019）等  
- Load：**Kafka、Doris、ClickHouse** 等  
- 模組含 **inlong-dashboard**（前端管控）、Manager、Sort、Agent 等  

最新 release 標籤 API 見 `2.4.0-RC0`（2026-08-21）；倉庫 push 至 2026-09-16，活躍度中等。

**適合**：需要「接入 + 同步 + 訂閱」一體、已有或願意上 TubeMQ/Pulsar/Kafka 作為中間緩衝、偏騰訊系大數據棧。  
**坑**：完整架構元件多（Agent/DataProxy/MQ/Sort/Manager/Dashboard/Audit），輕量模式可用 Sort+Manager+Dashboard，但仍比「單二進位制同步工具」重；維運成本要提前評估。

### 3. Flink CDC + Dinky / StreamPark（組合方案）

<figure>
<img src="https://d94807974-commits.github.io/images/realtime-cdc-sync-opensource-comparison/flink-cdc-architecture.jpg" alt="Apache Flink CDC 架構圖" loading="lazy" width="900">
<figcaption>圖片：來自 <a href="https://github.com/apache/flink-cdc">apache/flink-cdc</a> 文件靜態資源 <code>docs/static/fig/architecture.png</code>（Apache-2.0）。本地壓縮託管。</figcaption>
</figure>

**能力依據**：

- Flink CDC **本身無 Web UI**（官方定位是庫/連接器 + Pipeline YAML）。
- Pipeline Connectors Overview（stable / 3.7）：Source 含 **MySQL、SQL Server**；Sink 含 **Kafka、Doris** 等；**未列出 ClickHouse pipeline sink**。[Overview](https://nightlies.apache.org/flink/flink-cdc-docs-stable/docs/connectors/pipeline-connectors/overview/)
- 寫 ClickHouse 需走 **Flink SQL Connector / JDBC / 第三方**，不算「Pipeline 一等 sink」，本文標 ⚠️。
- Web 層：
  - [Dinky](https://github.com/DataLinkDC/dinky)（3779★）：FlinkSQL 開發平臺，支援 CDCSOURCE 等作業管理（中文社群活躍）。
  - [StreamPark](https://github.com/apache/streampark)（4346★，Apache-2.0）：Flink/Spark 應用管理平臺。

最新 Flink CDC release：`release-3.7.0`（2026-10-10）。

**適合**：已有 Flink 平臺、要強一致性/複雜轉換、接受「CDC 引擎 + 作業平臺」兩件套。  
**坑**：ClickHouse 需額外確認連接器版本矩陣；SQL Server CDC 要開庫級 CDC、清理 job；平臺與 Flink/CDC 版本矩陣要鎖死。

### 4. TIS（datavane/tis）

**能力依據**：[tis.pub 文件「支援的端型別」](https://tis.pub/docs/) 表格（核實於 2026-11-05）顯示：

- **SqlServer**：DataX 讀/寫 + **Flink-CDC Source** + ChunJun Sink  
- **MySQL**：同上  
- **Kafka**：DataX 與即時 Pipeline sink  
- **Doris**：DataX 讀/寫 + ChunJun 即時 Sink；文件有「MySQL 同步 Doris」即時 Demo  
- **Clickhouse**：DataX Writer + ChunJun 即時 Sink；有「MySQL 匯入 Clickhouse」影片範例  

底層明確寫為 **DataX / Flink-CDC**（及 ChunJun）+ Web Console，滿足「有 Web 配全量+增量」。

**適合**：要強 Web/零程式碼、中文資料全、願意繫結 TIS 外掛生態與 Flink 叢集。  
**坑**：外掛/SPI 體系學習曲線；即時鏈路實際依賴 Flink 叢集品質；社群體量小於 SeaTunnel/Flink CDC 主倉。

## 三、橫向對比（工程維度）

| 維度 | SeaTunnel+Web | InLong | Flink CDC+Dinky/StreamPark | TIS | DBSyncer 2.1.6（★954，未過 1k） |
|------|---------------|--------|----------------------------|-----|--------------------------------|
| Web 配置成熟度 | 可用，但 Web 倉活躍偏弱 | Dashboard 一等公民 | 取決於 Dinky/StreamPark | Console 強，偏零程式碼 | ✅ 內建 Thymeleaf 控制台 |
| 源：MySQL / SQL Server | 雙 CDC 文件齊全 | Extract 節點表齊全 | Pipeline 3.7 雙源 | Flink-CDC + DataX | ✅ binlog / ✅ CDC(LSN) |
| 目標三件套 | 三 sink 均有官方頁 | 三 Load 均列出 | Kafka+Doris 原生；CH ⚠️ | 三端均在端型別表 | ✅ Kafka；✅ Doris Stream Load；✅ CH 寫（源=游標） |
| 全量→增量 | `startup.mode=initial` | 同步場景支援 | Snapshot + binlog/CDC | DataX 全量 + Flink 增量 | ✅ FullIncrementPuller 釘位點 |
| DDL / Schema | `schema-changes.enabled`（有限） | 視 Sort/連接器 | Pipeline 演進中，需查版本文件 | 依賴底層 Flink-CDC/ChunJun | MySQL ALTER 事件有程式碼；e2e 未測 |
| 部署依賴 | Zeta 或 Flink + Web + DB | 多元件或輕量 Sort | Flink 叢集 + 平臺 | TIS + Flink(+DataX) | 單程序 + H2/MySQL；叢集需驗證 |
| 許可證 | Apache-2.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 |
| 中文/繁中資料 | 官網中英；iT 邦幫忙等有 CDC 概念文 | 中英文件 | 中文社群很強（Dinky） | 中文文件/影片豐富 | 中文官網/社群友好 |
| 典型坑 | Web/引擎版本鎖；外掛路徑 | 元件多、維運重 | CH sink 非 pipeline；版本矩陣 | 外掛版本與 Flink 對齊 | 預設 admin/admin；Boot 2.5/Kafka 0.11；無 GH Actions |

**繁體資料**：臺灣 iT 邦幫忙有多篇 CDC 概念文（如 [變更資料擷取 CDC](https://ithelp.ithome.com.tw/articles/10362458)、[Change Data Capture](https://ithelp.ithome.com.tw/articles/10275451)），偏原理而非上述具體產品對比；產品級繁中文件仍以各專案官網中英為主。

## 四、未入選及原因（同樣經核實）

| 專案 | Stars / License（2026-11-05） | 未入選主因 |
|------|-------------------------------|------------|
| [alibaba/canal](https://github.com/alibaba/canal)（+canal-admin） | 29746 / Apache-2.0 | **源端以 MySQL 為主**，不滿足 SQL Server 一等需求 |
| [debezium/debezium](https://github.com/debezium/debezium) | 13317 / Apache-2.0 | 引擎強大，但官方 [debezium-ui](https://github.com/debezium/debezium-ui) **已 archived**（351★，MIT），無活躍一等 Web |
| [alibaba/DataX](https://github.com/alibaba/DataX) + [WeiYe-Jing/datax-web](https://github.com/WeiYe-Jing/datax-web) | 17382 / NOASSERTION；6023 / MIT | **批式同步為主，無真正 CDC 增量**（增量需另棧）；DataX-Web 最近 push 停在 2024-06 |
| [brokercap/Bifrost](https://github.com/brokercap/Bifrost) | 2092 / Apache-2.0 | **MySQL 生態**，不覆蓋 SQL Server 條件 |
| [DTStack/chunjun](https://github.com/DTStack/chunjun) | 4106 / Apache-2.0 | 連接器能力強，但**本身不是完整 Web 配置平臺**（常被 TIS 等託管） |
| [airbytehq/airbyte](https://github.com/airbytehq/airbyte) | 22317 / **ELv2 (NOASSERTION)** | **非 OSI 傳統開源**（Elastic License 2.0）；Doris destination 社群狀態需單獨核（有 ARCHIVED 記錄），不作為本清單首選 |
| [tapdata/tapdata](https://github.com/tapdata/tapdata) | **662** / Apache-2.0 | **star < 1k** |
| [86dbs/dbsyncer](https://github.com/86dbs/dbsyncer)（DBSyncer **v2.1.6**） | **954** / Apache-2.0 | **star < 1k**（門檻未過）；功能上 Web + MySQL binlog / SQL Server CDC → Kafka/Doris/ClickHouse + 全量+增量均有原始碼依據，見下文「特別分析」；勿與 [Zhh9126/DBSYNC](https://github.com/Zhh9126/DBSYNC)（2.1.4 重新命名衍生）混淆 |
| [apache/hop](https://github.com/apache/hop) | 1519 / Apache-2.0 | ETL/視覺化強，**CDC 通常外掛 Debezium**，非本條件「內建訂閱三目標」主線 |
| CloudCanal（ClouGence） | 官方定位見 [cloudcanal-issue](https://github.com/ClouGence/cloudcanal-issue) | **社群版可免費使用但非開源**；明確剔除 |
| [lensesio/kafka-connect-ui](https://github.com/lensesio/kafka-connect-ui) | 519 | **star < 1k**，且只是 Connect 管理 UI，不是端到端 CDC 平臺 |
| seatunnel-web **單獨** | 888 | 不單獨過 1k；僅作為引擎配套計入 |

## 四附、特別分析：DBSyncer 2.1.6（功能匹配，star 未過 1k）

> **口徑**：仍列在「未入選」——本文硬門檻是 GitHub **stars > 1000**；2026-11-05 核實 [86dbs/dbsyncer](https://github.com/86dbs/dbsyncer) 為 **954★ / 156 forks / Apache-2.0**，最新 tag **`v2.1.6`**（annotated tag → commit `d0aa992e`，提交說明 `!621 pg增量调整`，UTC 2026-11-03 11:04；換算 Asia/Singapore = 2026-11-03 19:04）。`master` 在 tag 之後仍有數個「外掛增加刪除」提交（分析以 **v2.1.6** 檢出為準）。GitHub **Releases 列表為空**（發版靠 tag）；**無 `.github/workflows` CI**。貢獻者 API 前列：`86dbs`(2210)、`Pengnd`(519)、`lifejwang11`(66) 等。官網 [dbsyncer.cn](https://dbsyncer.cn)。

### 為何單獨寫一節

在「Web + MySQL/SQL Server → Kafka/Doris/ClickHouse + 全量+增量」這條鏈路上，DBSyncer **原始碼能力與本文入選條件高度重合**，僅差 star 門檻；中小團隊「單機開箱」場景經常會搜到它。下面結論均來自對 `v2.1.6` 樹的只讀走讀與一次 Linux `mvn -DskipTests package` 嘗試（不對上游倉庫做任何寫操作）。

### 架構（ASCII）

```
Browser ──► dbsyncer-web (Thymeleaf, 預設 :18686, README: admin/admin)
                │
                v
           dbsyncer-biz / parser / plugin
                │
                v
           dbsyncer-manager
        FullPuller | IncrementPuller | FullIncrementPuller
        (先 capture 位點 → 全量 → 增量；Meta snapshot 斷點)
                │
                v
           dbsyncer-connector (SPI)
   MySQL(binlog/shyiko)  SqlServer(CDC/LSN)
   Kafka(writer)  Doris(Stream Load|JDBC)  ClickHouse(源=PK游標輪詢 / 目標=寫)
                │
                v
           dbsyncer-storage
        預設 H2 檔案庫；生產建議 MySQL
        叢集開關 dbsyncer.cluster.enabled（須 storage=mysql）
```

### 核心同步鏈路（檔案路徑）

| 環節 | 路徑 / 行為（v2.1.6） |
|------|----------------------|
| 全量+增量編排 | `dbsyncer-manager/.../FullIncrementPuller.java`：註釋寫明「先捕獲位點 → 全量 → 從位點啟動增量」；`prepareFullPhase` → `incrementPuller.captureAndSaveOffset` |
| MySQL 增量 | `dbsyncer-connector-mysql/.../cdc/MySQLListener.java`：`BinaryLogClient` + binlog filename/position；`QueryEvent` → `DDLChangedEvent`（含 `parseAlter`） |
| SQL Server 增量 | `.../sqlserver/cdc/SqlServerListener.java`：`sys.fn_cdc_get_max_lsn` / `fn_cdc_get_all_changes_*`，位點鍵 `position` |
| → Kafka | `.../kafka/KafkaConnector.java`：`writer` 逐條 `producer.send` + `flush`；`reader` 拋「Kafka暫不支援全量讀取」 |
| → Doris | `.../doris/DorisConnector.java` + `load/DorisStreamLoadWriter.java`：配置為 Stream Load 模式且事件匹配時走 Stream Load，否則 JDBC；`buildUpsertSql` 退化為普通 INSERT（Unique/PK 覆蓋語義，程式碼註釋寫明） |
| ClickHouse | 源：`.../clickhouse/cdc/ClickHouseListener.java` **單數值主鍵游標輪詢**（非 binlog/CDC）；目標：JDBC 寫 / 按 PK 批刪等。README 連接器表：源✔ 目標✔ |
| Web / 埠 | `dbsyncer-web/.../application.properties`：`server.port=18686`；README 明確 **admin/admin** |
| 後設資料 | `dbsyncer.storage.type=h2`（預設）或 `mysql`；範例 `dbsyncer.storage.mysql.password=123` |

### 一致性與可靠性（程式碼觀感）

- **Kafka**：無交易 producer / 未見預設冪等強制；偏 **at-least-once**，需目標或業務冪等。客戶端依賴鎖定為 **`kafka.version=0.11.0.0`**（與現代 Broker 的相容與安全補丁均偏舊）。
- **Doris**：依賴 Unique/Primary Key 模型下 INSERT 覆蓋；Stream Load 與 JDBC 雙路徑，配置錯誤時行為不同。
- **全量切增量**：經典「先釘位點再灌歷史」；全量視窗內變更靠增量回放——需壓測與對帳。
- **DDL**：MySQL 側有 ALTER 解析與事件下發；對 Kafka/Doris/CH 目標的端到端 DDL 同步**未在本環境做 e2e**（標未能核實）。
- **HA**：存在 `ClusterService` SPI、`ClusterController`、以及 `application-node*.properties`（`dbsyncer.cluster.enabled=true`）；開源樹內預設實現為 `StandaloneService`（`@ConditionalOnMissingBean` + SPI）。**多節點排程完整行為依賴 MySQL 儲存與 SPI 實現，本分析未拉起多節點叢集驗證。** `MetaLockUtil` 僅為程序內 `String.intern` 鎖。

### 安全與依賴

- 預設帳號：**admin / admin**（README；`UserConfigServiceImpl.DEFAULT_PASSWORD` = SHA1(`admin`) 的 Base64：`0DPiKuNIrrVmD8IUCuw1hQxNqZc=`，已本地複核）。
- SSL keystore 密碼預設 **`dbsyncer`**（`application.properties`）。
- 棧：Java 8、**Spring Boot 2.5.14**（社群 EOL）、log4j2 **2.17.1**、mysql-binlog-connector **0.30.1**；`mysql.version=8.0.21`（偏舊）。無 NOTICE 檔案，有 Apache-2.0 `LICENSE`。
- 測試：約 **15** 個 `src/test` Java 檔案 vs 約 **1073** 個 main；**無 GitHub Actions**。

### 構建結果（本機 Linux，Asia/Singapore 時段）

在隔離環境安裝 Temurin 11 + Maven 3.9.9，於 `/tmp/dbsyncer-up`（`git checkout v2.1.6`）執行 `mvn -DskipTests package`：**成功**，產出 `dbsyncer-web/target/dbsyncer-2.1.6.jar`（約 1.2MB 模組 jar；完整可執行包以專案 `build.sh`/assembly 為準）。模組名與目錄均為小寫 `dbsyncer-connector-*`，**不存在** Zhh9126/DBSYNC 那種 `DBDataSync-*` / `dbdatasync-*` 大小寫錯配。

### 優劣與適用場景

**優勢**：真正「下載解壓 + Web」；MySQL binlog 與 SQL Server CDC 同產品內；Kafka/Doris/CH 目的碼齊全；全量+增量編排清晰；中文社群與 [dbsyncer.cn](https://dbsyncer.cn) 文件友好；相對 Flink/InLong 維運面小。

**劣勢**：star < 1k、無 GitHub Releases/Actions；Boot 2.5 / Kafka 0.11 技術債；預設口令與範例庫密碼；ClickHouse「增量源」實為游標輪詢；一致性語義弱於 Flink CDC exactly-once 體系；開源叢集能力需自行驗證。

**適合**：中小團隊、單機/小叢集、要快速建立 MySQL 或 SQL Server → Kafka/Doris（及 CH 目標寫入）、可接受自行加固安全與依賴升級。  
**更適合改用入選方案的情況**：要強 exactly-once、大規模多租戶、或必須 star/ASF 生態背書 → SeaTunnel / InLong / Flink CDC+Dinky / TIS。

### 生產補強清單（若採用 2.1.6）

1. 立即改掉 admin 密碼；關閉或保護 `reset-pwd`；改掉範例 MySQL storage 密碼與 keystore 口令。  
2. 生產後設資料改 **MySQL**；按需評估 `cluster.enabled` 並做故障切換演練。  
3. 升級路徑評估：Boot / Kafka client / JDBC 驅動；補依賴 CVE 掃描與 CI。  
4. 明確 Kafka acks/冪等與 Doris 模型；全量切增量與失敗重試做對帳。  
5. ClickHouse 若作**源**，不要按真正 CDC 預期排期。  
6. **優先跟蹤官方 [86dbs/dbsyncer](https://github.com/86dbs/dbsyncer)**，勿預設採用第三方重新命名倉（見下）。

### 衍生倉警示：Zhh9126/DBSYNC（2.1.4 重新命名）

同日只讀分析 [Zhh9126/DBSYNC](https://github.com/Zhh9126/DBSYNC)（26★，Apache-2.0，創建於 2026-10-19）：GitHub `fork=false`，但唯一提交自述「原 dbsyncer 重新命名升級」，包名 `dbsyncer`→`dbdatasync`，版本停在 **2.1.4**，並多出 `dbdatasync-engine` 模組；**缺少**上游已有的 `gaussdb` 等連接器。關鍵問題：

- `dbdatasync-connector/pom.xml` 模組寫成 `DBDataSync-connector-*`，磁碟目錄為 `dbdatasync-connector-*` → **Linux 下 Maven 直接失敗**（與其 issue #1 一致）；本環境復現。  
- **無** `.github` CI、**無** Release；貢獻者僅上傳者 1 人。  
- 同樣預設 **admin/admin**，keystore 密碼改為 `DBDataSync`，範例 `mysql.password=123`。  
- 無 NOTICE；署名弱於對上游 AE86/86dbs 的常見歸因實踐。

**結論**：若看中 DBSyncer 能力，請用 **86dbs/dbsyncer @ v2.1.6+**；DBSYNC 僅適合對照原始碼，**不建議生產**。

## 五、場景選型建議

1. **要用一條鏈路串起 MySQL+SQL Server → Kafka/Doris/ClickHouse，且希望少疊平臺**  
   → 優先 **SeaTunnel + SeaTunnel Web**；上線前做 Web↔引擎版本相容矩陣與一次全量切增量演練。

2. **已有 Flink 平臺，要強流處理/複雜 SQL**  
   → **Flink CDC 3.7+**，Web 用 **Dinky**（開發友好）或 **StreamPark**（作業治理）；若目標含 ClickHouse，立項時單獨立項驗證 sink。

3. **要「資料接入平臺」而不只是同步作業**（多租戶、審計、MQ 緩衝）  
   → **Apache InLong**；評估是否上完整元件還是輕量 Sort 模式。

4. **強依賴中文零程式碼 Web、批次+即時同一控制台**  
   → **TIS**；把 Flink 叢集與外掛版本納入變更管理。

5. **跨地域（如新加坡作業 ↔ 墨西哥 Doris）**  
   → 同步作業與 Stream Load **靠近 Doris 區域**；或先入**同區域 Kafka**再非同步消費。跨洋直寫會把延遲與超時放大到 checkpoint / 交易語義上。

6. **僅 MySQL、要極簡**  
   → Canal / Bifrost 等仍可用，但**不符合本文 SQL Server 門檻**，勿混用結論。

7. **要極簡 Web 單機、功能鏈完整，可接受 star < 1k 與自行加固**  
   → **DBSyncer 2.1.6（86dbs/dbsyncer）**；詳見「特別分析」。**不要**用 Zhh9126/DBSYNC 替代上游。

## 六、上線前檢查清單

- [ ] 源庫：MySQL `binlog_format=ROW`、`binlog_row_image=FULL`；SQL Server 已啟用 CDC 且清理作業正常  
- [ ] 帳號權限：SELECT + 複製/CDC 相關權限（按各連接器文件，勿只開業務 CRUD）  
- [ ] 全量視窗：大表 snapshot 對源庫 IO 的影響；是否可從從庫讀  
- [ ] 增量斷點：checkpoint / offset 持久化介質與恢復演練  
- [ ] 目標：Doris 模型（Unique/Duplicate）與部分列更新；ClickHouse 去重/ReplacingMergeTree 策略；Kafka 分區鍵與冪等  
- [ ] DDL：增刪改列是否自動同步；不支援時的告警與人工流程  
- [ ] 許可證：二次分發/SaaS 是否觸碰 ELv2 等限制（Airbyte）  
- [ ] 可觀測：延遲、髒資料、對帳（行數/取樣 checksum）  
- [ ] 跨區域：RTT、頻寬、是否同城/同區落地 sink  

## 七、未能核實 / 閱讀邊界

- CloudCanal **商業控制台內部**功能細節未登入產品試用，僅依據其 GitHub issue 倉說明「非開源」剔除。  
- Airbyte 各 destination 的當前維護狀態（含 Doris）變化快，本文以 ELv2 + 歷史 ARCHIVED 線索作風險提示，**未逐一跑通連接器 e2e**。  
- SeaTunnel / Flink CDC 對 **每一種 DDL 子型別**（改主鍵、改分區等）的支援矩陣未逐條迴歸，只引用文件中已寫明的選項。  
- StreamPark / Dinky 與 Flink CDC 3.7 的**組合相容性**未在本機拉起叢集驗證，選型需按各自 release note 鎖版本。  
- 未評測各方案在「新加坡→墨西哥」鏈路上的實測延遲，跨地域僅為架構建議。  
- DBSyncer：**多節點叢集**（`cluster.enabled=true` + MySQL storage）與 **DDL→Doris/Kafka/CH** 端到端未在本環境拉起驗證；Kafka `acks`/冪等以業務配置為準，倉庫測試樣例可見 `acks=1`、`enable.idempotence=false`。  
- DBSyncer `master` 相對 `v2.1.6` 的後續提交未逐條迴歸。

## 參考資料

1. GitHub API 快照（2026-11-05）：`gh api repos/apache/seatunnel` 等，本文表格 star/license/pushed_at 來源。  
2. [SeaTunnel MySQL-CDC](https://seatunnel.apache.org/docs/2.3.11/connector-v2/source/MySQL-CDC/) · [SqlServer-CDC](https://seatunnel.apache.org/docs/2.3.11/connector-v2/source/SqlServer-CDC/) · [Kafka/Doris/Clickhouse sink](https://seatunnel.apache.org/docs/2.3.11/connector-v2/sink/Kafka/)  
3. [SeaTunnel Web README](https://github.com/apache/seatunnel-web/blob/main/README.md)  
4. [Apache InLong Introduction / Supported Data Nodes](https://inlong.apache.org/docs/introduction/)  
5. [Flink CDC Pipeline Connectors Overview](https://nightlies.apache.org/flink/flink-cdc-docs-stable/docs/connectors/pipeline-connectors/overview/)  
6. [TIS 文件與端型別表](https://tis.pub/docs/) · [datavane/tis](https://github.com/datavane/tis)  
7. [Dinky](https://github.com/DataLinkDC/dinky) · [StreamPark](https://github.com/apache/streampark)  
8. [Airbyte LICENSE (ELv2)](https://github.com/airbytehq/airbyte/blob/master/LICENSE) · [ClouGence/cloudcanal-issue](https://github.com/ClouGence/cloudcanal-issue)  
9. 繁中 CDC 概念：[iT邦幫忙 - 變更資料擷取](https://ithelp.ithome.com.tw/articles/10362458)  
10. [86dbs/dbsyncer](https://github.com/86dbs/dbsyncer) · tag [v2.1.6](https://github.com/86dbs/dbsyncer/releases/tag/v2.1.6) · [dbsyncer.cn](https://dbsyncer.cn) · [LICENSE (Apache-2.0)](https://github.com/86dbs/dbsyncer/blob/master/LICENSE)  
11. 衍生倉對照（不推薦生產）：[Zhh9126/DBSYNC](https://github.com/Zhh9126/DBSYNC) · [issue #1 Linux 編譯大小寫](https://github.com/Zhh9126/DBSYNC/issues/1)

---

*發布說明：配圖均來自 Apache 專案官方資源（Apache-2.0），壓縮後 <300KB，託管於本站靜態目錄。初版核實 2026-11-05；同日增補 DBSyncer 2.1.6 原始碼特別分析（Asia/Singapore）。*

