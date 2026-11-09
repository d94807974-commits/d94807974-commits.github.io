---
title: "用 Debezium 把多個 MySQL 的行級變更同步到 Kafka：場景、部署與工程問題"
date: 2026-11-09T15:00:00+08:00
lastmod: 2026-11-09T15:00:00+08:00
draft: false
tags: ["CDC", "Debezium", "Kafka Connect", "MySQL", "binlog", "Kafka", "GTID", "資料同步", "維運"]
categories: ["工程實踐"]
description: "把多個 MySQL 的每一行變更（INSERT/UPDATE/DELETE）即時推到 Kafka：為什麼用 binlog CDC、為什麼選零程式碼的 Debezium、Kafka Connect 分散式部署的完整步驟（MySQL 參數、最小權限、內部 topic、外掛、密碼不落明文、systemd、多源 connector、表到 topic 路由），以及上線後真正會遇到的工程問題：重複與冪等、順序與分區、快照壓力、binlog 過期、新增表、schema history、型別精度與時區、server_id 衝突、主從切換。"
summary: "多個 MySQL → Kafka 行級 CDC 的落地筆記：場景與選型、Kafka Connect + Debezium 的部署流程（含多源、表級路由、密碼檔案化），以及至少一次投遞、保序、快照、binlog 保留、新增表、型別精度、主從切換等工程問題的現象、原因與做法。"
ShowToc: true
---

> 📊 **圖解版**：[架構圖、事件流、部署形態對比與工程問題清單](/visual/debezium-mysql-multi-source-kafka.html)（單頁 HTML，由 answer-me-with-html 產生）

> **一句話結論**：要把多個 MySQL 的每一行變更推到 Kafka，並且希望「寫的程式碼越少越好、接手的人好維護」，優先考慮 **Kafka Connect（分散式模式）+ Debezium MySQL Connector**：純設定、按 GTID 記位點、多源統一管理。真正的難點不在部署，而在**消費端冪等、保序、快照和 binlog 保留**這些工程問題。
>
> 文中的主機、密碼、庫表一律是佔位符或通用範例（`<mysql1-host>`、`<kafka>:9092`、`<內網IP>`、`<強密碼>`、`shop.orders`），版本號請按實際情況調整。設定項名稱以 Debezium 2.x/3.x 官方文件為準。

## 一、場景：為什麼需要行級 CDC

### 1.1 典型用途

「把資料庫的每一行變更變成一條訊息」這件事，在很多架構裡都會出現：

| 場景 | 需要什麼 |
|---|---|
| 即時資料倉儲 / 湖倉入湖 | 業務庫的增刪改以秒級延遲進入 OLAP（Doris、ClickHouse、Iceberg 等） |
| 搜尋索引同步 | 商品、訂單變更後，Elasticsearch / OpenSearch 中的文件跟著更新或刪除 |
| 快取失效 | 行變更後精確刪除對應的快取 key，而不是靠 TTL 碰運氣 |
| 多庫匯聚 | 分庫分表、多個業務系統的同類資料匯聚到一個 topic 或一張寬表 |
| 審計與回溯 | 保留「誰在什麼時候把哪一行從什麼改成了什麼」（before / after） |
| 微服務事件 | 用 outbox 表或業務表變更驅動下游服務，避免雙寫 |

### 1.2 定時拉取的問題

最樸素的做法是定時 `SELECT ... WHERE updated_at > ?`。它能跑，但有幾個繞不過去的問題：

- **延遲和壓力是一對矛盾**：拉得勤，源庫壓力大；拉得慢，延遲高。
- **感知不到刪除**：物理刪除的行查不出來，只能改成邏輯刪除，要動業務表結構。
- **依賴 `updated_at` 的紀律**：有人直接改庫、批次 SQL 忘了更新時間戳，資料就漏了。
- **丟失中間狀態**：兩次拉取之間一行被改了三次，只能看到最後一次。
- **一條 SQL 改 1000 行**，你很難知道到底改了哪 1000 行。

### 1.3 為什麼是 binlog

MySQL 的 binlog 在 `ROW` 格式下記錄了每一行的前後映像，而且是**交易提交後**才寫入的。基於 binlog 的 CDC 相當於把自己偽裝成一個從庫：

- 每一行變更都有一條記錄，刪除也有；
- 回滾的交易不會出現；
- 對源庫只是一個複製連線，壓力遠小於頻繁掃表；
- 帶有位點（file/pos 或 GTID），可以斷點續傳。

代價是：需要開啟並保留 binlog、需要複製權限、需要處理表結構變更（DDL）。這些都放到後面的工程問題裡講。

## 二、選型思路

我在選型時給自己定了幾條硬標準：

1. **維護活躍**：近一年有持續的釋出和提交，issue 有人回覆。
2. **零程式碼或接近零程式碼**：同步任務以後要交給別人維護，設定比程式碼好交接。
3. **支援 GTID**：位點按 GTID 記錄，主從切換後能接著同步。
4. **多源與表級路由**：同時接多個 MySQL，能指定哪些表、發到哪個 topic。
5. **密碼不落明文**：設定檔、程序命令列、日誌、管理介面裡都看不到密碼。

按這幾條比較了三類方案（結論來自一次單機實測加官方文件，數值不代表生產）：

| 方案 | 需要寫的程式碼 | GTID | 評價 |
|---|---|---|---|
| **Debezium** | 0 行，純設定 | 位點中帶 GTID 集合，主從切換可續傳 | 社群最活躍、文件最全；設定項多，需要懂 JVM 和 Kafka Connect |
| **Maxwell** | 0 行；自訂 topic 對映需要幾行 JS | 需手動設定 `gtid_mode=true`，預設按 file/pos | 輕量、JSON 簡單；但 `ignore_producer_error` 預設為 `true`（Kafka 傳送失敗只記日誌），生產必須改成 `false`；UPDATE 的 `old` 只包含被修改的列；位點存在源庫的 `maxwell` 庫，需要寫權限 |
| **自研**（go-mysql、python-mysql-replication 等庫） | 數百行起 | 庫支援，需自己實現 | 靈活、吞吐可以很高；但位點一致性、型別對映、DDL、快照、高可用、監控全部要自己長期維護，交接成本最高 |

實測裡還有幾條對所有方案都成立的結論：

- 程序被 `kill -9` 後重啟，都**不丟資料但會有重複**——這是至少一次（at-least-once）語義的正常表現。
- 一條 `UPDATE` 命中 1000 行，會產生 1000 條訊息；`ROLLBACK` 的交易不產生訊息。
- 「白名單之外的表不產生任何訊息」「指定表到指定 topic」「多源互不串位點」都能用設定實現。

所以結論是：**生產首選 Debezium**；如果只要最輕量的 JSON、能接受上面幾條需要手動調整的預設值，Maxwell 也可以；自研只在有特殊客製需求時作為起點。

## 三、兩種部署形態

Debezium 的 MySQL Connector 有兩種常見跑法：

| | Kafka Connect（分散式模式） | Debezium Server |
|---|---|---|
| 程序模型 | 一個 Connect 叢集執行多個 connector，每個源一個 connector | 一個程序只跑一個源 |
| 管理方式 | REST API 增刪改查、暫停恢復、檢視狀態 | 改 `application.properties` 後重啟 |
| 位點儲存 | Kafka 內部 topic（`connect-offsets`） | 預設本地檔案（也可設定到 Kafka 等） |
| 高可用 | 多個 worker 用同一個 `group.id` 組成叢集，task 自動漂移 | 需要外部程序管理和持久卷 |
| 適合 | 多個源、需要統一維運 | 單源、輕量、或者不想維護 Connect 叢集 |

**多源的場景推薦 Kafka Connect**：源越多，統一的 REST 管理和集中存放的位點就越省心，接手的人也只需要掌握一套操作。下面的部署流程以它為主，Debezium Server 的單源設定放在附錄。

整體架構（圖形版見[圖解版](/visual/debezium-mysql-multi-source-kafka.html)的「整體架構」面板）：

```text
  +-----------+    +-----------+    +-----------+
  |  MySQL 1  |    |  MySQL 2  |    |  MySQL N  |
  | (binlog,  |    | (binlog,  |    | (binlog,  |
  |  GTID on) |    |  GTID on) |    |  GTID on) |
  +-----+-----+    +-----+-----+    +-----+-----+
        |                |                |
        | replication protocol (one connector per source)
        v                v                v
  +---------------------------------------------------+
  |   Kafka Connect cluster (group.id=cdc-connect)    |
  |   worker A  <-- tasks rebalance -->  worker B     |
  |   connectors: mysql-src1 | mysql-src2 | ...       |
  |   SMT: RegexRouter / ExtractNewRecordState        |
  +-------------------------+-------------------------+
                            |
                            v
  +---------------------------------------------------+
  | Kafka                                             |
  |  cdc.shop.orders  cdc.shop.users  cdc.crm.*       |
  |  connect-configs / connect-offsets / -status      |
  |  schemahistory.src1 / schemahistory.src2          |
  +-------------------------+-------------------------+
                            |
          +-----------------+-----------------+
          v                 v                 v
     OLAP / lake      search index      cache invalidation
     (idempotent consumers: primary key + GTID/position)
```

## 四、部署流程

下面以 Linux 主機 + systemd 部署為例；Kubernetes 上可以用 Strimzi 等 Operator，思路相同。

### 4.1 MySQL 側準備（每個源都要做）

先確認 binlog 相關參數（以 MySQL 8.0 為例）：

```sql
SHOW VARIABLES WHERE Variable_name IN
 ('log_bin', 'binlog_format', 'binlog_row_image',
  'gtid_mode', 'enforce_gtid_consistency', 'binlog_expire_logs_seconds');
-- expected: log_bin=ON, binlog_format=ROW, binlog_row_image=FULL,
--           gtid_mode=ON, enforce_gtid_consistency=ON
```

幾點說明：

- `binlog_format=ROW` 和 `binlog_row_image=FULL` 是行級 CDC 的前提，否則拿不到完整的前後映像。
- 開啟 GTID 不是必須的，但強烈建議：位點會帶上 GTID 集合，主從切換後可以續傳。
- `binlog_expire_logs_seconds` 至少保留 **3～7 天**，要大於「同步鏈路最長可能停機時間 + 追平時間」。保留太短，停機一個長假回來位點已經被清理，就只能重新快照。
- 如果從**只讀副本**採集（推薦，避開主庫壓力），副本必須開啟 `log_replica_updates=ON`（8.0.26 之前叫 `log_slave_updates`），否則副本自己的 binlog 裡沒有複製過來的變更。

建立專用帳號：

```sql
CREATE USER 'cdc_min'@'<connect-ip>' IDENTIFIED BY '<強密碼>';
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT
  ON *.* TO 'cdc_min'@'<connect-ip>';
```

各權限的用途：

| 權限 | 用途 |
|---|---|
| `REPLICATION SLAVE` | 以複製協議讀取 binlog |
| `REPLICATION CLIENT` | 執行 `SHOW MASTER STATUS` / `SHOW BINARY LOGS` 等查詢位點 |
| `SELECT` | 快照時讀取表資料 |
| `RELOAD` | 快照時執行 `FLUSH TABLES ... WITH READ LOCK` 等全域鎖操作 |
| `SHOW DATABASES` | 發現庫表 |

如果不需要全量快照（例如 `snapshot.mode=no_data` 且 `snapshot.locking.mode=none`），可以按官方文件進一步收斂：去掉 `RELOAD`，把 `SELECT` 收窄到需要的庫。是否需要 `LOCK TABLES` 等權限，以所用快照模式的文件為準。

### 4.2 Kafka 側準備

Kafka Connect 分散式模式依賴三個內部 topic，生產上建議**預先建立**，而不是交給自動建立：

```bash
kafka-topics.sh --bootstrap-server <kafka>:9092 --create --topic connect-configs \
  --partitions 1  --replication-factor 3 --config cleanup.policy=compact
kafka-topics.sh --bootstrap-server <kafka>:9092 --create --topic connect-offsets \
  --partitions 25 --replication-factor 3 --config cleanup.policy=compact
kafka-topics.sh --bootstrap-server <kafka>:9092 --create --topic connect-status \
  --partitions 5  --replication-factor 3 --config cleanup.policy=compact

# schema history topic: one per source, single partition, never expire
kafka-topics.sh --bootstrap-server <kafka>:9092 --create --topic schemahistory.src1 \
  --partitions 1 --replication-factor 3 --config retention.ms=-1
```

- `connect-configs` 必須單分區；三個 topic 都必須是 `compact`。
- **schema history topic 必須單分區、永不過期**，原因見第五節。
- 業務 topic（如 `cdc.shop.orders`）也建議預建，自己決定分區數、副本數和保留期。分區數一旦定下，就不要隨意增加（見第五節「順序」）。

### 4.3 安裝 Debezium 外掛

用 Kafka 自帶的 `connect-distributed.sh` 即可，把 Debezium 外掛解壓到 `plugin.path`：

```bash
mkdir -p /opt/connect-plugins && cd /opt/connect-plugins
curl -LO https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/<version>/debezium-connector-mysql-<version>-plugin.tar.gz
tar xzf debezium-connector-mysql-<version>-plugin.tar.gz
```

注意 Debezium 版本與 Kafka / Java 版本的相容關係，以釋出說明為準。

### 4.4 Worker 設定

`/opt/kafka/config/connect-distributed.properties`：

```properties
bootstrap.servers=<kafka1>:9092,<kafka2>:9092
group.id=cdc-connect
plugin.path=/opt/connect-plugins

config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status
config.storage.replication.factor=3
offset.storage.replication.factor=3
status.storage.replication.factor=3

key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false

# read secrets from files instead of plain text in connector configs
config.providers=file
config.providers.file.class=org.apache.kafka.common.config.provider.FileConfigProvider

# REST API on the internal network only
listeners=http://<內網IP>:8083
```

- `schemas.enable=false` 讓訊息只包含資料本身，體積小、消費端好處理；如果下游需要 schema，建議用 Avro / Protobuf + Schema Registry，而不是 JSON 內嵌 schema。
- REST 介面**沒有預設鑑權**，任何能訪問 8083 的人都能讀寫 connector 設定，所以只監聽內網，並用防火牆限制來源。

### 4.5 密碼不落明文

目標是：connector 設定、程序命令列、日誌、REST 返回結果裡都看不到密碼。

```bash
useradd -r -s /sbin/nologin kafkaconnect
install -d -m 0700 -o kafkaconnect /opt/secrets
printf 'password=%s\n' '<mysql1密碼>' > /opt/secrets/mysql1.properties
chown kafkaconnect /opt/secrets/*.properties && chmod 0400 /opt/secrets/*.properties
```

connector 設定裡只寫引用：

```json
"database.password": "${file:/opt/secrets/mysql1.properties:password}"
```

- `FileConfigProvider` 在執行時解析這個佔位符；透過 REST `GET /connectors/<name>/config` 看到的仍然是佔位符本身，而不是密碼。
- Kubernetes 上，把 Secret 以 volume 方式掛到同一路徑（`defaultMode: 0400`）即可；也可以換成 Vault 等外部金鑰系統的 config provider。
- **為什麼不直接用環境變數**：環境變數對同一使用者和 root 可以透過 `/proc/<pid>/environ` 讀到，`docker inspect`、`kubectl describe` 也可能把它顯示出來。檔案掛載 + 專用帳號 + 最小檔案權限更可控。
- Kafka 側如果開了 SASL，producer / consumer 的 JAAS 密碼同樣用 `${file:...}` 引用，不要寫進設定。

### 4.6 用 systemd 託管 Connect

`/etc/systemd/system/kafka-connect.service`：

```ini
[Unit]
Description=Kafka Connect (Debezium)
After=network.target

[Service]
User=kafkaconnect
Environment=KAFKA_HEAP_OPTS=-Xms2g -Xmx2g
ExecStart=/opt/kafka/bin/connect-distributed.sh /opt/kafka/config/connect-distributed.properties
Restart=always
RestartSec=10
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now kafka-connect
curl -s http://<內網IP>:8083/connector-plugins | grep -i mysql
```

要高可用，就在兩臺以上的機器上用**同一個 `group.id`** 各起一個 worker，它們會自動組成叢集；某臺機器掛掉時，它上面的 task 會被重新分配到其它 worker。MySQL connector 每個只有一個 task（binlog 是單流），所以「高可用」指的是故障轉移，而不是並行加速。

### 4.7 多個源：每個源一個 connector

`mysql-src1.json`：

```json
{
  "name": "mysql-src1",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "<mysql1-host>",
    "database.port": "3306",
    "database.user": "cdc_min",
    "database.password": "${file:/opt/secrets/mysql1.properties:password}",
    "database.server.id": "9301",
    "topic.prefix": "src1",
    "database.include.list": "shop,crm",
    "table.include.list": "shop\\.orders,shop\\.users,crm\\.customer_.*",
    "schema.history.internal.kafka.bootstrap.servers": "<kafka>:9092",
    "schema.history.internal.kafka.topic": "schemahistory.src1",
    "include.schema.changes": "false",
    "decimal.handling.mode": "string",
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "src1\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "cdc.$1.$2"
  }
}
```

關鍵項：

- `database.server.id`：這個 connector 偽裝成從庫時使用的 server id，**在整個 MySQL 複製拓撲裡必須全域唯一**，不能和任何真實從庫或其它 CDC 實例重複。
- `topic.prefix`：邏輯名稱，會出現在預設 topic 名（`<prefix>.<db>.<table>`）、訊息的 `source.name` 和指標裡。**上線後不要改**，改了會被視為一個新的源。
- `database.include.list` / `table.include.list`：都是**正規表示式**，按「庫名\\.表名」匹配。JSON 裡反斜槓本身要轉義，所以寫成 `shop\\.orders`，實際生效的正則是 `shop\.orders`。
- 預設 topic 名是 `src1.shop.orders`，`RegexRouter` 把它改成 `cdc.shop.orders`。這裡 `$1`、`$2` 是正則捕獲組，不是設定佔位符。
- `decimal.handling.mode=string`：見第五節「型別精度」。

第二個源複製一份，至少要改：

| 項 | src1 | src2 |
|---|---|---|
| `name` | `mysql-src1` | `mysql-src2` |
| `database.hostname` | `<mysql1-host>` | `<mysql2-host>` |
| `database.password` | `.../mysql1.properties:password` | `.../mysql2.properties:password` |
| `database.server.id` | `9301` | `9302` |
| `topic.prefix` | `src1` | `src2` |
| `schema.history.internal.kafka.topic` | `schemahistory.src1` | `schemahistory.src2` |
| 庫表列表、`transforms.route.regex` | 按需 | 按需（字首改成 `src2`） |

註冊：

```bash
curl -X POST -H 'Content-Type: application/json' --data @mysql-src1.json http://<內網IP>:8083/connectors
curl -X POST -H 'Content-Type: application/json' --data @mysql-src2.json http://<內網IP>:8083/connectors
```

各源的位點由 Kafka Connect 按 connector 分別儲存在 `connect-offsets` 裡，互不影響。

### 4.8 指定表到指定 topic

**單表對映**（多個路由按順序串聯）：

```json
"transforms": "orders,users",
"transforms.orders.type": "org.apache.kafka.connect.transforms.RegexRouter",
"transforms.orders.regex": "src1\\.shop\\.orders",
"transforms.orders.replacement": "topic_orders",
"transforms.users.type": "org.apache.kafka.connect.transforms.RegexRouter",
"transforms.users.regex": "src1\\.shop\\.users",
"transforms.users.replacement": "topic_users"
```

**正則合併**：把按年分表的 `orders_2024`、`orders_2025`……合併到一個 topic：

```json
"table.include.list": "shop\\.orders,shop\\.orders_20[0-9]{2}",
"transforms": "hist",
"transforms.hist.type": "org.apache.kafka.connect.transforms.RegexRouter",
"transforms.hist.regex": "src1\\.shop\\.orders_20[0-9]{2}",
"transforms.hist.replacement": "topic_orders_history"
```

注意幾點：

- `RegexRouter` 的正則是**整串匹配**，`src1\\.shop\\.orders` 不會誤傷 `orders_2024`。
- `table.include.list` 和 `table.exclude.list` **不能同時使用**，庫級的 include / exclude 同理。
- 合併到同一個 topic 的多張表，key 的結構必須一致（主鍵列相同），否則下游按 key 處理會出問題。Debezium 也提供 `io.debezium.transforms.ByLogicalTableRouter`，可以在 key 裡額外加上來源表，用於分表合併。
- 白名單之外的表不會產生訊息，也不會建立 topic。

**只想要「變更後的行」**：預設訊息是包含 `before`、`after`、`source`、`op` 的信封結構。下游只關心最新狀態時，可以加 `ExtractNewRecordState` 把它展平：

```json
"transforms": "unwrap,route",
"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
"transforms.unwrap.add.fields": "op,source.ts_ms,source.gtid",
"transforms.unwrap.delete.tombstone.handling.mode": "rewrite"
```

`rewrite` 會把刪除事件改寫成帶 `__deleted=true` 的記錄，下游可以據此刪除；`add.fields` 把操作型別和 GTID 保留下來，供冪等使用。轉換按 `transforms` 列出的順序執行。

### 4.9 驗證

```bash
# connector and task should both be RUNNING
curl -s http://<內網IP>:8083/connectors/mysql-src1/status

kafka-console-consumer.sh --bootstrap-server <kafka>:9092 \
  --topic cdc.shop.orders --from-beginning --property print.key=true
```

建議至少驗證這幾項：

1. 單行 `INSERT` / `UPDATE` / `DELETE` 分別產生 `op=c` / `op=u` / `op=d` 的訊息，UPDATE 帶完整的 `before` 和 `after`。
2. 一條 `UPDATE` 命中 1000 行，產生 1000 條訊息，主鍵集合與預期一致。
3. `BEGIN; ...; ROLLBACK;` 不產生任何訊息。
4. 白名單外的表寫入後沒有訊息。
5. 同一主鍵的多次修改落在同一分區且順序正確。
6. 強制重啟 Connect 程序後，不丟訊息（允許重複）。

預設情況下，刪除事件後面還會跟一條 value 為 `null` 的墓碑（tombstone）訊息，用於 compact topic 的清理。消費端要能正確處理 `null` value。

### 附：Debezium Server 單源設定

只有一個源、不想維護 Connect 叢集時，可以用 Debezium Server。`conf/application.properties`：

```properties
debezium.sink.type=kafka
debezium.sink.kafka.producer.bootstrap.servers=<kafka>:9092
debezium.sink.kafka.producer.key.serializer=org.apache.kafka.common.serialization.StringSerializer
debezium.sink.kafka.producer.value.serializer=org.apache.kafka.common.serialization.StringSerializer

debezium.source.connector.class=io.debezium.connector.mysql.MySqlConnector
debezium.source.database.hostname=<mysql1-host>
debezium.source.database.port=3306
debezium.source.database.user=cdc_min
debezium.source.database.password=${MYSQL1_CDC_PASSWORD}
debezium.source.database.server.id=9301
debezium.source.topic.prefix=src1
debezium.source.database.include.list=shop
debezium.source.table.include.list=shop\\.orders,shop\\.users
debezium.source.decimal.handling.mode=string

debezium.source.offset.storage=org.apache.kafka.connect.storage.FileOffsetBackingStore
debezium.source.offset.storage.file.filename=/data/src1/offsets.dat
debezium.source.offset.flush.interval.ms=10000
debezium.source.schema.history.internal=io.debezium.storage.file.history.FileSchemaHistory
debezium.source.schema.history.internal.file.filename=/data/src1/schema_history.dat

debezium.transforms=orders,users
debezium.transforms.orders.type=org.apache.kafka.connect.transforms.RegexRouter
debezium.transforms.orders.regex=src1\\.shop\\.orders
debezium.transforms.orders.replacement=topic_orders
debezium.transforms.users.type=org.apache.kafka.connect.transforms.RegexRouter
debezium.transforms.users.regex=src1\\.shop\\.users
debezium.transforms.users.replacement=topic_users
```

- 一個程序一個源；多個源就多個目錄、多個程序，`server.id`、`topic.prefix`、offset 檔案、schema history 檔案全部獨立。
- `${MYSQL1_CDC_PASSWORD}` 由 Quarkus 在啟動時從環境變數展開。上面說過環境變數的風險，更穩妥的做法是由啟動指令碼從 0400 的金鑰檔案讀取後再注入，並用專用帳號執行。
- offset 和 schema history 檔案必須放在持久化儲存上，丟了就等於丟了位點。

## 五、工程問題（重點）

部署只是開始。下面這些問題，基本每條鏈路上線後都會遇到。

### 5.1 重複訊息：至少一次投遞

- **現象**：Connect 程序被強殺或 worker 重平衡後，消費端看到一段重複的訊息。
- **原因**：Debezium 把訊息發到 Kafka 後，位點是**週期性**提交的（`offset.flush.interval.ms`，預設 60 秒）。崩潰時最後一次提交之後已經發出的訊息，重啟後會再發一遍。這是設計上的 at-least-once。
- **做法**：
  - 消費端必須冪等。最常見的是**按主鍵 upsert**；需要嚴格去重時，用「主鍵 + GTID（或 binlog file/pos/row）」判斷是否處理過。
  - 落 OLAP 時使用支援主鍵覆蓋的表模型（如 Doris 的 Unique Key、ClickHouse 的 ReplacingMergeTree），天然吸收重複。
  - 新版 Kafka Connect 支援 source connector 的 exactly-once（`exactly.once.source.support`），但需要 Kafka 和 connector 兩邊都滿足條件，不要預設依賴它，消費端冪等仍然是底線。

### 5.2 順序與分區

- **現象**：同一行先 UPDATE 後 DELETE，下游卻先收到 DELETE，最後留下一行「復活」的資料。
- **原因**：Kafka 只保證**單個分區內**有序。Debezium 預設用主鍵作為訊息 key，同一主鍵會進同一分區，所以同一行的變更是有序的；但：
  - 中途**增加 topic 分區數**，key 到分區的對映會變，切換前後的訊息可能被不同消費者並行處理；
  - 無主鍵的表 key 為空，訊息會被分散到各分區，順序無法保證；
  - 多個源合併到一個 topic 時，不同源裡主鍵相同的行會被當成同一個 key。
- **做法**：
  - 業務 topic 預建時就把分區數定好，之後不要隨意增加；確實要擴，就新建 topic 並做一次切換。
  - 無主鍵的表，用 `message.key.columns` 指定業務唯一鍵作為 key，或者在源頭補主鍵。
  - 多源合併到同一個 topic 時，在 key 裡加上來源（例如用 `ByLogicalTableRouter` 的 key 增強），或者乾脆每個源一個 topic。
  - 跨表、跨主鍵的全域順序，Kafka 本身不提供，不要在消費端假設它。

### 5.3 首次快照對源庫的壓力

- **現象**：新建 connector 後，源庫 CPU、IO 飆升，甚至短暫出現鎖等待。
- **原因**：預設 `snapshot.mode=initial` 會先把白名單內的表全量讀一遍，再從快照時刻的位點開始讀 binlog。大表全表掃描、以及部分模式下的全域讀鎖，都會影響源庫。
- **做法**：
  - 從只讀副本採集。
  - 選擇合適的快照模式：只要增量時用 `no_data`（舊版本叫 `schema_only`）；需要全量時用 `initial`，並在低峰期進行。
  - 用 `snapshot.locking.mode` 控制鎖的行為（如 `minimal`、`none`），理解它對一致性的影響後再選。
  - 用 `snapshot.fetch.size`、`snapshot.max.threads` 等參數控制節奏；超大表可以先 `no_data` 啟動增量，再用增量快照（見 5.5）分批補歷史。

### 5.4 binlog 被清理，位點失效

- **現象**：connector 停了幾天，重啟後報錯，大意是需要的 binlog 已經不在伺服器上。
- **原因**：`binlog_expire_logs_seconds` 到期，或者有人手動 `PURGE BINARY LOGS`，connector 記錄的位點之後的 binlog 已經沒了。
- **做法**：
  - 保證 binlog 保留時長 > 最長停機時間 + 追平時間，並對「connector 停止時長」設定告警。
  - 萬一發生，只能重新快照：可以用 `snapshot.mode=when_needed` 讓 connector 在位點失效時自動重新快照，或者人工刪除位點後重建。重新快照意味著下游會收到一遍全量資料，消費端的冪等此時尤為重要。

### 5.5 新增表只有增量，沒有歷史

- **現象**：把新表加進 `table.include.list` 之後，只收到了之後的變更，歷史資料沒有。
- **原因**：快照只在 connector 第一次啟動時執行；之後修改白名單，新表預設只同步增量。
- **做法**：使用**增量快照（incremental snapshot）**。在源庫建一張 signal 表，設定 `signal.data.collection`（例如 `shop.debezium_signal`，並加入 `table.include.list`），然後插入一條訊號：

```sql
INSERT INTO shop.debezium_signal (id, type, data)
VALUES ('snapshot-orders-2024', 'execute-snapshot',
        '{"data-collections": ["shop.orders_2024"], "type": "incremental"}');
```

增量快照按主鍵分塊讀取，並與 binlog 流交錯去重，不需要長時間鎖表，也不會中斷增量同步。也可以透過 Kafka 訊號 topic 觸發，具體以文件為準。

### 5.6 schema history topic 被刪或被清理

- **現象**：connector 無法啟動，報 schema history 缺失或不完整。
- **原因**：Debezium 需要按 binlog 位點回放 DDL，才能正確解析某個時間點的行資料。這些 DDL 存在 schema history topic 裡。如果這個 topic 設定了過期時間、被 compact、被多分區化，或者被誤刪，歷史就不完整了。
- **做法**：
  - 每個 connector 獨立一個 schema history topic，單分區、`retention.ms=-1`、不 compact。
  - 把它和 `connect-*` 內部 topic 一起列為「禁止刪除」的資源。
  - 真的丟了，可以用 `snapshot.mode=recovery`（舊版叫 `schema_only_recovery`）從當前表結構重建，前提是丟失期間沒有發生過 DDL；不確定就重新快照。
  - 可以開啟 `schema.history.internal.store.only.captured.tables.ddl=true`，只存白名單內表的 DDL，減小體積。

### 5.7 型別精度、時區和大欄位

- **DECIMAL**：預設 `decimal.handling.mode=precise`，在 JSON 裡會被編碼成 base64 位元組，消費端不好處理；設成 `double` 會丟精度。金額類欄位建議用 `string`，消費端按 decimal 解析。
- **時間精度**：保持預設的 `time.precision.mode=adaptive_time_microseconds`。**不要改成 `connect`**，否則 `DATETIME(6)` 會被截斷到毫秒，實測可復現。
- **時區**：`DATETIME` 沒有時區，Debezium 按「字面值當作 UTC」轉成 epoch 數值；`TIMESTAMP` 會被轉換成 UTC 的 ISO 字串。消費端要分別處理，不要一律加 8 小時。
- **JSON**：以字串輸出，消費端自行解析。
- **BLOB / BINARY**：用 `binary.handling.mode`（`bytes`、`base64`、`hex` 等）選擇編碼；大欄位會讓訊息變大，必要時調大 Kafka 的 `max.request.size` 和 topic 的 `max.message.bytes`，或者乾脆排除這些列（`column.exclude.list`）。

### 5.8 server_id 衝突，互相踢下線

- **現象**：兩個 connector（或 connector 與某個真實從庫）反覆斷開重連，日誌裡出現連線被伺服器關閉的錯誤。
- **原因**：MySQL 以 server_id 識別複製客戶端，同一個 server_id 的新連線會把舊連線踢掉。
- **做法**：給每個 CDC 實例分配一段專用的 server_id 號段（例如 9301～9399），登記在案；複製 connector 設定時第一件事就是改 `database.server.id`。

### 5.9 主從切換

- **現象**：源庫發生主從切換後，connector 無法繼續，或者從錯誤的位置開始讀。
- **原因**：file/pos 位點只在同一臺伺服器上有意義；換到新主庫後，檔名和偏移量對不上。
- **做法**：
  - 開啟 GTID。Debezium 的位點裡帶有已處理的 GTID 集合，連線到新主庫後會按 GTID 找到續傳位置。
  - 新主庫必須**仍保留**對應 GTID 之後的 binlog，並且開啟了 `log_replica_updates`，否則同樣會進入 5.4 的情況。
  - 連線地址用 VIP 或域名，而不是寫死某臺機器。
  - 需要過濾某些來源的 GTID 時，可以用 `gtid.source.includes` / `gtid.source.excludes`。

### 5.10 安全

- CDC 帳號只授予必要的權限，按來源 IP 限制。
- 所有密碼都透過 config provider 從檔案或金鑰系統讀取，設定、日誌、REST 返回裡只有佔位符。
- Connect 的 REST 介面只對內網開放；需要對外時放在帶鑑權的閘道後面。
- Kafka 使用 SASL_SSL，Connect worker、connector 內部的 producer / consumer、schema history 的 producer / consumer 都要設定認證，密碼同樣用 `${file:...}`。
- MySQL 連線開啟 TLS（`database.ssl.mode`）。
- binlog 裡有全量的行資料，CDC topic 的訪問權限應當和源庫同等對待。

## 六、日常維運

| 操作 | 命令或做法 |
|---|---|
| 修改設定（熱更新） | `PUT /connectors/<name>/config`，提交完整設定；connector 會自動重啟，不需要重啟程序 |
| 暫停 / 恢復 | `PUT /connectors/<name>/pause`、`PUT /connectors/<name>/resume` |
| 重啟失敗的 task | `POST /connectors/<name>/restart?includeTasks=true&onlyFailed=true` |
| 檢視狀態 | `GET /connectors/<name>/status`；connector 或 task 為 `FAILED` 時立即告警 |
| 同步延遲 | JMX 指標 `debezium.mysql:type=connector-metrics,context=streaming,server=<topic.prefix>` 中的 `MilliSecondsBehindSource` |
| binlog 保留 | 監控源庫最早 binlog 時間與 connector 當前位點的差距 |
| 升級 | 先在測試環境驗證新外掛；生產上逐臺替換 `plugin.path` 下的外掛並滾動重啟 worker，位點在 Kafka 中，不會丟 |
| 下線一個源 | 刪除 connector 之前先確認下游已消費完；schema history topic 和位點按需保留一段時間 |

**上線順序建議**：

1. 測試環境用一個源、一兩張表跑通，按 4.9 的清單逐項驗證，包括強殺重啟。
2. 在接近生產的資料量上觀察首次快照的耗時和對源庫的影響，確定快照策略。
3. 消費端完成冪等與墓碑訊息處理，並在重複注入的情況下驗證結果正確。
4. 逐個接入生產源，每接一個觀察一段時間的延遲與錯誤率。
5. 把 server_id 號段、topic 命名規則、金鑰檔案位置、告警項寫進維運文件，方便交接。

## 七、小結

- 行級 CDC 解決的是「每一行變更都不漏、刪除也能感知、對源庫壓力小」的問題，binlog 是 MySQL 上最可靠的來源。
- 多源 MySQL → Kafka，**Kafka Connect + Debezium** 是寫程式碼最少、最好交接的組合：每個源一個 connector，白名單 + `RegexRouter` 決定哪些表進哪個 topic，`FileConfigProvider` 讓密碼不出現在設定裡。
- 部署按清單走並不難，真正需要提前設計的是：**消費端冪等、按主鍵保序、快照策略、binlog 保留、schema history 保護、型別精度、server_id 管理和 GTID 主從切換**。
- 這些問題想清楚之後，加一個新源通常只是「複製一份 JSON、改五六個欄位、POST 一下」。

## 參考

- Debezium MySQL Connector：<https://debezium.io/documentation/reference/stable/connectors/mysql.html>
- Debezium Server：<https://debezium.io/documentation/reference/stable/operations/debezium-server.html>
- 增量快照與訊號：<https://debezium.io/documentation/reference/stable/configuration/signalling.html>
- ExtractNewRecordState：<https://debezium.io/documentation/reference/stable/transformations/event-flattening.html>
- Topic 路由：<https://debezium.io/documentation/reference/stable/transformations/topic-routing.html>
- Kafka Connect（分散式模式、RegexRouter、ConfigProvider）：<https://kafka.apache.org/documentation/#connect>
- Maxwell 設定與 producer：<https://maxwells-daemon.io/config/>、<https://maxwells-daemon.io/producers/>
