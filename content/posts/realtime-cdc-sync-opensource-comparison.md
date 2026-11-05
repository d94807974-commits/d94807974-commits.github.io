---
title: "开源实时 CDC 同步选型：带 Web 配置、MySQL/SQL Server→Kafka/Doris/ClickHouse、全量+增量（2026-11 核实）"
date: 2026-11-05T13:52:00+08:00
draft: false
tags: ["CDC", "数据同步", "SeaTunnel", "Flink CDC", "InLong", "Doris", "ClickHouse", "Kafka", "开源选型", "数据集成"]
summary: "按「Web 可配 + MySQL/SQL Server 源 + Kafka/Doris/ClickHouse 目标 + 全量+增量 + star>1k + 开源」条件，用 GitHub API 与官方文档逐一核实候选，对比 SeaTunnel(+Web)、InLong、Flink CDC(+Dinky/StreamPark)、TIS 等，给出场景选型、检查清单与未入选原因。数据核实日期：2026-11-05（Asia/Singapore）。"
ShowToc: true
---

> **调研边界（请先读）**：本文只讨论同时满足下列条件的开源项目——① 有 Web 页面可配置任务；② 源端至少覆盖 **MySQL 与 SQL Server**；③ 目标端覆盖 **Kafka、Apache Doris、ClickHouse**（或可明确组合达成）；④ 支持 **全量 + 增量**（CDC / binlog / CDC 日志类，而非仅定时 JDBC 批抽）；⑤ GitHub **stargazers_count > 1000**；⑥ OSI/ASF 友好开源许可（Apache-2.0 等）。Star、license、最近 push 一律以 `gh api repos/<owner>/<repo>` 在 **2026-11-05** 拉取为准；功能以官方文档页面为准，**不凭记忆断言**。未能打开或无法交叉验证的点，会在文末「未能核实」列出。

<figure>
<img src="/images/realtime-cdc-sync-opensource-comparison/seatunnel-website-logo.jpg" alt="Apache SeaTunnel 官网 Logo" loading="lazy" width="720">
<figcaption>图片：Apache SeaTunnel 官网 <a href="https://seatunnel.apache.org/image/logo.png">logo.png</a>；项目许可证 <a href="https://github.com/apache/seatunnel/blob/dev/LICENSE">Apache-2.0</a>。已压缩并本地托管于本站 <code>static/images/…</code>。</figcaption>
</figure>

## 一、问题怎么拆

业务侧常见诉求可以写成一条链路：

```
[OLTP] MySQL / SQL Server
        |  (snapshot 全量 + binlog/CDC 增量)
        v
   同步引擎 / 平台 (+ Web 配置)
        |
        +--> Kafka（总线 / 解耦）
        +--> Apache Doris（分析型 OLAP）
        +--> ClickHouse（列存分析）
```

工程上真正卡人的通常不是「能不能连上」，而是：

1. **有没有 Web**：运维/数据同学是否必须手写 YAML / Flink SQL。
2. **SQL Server 是否一等公民**：很多「国产 CDC」其实只做深了 MySQL。
3. **三目标是否原生 sink**：Kafka / Doris / ClickHouse 缺一就会被迫叠两套栈。
4. **全量切增量是否平滑**：`startup.mode=initial`、checkpoint、exactly-once、DDL。
5. **许可证与 star**：商业分发、二次开发、社区活跃度。

跨地域补充一句：若同步链路两端跨大洲（例如控制面/作业在新加坡、Doris 在墨西哥），RTT 与跨洋带宽会放大 checkpoint 与 Stream Load 延迟；选型时优先选**可把 sink 写路径贴近目标集群**、或**中间以同区域 Kafka 解耦**的架构，而不是只看功能 checklist。

## 二、入选项目（满足或「组合满足」）

下表 star / license / 最近 push 均为 **2026-11-05** 经 GitHub REST API 核实（UTC 时间已换算备注）。

| 方案 | 仓库 | Stars | License | 最近 push (UTC) | Web | MySQL | SQL Server | Kafka | Doris | ClickHouse | 全量+增量 |
|------|------|------:|---------|-----------------|-----|-------|------------|-------|-------|------------|-----------|
| SeaTunnel + SeaTunnel Web | [apache/seatunnel](https://github.com/apache/seatunnel) + [apache/seatunnel-web](https://github.com/apache/seatunnel-web) | **9788** / **888** | Apache-2.0 | 2026-11-04 / 2026-01-22 | ✅ Web（独立仓） | ✅ MySQL-CDC | ✅ SqlServer-CDC | ✅ | ✅ | ✅ | ✅ `startup.mode=initial` |
| Apache InLong | [apache/inlong](https://github.com/apache/inlong) | **1507** | Apache-2.0 | 2026-09-16 | ✅ Dashboard | ✅ Extract | ✅ Extract | ✅ Load | ✅ Load | ✅ Load | ✅ 同步场景 |
| Flink CDC + Dinky | [apache/flink-cdc](https://github.com/apache/flink-cdc) + [DataLinkDC/dinky](https://github.com/DataLinkDC/dinky) | **6492** / **3779** | Apache-2.0 | 2026-11-05 / 2026-10-22 | ⚠️ CDC 本身无 Web；Dinky 提供 | ✅ | ✅ Pipeline 3.7 | ✅ Pipeline sink | ✅ Pipeline sink | ⚠️ Pipeline 无官方 CH sink，需 Flink SQL/其他 | ✅ |
| Flink CDC + StreamPark | 同上 + [apache/streampark](https://github.com/apache/streampark) | **6492** / **4346** | Apache-2.0 | 同上 / 2026-10-31 | ⚠️ 同上，StreamPark 管 Flink 作业 | ✅ | ✅ | ✅ | ✅ | ⚠️ 同上 | ✅ |
| TIS | [datavane/tis](https://github.com/datavane/tis) | **1440** | Apache-2.0 | 2026-11-04 | ✅ Console | ✅ Flink-CDC + DataX | ✅ Flink-CDC + DataX | ✅ | ✅ | ✅ | ✅ 批量 DataX + 实时 Flink-CDC |

> **关于 SeaTunnel Web 888★**：单独未过 1k，但它是 ASF 官方 Web 控制台，与引擎同属 SeaTunnel 生态；本文按「引擎 star>1k + 官方 Web」入选，并在对比里标明 Web 仓活跃度（最近 push 停在 2026-01，需关注版本绑定）。

### 1. Apache SeaTunnel（+ SeaTunnel Web）

**能力依据（文档）**：

- MySQL CDC：支持 snapshot + 增量，`startup.mode` 含 `initial`（先历史后增量），可选 exactly-once、schema-changes。[MySQL-CDC 文档](https://seatunnel.apache.org/docs/2.3.11/connector-v2/source/MySQL-CDC/)
- SQL Server CDC：[SqlServer-CDC](https://seatunnel.apache.org/docs/2.3.11/connector-v2/source/SqlServer-CDC/)
- Sink： [Kafka](https://seatunnel.apache.org/docs/2.3.11/connector-v2/sink/Kafka/)、[Doris](https://seatunnel.apache.org/docs/2.3.11/connector-v2/sink/Doris/)、[Clickhouse](https://seatunnel.apache.org/docs/2.3.11/connector-v2/sink/Clickhouse/) 均有独立文档页（HTTP 301→正式路径，已确认存在）。
- Web： [seatunnel-web README](https://github.com/apache/seatunnel-web/blob/main/README.md) 描述为管理脚本、数据源、任务并提交到 Zeta Engine；默认 UI `http://127.0.0.1:8801/ui/`（admin/admin）。**Web 与引擎版本强绑定**，部署要求客户端与 Server 同 OS/同目录结构。

**架构示意**：

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

**适合**：希望「一个引擎覆盖三目标」、团队接受 Zeta/Flink 引擎运维、中文文档友好。  
**坑**：Web 仓 star 与活跃度弱于引擎；插件 jar / `plugin-mapping.properties` / `SEATUNNEL_HOME` 配置易踩坑；CDC 权限（REPLICATION）与 GTID/binlog 格式要按文档开。

### 2. Apache InLong

<figure>
<img src="/images/realtime-cdc-sync-opensource-comparison/inlong-architecture.jpg" alt="Apache InLong 标准架构图" loading="lazy" width="900">
<figcaption>图片：Apache InLong 官方文档架构图 <a href="https://inlong.apache.org/img/inlong-structure-en.png">inlong-structure-en.png</a>（见 <a href="https://inlong.apache.org/docs/introduction/">Introduction</a>）；Apache-2.0。本地压缩托管。</figcaption>
</figure>

**能力依据**：[Introduction - Supported Data Nodes](https://inlong.apache.org/docs/introduction/) 明确列出：

- Extract：MySQL、**SQLServer**（2012–2019）等  
- Load：**Kafka、Doris、ClickHouse** 等  
- 模块含 **inlong-dashboard**（前端管控）、Manager、Sort、Agent 等  

最新 release 标签 API 见 `2.4.0-RC0`（2026-08-21）；仓库 push 至 2026-09-16，活跃度中等。

**适合**：需要「接入 + 同步 + 订阅」一体、已有或愿意上 TubeMQ/Pulsar/Kafka 作为中间缓冲、偏腾讯系大数据栈。  
**坑**：完整架构组件多（Agent/DataProxy/MQ/Sort/Manager/Dashboard/Audit），轻量模式可用 Sort+Manager+Dashboard，但仍比「单二进制同步工具」重；运维成本要提前评估。

### 3. Flink CDC + Dinky / StreamPark（组合方案）

<figure>
<img src="/images/realtime-cdc-sync-opensource-comparison/flink-cdc-architecture.jpg" alt="Apache Flink CDC 架构图" loading="lazy" width="900">
<figcaption>图片：来自 <a href="https://github.com/apache/flink-cdc">apache/flink-cdc</a> 文档静态资源 <code>docs/static/fig/architecture.png</code>（Apache-2.0）。本地压缩托管。</figcaption>
</figure>

**能力依据**：

- Flink CDC **本身无 Web UI**（官方定位是库/连接器 + Pipeline YAML）。
- Pipeline Connectors Overview（stable / 3.7）：Source 含 **MySQL、SQL Server**；Sink 含 **Kafka、Doris** 等；**未列出 ClickHouse pipeline sink**。[Overview](https://nightlies.apache.org/flink/flink-cdc-docs-stable/docs/connectors/pipeline-connectors/overview/)
- 写 ClickHouse 需走 **Flink SQL Connector / JDBC / 第三方**，不算「Pipeline 一等 sink」，本文标 ⚠️。
- Web 层：
  - [Dinky](https://github.com/DataLinkDC/dinky)（3779★）：FlinkSQL 开发平台，支持 CDCSOURCE 等作业管理（中文社区活跃）。
  - [StreamPark](https://github.com/apache/streampark)（4346★，Apache-2.0）：Flink/Spark 应用管理平台。

最新 Flink CDC release：`release-3.7.0`（2026-10-10）。

**适合**：已有 Flink 平台、要强一致性/复杂转换、接受「CDC 引擎 + 作业平台」两件套。  
**坑**：ClickHouse 需额外确认连接器版本矩阵；SQL Server CDC 要开库级 CDC、清理 job；平台与 Flink/CDC 版本矩阵要锁死。

### 4. TIS（datavane/tis）

**能力依据**：[tis.pub 文档「支持的端类型」](https://tis.pub/docs/) 表格（核实于 2026-11-05）显示：

- **SqlServer**：DataX 读/写 + **Flink-CDC Source** + ChunJun Sink  
- **MySQL**：同上  
- **Kafka**：DataX 与实时 Pipeline sink  
- **Doris**：DataX 读/写 + ChunJun 实时 Sink；文档有「MySQL 同步 Doris」实时 Demo  
- **Clickhouse**：DataX Writer + ChunJun 实时 Sink；有「MySQL 导入 Clickhouse」视频示例  

底层明确写为 **DataX / Flink-CDC**（及 ChunJun）+ Web Console，满足「有 Web 配全量+增量」。

**适合**：要强 Web/零代码、中文资料全、愿意绑定 TIS 插件生态与 Flink 集群。  
**坑**：插件/SPI 体系学习曲线；实时链路实际依赖 Flink 集群质量；社区体量小于 SeaTunnel/Flink CDC 主仓。

## 三、横向对比（工程维度）

| 维度 | SeaTunnel+Web | InLong | Flink CDC+Dinky/StreamPark | TIS |
|------|---------------|--------|----------------------------|-----|
| Web 配置成熟度 | 可用，但 Web 仓活跃偏弱 | Dashboard 一等公民 | 取决于 Dinky/StreamPark | Console 强，偏零代码 |
| 源：MySQL / SQL Server | 双 CDC 文档齐全 | Extract 节点表齐全 | Pipeline 3.7 双源 | Flink-CDC + DataX |
| 目标三件套 | 三 sink 均有官方页 | 三 Load 均列出 | Kafka+Doris 原生；CH ⚠️ | 三端均在端类型表 |
| 全量→增量 | `startup.mode=initial` | 同步场景支持 | Snapshot + binlog/CDC | DataX 全量 + Flink 增量 |
| DDL / Schema | `schema-changes.enabled`（有限） | 视 Sort/连接器 | Pipeline 演进中，需查版本文档 | 依赖底层 Flink-CDC/ChunJun |
| 部署依赖 | Zeta 或 Flink + Web + DB | 多组件或轻量 Sort | Flink 集群 + 平台 | TIS + Flink(+DataX) |
| 许可证 | Apache-2.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 |
| 中文/繁中资料 | 官网中英；iT 邦帮忙等有 CDC 概念文 | 中英文档 | 中文社区很强（Dinky） | 中文文档/视频丰富 |
| 典型坑 | Web/引擎版本锁；插件路径 | 组件多、运维重 | CH sink 非 pipeline；版本矩阵 | 插件版本与 Flink 对齐 |

**繁体资料**：台湾 iT 邦帮忙有多篇 CDC 概念文（如 [变更资料撷取 CDC](https://ithelp.ithome.com.tw/articles/10362458)、[Change Data Capture](https://ithelp.ithome.com.tw/articles/10275451)），偏原理而非上述具体产品对比；产品级繁中文档仍以各项目官网中英为主。

## 四、未入选及原因（同样经核实）

| 项目 | Stars / License（2026-11-05） | 未入选主因 |
|------|-------------------------------|------------|
| [alibaba/canal](https://github.com/alibaba/canal)（+canal-admin） | 29746 / Apache-2.0 | **源端以 MySQL 为主**，不满足 SQL Server 一等需求 |
| [debezium/debezium](https://github.com/debezium/debezium) | 13317 / Apache-2.0 | 引擎强大，但官方 [debezium-ui](https://github.com/debezium/debezium-ui) **已 archived**（351★，MIT），无活跃一等 Web |
| [alibaba/DataX](https://github.com/alibaba/DataX) + [WeiYe-Jing/datax-web](https://github.com/WeiYe-Jing/datax-web) | 17382 / NOASSERTION；6023 / MIT | **批式同步为主，无真正 CDC 增量**（增量需另栈）；DataX-Web 最近 push 停在 2024-06 |
| [brokercap/Bifrost](https://github.com/brokercap/Bifrost) | 2092 / Apache-2.0 | **MySQL 生态**，不覆盖 SQL Server 条件 |
| [DTStack/chunjun](https://github.com/DTStack/chunjun) | 4106 / Apache-2.0 | 连接器能力强，但**本身不是完整 Web 配置平台**（常被 TIS 等托管） |
| [airbytehq/airbyte](https://github.com/airbytehq/airbyte) | 22317 / **ELv2 (NOASSERTION)** | **非 OSI 传统开源**（Elastic License 2.0）；Doris destination 社区状态需单独核（有 ARCHIVED 记录），不作为本清单首选 |
| [tapdata/tapdata](https://github.com/tapdata/tapdata) | **662** / Apache-2.0 | **star < 1k** |
| [86dbs/dbsyncer](https://github.com/86dbs/dbsyncer) | **954** / Apache-2.0 | **star < 1k**（有 Web、含 SqlServer/Kafka，差在 star 门槛） |
| [apache/hop](https://github.com/apache/hop) | 1519 / Apache-2.0 | ETL/可视化强，**CDC 通常外挂 Debezium**，非本条件「内建订阅三目标」主线 |
| CloudCanal（ClouGence） | 官方定位见 [cloudcanal-issue](https://github.com/ClouGence/cloudcanal-issue) | **社区版可免费使用但非开源**；明确剔除 |
| [lensesio/kafka-connect-ui](https://github.com/lensesio/kafka-connect-ui) | 519 | **star < 1k**，且只是 Connect 管理 UI，不是端到端 CDC 平台 |
| seatunnel-web **单独** | 888 | 不单独过 1k；仅作为引擎配套计入 |

## 五、场景选型建议

1. **要一条链路打通 MySQL+SQL Server → Kafka/Doris/ClickHouse，且希望少叠平台**  
   → 优先 **SeaTunnel + SeaTunnel Web**；上线前做 Web↔引擎版本兼容矩阵与一次全量切增量演练。

2. **已有 Flink 平台，要强流处理/复杂 SQL**  
   → **Flink CDC 3.7+**，Web 用 **Dinky**（开发友好）或 **StreamPark**（作业治理）；若目标含 ClickHouse，立项时单独立项验证 sink。

3. **要「数据接入平台」而不只是同步作业**（多租户、审计、MQ 缓冲）  
   → **Apache InLong**；评估是否上完整组件还是轻量 Sort 模式。

4. **强依赖中文零代码 Web、批量+实时同一控制台**  
   → **TIS**；把 Flink 集群与插件版本纳入变更管理。

5. **跨地域（如新加坡作业 ↔ 墨西哥 Doris）**  
   → 同步作业与 Stream Load **靠近 Doris 区域**；或先入**同区域 Kafka**再异步消费。跨洋直写会把延迟与超时放大到 checkpoint / 事务语义上。

6. **仅 MySQL、要极简**  
   → Canal / Bifrost 等仍可用，但**不符合本文 SQL Server 门槛**，勿混用结论。

## 六、上线前检查清单

- [ ] 源库：MySQL `binlog_format=ROW`、`binlog_row_image=FULL`；SQL Server 已启用 CDC 且清理作业正常  
- [ ] 账号权限：SELECT + 复制/CDC 相关权限（按各连接器文档，勿只开业务 CRUD）  
- [ ] 全量窗口：大表 snapshot 对源库 IO 的影响；是否可从从库读  
- [ ] 增量断点：checkpoint / offset 持久化介质与恢复演练  
- [ ] 目标：Doris 模型（Unique/Duplicate）与部分列更新；ClickHouse 去重/ReplacingMergeTree 策略；Kafka 分区键与幂等  
- [ ] DDL：增删改列是否自动同步；不支持时的告警与人工流程  
- [ ] 许可证：二次分发/SaaS 是否触碰 ELv2 等限制（Airbyte）  
- [ ] 可观测：延迟、脏数据、对账（行数/采样 checksum）  
- [ ] 跨区域：RTT、带宽、是否同城/同区落地 sink  

## 七、未能核实 / 阅读边界

- CloudCanal **商业控制台内部**功能细节未登录产品试用，仅依据其 GitHub issue 仓说明「非开源」剔除。  
- Airbyte 各 destination 的当前维护状态（含 Doris）变化快，本文以 ELv2 + 历史 ARCHIVED 线索作风险提示，**未逐一跑通连接器 e2e**。  
- SeaTunnel / Flink CDC 对 **每一种 DDL 子类型**（改主键、改分区等）的支持矩阵未逐条回归，只引用文档中已写明的选项。  
- StreamPark / Dinky 与 Flink CDC 3.7 的**组合兼容性**未在本机拉起集群验证，选型需按各自 release note 锁版本。  
- 未评测各方案在「新加坡→墨西哥」链路上的实测延迟，跨地域仅为架构建议。

## 参考资料

1. GitHub API 快照（2026-11-05）：`gh api repos/apache/seatunnel` 等，本文表格 star/license/pushed_at 来源。  
2. [SeaTunnel MySQL-CDC](https://seatunnel.apache.org/docs/2.3.11/connector-v2/source/MySQL-CDC/) · [SqlServer-CDC](https://seatunnel.apache.org/docs/2.3.11/connector-v2/source/SqlServer-CDC/) · [Kafka/Doris/Clickhouse sink](https://seatunnel.apache.org/docs/2.3.11/connector-v2/sink/Kafka/)  
3. [SeaTunnel Web README](https://github.com/apache/seatunnel-web/blob/main/README.md)  
4. [Apache InLong Introduction / Supported Data Nodes](https://inlong.apache.org/docs/introduction/)  
5. [Flink CDC Pipeline Connectors Overview](https://nightlies.apache.org/flink/flink-cdc-docs-stable/docs/connectors/pipeline-connectors/overview/)  
6. [TIS 文档与端类型表](https://tis.pub/docs/) · [datavane/tis](https://github.com/datavane/tis)  
7. [Dinky](https://github.com/DataLinkDC/dinky) · [StreamPark](https://github.com/apache/streampark)  
8. [Airbyte LICENSE (ELv2)](https://github.com/airbytehq/airbyte/blob/master/LICENSE) · [ClouGence/cloudcanal-issue](https://github.com/ClouGence/cloudcanal-issue)  
9. 繁中 CDC 概念：[iT邦幫忙 - 变更资料撷取](https://ithelp.ithome.com.tw/articles/10362458)

---

*发布说明：配图均来自 Apache 项目官方资源（Apache-2.0），压缩后 <300KB，托管于本站静态目录。核实日期：2026-11-05（Asia/Singapore）。*
