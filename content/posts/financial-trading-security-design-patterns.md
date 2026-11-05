---
title: "让每一笔钱都可证明：金融交易系统安全设计模式全景"
date: 2026-11-04T14:36:21+08:00
lastmod: 2026-11-05T13:25:00+08:00
draft: false
tags: ["安全架构", "金融科技", "设计模式", "支付", "交易系统", "Go"]
summary: "从孟加拉国央行、Knight Capital、Capital One，以及台湾第一银行 ATM 盗领与远东商银 SWIFT 遭骇五起事故出发，提出资金动作的六个不变量，并整理 SCA、所见即所签、签名防重放、幂等记账、复式账本、Maker-Checker、HSM/信封加密、事前风控与熔断等 13 个安全设计模式，并对照台湾金管会、银行公会、证交所与香港金管局、证监会的繁体中文监管文件，附可编译 Go 示例与落地清单。"
ShowToc: true
---

> 本文讨论的是金融交易类软件（支付、转账、证券/期货下单、清结算、账务核心）的**安全设计模式**。文中的事实、数字与法规条款均来自文末列出的公开资料；代码为可编译的 Go 示意实现（`go vet` 通过），用于说明思路而非直接上线。
>
> 本文首发后，补充了繁体中文资料并重写：台湾金管会新闻稿与裁罚公告、银行公会《金融機構辦理電子銀行業務安全控管作業基準》、台湾证交所《建立證券商資通安全檢查機制》、香港金管局与证监会的公告和指引，以及中央社、iThome 对两起台湾事故的报道。繁体原文保留繁体字。


<figure>
<img src="/images/financial-trading-security-design-patterns/padlock.jpg" alt="挂锁" loading="lazy" width="450">
<figcaption>图片：Alyson Hurt / <a href="https://commons.wikimedia.org/wiki/File:Padlock.jpg">Wikimedia Commons</a>，<a href="https://creativecommons.org/licenses/by/2.0/">CC BY 2.0</a>。金融安全不只是一把锁，而是可证明的闭环。</figcaption>
</figure>

## 引子：五起事故，五种"正确的失败"

先看三个被反复引用、但常被讲错重点的案例，再看两起发生在台湾、繁体中文资料很完整的案例。

**2016 年孟加拉国央行事件。** 攻击者攻破孟加拉国央行网络、观察转账操作并拿到凭证后，通过 SWIFT 向纽约联储发出三十多笔指令，试图从其账户转走 9.51 亿美元。5 笔共 1.01 亿美元得手，其中流向斯里兰卡的 2000 万美元因收款方名称拼写错误（"Fandation"）被拦下，流向菲律宾的 8100 万美元截至 2025 年只追回约 1500 万；其余 30 笔约 8.5 亿美元被纽约联储阻止。作案时间选在周末并叠加春节假期，延误了止付。事后 SWIFT 推出了客户安全计划（CSP）。

这起事件最值得玩味的是：**从系统视角看，那些指令都是"合法"的**——凭证真实，报文格式正确。认证没有失败，失败的是"认证之后的一切"。

**2012 年 Knight Capital 事件。** 根据美国 SEC 的处罚令，8 月 1 日开盘后约 45 分钟里，Knight 的路由系统 SMARS 在处理 212 笔小额零售订单时向市场发出了数百万笔订单，在 154 只股票上成交 400 万笔、超过 3.97 亿股，形成约 35 亿美元的净多头，最终亏损超过 4.6 亿美元。根因是一段 2003 年就停用却仍可调用的 "Power Peg" 旧代码、一个被新功能复用的旧标志位，以及一次漏掉 8 台服务器中 1 台的部署；此外还缺少订单比对、停机流程和有效告警（后文逐一展开）。这是 SEC 依据市场准入规则（Rule 15c3-5）采取的第一起执法行动，罚款 1200 万美元。

Knight 没有被"黑"。它说明了金融系统安全的另一半：**完整性和可用性的敌人，常常是你自己的代码与发布流程。**

**2019 年 Capital One 事件。** Capital One 公告称，攻击者利用一个配置漏洞，于 2019 年 3 月 22—23 日获取数据，涉及美国约 1 亿、加拿大约 600 万人，包括约 14 万个社会安全号和约 8 万个关联银行账号，未涉及卡号和登录凭证。美国司法部的通报提到，被告用自己编写的工具扫描 AWS 上配置错误的账户并借此入侵。美国货币监理署（OCC）对其罚款 8000 万美元，理由是在把重要 IT 业务迁移到公有云之前，**未能建立有效的风险评估流程**。

三起事故的共同点是：**单点控制都"在"，只是没有形成闭环。**

**2016 年第一银行 ATM 盗领事件。** 据 iThome 汇整的调查局鉴识结果，2016 年 7 月 10、11 日清晨，车手在第一银行台北、台中 22 家分行的 41 台 ATM 上，不插卡、不输密码就领走了 8,327 万 7,600 元新台币。入侵路径大致是：

1. 攻击者先控制一银**伦敦分行**一台"锁在铁柜中的电话录音伺服器主机"，以此为跳板进入总行内网；
2. 7 月 4 日，经由 **ATM 软件派送伺服器**发出一个"更新包"，把 ATM 的 Telnet 服务改为自动开启，ATM 重开机后即可被远程登录；
3. 7 月 9 日，派送木马 `cnginfo.exe`、`cngdisp.exe` 等，吐钞时每次 60 张；
4. 事后用批处理 `cleanup.bat` 调用微软的 `sdelete.exe` 删除木马和日志。调查局是在一台"自毁程式执行失效"的 ATM 上才找到样本。

调查局还发现，这些恶意程序只在 2016 年 7 月有效，逾期失效，这是其鉴识实验室"成立十年來，少見會指定發作月份的惡意程式"。警方 7 月 18 日逮捕 3 名洗钱嫌犯，追回 6,024 万元，两天后又在内湖西湖公园找到一千多万元，"僅剩數百萬元未尋獲"。

这起事件里最值得记住的一句话，来自调查局主管私下的透露：一银保护客户资料的数据库防护"非常嚴謹"，但内部系统之间可能基于「都是自己人」的心态，海外分行连回总行的系统"往往只需要簡單的帳號、密碼就可以順利登入"。**被攻破的不是 ATM，而是"内网互信"这个假设；被滥用的也不是漏洞，而是正常的软件更新通道。**

**2017 年远东商银 SWIFT 遭骇事件。** 金管会 2017 年 12 月 12 日的裁罚公告写道：远东商银于 2017 年 10 月 3 日及 10 月 5 日发现 SWIFT 系统遭电脑病毒及骇客入侵，产生虚伪交易汇出款项，总额约 6,010 万美元（合新台币 18 亿元），"嗣後已追回多數款項"，损失在 16 万美元以内。金管会认定其违反银行法第 45 条之 1 第 1 项，核处新台币 800 万元罚款，列出的缺失包括：

- 资安防御机制未完整建置；
- 系统管理者账号管理欠当；
- 未落实强化 SWIFT 系统安全；
- 未有效传达法令；
- 未发挥内部控制第三道防线功能。

中央社的报道更具体。金管会检查局副局长叶淑媛说，远银"沒有符合最小授權原則，造成高權限的SWIFT（環球財務通訊系統）帳號、密碼遭盜取"。金管会在孟加拉央行事件后已于 2016 年 9 月发函银行公会，要求各银行做好网段隔离，但远银「只做半套」：

> 工作站的做法有合規，但主機卻沒有做到網段的隔離。

iThome 还记录了金管会的一段解释：远银为何没有人工稽核每笔 SWIFT 交易？因为"如果作好SWIFT系統實體隔離，交易均由系統自動處理完成，因此沒有人工再次稽核的程序"。也就是说，**整个控制链建立在"隔离一定有效"这个前提上，前提一破，后面没有第二道闸。**

五起事故放在一起，共同点更清楚了：**单点控制都"在"，只是没有形成闭环。**

## 一、核心论点：给每一个"动钱的动作"定义不变量

传统安全叙事喜欢讲边界。金融系统的经验恰恰相反：边界会被穿透，凭证会被盗，代码会出错。真正可依赖的，是围绕**每一个资金动作**建立的一组不变量：

1. **可认证（Authenticated）**：发起者是谁，必须有多因素证据，而不仅是"会话还在"。
2. **内容绑定（Bound）**：被授权的是"这一笔、这个金额、这个收款方"，任何字段被改动，授权即失效。
3. **恰好一次（Once）**：无论网络如何重试、消息如何重复，账上只发生一次。
4. **不可篡改地留痕（Recorded）**：只追加的流水，余额可由流水推导，可以独立对账。
5. **有界（Bounded）**：单笔、单日、频率、总敞口都有上限，而且在事前检查。
6. **可停止（Stoppable）**：任何时候都存在一个比故障传播更快的"刹车"。

孟加拉国央行事件是第 1 条成立而第 5 条缺失；Knight 缺的是第 4、5、6 条；Capital One 则是这一切的"环境前提"出了问题。远银和孟加拉央行几乎是同一个剧本：凭证是真的、报文是对的，缺的是第 5 条的事前上限和独立复核；一银则说明第 4 条的"留痕"必须离开被攻击的主机，否则 `sdelete` 一跑，证据就没了。下面的模式目录，就是把六个不变量落到架构上的具体手段。

一笔转账在系统中的理想路径如下：

```text
 客户端/柜员端                        网关层                            交易核心
┌──────────────┐  ①签名+时间戳+nonce ┌──────────────────┐            ┌──────────────────────┐
│ SCA:知识/持有 │ ──────────────────▶ │ 验签 / 防重放     │            │ 幂等键查重(唯一约束)  │
│ /生物 + 设备  │                     │ 限流 / 反自动化   │──②────────▶│ 事前风控:限额/频率/敞口│
│ 动态链接码    │ ◀── 展示金额+收款方 │ BOLA/BFLA 授权    │            │ Maker-Checker(大额)   │
└──────────────┘                     └──────────────────┘            │ 复式记账:借贷平衡      │
        ▲                                     │                       └──────────┬───────────┘
        │③ 结果(可安全重试)                    │ 审计事件                        │ 只追加流水
        │                                     ▼                                  ▼
        │                           ┌──────────────────┐          ┌──────────────────────────┐
        └───────────────────────────│ 检测与响应(SIEM)  │◀─────────│ 对账:内部流水 vs 外部清算 │
                                    │ 告警→自动熔断     │─Kill────▶│ 敞口实时汇总 / 熔断开关  │
                                    └──────────────────┘  Switch  └──────────────────────────┘
          密钥:HSM/KMS 托管主密钥,信封加密数据;敏感号码令牌化;所有服务间调用零信任认证
```

## 二、威胁建模：把 STRIDE 翻译成金融语言

微软的 STRIDE 模型把威胁分成六类：仿冒（Spoofing）、篡改（Tampering）、抵赖（Repudiation）、信息泄露（Information Disclosure）、拒绝服务（DoS）和权限提升（Elevation of Privilege）。它的价值不在分类本身，而在于强迫你对**每一条数据流**逐项发问。落到金融场景，我建议这样对照：

| STRIDE | 金融场景中的典型形态 | 对应的主要模式 |
|---|---|---|
| 仿冒 | 盗用凭证发起转账、伪造商户回调、冒充内部服务 | SCA、设备绑定、请求签名、服务间 mTLS |
| 篡改 | 中间人改收款账号、改金额、改交易参数；篡改流水 | 动态链接、签名覆盖报文体、只追加账本 |
| 抵赖 | "这笔不是我做的"、柜员否认审批 | 不可否认签名、审计日志、Maker-Checker 留痕 |
| 信息泄露 | 卡号、证件号、持仓泄露 | 令牌化、信封加密、最小权限、数据最小化 |
| 拒绝服务 | 撮合/支付通道被打爆；失控算法耗尽资源 | 限流、熔断、容量隔离、灾备 |
| 权限提升 | 越权查看他人账户（BOLA）、普通用户调用管理接口（BFLA） | 对象级/功能级授权、职责分离 |

还要注意 STRIDE 不容易覆盖的一类：**合法但有害**的业务流。OWASP API Security Top 10（2023）把"对敏感业务流的无限制访问"列为 API6——攻击者每一步都没越权，只是用自动化把业务逻辑变成武器，比如薅羊毛和撞库后的小额试探。

## 三、模式目录

以下每个模式按"问题 / 方案 / 实现要点 / 代价"组织。

### 模式 1：强客户认证与设备绑定

**问题。** 密码会泄露，会话会被劫持，孟加拉国央行事件中攻击者甚至拿到了真实的操作凭证。

**方案。** 欧盟 PSD2 的监管技术标准（RTS，条例 2018/389）第 4 条给出了很好的工程化定义：认证需基于知识、持有、固有特征中的**两个或以上**要素；生成的认证码只能被接受一次，不可伪造，也无法由认证码反推出任何要素；认证失败时不得暴露是哪个要素出错；连续失败最多 5 次即应锁定；会话无操作的超时不超过 5 分钟。

**香港和台湾的写法。** 香港证监会 2017 年 10 月 27 日发布《降低及紓減與互聯網交易相關的黑客入侵風險指引》，列出 20 项基本监控规定。其中客户登入互联网交易账户的**双重认证**于 2018 年 4 月 27 日生效，其余规定于 2018 年 7 月 27 日生效。证监会的常见问题有两条特别工程化：

- "雙重密碼"不算双重认证，因为"雙重密碼只構成認證程序中的單一因素（即"客戶所知的"）"；
- 用短信一次性密码的经纪行，"便應告誡其客戶不要設定短訊轉發"。

香港金管局《监管政策手册》TM-E-1（V.4，2024 年 10 月 25 日，只备英文版本）则要求：网银在进行高风险交易前，每个登录会话至少做一次 2FA；高风险交易至少包括向客户**未登记的第三方收款人**转账。

台湾银行公会的《金融機構辦理電子銀行業務安全控管作業基準》走得更远。它参考 ISO 29115，"以身分登錄、信物管理、身分驗證三個面向"，把身份核验机制分为最高、高、中、低、最低五个信赖等级，再与交易风险对应。

**实现要点。** "持有"要素在移动端通常是**设备绑定**：在设备安全区生成密钥对、登记公钥，之后每次敏感操作用私钥对挑战签名。密码泄露了，没有那台设备也无法认证。

**代价。** 摩擦会降低转化率；换机、丢机的恢复流程常是社会工程的突破口，应与首次绑定同等严格。

### 模式 2：动态链接，所见即所签

**问题。** 认证了"人"，不等于认证了"这笔交易"。恶意软件可以在用户确认之后、提交之前替换收款账号。

**方案。** RTS 第 5 条要求"动态链接"：付款人在认证时被告知金额和收款方；认证码与该金额、该收款方一一对应；**金额或收款方的任何变化都会使认证码失效**；并且要保护展示给用户的信息的机密性与完整性。这就是"所见即所签"（WYSIWYS）的监管表述。

**实现要点。** 签名输入是"挑战 + 规范化后的交易摘要"，服务端用同样规则重算比对。关键在**展示通道**：展示的金额与被签名的数据若来自不同代码路径，绑定就形同虚设。

**代价。** 规范化规则（字段顺序、金额精度、编码）一旦两端不一致，就会出现"签了却验不过"的故障，需要版本化并配套测试向量。

### 模式 3：请求签名与防重放

**问题。** API 层面，攻击者可以截获一条合法请求原样重发，或在传输中改动报文体。TLS 保护的是链路，不保护"同一条请求被发两次"，也不保护网关之后的内部链路。

**方案。** 对"方法 + 路径 + 时间戳 + 随机数 nonce + 报文体摘要"做 HMAC 签名。服务端做三件事：校验时间戳在窗口内、以常量时间比较签名、确认 nonce 在窗口内未出现过。AWS 的 SigV4 就是这一思路的成熟范例，其文档明确：请求必须在请求中时间戳的五分钟之内到达 AWS，否则会被拒绝。

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

运行结果：首次请求通过；原样重放返回 `nonce replayed`；改了收款账号的请求返回 `bad signature`；十分钟前的请求返回 `timestamp outside window`。

**实现要点。** 时间窗口让 nonce 只需保存有限时长（生产中用 Redis `SET NX` 加过期即可）；签名必须覆盖报文体摘要；比较必须用 `hmac.Equal` 这类常量时间函数。

**代价。** 依赖时钟同步；多活部署时 nonce 存储若不跨机房一致，可在两地各重放一次。对称密钥意味着服务端也能伪造签名，**不提供不可否认性**，需要时应改用客户端私钥签名（模式 1）。

### 模式 4：幂等键与"恰好一次"记账

**问题。** 最危险的失败是**不确定的失败**：请求超时了，钱扣没扣？不重试会丢单，盲目重试会重复扣款。

**方案。** Stripe 的工程博客把这个问题讲得很透：网络失败是模糊的，客户端应该重试直到成功，而幂等键保证了"恰好一次"的语义；重试应使用带随机抖动的指数退避。其 API 文档进一步规定：服务端保存首次请求的结果，相同幂等键的后续请求直接返回这个结果；如果同一个键携带了不同的参数，会返回错误；键在至少 24 小时后才会被清理。

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

运行结果：`tx-42` 首次记账与网络超时后的重试都返回序号 0、余额只增加一次；同一个键换了金额会被拒绝；借贷不平的分录直接拒绝。

**实现要点。** 演示中的互斥锁，在生产中对应：**在同一个数据库事务里**插入幂等记录（键有唯一约束）并写入分录。在缓存里查重、在另一个事务里记账，等于没有幂等。保存请求摘要，可以识别"同一个键被用于不同请求"的客户端 bug。

**代价。** 幂等表持续增长，需要过期策略，而过期时间应远大于客户端的最大重试周期。跨系统的"恰好一次"还要求下游也支持幂等。

### 模式 5：复式记账、只追加流水与对账

**问题。** 直接 `UPDATE balance = balance - 100` 的系统，一旦出错，既无法知道错在哪里，也无法证明余额是对的。

**方案。** Modern Treasury 的开发者记账系列给出了两条原则：存储不可变的交易流水并**始终由流水计算余额**，因为直接修改余额的系统容易出错，且难以发现和对账；复式记账的力量在于所有借方余额之和始终等于所有贷方余额之和——不相等就意味着系统凭空造出或丢失了钱。上面的 `Post` 函数在入口强制分录和为零，`Balance` 由流水推导，正是这两条原则的最小实现。

**实现要点。** 更正只通过追加冲正分录完成，并在数据库层面收回业务账号对流水表的 `UPDATE/DELETE` 权限。**对账**是另一半：内部流水与清算机构、银行、交易所逐笔比对。Knight 事件中，SEC 特别指出公司没有任何控制去比对离开 SMARS 的订单与进入 SMARS 的订单——这正是一个缺失的实时对账。

**代价。** 读放大，需要引入余额快照，而快照又要与流水定期核对。

### 模式 6：Maker-Checker 与职责分离

**问题。** 单个人——无论是内鬼、被盗号的员工，还是一个疲惫的运维——能独立完成一个高风险动作。

**方案。** NIST 对职责分离的定义是：不应给任何单个用户足以独自滥用系统的权限，例如批准工资单的人不应同时是制作工资单的人；它可以静态实施（定义互斥角色），也可以动态实施（在访问时强制）。在交易系统里，它通常表现为"制单—复核"的四眼原则。

远银事件给这个模式补了一个反面注脚：SWIFT 交易"均由系統自動處理完成"，没有人工再次稽核。自动化本身没错，错在**自动化路径上没有任何一个不依赖同一组凭证的复核点**。四眼原则不一定要靠两个人，也可以是一个独立的、拿不到 SWIFT 操作员凭证的规则引擎，比如对大额、新收款行、非营业时间的报文二次核对。

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

**实现要点。** 第一，复核人批准的必须是**他看到的那个版本**：审批绑定内容摘要，指令被改则审批失效——这是模式 2 在内部流程上的投影。第二，四眼原则要覆盖**变更与发布**：Knight 的部署由一人完成、无人复核，8 台服务器漏了 1 台。

**代价。** 最大风险是"橡皮图章"。应按风险分层，只把真正高风险的动作交给人，并抽样回溯复核质量。

### 模式 7：密钥分层——HSM/KMS、信封加密与令牌化

**问题。** 难的不是加密，而是密钥放在哪、谁能用。

**方案。** 三层结构：

- **根密钥留在硬件里。** AWS KMS 的文档说明，KMS 密钥在其硬件安全模块（HSM）中生成，非对称私钥永远不会以未加密的形式离开 KMS。
- **信封加密。** AWS Encryption SDK 为每条消息生成唯一的数据密钥，用它加密数据，再用包装密钥（密钥加密密钥）加密这把数据密钥，并把加密后的数据密钥与密文存在一起。这样对主密钥的调用次数与数据量无关，轮换主密钥也无需重加密全部数据。加密上下文（encryption context）可以把"这份密文属于哪个租户、哪张表"绑定进认证数据中，防止密文被挪用。
- **令牌化。** PCI 安全标准委员会的令牌化指南（2011 年 8 月）指出，令牌化可以减少环境中的持卡人数据，从而可能缩小 PCI DSS 的适用范围；但如果商户能取回主账号（PAN），其环境仍在范围内。

**实现要点。** 业务系统只接触令牌，只有极少数服务能调用去令牌化接口，且每次调用都记审计日志。签名密钥（模式 3）和数据密钥都由 KMS/HSM 托管，应用进程里只有短期凭证。更上游的一步是**数据最小化**：泄露的影响与"存了多少"成正比——Capital One 事件中被访问的最大一类数据，是 2005 年至 2019 年初申请信用卡时提交的信息。不需要的不采集，日志框架统一掩码，测试环境只用脱敏数据。

**代价。** HSM 的吞吐和延迟对高频交易是硬约束，通常只用于签发会话密钥；令牌库本身成为最高价值目标，需要独立隔离。

### 模式 8：最小权限与零信任

**问题。** 内网默认可信，一次入侵就能横向扩散。

**方案。** NIST SP 800-207 把零信任描述为：把防御从静态的网络边界转移到用户、资产和资源；不因资产或账户所处的物理或网络位置、或资产归属而授予隐式信任。SWIFT 的客户安全控制框架（CSCF）也按同一逻辑组织为三大目标：保护你的环境（限制互联网访问、隔离关键系统、减少攻击面、物理安全），了解并限制访问（防止凭证泄露、管理身份并分离权限），检测与响应（检测对系统或交易记录的异常活动、制定事件响应与信息共享计划）。

台湾金管会 2022 年 12 月 27 日发布的「金融資安行動方案」2.0 也写进了零信任：鼓励金融机构逐步导入"身分鑑別、設備鑑別及信任推斷等零信任網路3大核心機制，搭配網路及資源的細化權限管控"。

一银和远银恰好是两个反例：

- 一银的问题是内网互信：海外分行与总行"並沒有實質且明顯的分野"，一台录音服务器就能走到 ATM 派送系统；
- 远银的问题是授权过大：叶淑媛解释，最小授权原则是"系統管理超過授權範圍時，須經過內部一定的控制程序後，才能一步步開放局部的權限"，而远银"一開始給的管理權限卻是最高權限"。

**实现要点。** 支付报文网关与办公网隔离；服务间用工作负载身份双向认证；云权限"每服务一个最小角色"，并自动扫描过宽策略。Capital One 的教训在于：在云上，**配置就是边界**，而迁移上云前的风险评估本身就是一项安全控制。

**代价。** 权限申请过慢时，工程师会倾向于申请"一劳永逸"的大权限，需要用临时提权、自动过期来平衡。

### 模式 9：事前风控——限额、频率与总敞口

**问题。** 合法凭证 + 合法格式 + 异常规模 = 灾难。孟加拉国央行一次发出的指令总额接近 10 亿美元；Knight 在 45 分钟里累积了 35 亿美元的头寸。

**方案。** SEC Rule 15c3-5 要求有市场准入的券商具备系统化的风险管理控制，合理设计以**防止错误订单的进入**，以及防止超过预设信用和资本阈值的订单——这些阈值要**按每个客户和券商自身汇总**计算。关键词是"事前"和"汇总"。Knight 的处罚令指出，其 9.5% 的价格区间控制不覆盖开盘前的订单；出问题的账户也没有与公司层面的资本阈值挂钩；而它的监控工具只在成交后显示头寸，完全依赖人工盯盘，也不显示限额。

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

**实现要点。** 示例里有三个刻意的设计：订单**发出即计入敞口**，不等成交回报（Knight 正是在成交后才看到头寸）；限额检查在发单路径上同步执行；触及总阈值时**自动熔断**，而不是发一封告警邮件——Knight 开盘前收到的 97 封自动邮件无人处置。

**代价。** 事前检查增加延迟；限额过紧误杀业务，过松形同虚设，需要有人对其负责并定期校准。

**台港监管中的"约定账户"。** 台湾银行公会基准把交易指示分成高风险和低风险两类："高風險交易"指执行结果"對客戶權益有重大影響之各類電子轉帳及交易指示，包含非約定轉帳交易超過辦理低風險交易最高限額之交易指示"，而"約定轉入帳戶之設定及轉帳"被列为低风险。香港 TM-E-1 用的是同一个概念：向"未登记收款人"转账才是高风险。两地的做法可以翻译成一条工程规则：**收款方是否事先登记，是风控里最便宜也最有效的特征之一**。登记本身要走高信赖等级的认证，登记之后的转账才可以降低摩擦。

### 模式 10：熔断开关与安全降级

**问题。** 当故障传播的速度超过人类反应速度时，"发现问题再处理"已经来不及。

**方案。** 每个能动钱的组件都要有一个独立的、经过演练的"停止"能力：按账户、按策略、按通道、全局四个粒度。SEC 就 Knight 事件发布的新闻稿中，执法部门官员的一句话值得每个架构师抄在墙上："Brokers and dealers must look at each component in each of their systems and ask themselves what would happen if the component malfunctions and what safety nets are in place to limit the harm it could cause."

**实现要点。** 熔断开关必须是**最简单的代码路径**：一个原子布尔值，在发单入口第一个检查，不依赖可能已经出故障的组件。停止之后要能"安全地维持现状"，而不是匆忙回滚——Knight 在应急中卸载新代码反而扩大了损失，说明应急动作本身也需要预案和演练。

**代价。** 误触发同样有损失，触发条件、授权人和恢复流程需事先约定并演练。

### 模式 11：反自动化与业务流保护

**问题。** 攻击者不越权，只是比人快一万倍。

**方案。** OWASP API4（无限制的资源消耗）提醒：执行超时、可分配内存、上传文件大小、单次请求内的操作数、每页返回的记录数，甚至第三方服务的花费上限，任何一项缺失或设置不当都构成漏洞。API6（对敏感业务流的无限制访问）给出的缓解措施包括：设备指纹（拒绝无头浏览器等非预期客户端）、人机识别（验证码或打字节奏等生物行为特征）、识别非人类模式（例如在一秒内完成"加入购物车"和"完成购买"）、考虑封禁 Tor 出口节点和知名代理。

台湾证交所对证券商的要求也落在同一处："應針對帳號登入及IP異常進行紀錄及分析，符合特定態樣應立即通知客戶及產生示警，以降低入侵或帳號撞庫攻擊之風險。"

**实现要点。** 限流按 IP、设备、账户、收款方、业务流分层，"收款方维度"常比"付款方维度"更能发现团伙作案。还要警惕 API9（资产清单管理不当）：被遗忘的旧版接口往往没有这些保护。

**代价。** 设备指纹与行为数据本身是敏感个人信息；误判会挡住真实用户，需要申诉通道。

### 模式 12：安全交付——从代码到二进制的可信链

**问题。** Knight 的根因不是攻击，而是一段停用了九年的代码、一个被复用的标志位和一次漏掉一台服务器的部署。

**方案。** NIST SP 800-218（SSDF）指出，大多数软件开发生命周期模型没有详细处理安全，因此需要把一组核心的安全开发实践嵌入每个模型。SLSA 框架则关注供应链：即使代码经过扫描，从源码到构建、打包、分发的每一环都存在被篡改的风险，SLSA 通过自动化追踪代码从源码到二进制的处理过程，确保对源码做过的审查仍然适用于最终运行的二进制。应用层的验证要求可以参考 OWASP ASVS，当前稳定版本为 5.0.0。

一银事件说明，可信链还要覆盖**分发与更新通道**：攻击者没有改 ATM 上的任何正式程序，而是借用 ATM 软件派送伺服器发出一个"更新包"，ATM "自動按照例行系統更新程序執行"。更新包如果必须带有离线签名密钥的签名、终端在安装前验签，这条路就走不通了。

**实现要点。** 三条"Knight 条款"：**删除死代码**而非用开关屏蔽；**永不复用**有历史语义的标志位；部署后自动核对每台主机的运行版本，不一致就不放流量。再加一条"一银条款"：**终端只安装经过验签的更新包，日志实时送到终端之外**，本地的 `sdelete` 删不掉远端副本。

**代价。** 构建溯源和准入检查会拖慢紧急修复；解决办法不是绕过，而是让紧急通道同样自动化，只是审批更快。

### 模式 13：韧性与灾难恢复

**问题。** 可用性本身就是金融安全：支付中断、无法平仓同样是真实损失。

**方案。** 新加坡金管局（MAS）的技术风险管理指南（2021 年 1 月 18 日发布）把目标定为建立健全的技术风险治理，并保持 IT 与网络的韧性；我国的网络安全等级保护基本要求（GB/T 22239-2019）也把安全运维管理等内容纳入基本要求。工程上，关键是把恢复目标（可容忍丢失多少数据、多久恢复）落实到架构，并定期做真实的切换演练。

金管会行动方案 2.0 把这件事写得很具体：鼓励重要金融机构强化核心资料保全，"包含核心資料檔案、資料庫加密與分持，儲存於第三地或雲端備份等機制"，并依行业特性订定核心业务系统备援演练指引，鼓励演练时"納入對外服務實際運作，驗證其有效性"。台湾证交所则要求证券商每年评估核心系统可容忍中断时间、RTO 和 RPO 的妥适性。

**实现要点。** 切换时最容易出现"同一笔交易两边各执行一次"，只有账本具备幂等键和唯一约束，切换后的重放才安全。此外要有**不能被生产凭证删除**的不可变备份。

**代价。** 多活架构会极大增加一致性问题的复杂度。

## 四、合规框架是地图，不是领土

PCI DSS（当前为 v4.0.1；v4.0 的 64 项新要求中有 51 项已于 2025 年 3 月 31 日生效）、SWIFT CSCF（强制参与，每年 7 月发布新版，7 月 1 日至 12 月 31 日间通过 KYC-SA 提交证明，控制项与 ISO 27002、PCI DSS、NIST CSF 等对齐）、等保 2.0（GB/T 22239-2019，2019 年 12 月 1 日实施，含云计算、移动互联等扩展要求）及金融行业的 JR/T 0071-2020 实施指引，再加上前文的 PSD2 RTS 和 SEC Rule 15c3-5，构成了一张很好的覆盖面地图。

**台湾与香港的繁体中文监管文件**，可以补上这张地图在亚洲的一角：

| 辖区 | 文件 | 要点（据原文） |
|---|---|---|
| 台湾 金管会 | 「金融資安行動方案」2.0（2022-12-27） | 40 项措施：新增 12 项、扩大适用 5 项、持续 23 项；三年为期、每季检讨；要求一定规模以上金融机构设置"副總經理層級以上之資安長"；推动 ISMS、SOC、零信任、MITRE ATT&CK 演训、F-ISAC 情资分享 |
| 台湾 银行公会 | 《金融機構辦理電子銀行業務安全控管作業基準》（我核对的是 1120614 版本） | 参考 ISO 29115 的五级信赖等级；交易指示按高、低风险分类，约定转入账户与非约定转账区别对待 |
| 台湾 证交所 | 《建立證券商資通安全檢查機制》 | 参考 ISO 27001 架构，共 14 个控制领域；网络下单须加密连线并使用多因子验证；最高权限账号须经权责主管同意并留存纪录；提供网络下单的核心系统异动时须执行源码扫描 |
| 香港 金管局 | 「網絡防衛計劃」（2016）与 2.0（2020-11-03 公布，2021-01-01 起分阶段生效） | 三大支柱：網絡防衛評估框架（C-RAF）、專業培訓計劃、網絡風險資訊共享平台；超过 90% 的银行认为 C-RAF 有助发现以往未能识别的缺口 |
| 香港 金管局 | TM-E-1「電子銀行的風險管理」（V.4，2024-10-25，只备英文） | 高风险交易前每会话至少一次 2FA；对可疑交易要求额外认证 |
| 香港 证监会 | 《降低及紓減與互聯網交易相關的黑客入侵風險指引》（2017-10-27） | 20 项基本监控规定；登入双重认证 2018-04-27 生效，其余 2018-07-27 生效；"双重密码"不算 2FA |

但合规清单天然是"控制项是否存在"的视角。Knight 有价格区间控制，有监控工具，有告警邮件——每一项单独看都"存在"，组合起来却没有一项能在 45 分钟内止损。**审计问的是"有没有"，事故问的是"闭不闭环"。** 远银是更直接的例子：监管 2016 年就要求网段隔离，远银的工作站合规、主机没有隔离，"只做半套"；而检查局通常每两年才对一家金融机构做一次一般金检，远银当年不在名单上。地图画得再细，也要有人在现场走一遍。

## 五、我的反思：这些模式的边界与权衡

**第一，安全与延迟在交易系统里是真实的零和。** 我的做法是按"能否造成不可逆损失"给检查分层：不可逆的一律同步、事前、自动；可逆的可以异步、事后、抽样。

**第二，幂等和对账是最被低估的安全控制。** 它们常被当作"可靠性"问题，但重复扣款、凭空造钱在客户和监管眼里就是安全事件。不能由流水重算余额、不能与外部逐笔对账的系统，谈不上完整性。

**第三，人是控制，也是漏洞。** 注意力会在第一百笔审批后耗尽；97 封告警邮件也说明"通知了人"不等于"控制了风险"。能写成规则的，就写成会自动阻断的代码。

**第四，最危险的往往是"合法"的输入。** 真实凭证、正确格式、自家代码发出的订单——所以检测不能只看攻击特征，还要看是否偏离了这个账户、这个策略的正常形态。

**第五，模式互相依赖，单独使用会带来虚假的安全感。** 防重放依赖时钟与密钥，熔断依赖准确的敞口汇总，灾备依赖幂等。评审时应画出依赖图，找出共享密钥服务、时间源这类"一处失效、多处失守"的单点。

**最后，本文的局限：** 案例与条款以欧美为主，台港部分补充了监管新闻稿、裁罚公告和指引原文，但金管会行动方案 2.0 我只读了新闻稿，没有逐项读完 PDF；香港 C-RAF 2.0 的附件只备英文，我没有展开；一银和远银的技术细节来自调查局披露与媒体报道，远银的入侵手法金管会以警方调查中为由没有公开；国内标准只核实到名称与适用范围；代码是单机教学实现，省略了持久化、分布式一致性和密钥轮换。

## 六、可执行清单

**认证与授权**
- [ ] 资金类操作强制多因素认证，失败不提示具体要素，连续失败锁定；账户恢复流程与首次绑定同等严格。
- [ ] 签名输入包含金额与收款方摘要，任何字段改动导致验证失败；展示与签名数据来自同一来源。
- [ ] 所有 API 做对象级与功能级授权测试，维护接口资产清单。
- [ ] 收款方先登记、后转账；登记本身走最高信赖等级认证，向未登记收款人转账一律按高风险处理。
- [ ] "双重密码"不算 2FA；用短信 OTP 时提醒客户不要设置短信转发。

**请求与交易语义**
- [ ] 外部与内部关键接口使用"时间戳 + nonce + 报文体摘要"签名，常量时间比较，nonce 有界存储。
- [ ] 所有写操作接受幂等键；幂等记录与分录在同一事务中写入，键有唯一约束，并保存请求摘要。
- [ ] 账本只追加，分录借贷平衡在写入时强制；更正只用冲正；业务账号无流水表的 UPDATE/DELETE 权限。
- [ ] 每日（关键链路实时）与外部清算、银行、交易所逐笔对账，差异自动告警并阻断。

**治理与密钥**
- [ ] 大额与敏感操作执行 Maker-Checker，审批绑定内容摘要；变更与发布同样四眼。
- [ ] 根密钥在 HSM/KMS，数据用信封加密并带加密上下文；卡号、证件号令牌化，去令牌化调用全量审计。
- [ ] 服务间零信任认证，云权限最小化并持续扫描配置漂移；迁移上云前完成正式风险评估。
- [ ] 海外分行、子公司、外包与内网之间不默认互信；管理员账号默认低权限、按需提权并留存纪录。

**运行时防护**
- [ ] 单笔、单日、频率、按客户与公司汇总的敞口限额在事前同步检查，订单发出即计入敞口。
- [ ] 账户、策略、通道、全局四级熔断开关，触发条件、授权人、恢复流程成文并每季度演练。
- [ ] 分层限流与反自动化（设备、账户、收款方、业务流维度），并为误判提供申诉通道。

**交付与韧性**
- [ ] 构建可溯源、制品签名；删除死代码、不复用旧标志位；部署后自动核对每台主机的运行版本。
- [ ] 终端（ATM、交易终端）只安装验签通过的更新包；日志实时外送，不能只存在本机。
- [ ] 定义并演练恢复目标；保留不可被生产凭证删除的不可变备份；切换后依赖幂等安全重放。

## 参考资料

**事故与执法**
- SEC, In the Matter of Knight Capital Americas LLC（Release No. 34-70694）：https://www.sec.gov/litigation/admin/2013/34-70694.pdf
- SEC 新闻稿 2013-222，SEC Charges Knight Capital With Violations of Market Access Rule：https://www.sec.gov/newsroom/press-releases/2013-222
- SEC, Risk Management Controls for Brokers or Dealers with Market Access（Rule 15c3-5）：https://www.sec.gov/rules-regulations/2011/06/risk-management-controls-brokers-or-dealers-market-access
- Capital One, Information on the Capital One Cyber Incident：https://www.capitalone.com/digital/facts2019/
- OCC 新闻稿 2020-101（Capital One 罚款）：https://www.occ.gov/news-issuances/news-releases/2020/nr-occ-2020-101.html
- 美国司法部华盛顿西区检察官办公室，Former Seattle tech worker convicted of wire fraud and computer intrusions：https://www.justice.gov/usao-wdwa/pr/former-seattle-tech-worker-convicted-wire-fraud-and-computer-intrusions
- Wikipedia, Bangladesh Bank robbery：https://en.wikipedia.org/wiki/Bangladesh_Bank_robbery

**台湾与香港（繁体中文）**
- 金管会新闻稿：〈金管會發布「金融資安行動方案」2.0，引導金融資安持續精進〉，2022-12-27：https://www.fsc.gov.tw/ch/home.jsp?dataserno=202212270001&dtable=News&id=96&mcustomize=news_view.jsp&parentpath=0%2C2
- 金管会裁罚公告：〈遠東國際商業銀行SWIFT系統遭駭重大偶發事件所涉缺失…核處新臺幣800萬元罰鍰〉，2017-12-12：https://www.fsc.gov.tw/ch/home.jsp?aplistdn=ou%3Dnews%2Cou%3Dmultisite%2Cou%3Dchinese%2Cou%3Dap_root%2Co%3Dfsc%2Cc%3Dtw&dataserno=201712120005&dtable=News&id=96&mcustomize=news_view.jsp&parentpath=0%2C2
- 中央社：〈遠銀遭駭18億元 金管會列三大缺失〉，2017-10-12：https://www.cna.com.tw/news/afe/201710120379.aspx
- iThome：〈【遠銀遭駭追追追】遠銀被駭案，金管會初步調查：銀行未做好內控管理〉，2017-10-12：https://www.ithome.com.tw/news/117380
- iThome：〈【詳細圖解】駭客入侵一銀ATM流程追追追〉，2016-07-25：https://www.ithome.com.tw/news/107294
- iThome：〈《獨家》調查局揭露：一銀ATM盜領惡意程式指定在7月發作，逾期失效〉，2016-07-15：https://www.ithome.com.tw/news/107136
- 银行公会：《金融機構辦理電子銀行業務安全控管作業基準》（1120614 版本，信托公会转载）：https://www.trust.org.tw/upload/1120001034-0-0.pdf
- 台湾证交所：〈簡介建立證券商資通安全檢查機制〉：https://www.twse.com.tw/market_insights/zh/detail/8a8216d6933460a4019371c2555e0222
- 香港金管局新闻稿：〈金管局宣布推出「網絡防衛計劃2.0」〉，2020-11-03：https://www.hkma.gov.hk/chi/news-and-media/press-releases/2020/11/20201103-4/
- 香港金管局：《監管政策手冊》TM-E-1「電子銀行的風險管理」（V.4，只备英文）：https://brdr.hkma.gov.hk/chi/doc-ldg/docId/20241101-3-TC
- 香港证监会通函：〈實施《降低及紓減與互聯網交易相關的黑客入侵風險指引》〉，2017-10-27：https://apps.sfc.hk/edistributionWeb/gateway/TC/circular/intermediaries/supervision/doc?refNo=17EC72
- 香港证监会常见问题：〈網絡保安〉：https://www.sfc.hk/TC/faqs/intermediaries/supervision/Cybersecurity/Cybersecurity

**标准与法规**
- 欧盟委员会授权条例 (EU) 2018/389（PSD2 RTS）第 4 条：https://www.legislation.gov.uk/eur/2018/389/article/4
- 同上，第 5 条（动态链接）：https://www.legislation.gov.uk/eur/2018/389/article/5
- Swift, Understand Controls（CSCF）：https://www.swift.com/myswift/customer-security-programme-csp/security-controls
- PCI SSC, PCI DSS v4.x Resource Hub：https://blog.pcisecuritystandards.org/pci-dss-v4-0-resource-hub
- PCI SSC, Now is the Time to Adopt the Future-Dated Requirements of PCI DSS v4.x：https://blog.pcisecuritystandards.org/now-is-the-time-for-organizations-to-adopt-the-future-dated-requirements-of-pci-dss-v4-x
- PCI SSC, Information Supplement: PCI DSS Tokenization Guidelines（2011）：https://www.pcisecuritystandards.org/documents/Tokenization_Guidelines_Info_Supplement.pdf
- Wikipedia, Payment Card Industry Data Security Standard：https://en.wikipedia.org/wiki/Payment_Card_Industry_Data_Security_Standard
- MAS, Technology Risk Management Guidelines：https://www.mas.gov.sg/regulation/guidelines/technology-risk-management-guidelines
- 国家标准 GB/T 22239-2019《信息安全技术 网络安全等级保护基本要求》：https://openstd.samr.gov.cn/bzgk/std/newGbInfo?hcno=BAFB47E8874764186BDB7865E8344DAF
- 全国标准信息公共服务平台，JR/T 0071—2020 发布公告：https://hbba.sacinfo.org.cn/snDetail/8951901cd64d412b54b858c7bb903baa7d4595b196dd07aaa1232d26804119fd
- 电子银行网，中国人民银行发布《金融行业网络安全等级保护实施指引》：https://www.cebnet.com.cn/20201124/102704588.html

**安全框架与工程实践**
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
