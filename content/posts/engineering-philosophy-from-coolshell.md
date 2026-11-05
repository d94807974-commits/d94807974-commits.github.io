---
title: "技术人的底层修养：从左耳朵耗子的文章里读出的工程哲学"
date: 2026-11-04T14:20:00+08:00
lastmod: 2026-11-05T13:25:00+08:00
draft: false
tags: ["左耳朵耗子", "工程哲学", "架构", "程序员成长", "CoolShell"]
summary: "重读陈皓（左耳朵耗子）在 CoolShell 与《左耳听风》里留下的文字，把散落的观点串成一套框架：学习观、编程观、架构观与职业观。核心只有一句话——把精力花在本质上，把控制交给系统。"
ShowToc: true
---

> 芝兰生于空谷，不以无人而不芳。
> —— 陈皓的 GitHub 个人简介


<figure>
<img src="/images/engineering-philosophy-from-coolshell/bookcase.jpg" alt="公共书柜" loading="lazy" width="700">
<figcaption>图片：Sir James / <a href="https://commons.wikimedia.org/wiki/File:Public_bookcase_germany_bonn_poppelsdorf_2008_08_10a.jpg">Wikimedia Commons</a>，<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC BY-SA 3.0</a>。技术文章像开放书柜：精华在于被反复取用的那几本。</figcaption>
</figure>

## 引子：为什么今天还要读 CoolShell

2023 年 5 月，陈皓（网名“左耳朵耗子”，很多人叫他“耗子叔”）因突发心梗离世，年仅 47 岁。他留下的 CoolShell（酷壳）博客，从 C++ 虚函数表一直写到 Kubernetes 网络排障、Go 编程模式、系统架构原则，二十年里影响了一代中国程序员。

三年后的今天，技术圈的热词换了一轮又一轮，AI 写代码已是日常。重读他的文章，我发现真正“不过时”的不是某个具体技术，而是他反复讲的一套**底层修养**：怎么学、怎么写、怎么设计、怎么对待自己的职业。这些观点散落在几百篇文章里，单独看像是经验之谈，放在一起看，背后其实是同一个思路：

**分清什么是本质、什么是控制；把人的精力花在本质上，把控制交给系统和工具。**

下面我试着用这条线，把他的学习观、编程观、架构观和职业观串起来，并在每部分写下我自己的理解和打算。

## 一、学习观：从“认知”到“技能”，中间没有捷径

### 1. 三级转换：认知 → 知识 → 技能

在《如何超过大多数人》里，耗子叔把一个人的能力拆成三层：先有零碎的**认知**，再通过系统学习变成**知识**，最后通过训练和实践变成**技能**。每一层都要花时间，大多数人就倒在半路上。

针对每一层，他都给了具体的方法：

- **认知层看信息源**：信息渠道（能否拿到一手信息）、信息质量（garbage in, garbage out）、信息密度（好的信息会逼着你去查资料、思考、动手验证）。
- **知识层看结构**：要建**知识树**，而不是背零散的知识点；要追问**知识的缘由**，弄清它的来龙去脉；最重要的是学到**方法套路**。他说：“学习不是为了找到答案，而是找到方法。”
- **技能层看训练方式**：精益求精（每次重复都要找更好的做法，否则只是“搬砖”）、允许自己犯错、找高手切磋。

这和 2011 年那篇著名的《程序员技术练级攻略》一脉相承。那篇文章开头就提醒：不要乱买书，不要乱追新技术新名词，基础的东西会在未来至少 10 年里通用；要回顾技术的发展历史；例子再简单也要亲手敲一遍；要多想“为什么是这样而不是那样”。

### 2. 一手资料与英文

他多次强调英文能力和一手资料的重要性。在《如何超过大多数人》里，他提醒只能拿到二手、三手信息的人，得到的是被别人解读过的东西，三传两递之后必然失真，甚至会被传递信息的“中间人”篡改；在《别让自己“墙”了自己》里，他又把英文能力看作能否融入世界的决定性因素。他推荐的学习材料几乎都是英文原著、论文和官方文档，比如在讲 Pipeline 时直接引用 Rob Pike 的博客，讲调用链追踪时直接让人去读 Google Dapper 论文。

### 3. 建一个自己的“实验室”

在介绍《程序员练级攻略（2018）》怎么用时，他给的第一条建议是：**建一个自己的实验室**。徒手把环境搭起来，写实验性的小程序去验证一个技术点，出了问题自己调试。他甚至说，不要在他的专栏里学习，要把专栏当向导和跳板，真正的学习要在线下系统地做。

### 我的反思

AI 时代，“拿到答案”的成本几乎降到了零。这让耗子叔那句“学习不是为了找到答案，而是找到方法”更加刺耳。如果我只会让 AI 给出答案，我得到的只是**认知**，连**知识**都算不上，更不用说**技能**。我的做法是：遇到 AI 给的方案，至少追问一层“为什么”，然后在本地实验室里把关键结论验证一遍。

## 二、编程观：程序 = 逻辑 + 控制

### 1. 编程的本质

《左耳听风》里的“编程范式游记”系列，从 C 的泛型讲到函数式、面向对象、Go 的委托，最后落到一篇《编程的本质》。他把 Niklaus Wirth 的“程序 = 算法 + 数据结构”和 Robert Kowalski 1979 年论文里的“算法 = 逻辑 + 控制”合在一起：

```
Program = Logic + Control + Data Structure
```

**Logic** 是业务要解决的问题，**Control** 是解决问题的策略：遍历还是递归、串行还是并行、同步还是异步、怎么重试、怎么调度。他的判断很直接：绝大多数程序混乱不堪，根本原因就是业务逻辑和控制逻辑纠缠在一起。他在文中把这句话连写了三遍：

> 有效地分离 Logic、Control 和 Data 是写出好程序的关键所在！

各种编程范式，本质上都是在做这种分离。Map/Reduce/Filter 是控制，传进去的 lambda 才是逻辑；接口抽象了逻辑，多态和依赖注入负责控制；用 DSL 描述表单校验规则（Logic），再写一个通用的 `check_form`（Control），代码立刻干净了。

### 2. 从 Guard Clauses 到 Functional Options

这条线贯穿了他很多讲代码的文章。

**《如何重构“箭头型”代码》**里，层层嵌套的 `if-else` 之所以难读，是因为错误检查（控制）把正常流程（逻辑）挤到了最深处。他的解法是 Guard Clauses：**让出错的代码先返回**，剩下的就是正常流程；再把语句块抽成函数。他还特意反驳了“代码不共享就不要抽函数”的说法：函数的作用是屏蔽细节，不只是复用。

**《Go 编程模式：Functional Options》**则是同一思想在 API 设计上的体现。一个 `Server` 有必填参数和一堆可选参数：写一组 `NewXXXServer` 很啰嗦，用 `Config` 结构体又得纠结传 `nil` 还是 `Config{}`，用 Builder 还要多一个包装类型。函数式选项把“怎么配置”变成了可组合的函数：

```go
type Option func(*Server)

func Timeout(d time.Duration) Option {
    return func(s *Server) { s.Timeout = d }
}

func NewServer(addr string, port int, opts ...Option) (*Server, error) {
    srv := Server{Addr: addr, Port: port, Protocol: "tcp", Timeout: 30 * time.Second}
    for _, opt := range opts {
        opt(&srv)
    }
    return &srv, nil
}

s, _ := NewServer("0.0.0.0", 8080, Timeout(300*time.Second))
```

**《Go 编程模式：Pipeline》**更进一步：用 channel 把 `echo → odd → sq → sum` 串起来，就像 Unix 管道 `echo $nums | sq | sum`；HTTP 中间件则用一个 `Handler(h, decors...)` 把鉴权、日志等横切关注点拼装起来。每个小函数只做一件事，组合方式就是控制。

### 3. 把两个模式合起来：重试就是“控制”

我自己做了个小练习，把耗子叔的两个观点合在一起：**重试是典型的控制逻辑**，它不应该散落在每个业务函数里，而应该是一个可配置、可复用的“控制器”，配置方式正好用 Functional Options。

```go
type retryPolicy struct {
    maxAttempts         int
    baseDelay, maxDelay time.Duration
    retryable           func(error) bool
}

type RetryOption func(*retryPolicy)

func MaxAttempts(n int) RetryOption { return func(p *retryPolicy) { p.maxAttempts = n } }
func Backoff(base, max time.Duration) RetryOption {
    return func(p *retryPolicy) { p.baseDelay, p.maxDelay = base, max }
}
func RetryIf(f func(error) bool) RetryOption { return func(p *retryPolicy) { p.retryable = f } }

// Retry 只关心“怎么重试”（Control），fn 只关心“做什么”（Logic）。
func Retry(ctx context.Context, fn func(context.Context) error, opts ...RetryOption) error {
    p := retryPolicy{maxAttempts: 3, baseDelay: 100 * time.Millisecond,
        maxDelay: 2 * time.Second, retryable: func(error) bool { return true }}
    for _, o := range opts {
        o(&p)
    }
    var err error
    for i := 0; i < p.maxAttempts; i++ {
        if err = fn(ctx); err == nil || !p.retryable(err) {
            return err
        }
        d := p.baseDelay << i // 指数退避
        if d > p.maxDelay {
            d = p.maxDelay
        }
        d = d/2 + time.Duration(rand.Int63n(int64(d/2)+1)) // 抖动，避免重试风暴
        select {
        case <-time.After(d):
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    return err
}
```

业务代码只需要写 `Retry(ctx, callPayment, MaxAttempts(5), RetryIf(isTimeout))`。以后想加熔断、加监控埋点，改的都是控制层，业务函数一行不动。

## 三、架构观：为失败而设计，把控制收口

### 1. 弹力设计：故障是常态

《左耳听风》的分布式系统系列里，“弹力设计”是我读得最多的一组。他的出发点很朴素：在分布式系统里，故障不可避免，所以设计的目标不是“不出故障”，而是**出了故障能自我保护、快速恢复**。这正是 AWS 说的 Design for Failure。

在总结篇里，他把弹力设计归成三大块：

1. **冗余服务**：消除单点，靠负载均衡、服务发现、动态路由和健康检查。
2. **服务解耦**：横向做隔离（Bulkheads，按业务分片、按用户做多租户），纵向用异步通讯；一致性问题靠补偿事务做“反向交易”。
3. **服务容错**：重试、幂等、熔断、限流、降级。

这几个模式之间是有依赖的，这是我觉得最有价值的地方：

- **重试必须配合幂等**。调用超时的时候，你不知道对方到底成功没有，安全的重试前提是“同一个请求执行多少次，结果都一样”。通常的做法是给每次调用一个全局唯一 ID，服务端据此判断请求是否已处理过：

```go
// 用 requestID 做幂等：重复请求直接返回第一次的结果，不会重复扣款
func Pay(ctx context.Context, s Store, requestID string, doPay func() (string, error)) (string, error) {
    if r, loaded, err := s.PutIfAbsent(ctx, requestID, "PROCESSING"); err != nil {
        return "", err
    } else if loaded {
        return r, nil
    }
    return doPay() // 完成后把最终结果回写到 requestID 对应的记录
}
```

- **熔断是对重试的保护**：下游已经挂了还在拼命重试，只会把它压得更死。熔断器在 Closed、Open、Half-Open 三种状态间切换，给下游喘息的时间。
- **限流和降级是对自己的保护**。他讲降级时举过一个例子：伦敦诺丁山狂欢节和阿森纳球赛散场时，公交地铁全部免费，牺牲一点收入，换来人群最快疏散。系统也一样，资源不够时要主动牺牲一些东西：一致性（强一致退为最终一致）、次要功能、返回数据的完整度。他特别提醒，降级必须事先梳理清楚哪些功能必须保住、哪些可以牺牲，并把预案写成可以自动或半自动执行的代码。

### 2. 架构原则：控制逻辑要“收口”

2021 年的《我做系统架构的一些原则》是他架构思想的集大成之作，共十一条。其中**原则七“对控制逻辑进行全面收口”**，和前面“程序 = 逻辑 + 控制”完全呼应，只是放大到了系统层面：服务发现、重试、熔断、限流、监控、调度、灰度发布，这些都是控制逻辑，应该统一收到网关、SDK、Service Mesh、Kubernetes 这类基础设施里，而不是让每个业务团队各写一遍。

他创办的 MegaEase 做的开源流量调度系统 Easegress（现在是 CNCF 项目），某种意义上就是这个理念的产品化：把流量控制从业务代码里剥离出来。

其余几条，我认为对日常工作最有指导意义的是：

- **关注真正的收益，而不是技术本身**：能否让团队并行开发、让系统更稳定、通过简化和自动化降低成本。三者都不沾的架构没有意义。
- **完备性比性能更重要**：先用支持 ACID 的关系型数据库，NoSQL 作补充，也就是“先紧后松”。他的经验是，性能问题总是有解，而丢掉的完备性很难找回来。
- **不要依赖自己的经验，要依赖数据和学习**：他说，如果有一天你开始凭以往经验做技术决定，你就已经不可能再成长了。
- **小心 X-Y 问题，追问原始需求**：很多人来问“大数据流式处理怎么做”，追问下去发现只需要做一下性能调优。
- **不要迁就技术债务**：与其花大力气迁就，不如直接还；还不了就建一个“新城区”，用防腐层隔开。

### 3. 性能：先定义，再测量，最后才优化

在《性能调优攻略》里，他先定义性能：性能就是**吞吐量（Throughput）**和**延迟（Latency）**两个指标，缺一不可。能扛一百万并发但延迟两分钟，毫无意义。然后是方法论：系统出问题时，先别急着看代码，先看操作系统报告：CPU、内存、磁盘 IO、网络 IO；再用 Profiler 找热点；常用的优化策略无非是空间换时间、时间换空间、简化代码、并行处理。

这一点和他“依赖数据而不是经验”的原则是同一件事。

### 我的反思

我以前做架构评审，最常犯的错是一上来就讨论“用什么”：上不上 MQ、用不用 Redis。耗子叔的原则逼着我先回答三个问题：**收益是什么？失败了怎么办？控制逻辑放在哪？**这三个问题答清楚了，选型往往自然就定了。

## 四、职业观：效率、投资与“别墙了自己”

### 1. 效率不是干得多，而是价值大

2013 年的《加班与效率》里，他对“靠加班超越对手”的说法批评得非常尖锐：这说明管理者已经想不出自己公司的核心价值了。他给效率下的定义是：

> 效率不是比谁干的活多，而是比谁干得活有更大的价值。

他用“有用功 / 总功”来拆解效率：增加有用功（多问需求方为什么要做、能不能简化，“有用功的来源不是拼命做需求，而是砍需求”），降低总功（减少支持性和重复性的工作），形成合力（整个团队对整个产品负责）。他还介绍了 Amazon 的 T-Shirt Size 估算：业务影响力和投入成本各估一个尺码，影响小、成本大的需求直接砍掉。

在《什么是工程师文化？》里，他把工程师文化概括为**自由 + 效率**，并认为在自由的环境下痴迷于提升效率，创新就会自然发生。效率的具体做法包括：简化（“简化不是简陋”）、无情地推行自动化、正确的抽象、高质量的产品、持续改进。

### 2. 20 到 30 岁，练学习能力和解决问题的能力

《技术人员的发展之路》写于他 40 岁那年。他认为 20 到 30 岁是最关键的十年，首要任务是练就**高效的学习能力**和**解决难题的能力**，标志是在团队里显现出 Leadership。他特别强调，Leadership 不是当领导和经理，而是一种特征：大家遇到问题时，总是你能站出来说该怎么办（帮人解问题）；大家做关键决定时，会来征求你的意见（被人所依赖）。一旦进入“能力强 → 解决更难的问题 → 学到更多”的正循环，人生的可能性就会大大增加。

《程序员如何把控自己的职业》延续了这个思路：看清趋势（他认为技术演进的规律就是自动化加规模化），打牢基础以不变应万变，提升成长的效率。他把技术人分成技工、特种工、工程师、设计师/架构师、经理几类，懂原理的“特种工”不容易随着某项技术一起被淘汰。

### 3. 投资而不是投机

《别让自己“墙”了自己》里有一段我很认同：大多数人只会算自己短期会失去什么，优秀的人算的是投入以后未来能得到什么，前者是投机，后者是投资。他劝人做有价值的事，打开眼界，站在更高的维度思考：与其焦虑薪水、地点、户口这些低维度的东西，不如关心自己的可能性是否更多、能否解决更大更难的问题。

### 不是圣经，是一种思考方式

公平地说，耗子叔的很多观点相当犀利，有些也很有争议，比如“企业级架构选 Java 基本不会错”，比如对国内一些互联网产品的批评。他自己也在文章里说明，这些架构原则适用于比较复杂的业务，简单应用可能会得出相反的结论。读他的文章，我觉得最好的方式不是全盘照搬结论，而是学他得出结论的方法：追问本质、看数据、看历史、看一手资料，然后自己判断。这本身也是他一直倡导的独立思考。

## 五、我的行动清单

把上面这些落到自己身上，我给自己定了几条可以检查的规矩：

1. **每学一个新技术，画出知识树，并追溯它要解决的原始问题**。说不清“它为什么出现”，就不算学会。
2. **一手资料优先**：官方文档、RFC、论文、原作者博客，二手解读只作索引。
3. **维护一个个人实验室仓库**，每个重要结论配一个能跑的最小实验。
4. **写代码时先问：这行是 Logic 还是 Control？**能抽出去的控制逻辑（重试、超时、校验、日志）尽量抽成可复用的组件。
5. **设计评审必答三问**：收益是什么？失败时如何降级？控制逻辑收口在哪？
6. **做决策前先拿数据**：压测、Profiler、监控，而不是凭感觉或凭经验。
7. **用“有用功 / 总功”审视自己的一周**：哪些需求该砍，哪些重复劳动该自动化。
8. **每年做一次“投资复盘”**：今年的时间投在了哪里，有没有让自己的可能性变多。

## 结语：灯熄了，光还在

耗子叔离开之后，很多人自发写文章悼念他。我和他素未谋面，只是一个读着 CoolShell 成长起来的普通程序员。他的文字里没有“速成”。在 2018 版练级攻略里，他坦言是故意把内容写得让人望而生畏，因为本来就没有捷径，很多知识是硬骨头，只能一口一口地啃。他相信基础、相信动手、相信效率、相信开放，也身体力行地把自己的技术写出来、开源出来，分享给所有人。

他在 GitHub 的个人简介写着：“芝兰生于空谷，不以无人而不芳。”我想，对一个技术人最好的纪念，大概就是把他讲过的那些朴素道理，认真地用在自己每天写的代码和做的设计里。

谨以此文，纪念陈皓（1976—2023）。

## 参考资料

- 陈皓，[《程序员技术练级攻略》](https://coolshell.cn/articles/4990.html)，CoolShell，2011
- 陈皓，[《程序员练级攻略（2018) 与我的专栏》](https://coolshell.cn/articles/18360.html)，CoolShell，2018
- 陈皓，[《如何超过大多数人》](https://coolshell.cn/articles/19464.html)，CoolShell，2019
- 陈皓，[《别让自己“墙”了自己》](https://coolshell.cn/articles/20276.html)，CoolShell，2019
- 陈皓，[《Go 编程模式：Functional Options》](https://coolshell.cn/articles/21146.html)，CoolShell，2020
- 陈皓，《Go 编程模式：Pipeline》，CoolShell，2020（[极客时间版](https://time.geekbang.org/column/article/332610)）
- 陈皓，[《如何重构“箭头型”代码》](https://coolshell.cn/articles/17757.html)，CoolShell，2017
- 陈皓，[《我做系统架构的一些原则》](https://coolshell.cn/articles/21672.html)，CoolShell，2021
- 陈皓，[《性能调优攻略》](https://coolshell.cn/articles/7490.html)，CoolShell，2012
- 陈皓，[《加班与效率》](https://coolshell.cn/articles/10217.html)，CoolShell，2013
- 陈皓，[《什么是工程师文化？》](https://coolshell.cn/articles/17497.html)，CoolShell，2016
- 陈皓，[《技术人员的发展之路》](https://coolshell.cn/articles/17583.html)，CoolShell，2016
- 陈皓，[《程序员如何把控自己的职业》](https://coolshell.cn/articles/20977.html)，CoolShell，2020
- 陈皓，[《左耳听风》专栏](https://time.geekbang.org/column/48)，极客时间：[《弹力设计：认识故障和弹力设计》](https://time.geekbang.org/column/article/3912)、[《弹力设计：隔离设计》](https://time.geekbang.org/column/article/3917)、[《编程范式游记（9）：编程的本质》](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/%e5%b7%a6%e8%80%b3%e5%90%ac%e9%a3%8e/038%20%20%e7%bc%96%e7%a8%8b%e8%8c%83%e5%bc%8f%e6%b8%b8%e8%ae%b0%ef%bc%889%ef%bc%89-%20%e7%bc%96%e7%a8%8b%e7%9a%84%e6%9c%ac%e8%b4%a8.md)、[《弹力设计总结》](https://www.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E5%B7%A6%E8%80%B3%E5%90%AC%E9%A3%8E/051%20%20%E5%BC%B9%E5%8A%9B%E8%AE%BE%E8%AE%A1%E7%AF%87%E4%B9%8B%E2%80%9C%E5%BC%B9%E5%8A%9B%E8%AE%BE%E8%AE%A1%E6%80%BB%E7%BB%93%E2%80%9D.md)、[《降级设计》](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E5%B7%A6%E8%80%B3%E5%90%AC%E9%A3%8E/050%20%20%E5%BC%B9%E5%8A%9B%E8%AE%BE%E8%AE%A1%E7%AF%87%E4%B9%8B%E2%80%9C%E9%99%8D%E7%BA%A7%E8%AE%BE%E8%AE%A1%E2%80%9D.md)（后三篇为第三方存档）
- Robert Kowalski, *Algorithm = Logic + Control*, Communications of the ACM, 1979
- Rob Pike, [Go Concurrency Patterns: Pipelines and cancellation](https://go.dev/blog/pipelines)
- [haoel 的 GitHub 主页](https://github.com/haoel)、[haoel/leetcode](https://github.com/haoel/leetcode)、[Easegress](https://github.com/easegress-io/easegress)
- [CoolShell 全站 Markdown 备份（slan-ning/coolshell-markdown）](https://github.com/slan-ning/coolshell-markdown)
- 陈皓离世相关报道：[澎湃新闻](https://www.thepaper.cn/newsDetail_forward_23100097)、[央广网](https://news.cnr.cn/native/gd/20230516/t20230516_526252880.shtml)（2023-05）；[维基百科：左耳朵耗子](https://zh.wikipedia.org/zh-cn/%E5%B7%A6%E8%80%B3%E6%9C%B5%E8%80%97%E5%AD%90)
