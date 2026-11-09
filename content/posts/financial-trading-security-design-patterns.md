---
title: "讓每一筆錢都可證明：金融交易系統安全設計模式全景"
date: 2026-11-04T14:36:21+08:00
lastmod: 2026-11-05T13:25:00+08:00
draft: false
tags: ["安全架構", "金融科技", "設計模式", "支付", "交易系統", "Go"]
summary: "從孟加拉國央行、Knight Capital、Capital One，以及臺灣第一銀行 ATM 盜領與遠東商銀 SWIFT 遭駭五起事故出發，提出資金動作的六個不變數，並整理 SCA、所見即所簽、簽名防重放、冪等記帳、複式帳本、Maker-Checker、HSM/信封加密、事前風控與熔斷等 13 個安全設計模式，並對照臺灣金管會、銀行公會、證交所與香港金管局、證監會的繁體中文監管檔案，附可編譯 Go 範例與落地清單。"
ShowToc: true
---

> 📊 **圖解版**：[六個不變量、轉帳路徑、冪等記帳與 STRIDE 對照](/visual/financial-trading-security-design-patterns.html)（單頁 HTML，由 answer-me-with-html 產生）

> 本文討論的是金融交易類軟體（支付、轉帳、證券/期貨下單、清結算、帳務核心）的**安全設計模式**。文中的事實、數字與法規條款均來自文末列出的公開資料；程式碼為可編譯的 Go 示意實現（`go vet` 通過），用於說明思路而非直接上線。
>
> 本文首發後，補充了繁體中文資料並重寫：臺灣金管會新聞稿與裁罰公告、銀行公會《金融機構辦理電子銀行業務安全控管作業基準》、臺灣證交所《建立證券商資通安全檢查機制》、香港金管局與證監會的公告和指引，以及中央社、iThome 對兩起臺灣事故的報道。繁體原文保留繁體字。


<figure>
<img src="/images/financial-trading-security-design-patterns/padlock.jpg" alt="掛鎖" loading="lazy" width="450">
<figcaption>圖片：Alyson Hurt / <a href="https://commons.wikimedia.org/wiki/File:Padlock.jpg">Wikimedia Commons</a>，<a href="https://creativecommons.org/licenses/by/2.0/">CC BY 2.0</a>。金融安全不只是一把鎖，而是可證明的閉環。</figcaption>
</figure>

## 引子：五起事故，五種"正確的失敗"

先看三個被反覆引用、但常被講錯重點的案例，再看兩起發生在臺灣、繁體中文資料很完整的案例。

**2016 年孟加拉國央行事件。** 攻擊者攻破孟加拉國央行網路、觀察轉帳操作並拿到憑證後，通過 SWIFT 向紐約聯儲發出三十多筆指令，試圖從其帳戶轉走 9.51 億美元。5 筆共 1.01 億美元得手，其中流向斯里蘭卡的 2000 萬美元因收款方名稱拼寫錯誤（"Fandation"）被攔下，流向菲律賓的 8100 萬美元截至 2025 年只追回約 1500 萬；其餘 30 筆約 8.5 億美元被紐約聯儲阻止。作案時間選在週末並疊加春節假期，延誤了止付。事後 SWIFT 推出了客戶安全計劃（CSP）。

這起事件最值得玩味的是：**從系統視角看，那些指令都是"合法"的**——憑證真實，報文格式正確。認證沒有失敗，失敗的是"認證之後的一切"。

**2012 年 Knight Capital 事件。** 根據美國 SEC 的處罰令，8 月 1 日開盤後約 45 分鐘裡，Knight 的路由系統 SMARS 在處理 212 筆小額零售訂單時向市場發出了數百萬筆訂單，在 154 只股票上成交 400 萬筆、超過 3.97 億股，形成約 35 億美元的淨多頭，最終虧損超過 4.6 億美元。根因是一段 2003 年就停用卻仍可呼叫的 "Power Peg" 舊程式碼、一個被新功能複用的舊標誌位，以及一次漏掉 8 臺伺服器中 1 臺的部署；此外還缺少訂單比對、停機流程和有效告警（後文逐一展開）。這是 SEC 依據市場準入規則（Rule 15c3-5）採取的第一起執法行動，罰款 1200 萬美元。

Knight 沒有被"黑"。它說明了金融系統安全的另一半：**完整性和可用性的敵人，常常是你自己的程式碼與釋出流程。**

**2019 年 Capital One 事件。** Capital One 公告稱，攻擊者利用一個配置漏洞，於 2019 年 3 月 22—23 日獲取資料，涉及美國約 1 億、加拿大約 600 萬人，包括約 14 萬個社會安全號和約 8 萬個關聯銀行帳號，未涉及卡號和登入憑證。美國司法部的通報提到，被告用自己編寫的工具掃描 AWS 上配置錯誤的帳戶並藉此入侵。美國貨幣監理署（OCC）對其罰款 8000 萬美元，理由是在把重要 IT 業務遷移到公有云之前，**未能建立有效的風險評估流程**。

三起事故的共同點是：**單點控制都"在"，只是沒有形成閉環。**

**2016 年第一銀行 ATM 盜領事件。** 據 iThome 彙整的調查局鑑識結果，2016 年 7 月 10、11 日清晨，車手在第一銀行臺北、臺中 22 家分行的 41 臺 ATM 上，不插卡、不輸密碼就領走了 8,327 萬 7,600 元新臺幣。入侵路徑大致是：

1. 攻擊者先控制一銀**倫敦分行**一臺"鎖在鐵櫃中的電話錄音伺服器主機"，以此為跳板進入總行內網；
2. 7 月 4 日，經由 **ATM 軟體派送伺服器**發出一個"更新包"，把 ATM 的 Telnet 服務改為自動開啟，ATM 重開機後即可被遠端登入；
3. 7 月 9 日，派送木馬 `cnginfo.exe`、`cngdisp.exe` 等，吐鈔時每次 60 張；
4. 事後用批處理 `cleanup.bat` 呼叫微軟的 `sdelete.exe` 刪除木馬和日誌。調查局是在一臺"自毀程式執行失效"的 ATM 上才找到樣本。

調查局還發現，這些惡意程式只在 2016 年 7 月有效，逾期失效，這是其鑑識實驗室"成立十年來，少見會指定發作月份的惡意程式"。警方 7 月 18 日逮捕 3 名洗錢嫌犯，追回 6,024 萬元，兩天後又在內湖西湖公園找到一千多萬元，"僅剩數百萬元未尋獲"。

這起事件裡最值得記住的一句話，來自調查局主管私下的透露：一銀保護客戶資料的資料庫防護"非常嚴謹"，但內部系統之間可能基於「都是自己人」的心態，海外分行連回總行的系統"往往只需要簡單的帳號、密碼就可以順利登入"。**被攻破的不是 ATM，而是"內網互信"這個假設；被濫用的也不是漏洞，而是正常的軟體更新通道。**

**2017 年遠東商銀 SWIFT 遭駭事件。** 金管會 2017 年 12 月 12 日的裁罰公告寫道：遠東商銀於 2017 年 10 月 3 日及 10 月 5 日發現 SWIFT 系統遭電腦病毒及駭客入侵，產生虛偽交易匯出款項，總額約 6,010 萬美元（合新臺幣 18 億元），"嗣後已追回多數款項"，損失在 16 萬美元以內。金管會認定其違反銀行法第 45 條之 1 第 1 項，核處新臺幣 800 萬元罰款，列出的缺失包括：

- 資安防禦機制未完整建置；
- 系統管理者帳號管理欠當；
- 未落實強化 SWIFT 系統安全；
- 未有效傳達法令；
- 未發揮內部控制第三道防線功能。

中央社的報道更具體。金管會檢查局副局長葉淑媛說，遠銀"沒有符合最小授權原則，造成高權限的SWIFT（環球財務通訊系統）帳號、密碼遭盜取"。金管會在孟加拉央行事件後已於 2016 年 9 月發函銀行公會，要求各銀行做好網段隔離，但遠銀「只做半套」：

> 工作站的做法有合規，但主機卻沒有做到網段的隔離。

iThome 還記錄了金管會的一段解釋：遠銀為何沒有人工稽核每筆 SWIFT 交易？因為"如果作好SWIFT系統實體隔離，交易均由系統自動處理完成，因此沒有人工再次稽核的程式"。也就是說，**整個控制鏈建立在"隔離一定有效"這個前提上，前提一破，後面沒有第二道閘。**

五起事故放在一起，共同點更清楚了：**單點控制都"在"，只是沒有形成閉環。**

## 一、核心論點：給每一個"動錢的動作"定義不變數

傳統安全敘事喜歡講邊界。金融系統的經驗恰恰相反：邊界會被穿透，憑證會被盜，程式碼會出錯。真正可依賴的，是圍繞**每一個資金動作**建立的一組不變數：

1. **可認證（Authenticated）**：發起者是誰，必須有多因素證據，而不僅是"會話還在"。
2. **內容繫結（Bound）**：被授權的是"這一筆、這個金額、這個收款方"，任何欄位被改動，授權即失效。
3. **恰好一次（Once）**：無論網路如何重試、訊息如何重複，帳上只發生一次。
4. **不可篡改地留痕（Recorded）**：只追加的流水，餘額可由流水推導，可以獨立對帳。
5. **有界（Bounded）**：單筆、單日、頻率、總敞口都有上限，而且在事前檢查。
6. **可停止（Stoppable）**：任何時候都存在一個比故障傳播更快的"剎車"。

孟加拉國央行事件是第 1 條成立而第 5 條缺失；Knight 缺的是第 4、5、6 條；Capital One 則是這一切的"環境前提"出了問題。遠銀和孟加拉央行幾乎是同一個劇本：憑證是真的、報文是對的，缺的是第 5 條的事前上限和獨立複核；一銀則說明第 4 條的"留痕"必須離開被攻擊的主機，否則 `sdelete` 一跑，證據就沒了。下面的模式目錄，就是把六個不變數落到架構上的具體手段。

一筆轉帳在系統中的理想路徑如下：

```text
 客戶端/櫃員端                        閘道層                            交易核心
┌──────────────┐  ①簽名+時間戳+nonce ┌──────────────────┐            ┌──────────────────────┐
│ SCA:知識/持有 │ ──────────────────▶ │ 驗簽 / 防重放     │            │ 冪等鍵查重(唯一約束)  │
│ /生物 + 裝置  │                     │ 限流 / 反自動化   │──②────────▶│ 事前風控:限額/頻率/敞口│
│ 動態連結碼    │ ◀── 展示金額+收款方 │ BOLA/BFLA 授權    │            │ Maker-Checker(大額)   │
└──────────────┘                     └──────────────────┘            │ 複式記帳:借貸平衡      │
        ▲                                     │                       └──────────┬───────────┘
        │③ 結果(可安全重試)                    │ 審計事件                        │ 只追加流水
        │                                     ▼                                  ▼
        │                           ┌──────────────────┐          ┌──────────────────────────┐
        └───────────────────────────│ 檢測與響應(SIEM)  │◀─────────│ 對帳:內部流水 vs 外部清算 │
                                    │ 告警→自動熔斷     │─Kill────▶│ 敞口即時彙總 / 熔斷開關  │
                                    └──────────────────┘  Switch  └──────────────────────────┘
          金鑰:HSM/KMS 託管主金鑰,信封加密資料;敏感號碼令牌化;所有服務間呼叫零信任認證
```

## 二、威脅建模：把 STRIDE 翻譯成金融語言

微軟的 STRIDE 模型把威脅分成六類：仿冒（Spoofing）、篡改（Tampering）、抵賴（Repudiation）、資訊洩露（Information Disclosure）、拒絕服務（DoS）和權限提升（Elevation of Privilege）。它的價值不在分類本身，而在於強迫你對**每一條資料流**逐項發問。落到金融場景，我建議這樣對照：

| STRIDE | 金融場景中的典型形態 | 對應的主要模式 |
|---|---|---|
| 仿冒 | 盜用憑證發起轉帳、偽造商戶回呼、冒充內部服務 | SCA、裝置繫結、請求簽名、服務間 mTLS |
| 篡改 | 中間人改收款帳號、改金額、改交易參數；篡改流水 | 動態連結、簽名覆蓋報文體、只追加帳本 |
| 抵賴 | "這筆不是我做的"、櫃員否認審批 | 不可否認簽名、審計日誌、Maker-Checker 留痕 |
| 資訊洩露 | 卡號、證件號、持倉洩露 | 令牌化、信封加密、最小權限、資料最小化 |
| 拒絕服務 | 撮合/支付通道被打爆；失控演算法耗盡資源 | 限流、熔斷、容量隔離、災備 |
| 權限提升 | 越權檢視他人帳戶（BOLA）、普通使用者呼叫管理介面（BFLA） | 物件級/功能級授權、職責分離 |

還要注意 STRIDE 不容易覆蓋的一類：**合法但有害**的業務流。OWASP API Security Top 10（2023）把"對敏感業務流的無限制訪問"列為 API6——攻擊者每一步都沒越權，只是用自動化把業務邏輯變成武器，比如薅羊毛和撞庫後的小額試探。

## 三、模式目錄

以下每個模式按"問題 / 方案 / 實現要點 / 代價"組織。

### 模式 1：強客戶認證與裝置繫結

**問題。** 密碼會洩露，會話會被劫持，孟加拉國央行事件中攻擊者甚至拿到了真實的操作憑證。

**方案。** 歐盟 PSD2 的監管技術標準（RTS，條例 2018/389）第 4 條給出了很好的工程化定義：認證需基於知識、持有、固有特徵中的**兩個或以上**要素；生成的認證碼只能被接受一次，不可偽造，也無法由認證碼反推出任何要素；認證失敗時不得暴露是哪個要素出錯；連續失敗最多 5 次即應鎖定；會話無操作的超時不超過 5 分鐘。

**香港和臺灣的寫法。** 香港證監會 2017 年 10 月 27 日釋出《降低及紓減與網際網路交易相關的駭客入侵風險指引》，列出 20 項基本監控規定。其中客戶登入網際網路交易帳戶的**雙重認證**於 2018 年 4 月 27 日生效，其餘規定於 2018 年 7 月 27 日生效。證監會的常見問題有兩條特別工程化：

- "雙重密碼"不算雙重認證，因為"雙重密碼只構成認證程式中的單一因素（即"客戶所知的"）"；
- 用簡訊一次性密碼的經紀行，"便應告誡其客戶不要設定短訊轉發"。

香港金管局《監管政策手冊》TM-E-1（V.4，2024 年 10 月 25 日，只備英文版本）則要求：網銀在進行高風險交易前，每個登入會話至少做一次 2FA；高風險交易至少包括向客戶**未登記的第三方收款人**轉帳。

臺灣銀行公會的《金融機構辦理電子銀行業務安全控管作業基準》走得更遠。它參考 ISO 29115，"以身分登入、信物管理、身分驗證三個面向"，把身份核驗機制分為最高、高、中、低、最低五個信賴等級，再與交易風險對應。

**實現要點。** "持有"要素在移動端通常是**裝置繫結**：在裝置安全區生成金鑰對、登記公鑰，之後每次敏感操作用私鑰對挑戰簽名。密碼洩露了，沒有那臺裝置也無法認證。

**代價。** 摩擦會降低轉化率；換機、丟機的恢復流程常是社會工程的突破口，應與首次繫結同等嚴格。

### 模式 2：動態連結，所見即所簽

**問題。** 認證了"人"，不等於認證了"這筆交易"。惡意軟體可以在使用者確認之後、提交之前替換收款帳號。

**方案。** RTS 第 5 條要求"動態連結"：付款人在認證時被告知金額和收款方；認證碼與該金額、該收款方一一對應；**金額或收款方的任何變化都會使認證碼失效**；並且要保護展示給使用者的資訊的機密性與完整性。這就是"所見即所簽"（WYSIWYS）的監管表述。

**實現要點。** 簽名輸入是"挑戰 + 規範化後的交易摘要"，服務端用同樣規則重算比對。關鍵在**展示通道**：展示的金額與被簽名的資料若來自不同程式碼路徑，繫結就形同虛設。

**代價。** 規範化規則（欄位順序、金額精度、編碼）一旦兩端不一致，就會出現"簽了卻驗不過"的故障，需要版本化並配套測試向量。

### 模式 3：請求簽名與防重放

**問題。** API 層面，攻擊者可以截獲一條合法請求原樣重發，或在傳輸中改動報文體。TLS 保護的是鏈路，不保護"同一條請求被發兩次"，也不保護閘道之後的內部鏈路。

**方案。** 對"方法 + 路徑 + 時間戳 + 隨機數 nonce + 報文體摘要"做 HMAC 簽名。服務端做三件事：校驗時間戳在視窗內、以常量時間比較簽名、確認 nonce 在視窗內未出現過。AWS 的 SigV4 就是這一思路的成熟範例，其文件明確：請求必須在請求中時間戳的五分鐘之內到達 AWS，否則會被拒絕。

```go
// ---------- 1. HMAC 请求签名 + 防重放 ----------

const maxSkew = 5 * time.Minute

type Request struct {
	Method, Path string
	Timestamp    int64 // Unix 秒
	Nonce        string
	Body         []byte
	Signature    string // hex(HMAC-SHA256)
}

func canonical(r *Request) string {
	h := sha256.Sum256(r.Body)
	return r.Method + "\n" + r.Path + "\n" +
		strconv.FormatInt(r.Timestamp, 10) + "\n" + r.Nonce + "\n" + hex.EncodeToString(h[:])
}

func Sign(key []byte, r *Request) string {
	m := hmac.New(sha256.New, key)
	m.Write([]byte(canonical(r)))
	return hex.EncodeToString(m.Sum(nil))
}

type Verifier struct {
	key   []byte
	mu    sync.Mutex
	seen  map[string]time.Time // nonce -> 过期时间（生产中用 Redis SETNX + TTL）
	clock func() time.Time
}

var (
	ErrSkew   = errors.New("timestamp outside window")
	ErrSig    = errors.New("bad signature")
	ErrReplay = errors.New("nonce replayed")
)

func (v *Verifier) Verify(r *Request) error {
	now := v.clock()
	ts := time.Unix(r.Timestamp, 0)
	if ts.Before(now.Add(-maxSkew)) || ts.After(now.Add(maxSkew)) {
		return ErrSkew
	}
	want, err := hex.DecodeString(Sign(v.key, r))
	if err != nil {
		return err
	}
	got, err := hex.DecodeString(r.Signature)
	if err != nil || !hmac.Equal(want, got) { // 常量时间比较
		return ErrSig
	}
	v.mu.Lock()
	defer v.mu.Unlock()
	for n, exp := range v.seen { // 演示用的惰性清理
		if now.After(exp) {
			delete(v.seen, n)
		}
	}
	if _, dup := v.seen[r.Nonce]; dup {
		return ErrReplay
	}
	v.seen[r.Nonce] = now.Add(2 * maxSkew) // 窗口外的请求已被时间戳拦下
	return nil
}
```

執行結果：首次請求通過；原樣重放返回 `nonce replayed`；改了收款帳號的請求返回 `bad signature`；十分鐘前的請求返回 `timestamp outside window`。

**實現要點。** 時間視窗讓 nonce 只需儲存有限時長（生產中用 Redis `SET NX` 加過期即可）；簽名必須覆蓋報文體摘要；比較必須用 `hmac.Equal` 這類常量時間函式。

**代價。** 依賴時鐘同步；多活部署時 nonce 儲存若不跨機房一致，可在兩地各重放一次。對稱金鑰意味著服務端也能偽造簽名，**不提供不可否認性**，需要時應改用客戶端私鑰簽名（模式 1）。

### 模式 4：冪等鍵與"恰好一次"記帳

**問題。** 最危險的失敗是**不確定的失敗**：請求超時了，錢扣沒扣？不重試會丟單，盲目重試會重複扣款。

**方案。** Stripe 的工程部落格把這個問題講得很透：網路失敗是模糊的，客戶端應該重試直到成功，而冪等鍵保證了"恰好一次"的語義；重試應使用帶隨機抖動的指數退避。其 API 文件進一步規定：服務端儲存首次請求的結果，相同冪等鍵的後續請求直接返回這個結果；如果同一個鍵攜帶了不同的參數，會返回錯誤；鍵在至少 24 小時後才會被清理。

```go
// ---------- 2. 幂等记账（复式、只追加） ----------

type Entry struct {
	Account string
	Amount  int64 // 最小货币单位；借为正，贷为负
}

type Ledger struct {
	mu      sync.Mutex
	journal []Entry           // 只追加，从不修改
	idem    map[string]string // 幂等键 -> 请求摘要
	results map[string]int    // 幂等键 -> 首次记账的流水号
}

var ErrUnbalanced = errors.New("entries do not sum to zero")
var ErrKeyReuse = errors.New("idempotency key reused with different payload")

func digest(es []Entry) string {
	h := sha256.New()
	for _, e := range es {
		fmt.Fprintf(h, "%s:%d;", e.Account, e.Amount)
	}
	return hex.EncodeToString(h.Sum(nil))
}

// Post 返回流水起始序号；同一幂等键的重试返回同一结果，不会重复入账。
func (l *Ledger) Post(key string, es []Entry) (int, error) {
	var sum int64
	for _, e := range es {
		sum += e.Amount
	}
	if sum != 0 || len(es) < 2 {
		return 0, ErrUnbalanced
	}
	d := digest(es)
	l.mu.Lock() // 生产中：同一数据库事务 + 幂等键唯一约束
	defer l.mu.Unlock()
	if old, ok := l.idem[key]; ok {
		if old != d {
			return 0, ErrKeyReuse
		}
		return l.results[key], nil
	}
	seq := len(l.journal)
	l.journal = append(l.journal, es...)
	l.idem[key], l.results[key] = d, seq
	return seq, nil
}

func (l *Ledger) Balance(acct string) int64 { // 余额由流水推导
	l.mu.Lock()
	defer l.mu.Unlock()
	var b int64
	for _, e := range l.journal {
		if e.Account == acct {
			b += e.Amount
		}
	}
	return b
}
```

執行結果：`tx-42` 首次記帳與網路超時後的重試都返回序號 0、餘額只增加一次；同一個鍵換了金額會被拒絕；借貸不平的分錄直接拒絕。

**實現要點。** 演示中的互斥鎖，在生產中對應：**在同一個資料庫交易裡**插入冪等記錄（鍵有唯一約束）並寫入分錄。在快取裡查重、在另一個交易裡記帳，等於沒有冪等。儲存請求摘要，可以識別"同一個鍵被用於不同請求"的客戶端 bug。

**代價。** 冪等表持續增長，需要過期策略，而過期時間應遠大於客戶端的最大重試週期。跨系統的"恰好一次"還要求下游也支援冪等。

### 模式 5：複式記帳、只追加流水與對帳

**問題。** 直接 `UPDATE balance = balance - 100` 的系統，一旦出錯，既無法知道錯在哪裡，也無法證明餘額是對的。

**方案。** Modern Treasury 的開發者記帳系列給出了兩條原則：儲存不可變的交易流水並**始終由流水計算餘額**，因為直接修改餘額的系統容易出錯，且難以發現和對帳；複式記帳的力量在於所有借方餘額之和始終等於所有貸方餘額之和——不相等就意味著系統憑空造出或丟失了錢。上面的 `Post` 函式在入口強制分錄和為零，`Balance` 由流水推導，正是這兩條原則的最小實現。

**實現要點。** 更正只通過追加衝正分錄完成，並在資料庫層面收回業務帳號對流水錶的 `UPDATE/DELETE` 權限。**對帳**是另一半：內部流水與清算機構、銀行、交易所逐筆比對。Knight 事件中，SEC 特別指出公司沒有任何控制去比對離開 SMARS 的訂單與進入 SMARS 的訂單——這正是一個缺失的即時對帳。

**代價。** 讀放大，需要引入餘額快照，而快照又要與流水定期核對。

### 模式 6：Maker-Checker 與職責分離

**問題。** 單個人——無論是內鬼、被盜號的員工，還是一個疲憊的維運——能獨立完成一個高風險動作。

**方案。** NIST 對職責分離的定義是：不應給任何單個使用者足以獨自濫用系統的權限，例如批准工資單的人不應同時是製作工資單的人；它可以靜態實施（定義互斥角色），也可以動態實施（在訪問時強制）。在交易系統裡，它通常表現為"制單—複核"的四眼原則。

遠銀事件給這個模式補了一個反面註腳：SWIFT 交易"均由系統自動處理完成"，沒有人工再次稽核。自動化本身沒錯，錯在**自動化路徑上沒有任何一個不依賴同一組憑證的複核點**。四眼原則不一定要靠兩個人，也可以是一個獨立的、拿不到 SWIFT 操作員憑證的規則引擎，比如對大額、新收款行、非營業時間的報文二次核對。

```go
// ---------- 4. Maker-Checker：审批绑定内容摘要 ----------

type Pending struct {
	ID, Maker, Hash string // Hash = sha256(规范化后的指令全文)
	Approved        bool
}

func Approve(p *Pending, checker, hashShownToChecker string) error {
	if checker == p.Maker {
		return errors.New("maker cannot approve own instruction")
	}
	if hashShownToChecker != p.Hash {
		return errors.New("instruction changed after review")
	}
	p.Approved = true
	return nil
}
```

**實現要點。** 第一，複核人批准的必須是**他看到的那個版本**：審批繫結內容摘要，指令被改則審批失效——這是模式 2 在內部流程上的投影。第二，四眼原則要覆蓋**變更與釋出**：Knight 的部署由一人完成、無人複核，8 臺伺服器漏了 1 臺。

**代價。** 最大風險是"橡皮圖章"。應按風險分層，只把真正高風險的動作交給人，並抽樣回溯複核品質。

### 模式 7：金鑰分層——HSM/KMS、信封加密與令牌化

**問題。** 難的不是加密，而是金鑰放在哪、誰能用。

**方案。** 三層結構：

- **根金鑰留在硬體裡。** AWS KMS 的文件說明，KMS 金鑰在其硬體安全模組（HSM）中生成，非對稱私鑰永遠不會以未加密的形式離開 KMS。
- **信封加密。** AWS Encryption SDK 為每條訊息生成唯一的資料金鑰，用它加密資料，再用包裝金鑰（金鑰加密金鑰）加密這把資料金鑰，並把加密後的資料金鑰與密文存在一起。這樣對主金鑰的呼叫次數與資料量無關，輪換主金鑰也無需重加密全部資料。加密上下文（encryption context）可以把"這份密文屬於哪個租戶、哪張表"繫結進認證資料中，防止密文被挪用。
- **令牌化。** PCI 安全標準委員會的令牌化指南（2011 年 8 月）指出，令牌化可以減少環境中的持卡人資料，從而可能縮小 PCI DSS 的適用範圍；但如果商戶能取回主帳號（PAN），其環境仍在範圍內。

**實現要點。** 業務系統只接觸令牌，只有極少數服務能呼叫去令牌化介面，且每次呼叫都記審計日誌。簽名金鑰（模式 3）和資料金鑰都由 KMS/HSM 託管，應用程序裡只有短期憑證。更上游的一步是**資料最小化**：洩露的影響與"存了多少"成正比——Capital One 事件中被訪問的最大一類資料，是 2005 年至 2019 年初申請信用卡時提交的資訊。不需要的不採集，日誌框架統一掩碼，測試環境只用脫敏資料。

**代價。** HSM 的吞吐和延遲對高頻交易是硬約束，通常只用於簽發會話金鑰；令牌庫本身成為最高價值目標，需要獨立隔離。

### 模式 8：最小權限與零信任

**問題。** 內網預設可信，一次入侵就能橫向擴散。

**方案。** NIST SP 800-207 把零信任描述為：把防禦從靜態的網路邊界轉移到使用者、資產和資源；不因資產或帳戶所處的物理或網路位置、或資產歸屬而授予隱式信任。SWIFT 的客戶安全控制框架（CSCF）也按同一邏輯組織為三大目標：保護你的環境（限制網際網路訪問、隔離關鍵系統、減少攻擊面、物理安全），瞭解並限制訪問（防止憑證洩露、管理身份並分離權限），檢測與響應（檢測對系統或交易記錄的異常活動、制定事件響應與資訊共享計劃）。

臺灣金管會 2022 年 12 月 27 日釋出的「金融資安行動方案」2.0 也寫進了零信任：鼓勵金融機構逐步匯入"身分鑑別、裝置鑑別及信任推斷等零信任網路3大核心機制，搭配網路及資源的細化權限管控"。

一銀和遠銀恰好是兩個反例：

- 一銀的問題是內網互信：海外分行與總行"並沒有實質且明顯的分野"，一臺錄音伺服器就能走到 ATM 派送系統；
- 遠銀的問題是授權過大：葉淑媛解釋，最小授權原則是"系統管理超過授權範圍時，須經過內部一定的控制程式後，才能一步步開放局部的權限"，而遠銀"一開始給的管理權限卻是最高權限"。

**實現要點。** 支付報文閘道與辦公網隔離；服務間用工作負載身份雙向認證；雲權限"每服務一個最小角色"，並自動掃描過寬策略。Capital One 的教訓在於：在雲上，**配置就是邊界**，而遷移上雲前的風險評估本身就是一項安全控制。

**代價。** 權限申請過慢時，工程師會傾向於申請"一勞永逸"的大權限，需要用臨時提權、自動過期來平衡。

### 模式 9：事前風控——限額、頻率與總敞口

**問題。** 合法憑證 + 合法格式 + 異常規模 = 災難。孟加拉國央行一次發出的指令總額接近 10 億美元；Knight 在 45 分鐘裡累積了 35 億美元的頭寸。

**方案。** SEC Rule 15c3-5 要求有市場準入的券商具備系統化的風險管理控制，合理設計以**防止錯誤訂單的進入**，以及防止超過預設信用和資本閾值的訂單——這些閾值要**按每個客戶和券商自身彙總**計算。關鍵詞是"事前"和"彙總"。Knight 的處罰令指出，其 9.5% 的價格區間控制不覆蓋開盤前的訂單；出問題的帳戶也沒有與公司層面的資本閾值掛鉤；而它的監控工具只在成交後顯示頭寸，完全依賴人工盯盤，也不顯示限額。

```go
// ---------- 3. 事前风控 + 熔断开关 ----------

type RiskGate struct {
	killed       atomic.Bool
	mu           sync.Mutex
	maxOrder     int64 // 单笔名义金额上限
	maxExposure  int64 // 全公司总敞口上限（含未成交挂单）
	exposure     int64 // 已成交持仓 + 在途订单
	maxPerSecond int
	window       time.Time
	count        int
}

var ErrKilled = errors.New("kill switch engaged")

func (g *RiskGate) Kill() { g.killed.Store(true) }

func (g *RiskGate) Check(notional int64, now time.Time) error {
	if g.killed.Load() {
		return ErrKilled
	}
	g.mu.Lock()
	defer g.mu.Unlock()
	if notional > g.maxOrder {
		return errors.New("order notional limit")
	}
	if now.Sub(g.window) >= time.Second {
		g.window, g.count = now.Truncate(time.Second), 0
	}
	if g.count >= g.maxPerSecond {
		return errors.New("order rate limit")
	}
	if g.exposure+notional > g.maxExposure {
		g.killed.Store(true) // 触及总阈值：自动熔断，而不是等人盯屏
		return ErrKilled
	}
	g.count++
	g.exposure += notional // 订单发出即计入敞口，不等成交回报
	return nil
}
```

**實現要點。** 範例裡有三個刻意的設計：訂單**發出即計入敞口**，不等成交回報（Knight 正是在成交後才看到頭寸）；限額檢查在發單路徑上同步執行；觸及總閾值時**自動熔斷**，而不是發一封告警郵件——Knight 開盤前收到的 97 封自動郵件無人處置。

**代價。** 事前檢查增加延遲；限額過緊誤殺業務，過鬆形同虛設，需要有人對其負責並定期校準。

**臺港監管中的"約定帳戶"。** 臺灣銀行公會基準把交易指示分成高風險和低風險兩類："高風險交易"指執行結果"對客戶權益有重大影響之各類電子轉帳及交易指示，包含非約定轉帳交易超過辦理低風險交易最高限額之交易指示"，而"約定轉入帳戶之設定及轉帳"被列為低風險。香港 TM-E-1 用的是同一個概念：向"未登記收款人"轉帳才是高風險。兩地的做法可以翻譯成一條工程規則：**收款方是否事先登記，是風控裡最便宜也最有效的特徵之一**。登記本身要走高信賴等級的認證，登記之後的轉帳才可以降低摩擦。

### 模式 10：熔斷開關與安全降級

**問題。** 當故障傳播的速度超過人類反應速度時，"發現問題再處理"已經來不及。

**方案。** 每個能動錢的元件都要有一個獨立的、經過演練的"停止"能力：按帳戶、按策略、按通道、全域四個粒度。SEC 就 Knight 事件釋出的新聞稿中，執法部門官員的一句話值得每個架構師抄在牆上："Brokers and dealers must look at each component in each of their systems and ask themselves what would happen if the component malfunctions and what safety nets are in place to limit the harm it could cause."

**實現要點。** 熔斷開關必須是**最簡單的程式碼路徑**：一個原子布林值，在發單入口第一個檢查，不依賴可能已經出故障的元件。停止之後要能"安全地維持現狀"，而不是匆忙回滾——Knight 在應急中移除新程式碼反而擴大了損失，說明應急動作本身也需要預案和演練。

**代價。** 誤觸發同樣有損失，觸發條件、授權人和恢復流程需事先約定並演練。

### 模式 11：反自動化與業務流保護

**問題。** 攻擊者不越權，只是比人快一萬倍。

**方案。** OWASP API4（無限制的資源消耗）提醒：執行超時、可分配記憶體、上傳檔案大小、單次請求內的操作次數、每頁返回的記錄數，甚至第三方服務的花費上限，任何一項缺失或設定不當都構成漏洞。API6（對敏感業務流的無限制訪問）給出的緩解措施包括：裝置指紋（拒絕無頭瀏覽器等非預期客戶端）、人機識別（驗證碼或打字節奏等生物行為特徵）、識別非人類模式（例如在一秒內完成"加入購物車"和"完成購買"）、考慮封禁 Tor 出口節點和知名代理。

臺灣證交所對證券商的要求也落在同一處："應針對帳號登入及IP異常進行紀錄及分析，符合特定態樣應立即通知客戶及產生示警，以降低入侵或帳號撞庫攻擊之風險。"

**實現要點。** 限流按 IP、裝置、帳戶、收款方、業務流分層，"收款方維度"常比"付款方維度"更能發現團夥作案。還要警惕 API9（資產清單管理不當）：被遺忘的舊版介面往往沒有這些保護。

**代價。** 裝置指紋與行為資料本身是敏感個人資訊；誤判會擋住真實使用者，需要申訴通道。

### 模式 12：安全交付——從程式碼到二進位制的可信鏈

**問題。** Knight 的根因不是攻擊，而是一段停用了九年的程式碼、一個被複用的標誌位和一次漏掉一臺伺服器的部署。

**方案。** NIST SP 800-218（SSDF）指出，大多數軟體開發生命週期模型沒有詳細處理安全，因此需要把一組核心的安全開發實踐嵌入每個模型。SLSA 框架則關注供應鏈：即使程式碼經過掃描，從原始碼到構建、打包、分發的每一環都存在被篡改的風險，SLSA 通過自動化追蹤程式碼從原始碼到二進位制的處理過程，確保對原始碼做過的審查仍然適用於最終執行的二進位制。應用層的驗證要求可以參考 OWASP ASVS，當前穩定版本為 5.0.0。

一銀事件說明，可信鏈還要覆蓋**分發與更新通道**：攻擊者沒有改 ATM 上的任何正式程式，而是借用 ATM 軟體派送伺服器發出一個"更新包"，ATM "自動按照例行系統更新程式執行"。更新包如果必須帶有離線簽名金鑰的簽名、終端在安裝前驗簽，這條路就走不通了。

**實現要點。** 三條"Knight 條款"：**刪除死程式碼**而非用開關遮蔽；**永不復用**有歷史語義的標誌位；部署後自動核對每臺主機的執行版本，不一致就不放流量。再加一條"一銀條款"：**終端只安裝經過驗簽的更新包，日誌即時送到終端之外**，本地的 `sdelete` 刪不掉遠端副本。

**代價。** 構建溯源和准入檢查會拖慢緊急修復；解決辦法不是繞過，而是讓緊急通道同樣自動化，只是審批更快。

### 模式 13：韌性與災難恢復

**問題。** 可用性本身就是金融安全：支付中斷、無法平倉同樣是真實損失。

**方案。** 新加坡金管局（MAS）的技術風險管理指南（2021 年 1 月 18 日釋出）把目標定為建立健全的技術風險治理，並保持 IT 與網路的韌性；我國的網路安全等級保護基本要求（GB/T 22239-2019）也把安全維運管理等內容納入基本要求。工程上，關鍵是把恢復目標（可容忍丟失多少資料、多久恢復）落實到架構，並定期做真實的切換演練。

金管會行動方案 2.0 把這件事寫得很具體：鼓勵重要金融機構強化核心資料保全，"包含核心資料檔案、資料庫加密與分持，儲存於第三地或雲端備份等機制"，並依行業特性訂定核心業務系統備援演練指引，鼓勵演練時"納入對外服務實際運作，驗證其有效性"。臺灣證交所則要求證券商每年評估核心系統可容忍中斷時間、RTO 和 RPO 的妥適性。

**實現要點。** 切換時最容易出現"同一筆交易兩邊各執行一次"，只有帳本具備冪等鍵和唯一約束，切換後的重放才安全。此外要有**不能被生產憑證刪除**的不可變備份。

**代價。** 多活架構會極大增加一致性問題的複雜度。

## 四、合規框架是地圖，不是領土

PCI DSS（當前為 v4.0.1；v4.0 的 64 項新要求中有 51 項已於 2025 年 3 月 31 日生效）、SWIFT CSCF（強制參與，每年 7 月釋出新版，7 月 1 日至 12 月 31 日間通過 KYC-SA 提交證明，控制項與 ISO 27002、PCI DSS、NIST CSF 等對齊）、等保 2.0（GB/T 22239-2019，2019 年 12 月 1 日實施，含雲端計算、行動互聯等擴充要求）及金融行業的 JR/T 0071-2020 實施指引，再加上前文的 PSD2 RTS 和 SEC Rule 15c3-5，構成了一張很好的覆蓋面地圖。

**臺灣與香港的繁體中文監管檔案**，可以補上這張地圖在亞洲的一角：

| 轄區 | 檔案 | 要點（據原文） |
|---|---|---|
| 臺灣 金管會 | 「金融資安行動方案」2.0（2022-12-27） | 40 項措施：新增 12 項、擴大適用 5 項、持續 23 項；三年為期、每季檢討；要求一定規模以上金融機構設定"副總經理層級以上之資安長"；推動 ISMS、SOC、零信任、MITRE ATT&CK 演訓、F-ISAC 情資分享 |
| 臺灣 銀行公會 | 《金融機構辦理電子銀行業務安全控管作業基準》（我核對的是 1120614 版本） | 參考 ISO 29115 的五級信賴等級；交易指示按高、低風險分類，約定轉入帳戶與非約定轉帳區別對待 |
| 臺灣 證交所 | 《建立證券商資通安全檢查機制》 | 參考 ISO 27001 架構，共 14 個控制領域；網路下單須加密連線並使用多因子驗證；最高權限帳號須經權責主管同意並留存紀錄；提供網路下單的核心繫統異動時須執行原始碼掃描 |
| 香港 金管局 | 「網路防衛計劃」（2016）與 2.0（2020-11-03 公佈，2021-01-01 起分階段生效） | 三大支柱：網路防衛評估框架（C-RAF）、專業培訓計劃、網路風險資訊共享平臺；超過 90% 的銀行認為 C-RAF 有助發現以往未能識別的缺口 |
| 香港 金管局 | TM-E-1「電子銀行的風險管理」（V.4，2024-10-25，只備英文） | 高風險交易前每會話至少一次 2FA；對可疑交易要求額外認證 |
| 香港 證監會 | 《降低及紓減與網際網路交易相關的駭客入侵風險指引》（2017-10-27） | 20 項基本監控規定；登入雙重認證 2018-04-27 生效，其餘 2018-07-27 生效；"雙重密碼"不算 2FA |

但合規清單天然是"控制項是否存在"的視角。Knight 有價格區間控制，有監控工具，有告警郵件——每一項單獨看都"存在"，組合起來卻沒有一項能在 45 分鐘內止損。**審計問的是"有沒有"，事故問的是"閉不閉環"。** 遠銀是更直接的例子：監管 2016 年就要求網段隔離，遠銀的工作站合規、主機沒有隔離，"只做半套"；而檢查局通常每兩年才對一家金融機構做一次一般金檢，遠銀當年不在名單上。地圖畫得再細，也要有人在現場走一遍。

## 五、我的反思：這些模式的邊界與權衡

**第一，安全與延遲在交易系統裡是真實的零和。** 我的做法是按"能否造成不可逆損失"給檢查分層：不可逆的一律同步、事前、自動；可逆的可以非同步、事後、抽樣。

**第二，冪等和對帳是最被低估的安全控制。** 它們常被當作"可靠性"問題，但重複扣款、憑空造錢在客戶和監管眼裡就是安全事件。不能由流水重算餘額、不能與外部逐筆對帳的系統，談不上完整性。

**第三，人是控制，也是漏洞。** 注意力會在第一百筆審批後耗盡；97 封告警郵件也說明"通知了人"不等於"控制了風險"。能寫成規則的，就寫成會自動阻斷的程式碼。

**第四，最危險的往往是"合法"的輸入。** 真實憑證、正確格式、自家程式碼發出的訂單——所以檢測不能只看攻擊特徵，還要看是否偏離了這個帳戶、這個策略的正常形態。

**第五，模式互相依賴，單獨使用會帶來虛假的安全感。** 防重放依賴時鐘與金鑰，熔斷依賴準確的敞口彙總，災備依賴冪等。評審時應畫出依賴圖，找出共享金鑰服務、時間源這類"一處失效、多處失守"的單點。

**最後，本文的侷限：** 案例與條款以歐美為主，臺港部分補充了監管新聞稿、裁罰公告和指引原文，但金管會行動方案 2.0 我只讀了新聞稿，沒有逐項讀完 PDF；香港 C-RAF 2.0 的附件只備英文，我沒有展開；一銀和遠銀的技術細節來自調查局披露與媒體報道，遠銀的入侵手法金管會以警方調查中為由沒有公開；國內標準只核實到名稱與適用範圍；程式碼是單機教學實現，省略了持久化、分散式一致性和金鑰輪換。

## 六、可執行清單

**認證與授權**
- [ ] 資金類操作強制多因素認證，失敗不提示具體要素，連續失敗鎖定；帳戶恢復流程與首次繫結同等嚴格。
- [ ] 簽名輸入包含金額與收款方摘要，任何欄位改動導致驗證失敗；展示與簽名資料來自同一來源。
- [ ] 所有 API 做物件級與功能級授權測試，維護介面資產清單。
- [ ] 收款方先登記、後轉帳；登記本身走最高信賴等級認證，向未登記收款人轉帳一律按高風險處理。
- [ ] "雙重密碼"不算 2FA；用簡訊 OTP 時提醒客戶不要設定簡訊轉發。

**請求與交易語義**
- [ ] 外部與內部關鍵介面使用"時間戳 + nonce + 報文體摘要"簽名，常量時間比較，nonce 有界儲存。
- [ ] 所有寫操作接受冪等鍵；冪等記錄與分錄在同一交易中寫入，鍵有唯一約束，並儲存請求摘要。
- [ ] 帳本只追加，分錄借貸平衡在寫入時強制；更正只用衝正；業務帳號無流水錶的 UPDATE/DELETE 權限。
- [ ] 每日（關鍵鏈路即時）與外部清算、銀行、交易所逐筆對帳，差異自動告警並阻斷。

**治理與金鑰**
- [ ] 大額與敏感操作執行 Maker-Checker，審批繫結內容摘要；變更與釋出同樣四眼。
- [ ] 根金鑰在 HSM/KMS，資料用信封加密並帶加密上下文；卡號、證件號令牌化，去令牌化呼叫全量審計。
- [ ] 服務間零信任認證，雲權限最小化並持續掃描配置漂移；遷移上雲前完成正式風險評估。
- [ ] 海外分行、子公司、外包與內網之間不預設互信；管理員帳號預設低權限、按需提權並留存紀錄。

**執行時防護**
- [ ] 單筆、單日、頻率、按客戶與公司彙總的敞口限額在事前同步檢查，訂單發出即計入敞口。
- [ ] 帳戶、策略、通道、全域四級熔斷開關，觸發條件、授權人、恢復流程成文並每季度演練。
- [ ] 分層限流與反自動化（裝置、帳戶、收款方、業務流維度），並為誤判提供申訴通道。

**交付與韌性**
- [ ] 構建可溯源、製品簽名；刪除死程式碼、不復用舊標誌位；部署後自動核對每臺主機的執行版本。
- [ ] 終端（ATM、交易終端）只安裝驗簽通過的更新包；日誌即時外送，不能只存在本機。
- [ ] 定義並演練恢復目標；保留不可被生產憑證刪除的不可變備份；切換後依賴冪等安全重放。

## 參考資料

**事故與執法**
- SEC, In the Matter of Knight Capital Americas LLC（Release No. 34-70694）：https://www.sec.gov/litigation/admin/2013/34-70694.pdf
- SEC 新聞稿 2013-222，SEC Charges Knight Capital With Violations of Market Access Rule：https://www.sec.gov/newsroom/press-releases/2013-222
- SEC, Risk Management Controls for Brokers or Dealers with Market Access（Rule 15c3-5）：https://www.sec.gov/rules-regulations/2011/06/risk-management-controls-brokers-or-dealers-market-access
- Capital One, Information on the Capital One Cyber Incident：https://www.capitalone.com/digital/facts2019/
- OCC 新聞稿 2020-101（Capital One 罰款）：https://www.occ.gov/news-issuances/news-releases/2020/nr-occ-2020-101.html
- 美國司法部華盛頓西區檢察官辦公室，Former Seattle tech worker convicted of wire fraud and computer intrusions：https://www.justice.gov/usao-wdwa/pr/former-seattle-tech-worker-convicted-wire-fraud-and-computer-intrusions
- Wikipedia, Bangladesh Bank robbery：https://en.wikipedia.org/wiki/Bangladesh_Bank_robbery

**臺灣與香港（繁體中文）**
- 金管會新聞稿：〈金管會發布「金融資安行動方案」2.0，引導金融資安持續精進〉，2022-12-27：https://www.fsc.gov.tw/ch/home.jsp?dataserno=202212270001&dtable=News&id=96&mcustomize=news_view.jsp&parentpath=0%2C2
- 金管會裁罰公告：〈遠東國際商業銀行SWIFT系統遭駭重大偶發事件所涉缺失…核處新臺幣800萬元罰鍰〉，2017-12-12：https://www.fsc.gov.tw/ch/home.jsp?aplistdn=ou%3Dnews%2Cou%3Dmultisite%2Cou%3Dchinese%2Cou%3Dap_root%2Co%3Dfsc%2Cc%3Dtw&dataserno=201712120005&dtable=News&id=96&mcustomize=news_view.jsp&parentpath=0%2C2
- 中央社：〈遠銀遭駭18億元 金管會列三大缺失〉，2017-10-12：https://www.cna.com.tw/news/afe/201710120379.aspx
- iThome：〈【遠銀遭駭追追追】遠銀被駭案，金管會初步調查：銀行未做好內控管理〉，2017-10-12：https://www.ithome.com.tw/news/117380
- iThome：〈【詳細圖解】駭客入侵一銀ATM流程追追追〉，2016-07-25：https://www.ithome.com.tw/news/107294
- iThome：〈《獨家》調查局揭露：一銀ATM盜領惡意程式指定在7月發作，逾期失效〉，2016-07-15：https://www.ithome.com.tw/news/107136
- 銀行公會：《金融機構辦理電子銀行業務安全控管作業基準》（1120614 版本，信託公會轉載）：https://www.trust.org.tw/upload/1120001034-0-0.pdf
- 臺灣證交所：〈簡介建立證券商資通安全檢查機制〉：https://www.twse.com.tw/market_insights/zh/detail/8a8216d6933460a4019371c2555e0222
- 香港金管局新聞稿：〈金管局宣佈推出「網路防衛計劃2.0」〉，2020-11-03：https://www.hkma.gov.hk/chi/news-and-media/press-releases/2020/11/20201103-4/
- 香港金管局：《監管政策手冊》TM-E-1「電子銀行的風險管理」（V.4，只備英文）：https://brdr.hkma.gov.hk/chi/doc-ldg/docId/20241101-3-TC
- 香港證監會通函：〈實施《降低及紓減與網際網路交易相關的駭客入侵風險指引》〉，2017-10-27：https://apps.sfc.hk/edistributionWeb/gateway/TC/circular/intermediaries/supervision/doc?refNo=17EC72
- 香港證監會常見問題：〈網路保安〉：https://www.sfc.hk/TC/faqs/intermediaries/supervision/Cybersecurity/Cybersecurity

**標準與法規**
- 歐盟委員會授權條例 (EU) 2018/389（PSD2 RTS）第 4 條：https://www.legislation.gov.uk/eur/2018/389/article/4
- 同上，第 5 條（動態連結）：https://www.legislation.gov.uk/eur/2018/389/article/5
- Swift, Understand Controls（CSCF）：https://www.swift.com/myswift/customer-security-programme-csp/security-controls
- PCI SSC, PCI DSS v4.x Resource Hub：https://blog.pcisecuritystandards.org/pci-dss-v4-0-resource-hub
- PCI SSC, Now is the Time to Adopt the Future-Dated Requirements of PCI DSS v4.x：https://blog.pcisecuritystandards.org/now-is-the-time-for-organizations-to-adopt-the-future-dated-requirements-of-pci-dss-v4-x
- PCI SSC, Information Supplement: PCI DSS Tokenization Guidelines（2011）：https://www.pcisecuritystandards.org/documents/Tokenization_Guidelines_Info_Supplement.pdf
- Wikipedia, Payment Card Industry Data Security Standard：https://en.wikipedia.org/wiki/Payment_Card_Industry_Data_Security_Standard
- MAS, Technology Risk Management Guidelines：https://www.mas.gov.sg/regulation/guidelines/technology-risk-management-guidelines
- 國家標準 GB/T 22239-2019《資訊安全技術 網路安全等級保護基本要求》：https://openstd.samr.gov.cn/bzgk/std/newGbInfo?hcno=BAFB47E8874764186BDB7865E8344DAF
- 全國標準資訊公共服務平臺，JR/T 0071—2020 釋出公告：https://hbba.sacinfo.org.cn/snDetail/8951901cd64d412b54b858c7bb903baa7d4595b196dd07aaa1232d26804119fd
- 電子銀行網，中國人民銀行釋出《金融行業網路安全等級保護實施指引》：https://www.cebnet.com.cn/20201124/102704588.html

**安全框架與工程實踐**
- Microsoft, Threat Modeling Tool threats（STRIDE）：https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool-threats
- OWASP API Security Top 10 2023：https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- OWASP API4:2023 Unrestricted Resource Consumption：https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/
- OWASP API6:2023 Unrestricted Access to Sensitive Business Flows：https://owasp.org/API-Security/editions/2023/en/0xa6-unrestricted-access-to-sensitive-business-flows/
- OWASP ASVS：https://github.com/OWASP/ASVS
- NIST SP 800-207, Zero Trust Architecture：https://csrc.nist.gov/pubs/sp/800/207/final
- NIST SP 800-218, Secure Software Development Framework：https://csrc.nist.gov/pubs/sp/800/218/final
- NIST CSRC Glossary, Separation of Duty：https://csrc.nist.gov/glossary/term/separation_of_duty
- SLSA v1.0, About SLSA：https://slsa.dev/spec/v1.0/about
- AWS, Signature Version 4 for API requests：https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv.html
- AWS KMS cryptography essentials：https://docs.aws.amazon.com/kms/latest/developerguide/kms-cryptography.html
- AWS Encryption SDK concepts（envelope encryption）：https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/concepts.html
- Stripe API, Idempotent requests：https://docs.stripe.com/api/idempotent_requests
- Stripe Blog, Designing robust and predictable APIs with idempotency：https://stripe.com/blog/idempotency
- Modern Treasury, Accounting for Developers, Part I：https://www.moderntreasury.com/journal/accounting-for-developers-part-i
