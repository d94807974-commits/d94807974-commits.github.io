---
title: "《混合訊號》×《驅動力》整合讀書筆記：激勵是一種訊號"
date: 2026-11-04T16:22:59+08:00
lastmod: 2026-11-05T13:25:00+08:00
draft: false
tags: ["讀書筆記", "混合訊號", "驅動力", "Uri Gneezy", "Daniel Pink", "激勵", "行為經濟學", "技術管理"]
summary: "把尤里·格尼茨的《混合訊號》（Mixed Signals）和丹尼爾·平克的《驅動力》（Drive）放在一起讀：平克說胡蘿蔔加大棒在創造性工作上會擠出內在動機，格尼茨用「激勵即訊號」解釋它什麼時候、為什麼會失效。內容包括托兒所罰款、「要麼給夠要麼別給」、Deci 1971、蠟燭問題等原始研究的核對，Cameron & Pierce 與 Deci 學派的元分析之爭，臺灣繁體版《誘因設計》《動機，單純的力量》的譯名與書評對照，並附上一個整合框架、一套工程團隊落地方法、兩本書各自的侷限和一份檢查清單。"
ShowToc: true
---

> 素材說明：兩本書的全文我都沒有讀完。這篇筆記依據的材料有：兩本書的出版社頁面和中英文目錄（耶魯大學出版社、中信出版社、浙江人民出版社/湛廬、臺灣天下文化、大塊文化），平克的官網和他在 TED 的演講（英文原稿，以及 TED 官方的繁體、簡體中文字幕），格尼茨在 Adam Grant 播客《ReThinking》裡的訪談文字稿，幾篇原始論文（格尼茨本人主頁上的 PDF、PubMed 和期刊摘要），還有天下文化與臺灣讀書博主的書評。凡是加引號的句子，都能在下文「參考資料」裡找到出處；書裡的原話我只引用出版社或可靠二手來源公開刊出的部分，沒有憑印象"復原"任何一句。


<figure>
<img src="/images/mixed-signals-drive-incentives-reading-notes/traffic-light.jpg" alt="交通訊號燈" loading="lazy" width="700">
<figcaption>圖片：Raizkgh / <a href="https://commons.wikimedia.org/wiki/File:Traffic_light.jpg">Wikimedia Commons</a>，<a href="https://creativecommons.org/publicdomain/zero/1.0/">CC0</a>。激勵像訊號燈：顏色清楚時有效，互相矛盾時事故就來了。</figcaption>
</figure>

## 一、兩本書的基本資訊

| | 《混合訊號》 | 《驅動力》 |
|---|---|---|
| 原書 | *Mixed Signals: How Incentives Really Work*，Yale University Press，2023（精裝 ISBN 9780300255539；平裝 2024-03-26，ISBN 9780300276749，320 頁） | *Drive: The Surprising Truth About What Motivates Us*，Riverhead Books，2009 |
| 作者 | Uri Gneezy，加州大學聖迭戈分校 Rady 管理學院行為經濟學講席教授，與 John List 合著《隱性動機》（*The Why Axis*） | Daniel H. Pink，美國商業與行為科學暢銷書作家，另著 *A Whole New Mind*、*When* 等 |
| 簡體版 | 《混合訊號：激勵是如何發揮作用的》，[美] 尤里·格尼茨著，江生、於華譯，中信出版社 2023-11，ISBN 9787521756319，328 頁 | 《驅動力：在獎勵與懲罰都已失效的當下，如何煥發人的熱情》，[美] 丹尼爾·平克著，龔怡屏譯，湛廬策劃、浙江人民出版社 2018-06（經典版），ISBN 9787213087486，223 頁 |
| 繁體版 | 《誘因設計：精準傳遞訊號，讓人照著你的想法行動》，尤里．葛尼奇著，周宜芳譯，天下文化 2023-10-27，ISBN 9786263554528，352 頁 | 《動機，單純的力量：把工作做得像投入嗜好一樣 有最單純的動機，才有最棒的表現》，丹尼爾．品克著，席玉蘋譯，大塊文化 2010 年初版，ISBN 9789570316445，280 頁 |
| 一句話 | 所有激勵本質上都是訊號，問題出在訊號互相打架 | 驅動力 3.0：自主、專精、目的 |

簡體版基本直譯原書名，繁體版則改動更大：《誘因設計》的副標題直接點出"訊號"，把全書的核心概念擺上了封面。《驅動力》在臺灣叫《動機，單純的力量》，副標題"把工作做得像投入嗜好一樣"，強調的是"心流"那一面。平克的三要素 autonomy、mastery、purpose，簡體版目錄譯作"自主、專精、目的"，臺灣讀者書評引用的繁體版譯法也是"自主、專精、目的"；TED 官方繁體字幕則譯作"自主性、掌握度、使命感"。同一套概念，中文措辭相差不小。

另外，《混合訊號》的簡體版和繁體版章節標題並不一一對應。比如簡體第 5 章叫"鼓勵創新卻懲罰失敗"，繁體對應章節叫"鼓勵創新 vs. 迴避風險"。繁體版用"A vs. B"的句式，衝突感更強。

## 二、兩本書各自講了什麼

### 《驅動力》：胡蘿蔔加大棒為什麼過時

平克把人類動機分為三代。驅動力 1.0 是生物本能（餓了要吃）。驅動力 2.0 是獎懲，也就是"做了 X 就給 Y"的 if-then 獎勵。驅動力 3.0 是內在動機。官網的摘要寫道，數十年的研究表明 if-then 獎勵"can reduce performance, crush creativity, and dampen long-term motivation – especially for complex or creative tasks"。

全書分三部分：
- **第一部分**：驅動力 3.0 時代來臨，包括"獎勵的懲罰，胡蘿蔔大棒失效了"和"做 I 型人還是 X 型人"。
- **第二部分**：三大要素。自主（"我做什麼，我決定"）、專精（"把想做的事情做得越來越好"）、目的（"超越自身的渴望"）。
- **第三部分**：I 型工具箱。

平克官網列出的 Kindle 高亮裡，有一句對工程管理很有用："Autonomy over four aspects of work: what people do, when they do it, how they do it, and whom they do it with." 也就是任務、時間、方法、團隊四個維度的自主。

2009 年的 TED 演講《The puzzle of motivation》是這本書的濃縮版。演講中他講了 Glucksberg 的蠟燭問題：一組被告知"最快的 25% 得 5 美元，最快的人得 20 美元"，結果比對照組平均慢了三分半鐘。繁體字幕是這樣譯的："答案：是慢了三分半鐘"。他的解釋是，獎勵會讓人聚焦，適合答案就在眼前的任務；可蠟燭問題需要四處尋找靈感，"獎勵只會收窄我們的視線，限制了可能性"。

演講最後的總結，繁體字幕是："第一：那些二十世紀的獎勵方式……只能在一個比我們想像更狹窄的環境下發揮作用，第二：那種「因果」式的獎勵往往摧毀創意，第三：提高工作表現的秘訣，不在獎與罰，而是在見不到的內在動力"。

### 《混合訊號》：激勵是會說話的

格尼茨的出發點完全不同。他不是心理學家，而是做田野實驗的經濟學家。耶魯出版社頁面引用了全書的一句核心話："Incentives send a signal ... and your objective is to make sure this signal is aligned with your goals."

天下文化的簡介給"混合訊號"下了定義："所提供的誘因，其發出的訊號原本應該是為了獎勵A行為，最後卻鼓勵了B行為"。

全書分七部分 28 章，引言叫"照我說的做，別學我的樣兒"。我最關心的是第二部分"避免混合訊號"，四章分別是：
- 以品質為代價激勵數量
- 鼓勵創新卻懲罰失敗
- 鼓勵長期目標卻獎勵短期成果
- 鼓勵團隊合作卻激勵個人成功

這幾乎就是技術團隊績效制度的四大病。後面幾部分講激勵如何塑造故事（風險與錯誤、心理帳戶、作為訊號的獎勵）、用激勵診斷問題（比如"有償離職"）、改變習慣，以及在馬賽社群用激勵改變獵獅、割禮等文化習俗。最後講談判中的訊號（錨定、價格傳遞品質訊號）。

在 Adam Grant 的播客裡，格尼茨舉了公交司機的例子：管理層說要安全駕駛、乘客滿意，但工資按載客數算，那麼司機讀到的資訊就是"其實就是要多拉客"。他說："I really pick up the signals of what does it mean to be a good driver or a good employee ... from the incentives that are out there." 他還補充："you cannot have a situation without incentives"。不設激勵也是一種激勵，現狀本身就帶著訊號。

臺灣博主"西恩的閱讀筆記"的總結很到位。書裡把激勵分成兩類：直接的經濟效應（錢、罰款），和間接的社會訊號與自我訊號。"社會訊號"是在意別人怎麼看我，"自我訊號"是維持內心的自我形象。

## 三、兩本書共享的證據，我逐條核對了

兩本書引用的經典研究大量重疊，而且其中幾篇格尼茨本人就是作者。我把原始文獻找出來對了一遍。

**1. 托兒所罰款：「A Fine is a Price」**（Gneezy & Rustichini，*Journal of Legal Studies* 2000）
- 實驗在海法 10 傢俬立托兒所進行，時間是 1998 年 1 至 6 月，共 20 周。
- 從第 5 周起，6 家托兒所對遲到 10 分鐘以上的家長罰款 10 新謝克爾，另外 4 家作對照。
- 結果遲到的家長不減反增，穩定在初始水平的"almost twice"。第 17 周取消罰款後，遲到率也沒有回落。
- 論文的解釋是：罰款被引入一個"不完全契約"，它改變了家長掌握的資訊。遲到原本是一件說不清有多糟的事，現在被標上了價格，"No guilt or shame"。
- 格尼茨在播客裡的說法更直白："Now that you told me that it's $3 bad, now I don't care." 他還說這件事最初來自自己女兒上托兒所的親身經歷。

**2. 「Pay Enough or Don't Pay at All」**（Gneezy & Rustichini，*QJE* 2000）
- IQ 測試：每答對一題付 10 美分的組，平均答對 23.07 題，比不付錢的組（28.4）還低。付 1 或 3 新謝克爾的組則升到 34 題左右，兩者之間沒有顯著差異。
- 募捐：高中生上門募捐，拿 1% 提成的組只募到 153.6，比純義務組的 238.6 少得多。提成提到 10%，募到 219.3，仍然沒超過義務組。
- 結論就是標題：要麼給夠，要麼別給。給一點點錢，等於告訴對方"這事就值這麼點"。

**3. Deci 1971**（*JPSP* 18(1):105–115）
- 實驗讓 24 名心理學導論課的學生（實驗組、對照組各 12 人）分三次玩 Parker Brothers 出的 Soma 立體拼圖。第二次只給實驗組按完成情況付錢，第三次撤掉報酬，觀察大家在自由時間裡還願不願意繼續玩。
- 摘要的結論是：用錢作外部獎勵時，內在動機傾向於下降；用口頭強化和正面反饋時，內在動機傾向於上升。
- 我在原文第 107 頁讀到一段幾乎就是格尼茨理論雛形的話：金錢獎勵的存在"suggests to the subjects that they 'should probably not render this activity without pay'"。早在 1971 年，Deci 就把擠出效應解釋成"獎勵傳遞了關於這件事性質的資訊"。這是兩本書最自然的交匯點。
- Deci 和 Ryan 此後發展出自我決定理論（SDT）。官網的表述是：支援自主（autonomy）、勝任（competence）、關係（relatedness）三種基本心理需要的環境，會帶來最高品質的動機。平克的"自主、專精、目的"基本是把 SDT 的前兩項拿來，再把"關係"換成了"目的"。

**4. 高額獎勵與"大錯"**（Ariely、Gneezy、Loewenstein & Mazar，*Review of Economic Studies* 2009）
- 這正是平克 TED 裡講的"麻省理工研究 + 印度馬杜賴復現"。論文在印度鄉村設了 4、40、400 盧比三檔獎勵，最高一檔單個遊戲的滿額獎金（400 盧比）已接近當時印度農村人均月消費支出（495 盧比）。
- 摘要原話："With some important exceptions, very high reward levels had a detrimental effect on performance."
- 但論文同時報告，在純體力的按鍵任務上，高獎勵確實提高了表現。所以準確的說法是：高額獎勵傷害需要認知技能的任務，而不是傷害一切任務。

## 四、爭議：擠出效應到底有多普遍

平克的書寫得像一場"科學對企業"的審判，但學術界對擠出效應爭論了二十多年，遠不止一邊倒。

- **Cameron & Pierce（1994，*Review of Educational Research*）**：對 96 項實驗做元分析，摘要結論是"overall, reward does not decrease intrinsic motivation"。口頭表揚會提高內在動機，唯一的負面效應出現在"為了完成任務本身而給予的預期有形獎勵"上，而且只是"minimal negative effect"。
- **Deci、Koestner & Ryan（1999，*Psychological Bulletin*）**：用 128 項研究反駁。結論是，與參與、完成、表現掛鉤的有形獎勵都會顯著削弱自由選擇情境下的內在動機（d 約為 −0.40、−0.36、−0.28），對兒童傷害更大；但正面反饋會增強內在動機（d = 0.33）。同一期附了大量評論與回應，可見爭論之激烈。
- **Cerasoli、Nicklin & Ford（2014，*Psychological Bulletin*）**：這是我認為最有調和價值的一項。它覆蓋 183 個樣本、21 萬餘人，發現內在動機能預測績效，而且在"品質"上預測力更強；外部激勵則更能預測"數量"。擠出效應主要出現在激勵與績效直接掛鉤、高度顯著的情況下。作者的結論是二者"not necessarily antagonistic"。Adam Grant 在播客裡也是這麼概括的：激勵更擅長讓人更努力，而不是更聰明。

所以，平克說"if-then 獎勵往往摧毀創意"，這個說法在實驗室的創造性任務上有證據，但推到"獎金和 KPI 在知識工作中普遍失效"就過度概括了。格尼茨的框架恰好能解釋兩邊的資料為什麼都成立：同樣是錢，傳遞的訊號不同，結果就不同。

## 五、整合：激勵 = 經濟價格 + 訊號

我把兩本書合成一張圖：

```
                 一項激勵（獎金 / 罰款 / 指標 / 晉升標準）
                                  │
          ┌───────────────────────┴───────────────────────┐
          ▼                                               ▼
   直接效應：價格                                    間接效應：訊號
   （格尼茨 / 標準經濟學）                           （格尼茨 + Deci + 平克）
   做 X 更划算 → 做更多 X                                  │
                               ┌──────────────────────┼──────────────────────┐
                               ▼                      ▼                      ▼
                         關於任務的訊號         關於我的訊號           關於你的訊號
                       "這事值多少/有多糟"   "我是因為錢才做"       "組織真正在乎什麼"
                        托兒所罰款              Deci 1971               公交司機、20% 時間
                        給一點點錢              自我訊號/身份           混合訊號四大病
                               │                      │                      │
                               └──────────────────────┼──────────────────────┘
                                                      ▼
                     訊號與內在動機（自主 / 專精 / 目的）一致 → 互相加強
                     訊號與內在動機衝突 → 擠出（平克說的"獎勵的懲罰"）
```

| 問題 | 平克的回答 | 格尼茨的回答 | 我的整合 |
|---|---|---|---|
| 激勵為什麼會失效？ | 外部獎勵把"玩"變成"活"，擠出內在動機 | 激勵發出的訊號和你嘴上說的目標相反 | 擠出是訊號失敗的一種特例：訊號告訴對方"你是為錢幹活的" |
| 什麼時候錢有用？ | 常規、演算法型任務 | 給夠、訊號清晰、與目標一致時 | 數量型、可度量的任務直接用；品質型、創造性任務，先檢查訊號 |
| 應該怎麼設計？ | 給自主、專精、目的；先把基本報酬給公平，"把錢的問題移出桌面" | 先畫出激勵會引發的行為樹，確保"放進誘因的，確實是你想要鼓勵的事物" | 先付公平的基本報酬，再用非控制性的反饋與認可做訊號，最後才謹慎用 if-then |
| 正面反饋呢？ | 鼓勵"now-that"式的事後、非預期認可 | 獎勵本身就是訊號，可以表達尊重 | Deci 等人 1999 年的元分析：正面反饋 d = 0.33，是最便宜的強訊號 |

一句話：**平克告訴你"什麼"會被擠出，格尼茨告訴你"為什麼"會被擠出，以及如何不被擠出。**

## 六、放到工程團隊裡

下面幾條是我結合兩本書，給技術負責人列的診斷方法。每一條都先問同一個問題：**這項激勵到底在對工程師"說"什麼？**

**1. OKR / KPI**
- 混合訊號：說"品質第一"，卻按功能交付數、故事點、PR 數量考核。這正是格尼茨第 4 章的"以品質為代價激勵數量"。
- 做法：KPI 用於觀察系統，不用於直接發錢。OKR 保持"有挑戰、可失敗"的定位，和獎金脫鉤；否則大家會把目標往低處定，這本身就是理性回應訊號。

**2. 獎金與績效評級**
- 兩本書在這裡意見一致：基本報酬要公平且有競爭力。平克要先"把錢的問題移出桌面"，格尼茨要"給夠"。只給一點點錢的"優秀獎"，最容易落入"給一點點等於告訴你這事不值錢"的陷阱。
- 團隊目標配個人獎金，就是第 7 章的"鼓勵團隊合作卻激勵個人成功"。如果工作高度依賴協作，就讓獎金池的一部分與團隊結果掛鉤，並在評審中明確評價幫助他人、帶新人、做 code review 這類"不顯眼的貢獻"。

**3. On-call 補貼**
- On-call 應該付錢，這是對打擾的補償。但要小心"按事故次數付錢"：這會發出"事故越多越好"的訊號，跟托兒所罰款反過來是同一個道理。
- 更好的做法是按值班時長付固定補貼，同時把"減少告警噪音、寫 runbook"算作專精和晉升證據，讓人從中得到"我在變得更好"的自我訊號。

**4. Bug bounty 與內部品質獎**
- 對外部白帽，賞金是清晰的價格訊號，效果很好。對內部員工按 bug 數發獎，則可能產生"自己埋雷自己挖"的經典反效果；即便沒人作弊，也在告訴大家"品質是額外的活"。
- 做法：內部更多用公開認可和事後的"now-that"獎勵，不事先承諾按件計酬。

**5. Code review 指標**
- 統計"每人 review 了多少 PR、平均響應時長"，很快就會出現秒批的 LGTM。這是典型的數量擠出品質。
- 做法：指標只給團隊看趨勢，不排名；把"一次有深度的 review"當作專精來認可，比如在週會上點評一條好評論。

**6. 20% 時間 / 駭客日**
- 平克在 TED 中盛讚 Atlassian 的"FedEx 日"和谷歌的 20% 時間。但 Quartz 在 2013 年報道，谷歌 20% 時間已經"as good as dead"。原因很有意思：專案要經理批准，而經理按團隊生產力被考核，考核又預設員工 100% 時間都在主業上。
- 這是一個教科書級的混合訊號：嘴上給自主，考核卻在懲罰自主。想做 20% 時間，必須同步調整經理的考核口徑，否則不如不做。

**7. 失敗與創新**
- 格尼茨第 5 章"鼓勵創新卻懲罰失敗"。他在播客裡說："you try to innovate, you fail ... and then they punish you"。
- 做法：事故復盤對事不對人，實驗失敗不進績效扣分項，而且要讓大家看到真實的例子。比如一個失敗的重構提案仍被寫進晉升材料，比一百次口號都管用。

**8. 職級階梯與專精**
- 專精需要"可見的進步"。一份清晰的工程職級說明，本身就是最強的專精訊號：它告訴大家組織在乎什麼樣的成長。
- 要注意，如果職級說明寫的是"技術深度"，實際晉升看的卻是"帶了多大的專案"，大家就會去搶專案而不是磨手藝。

**9. 自主的四個維度**
- 用平克的"任務、時間、方法、團隊"四維度做一次自查：工程師能否選擇技術方案？能否調整自己的工作時間？能否參與選題？能否換組？每多一個"不能"，就該問一句：這條限制是必要的，還是習慣？

## 七、我的思考：兩本書的侷限

**《驅動力》的問題**
1. **證據被講成了故事。** 平克把實驗室裡的創造性任務結論推廣到整個"21 世紀的工作"，跳過了 Cameron & Pierce 和 Cerasoli 等人的反面與調和證據。TED 裡"谷歌差不多一半的新產品都誕生於 20% 時間"這類說法，也缺少嚴格的資料支撐。
2. **I 型/X 型的二分太乾淨。** 格尼茨在播客裡說得好：心理學家說人上班是為了意義，經濟學家說是為了交房租，"these two strawmen are just strawmen"。
3. **"把錢移出桌面"說起來容易。** 在薪酬不透明、差距大的組織裡，錢永遠在桌面上。

**《混合訊號》的問題**
1. **框架太寬，幾乎什麼都能解釋。** "一切激勵都是訊號"很有洞察力，但幾乎可以事後解釋任何結果。要讓它有用，必須事先寫下"我預期對方會讀到什麼訊號"，並用實驗檢驗。格尼茨本人倡導小規模測試，讀者卻容易只記住故事。
2. **案例多，組織層面的系統設計少。** 臺灣讀者的書評也指出，書名像是教"怎麼設計"，"其實更多是用非常多的例子"，告訴你混合訊號會帶來什麼後果。
3. **田野實驗的外推。** 托兒所、募捐、馬賽社群的結論，搬到一家 500 人的軟體公司能保留多少，需要自己驗證。

**兩本書都沒有充分討論的**
- 訊號是被"讀"出來的，讀法受文化、信任和歷史影響。同一筆獎金，在一個剛裁過員的團隊裡，訊號完全不同。
- 激勵的設計者通常也處在激勵之中。經理有自己的 KPI，這才是 20% 時間失敗的真正原因。

## 八、可執行清單

設計或調整任何一項激勵前，逐條過一遍：

- [ ] **寫下目標行為**：我希望大家多做什麼、少做什麼？品質還是數量？
- [ ] **寫下訊號**：站在一名普通工程師的角度，這項激勵在"說"什麼？會不會和我們嘴上說的相反？
- [ ] **檢查四類混合訊號**：數量 vs 品質、創新 vs 懲罰失敗、長期 vs 短期、團隊 vs 個人。
- [ ] **金額檢查**：要麼給夠，要麼不要給錢，改用認可、機會或時間。
- [ ] **任務型別檢查**：常規、可度量的任務可以用 if-then；探索性、創造性的任務優先保障自主、專精、目的，獎勵放在事後、不事先承諾。
- [ ] **畫行為樹**：如果有人鑽空子，最便宜的鑽法是什麼？
- [ ] **檢查設計者自己的激勵**：經理、HR 的考核是否會讓這項政策名存實亡？
- [ ] **反饋優先**：有沒有更便宜的方法，例如具體、及時、非控制性的正面反饋？
- [ ] **小範圍試點**：先在一個團隊試一個週期，帶對照組和退出條件。
- [ ] **撤銷預案**：托兒所告訴我們，撤掉一個壞激勵並不能讓行為自動恢復，上線前要想好怎麼收場。

## 參考資料

**書籍資訊**
- Yale University Press, *Mixed Signals: How Incentives Really Work*（平裝與電子版頁面）：<https://yalebooks.yale.edu/book/9780300276749/mixed-signals/>
- Google Books, *Mixed Signals*（精裝 ISBN 9780300255539）：<https://books.google.com/books/about/Mixed_Signals.html?id=k7JI0QEACAAJ>
- 豆瓣讀書，《混合訊號：激勵是如何發揮作用的》（中信出版社）：<https://book.douban.com/subject/36623489/>
- 豆瓣讀書，《誘因設計》（天下文化）：<https://book.douban.com/subject/36622217/>
- 天下文化 Bookzone，《誘因設計》：<https://bookzone.cwgv.com.tw/book/BCB817>
- Daniel Pink 官網，*Drive*：<https://www.danpink.com/books/drive/>
- 湛廬文化，《驅動力》新版頁面：<https://www.cheerspublishing.com/book/detail/1651107448999563264>
- 誠品，《動機，單純的力量》（大塊文化）：<https://www.eslite.com/product/1001129721961311>
- 豆瓣讀書，《動機，單純的力量》：<https://book.douban.com/subject/5288336/>

**演講與訪談**
- Dan Pink, "The puzzle of motivation", TEDGlobal 2009（含繁體、簡體中文字幕）：<https://www.ted.com/talks/dan_pink_the_puzzle_of_motivation>
- ReThinking with Adam Grant, "How do incentives really work? with Uri Gneezy"（文字稿）：<https://www.ted.com/podcasts/rethinking-with-adam-grant/how-do-incentives-really-work-with-uri-gneezy-transcript>
- The Brainy Business 播客第 273 期，"Mixed Signals: How Incentives Really Work, with Uri Gneezy"：<https://thebrainybusiness.com/podcast/273-mixed-signals-how-incentives-really-work-with-uri-gneezy/>

**原始研究**
- Gneezy & Rustichini (2000), "A Fine is a Price", *Journal of Legal Studies* 29(1):1–17：<https://rady.ucsd.edu/_files/faculty-research/uri-gneezy/fine.pdf>（DOI 頁：<https://www.journals.uchicago.edu/doi/10.1086/468061>）
- Gneezy & Rustichini (2000), "Pay Enough or Don't Pay at All", *QJE* 115(3):791–810：<https://rady.ucsd.edu/_files/faculty-research/uri-gneezy/pay-enough.pdf>
- Gneezy, Meier & Rey-Biel (2011), "When and Why Incentives (Don't) Work to Modify Behavior", *JEP* 25(4):191–210：<https://www.aeaweb.org/articles?id=10.1257/jep.25.4.191>
- Ariely, Gneezy, Loewenstein & Mazar (2009), "Large Stakes and Big Mistakes", *Review of Economic Studies* 76(2):451–469：<https://people.duke.edu/~dandan/webfiles/PapersUpside/Large%20Stakes%20Big%20Mistakes.pdf>
- Deci (1971), "Effects of externally mediated rewards on intrinsic motivation", *JPSP* 18(1):105–115：<https://selfdeterminationtheory.org/SDT/documents/1971_Deci.pdf>（DOI：<https://doi.org/10.1037/h0030644>）
- Glucksberg (1962), "The influence of strength of drive on functional fixedness and perceptual recognition", *Journal of Experimental Psychology* 63:36–41：<https://pubmed.ncbi.nlm.nih.gov/13899303/>
- Cameron & Pierce (1994), "Reinforcement, Reward, and Intrinsic Motivation: A Meta-Analysis", *Review of Educational Research* 64(3):363–423：<https://journals.sagepub.com/doi/10.3102/00346543064003363>
- Deci, Koestner & Ryan (1999), *Psychological Bulletin* 125(6):627–668：<https://pubmed.ncbi.nlm.nih.gov/10589297/>
- Cerasoli, Nicklin & Ford (2014), *Psychological Bulletin* 140(4):980–1008：<https://pubmed.ncbi.nlm.nih.gov/24491020/>
- Self-Determination Theory 官網，Theory 概述：<https://selfdeterminationtheory.org/theory/>

**報道與書評**
- Quartz (2013), "Google's '20% time,' which brought you Gmail and AdSense, is now as good as dead"：<https://qz.com/115831/googles-20-time-which-brought-you-gmail-and-adsense-is-now-as-good-as-dead>
- 西恩的閱讀筆記，【超閱讀觀點】《誘因設計》：<https://seanreads.com/mixed-signals-how-incentives-really-work/>
- Angela Watson, "Responsible rewards: using the 'now-that' principle with students"（引述《驅動力》對 if-then 與 now-that 獎勵的區分）：<https://truthforteachers.com/truth-for-teachers-podcast/responsible-rewards-using-now-principle-students/>
- 托馬斯的社群咖啡館，閱讀心得《動機，單純的力量》：<https://vocus.cc/article/6906a023fd897800014f3c6a>
