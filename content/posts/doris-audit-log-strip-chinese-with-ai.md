---
title: "用 AI 给私有化 Doris 打补丁：重新编译 FE 让审计日志去中文（2.1.6 / 2.1.2 实战）"
date: 2026-11-05T23:45:00+08:00
lastmod: 2026-11-05T23:45:00+08:00
draft: false
tags: ["Doris", "审计日志", "Log4j2", "FE", "私有化部署", "AI 协作", "Java", "运维"]
categories: ["工程实践"]
description: "私有化 Apache Doris 的 fe.audit.log 需要去掉中文，但 conf/log4j2-spring.xml 是 FE 启动时代码生成的，手改会被覆盖。本文记录和 AI 协作的完整过程：定位 Log4jConfig、只改一个类、用官方二进制包做 classpath 单类编译、逐字节自证一致、jar uf 替换，并给出 2.1.6-rc04 与 2.1.2-rc04 两个版本的测试结果、免换 jar 方案及其局限。"
summary: "私有化 Doris 的 fe.audit.log 要去中文，可日志配置是 FE 每次启动代码生成的。记录如何与 AI 协作：读源码定位 Log4jConfig、最小改动加 %replace / RegexFilter、只编译一个类并逐字节自证、jar uf 替换，两个版本（2.1.6-rc04、2.1.2-rc04）实测四种模式。"
ShowToc: true
---

> **一句话结论**：审计日志的 log4j2 配置是 `org.apache.doris.common.Log4jConfig` 在 FE 启动时拼出来的，所以要“抗重启”，就得改这个生成器本身。整个补丁只替换 `doris-fe.jar` 里的 **一个 class**；不全量编译 Doris，而是用官方二进制包的 `lib/*` 做 classpath，单独编译这一个类。
>
> 本文所有数字都来自这次的真实操作记录。服务器地址、主机名、集群 ID、真实安装路径都已脱敏，命令里统一用 `${DORIS_HOME}`、`./work`、`./src`、`$JAVA_HOME` 这类占位符。

## 一、背景与需求

手上有两套私有化部署的 Apache Doris，都是单 FE（Master）：

| 集群 | FE 版本（`SHOW FRONTENDS` 的 Version） |
|---|---|
| A | `doris-2.1.6-rc04-653e315ba5` |
| B | `doris-2.1.2-rc04-b130df2488` |

需求很简单：**FE 的审计日志 `fe.audit.log` 里不要出现中文**。SQL 里的中文字符串常量、中文别名、中文注释都算。

麻烦在于，`${DORIS_HOME}/fe/conf/` 里那个 `log4j2-spring.xml` 头部写着：

```xml
<!-- Auto Generated. DO NOT MODIFY IT! -->
<Configuration status="info" packages="org.apache.doris.common">
```

FE **每次启动都会重新生成并覆盖它**，手改审计 appender 的 pattern 撑不过一次重启。把文件设成只读也不行：FE 写不进去，直接启动失败。

## 二、和 AI 协作的过程

这次的分工大致是这样的：

| 人提供 | AI 负责 |
|---|---|
| 准确版本号（`SHOW FRONTENDS\G` 的 `Version` 字段） | 按 tag 拉对应源码，定位生成 XML 的代码（`Log4jConfig`） |
| 线上自动生成的 `log4j2-spring.xml` 内容（appender 名、pattern） | 先判断有没有**不换 jar** 就能做到的办法（配置项、自定义模板） |
| 线上 `doris-fe.jar` 的 sha256 | 改代码、单类编译、证明“未改动源码编译结果 = 官方 class” |
| 最终决定和上线操作 | 起测试 FE，跑四种模式 + 重启 + 运行时改日志级别，生成 README / diff / 校验和 |

**一个真实的教训：版本号拿错过。**一开始对话里给的版本是 2.1.2，AI 就照着 2.1.2 改了。后来用户贴出线上生成的 XML，里面有 `AuditFile`、`AuditConsole` 和 `<!--REPLACED BY AUDIT LOG MARKER-->`。AI 一核对源码发现：**2.1.2 根本没有这些结构**（它只有一个叫 `Auditfile` 的审计 appender，小写 f），这个 marker 要到 2.1.4 之后才出现。再跑一次 `SHOW FRONTENDS`，确认线上其实是 2.1.6-rc04，于是 2.1.2 的产物全部作废重做。

所以第一步永远是：

```sql
SHOW FRONTENDS\G   -- 只看 Version 字段，例如 doris-2.1.6-rc04-653e315ba5
```

版本号最后那段是 commit 短 hash。拿它和官方二进制包里 `Version` 类记录的 `doris_build_short_hash` 对一下，就能确认源码 tag 选对了。

后来 B 集群（真的是 2.1.2-rc04）也要同样的补丁。这次 AI 没有直接套用 2.1.6 的文件，而是按 2.1.2 的源码结构重新移植了一遍（差异见第四节末尾）。

## 三、原理：为什么改生成代码就能抗重启

![FE 启动时审计日志配置的生成与加载流程（橙色为补丁新增步骤）](/images/doris-audit-log-strip-chinese-with-ai/flow.svg)

读 [2.1.6-rc04 的 Log4jConfig.java](https://github.com/apache/doris/blob/2.1.6-rc04/fe/fe-core/src/main/java/org/apache/doris/common/Log4jConfig.java) 可以看到，FE 启动时 `initLogging()` → `reconfig()` 会做这几件事：

1. 用代码拼出一个 XML 模板，里面带各种 `<!--REPLACED BY ...-->` 占位符；
2. 替换占位符，再用 `StrSubstitutor` 把 `${sys_log_dir}` 这类变量填进去；
3. `SpringLog4j2Config.writeSpringLogConf()` 把结果写到 `conf/log4j2-spring.xml`；
4. 直接在内存里 `context.start(config)` 加载这份配置；
5. 之后 [HttpServer](https://github.com/apache/doris/blob/2.1.6-rc04/fe/fe-core/src/main/java/org/apache/doris/httpv2/HttpServer.java) 启动 Spring Boot 时，通过 `logging.config` 指向这份文件，再加载一次。

另外，通过 HTTP `POST /rest/v1/log` 在运行时改日志级别，会调用 `Log4jConfig.updateLogging()` → `reconfig()`，把整套流程再走一遍。

结论很直接：**过滤规则只要写进这个生成器，无论是重启、Spring 重新加载，还是运行时 reconfig，生成出来的配置都会带上它。**

Log4j2 有两种现成手段可以用：

- **strip（删除中文）**：用 PatternLayout 的 [`%replace{pattern}{regex}{substitution}`](https://logging.apache.org/log4j/2.x/manual/pattern-layout.html) 对 `%m` 做正则替换；
- **drop（整行丢弃）**：用 [`RegexFilter`](https://logging.apache.org/log4j/2.x/manual/filters.html)，消息匹配就 `DENY`。

### 关键 diff（2.1.6-rc04，精简）

```diff
     private static final String AUDIT_LOG_PATTERN
-            = AUDIT_LOG_MARKER_PLACEHOLDER + "%d{yyyy-MM-dd HH:mm:ss,SSS} [%c{1}] %m%n";
+            = AUDIT_LOG_MARKER_PLACEHOLDER + "%d{yyyy-MM-dd HH:mm:ss,SSS} [%c{1}] "
+            + AUDIT_LOG_MESSAGE_PLACEHOLDER + "%n";
 ...
             .append("    <Console name=\"" + AUDIT_LOG_CONSOLE_APPENDER + "\" target=\"SYSTEM_OUT\">\n")
+            .append("      " + AUDIT_LOG_FILTER_PLACEHOLDER + "\n")
 ...
             .append("    <RollingFile name=\"" + AUDIT_LOG_FILE_APPENDER + "\" fileName=\"${audit_log_dir}/fe.audit.log\" ...>\n")
+            .append("      " + AUDIT_LOG_FILTER_PLACEHOLDER + "\n")
 ...
+    static String auditMessagePattern() {
+        if (auditFilterMode.equals("strip")) {
+            return xmlEscape("%replace{%m}{" + auditChineseRegex + "}{" + auditChineseReplacement + "}");
+        }
+        return "%m";
+    }
+
+    static String auditFilterElement() {
+        if (auditFilterMode.equals("drop")) {
+            return "<RegexFilter regex=\"" + xmlEscape("(?s).*(?:" + auditChineseRegex + ").*")
+                    + "\" useRawMsg=\"false\" onMatch=\"DENY\" onMismatch=\"NEUTRAL\"/>";
+        }
+        return "";
+    }
 ...
         newXmlConfTemplate = strSub.replace(newXmlConfTemplate);
+        // 在 ${} 替换之后、用 String.replace（不是 replaceAll）注入，正则里的反斜杠原样保留
+        newXmlConfTemplate = newXmlConfTemplate.replace(AUDIT_LOG_MESSAGE_PLACEHOLDER, auditMessagePattern());
+        newXmlConfTemplate = newXmlConfTemplate.replace(AUDIT_LOG_FILTER_PLACEHOLDER, auditFilterElement());
 ...
         customConfDir = Config.custom_config_dir;
+        loadAuditFilterConf(confDir, customConfDir);
         reconfig();
```

有几个细节：

- **配置项由补丁自己从 `fe.conf` / `fe_custom.conf` 读取**，不用改 `Config.java`，所以只需要替换一个 class。Doris 的 `ConfigBase` 会忽略不认识的配置项，回滚到原版 jar 时这些配置项留在 fe.conf 里也没影响。
- **注入时机放在 `${}` 替换之后**，并且用 `String.replace` 而不是 `replaceAll`。否则 `\p{IsHan}` 里的反斜杠会被当成替换串的转义符吃掉。
- **注入的内容会先做 XML 转义**（`& < > "`），不会因为正则里的特殊字符把 XML 写坏。
- **配置值非法时 FE 直接拒绝启动**，例如 `audit_log_filter_chinese = foo` 会在 fe.out 里报 `audit_log_filter_chinese config error: foo, must be none/strip/drop`。审计配置错了就该尽早暴露，而不是悄悄失效。

### 生成后的 pattern 对比（AuditFile）

```xml
<!-- 原版 / none -->
<Pattern><!--REPLACED BY AUDIT LOG MARKER-->%d{yyyy-MM-dd HH:mm:ss,SSS} [%c{1}] %m%n</Pattern>

<!-- strip（默认） -->
<Pattern><!--REPLACED BY AUDIT LOG MARKER-->%d{yyyy-MM-dd HH:mm:ss,SSS} [%c{1}] %replace{%m}{[\p{IsHan}\p{InCJKSymbolsAndPunctuation}\p{InHalfwidthAndFullwidthForms}\p{InCJKCompatibilityForms}\p{InBopomofo}]+}{}%n</Pattern>

<!-- drop：appender 内多一行 -->
<RegexFilter regex="(?s).*(?:[\p{IsHan}\p{InCJKSymbolsAndPunctuation}\p{InHalfwidthAndFullwidthForms}\p{InCJKCompatibilityForms}\p{InBopomofo}]+).*" useRawMsg="false" onMatch="DENY" onMismatch="NEUTRAL"/>
```

除此之外，生成文件的结构完全不变：logger、路径、滚动策略、marker 都一样。在 none / strip 模式下，原来放 filter 的位置会多出一行空白。

### 2.1.2-rc04 的差异

[2.1.2-rc04 的 Log4jConfig.java](https://github.com/apache/doris/blob/2.1.2-rc04/fe/fe-core/src/main/java/org/apache/doris/common/Log4jConfig.java) 结构更老，移植时做了这些调整：

- 模板是一整段字符串常量 `xmlConfTemplate`，不是 `StringBuilder`，所以直接在模板里加占位符；
- 只有一个审计 appender `Auditfile`，没有 `AuditConsole`，也没有 AUDIT LOG MARKER；
- 没有 `LogUtils`，启动信息改用 `System.out.println` 打印（fe.out 里没有 `StdoutLogger` 前缀）；
- 三个配置项、默认值、校验规则、注入方式和 2.1.6 版完全一致。

## 四、不全量编译：只编译一个类

全量编译 Doris FE 要拉一大堆依赖，耗时长，还可能因为构建环境不同，编出和官方不一样的字节码。这里只改了一个类，所以完全可以：

1. 拿**同版本官方二进制包**里的 `fe/lib/*` 做 classpath；
2. 用 JDK 8 只编译 `Log4jConfig.java`；
3. **先编译未改动的源码，证明结果和官方 jar 里的 class 逐字节一致**，再编译改过的版本；
4. 用 `jar uf` 只替换这一个条目。

官方 jar 是 JDK 8 编译的：`Version` 类里记录的编译器分别是 1.8.0_352（2.1.6）和 1.8.0_312（2.1.2）；class 文件 major version 为 52（Java 8，见 [JVM 规范第 4 章](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)）。补丁 class 也必须是 52，这样在 JDK 8/11/17 上都能加载。

可复用的命令（以 2.1.6-rc04 为例）：

```bash
# 0) 源码：只需要这一个文件
git clone --filter=blob:none --no-checkout https://github.com/apache/doris.git ./src
cd ./src && git fetch --depth 1 origin tag 2.1.6-rc04
mkdir -p ../work/orig/org/apache/doris/common
git show 2.1.6-rc04:fe/fe-core/src/main/java/org/apache/doris/common/Log4jConfig.java \
  > ../work/orig/org/apache/doris/common/Log4jConfig.java
cd ..

# 1) 官方二进制包解压后的 fe/lib 作为 classpath（这里记为 ./work/apache-doris-2.1.6-bin-x64）
LIB=./work/apache-doris-2.1.6-bin-x64/fe/lib
export PATH=$JAVA_HOME/bin:$PATH     # JDK 8

# 2) 自证：未改动源码 → 与官方 class 逐字节一致（注意 -g，否则调试信息不同）
javac -g -source 8 -target 8 -nowarn -cp "$LIB/*" -d ./work/cls/orig \
  ./work/orig/org/apache/doris/common/Log4jConfig.java
mkdir -p ./work/official && (cd ./work/official && unzip -qo "../../$LIB/doris-fe.jar" 'org/apache/doris/common/Log4jConfig*')
for f in ./work/official/org/apache/doris/common/Log4jConfig*.class; do
  cmp "$f" "./work/cls/orig/org/apache/doris/common/$(basename "$f")" && echo "same $(basename "$f")"
done
javap -v ./work/cls/orig/org/apache/doris/common/Log4jConfig.class | grep major   # major version: 52

# 3) 编译改过的源码（./work/new/... 是打过 patch 的 Log4jConfig.java）
javac -g -source 8 -target 8 -nowarn -cp "$LIB/*" -d ./work/cls/new \
  ./work/new/org/apache/doris/common/Log4jConfig.java

# 4) 复制一份官方 jar，只更新 Log4jConfig.class
cp "$LIB/doris-fe.jar" ./work/doris-fe.jar
(cd ./work/cls/new && jar uf ../../doris-fe.jar org/apache/doris/common/Log4jConfig.class)
```

两个踩过的小坑：

- 编译时**不加 `-g`**，内部类 `Log4jConfig$Tuple.class` 会和官方不一致；加上 `-g` 后两个 class 都逐字节一致。
- 2.1.2 版改动后新增了 import，导致行号变化，重新编译出来的 `Tuple.class` 只差行号表。所以 `jar uf` **只更新 `Log4jConfig.class`**，内部类继续用官方原文件。

## 五、配置项与测试结果

| 配置项（写在 `${DORIS_HOME}/fe/conf/fe.conf`） | 取值 | 默认 | 说明 |
|---|---|---|---|
| `audit_log_filter_chinese` | none / strip / drop | **strip** | none=原样；strip=删除中文；drop=含中文的审计记录整行不写 |
| `audit_log_chinese_replacement` | 字符串 | 空 | strip 时用来替换中文的串，例如 `?`；不能包含 `{ } % $ \` |
| `audit_log_chinese_regex` | Java 正则 | `[\p{IsHan}\p{InCJKSymbolsAndPunctuation}\p{InHalfwidthAndFullwidthForms}\p{InCJKCompatibilityForms}\p{InBopomofo}]+` | 默认覆盖汉字、中文标点、全角字符和注音；不能包含 `$`、`%`，花括号必须成对 |

读取顺序是 fe.conf → fe_custom.conf（后者覆盖前者），也可以用 JVM `-D` 参数覆盖。**注意：换上补丁 jar 后，即使什么都不配，默认也是 strip。**

测试 SQL：

```sql
select '你好，世界！abc' as `中文别名` /* 注释：测试 */;
```

两个版本分别用“官方二进制包 + 补丁 jar”起单 FE（JDK 1.8.0_504）测试，fe.audit.log 里 `Stmt=` 的实测结果一致：

| 模式 | fe.audit.log 中的 Stmt |
|---|---|
| none | <code>select &#39;你好，世界！abc&#39; as &#96;中文别名&#96; /&#42; 注释：测试 &#42;/</code> |
| strip（默认，不写任何配置） | <code>select &#39;abc&#39; as &#96;&#96; /&#42;&nbsp;&nbsp;&#42;/</code> |
| strip + replacement=`?` | <code>select &#39;?abc&#39; as &#96;?&#96; /&#42; ? &#42;/</code> |
| drop | 这条不写入；同批的英文 SQL（`select 'hello world' as en_only`）正常写入 |

另外还验证了这些情况（两个版本都测过）：

- 每次重启后 `conf/log4j2-spring.xml` 的 mtime 都会变，内容带 `%replace` 或 `RegexFilter`；
- `POST /rest/v1/log?add_verbose=org.apache.doris.qe` 在运行时触发 reconfig 后，过滤仍然生效；
- 跨两行的 SQL 也能正确处理（Doris 写审计日志前会把换行替换成空格）；
- 2.1.2 版在 `sys_log_mode = ASYNC` 下同样生效；
- 配置非法值时 FE 拒绝启动。

上线后的验证命令：

```bash
grep "audit log chinese filter" ${DORIS_HOME}/fe/log/fe.out | tail -1
ls -l ${DORIS_HOME}/fe/conf/log4j2-spring.xml          # mtime 应该是本次启动时间
grep -n 'replace{%m}\|RegexFilter' ${DORIS_HOME}/fe/conf/log4j2-spring.xml
tail -n 3 ${DORIS_HOME}/fe/log/fe.audit.log           # 以 fe.conf 里实际的审计日志目录为准
```

## 六、交付前的防错

**1）线上 jar 必须是官方原版。**如果线上的 `doris-fe.jar` 被别人改过，拿官方 jar 打补丁再替换上去，就会把那些改动悄悄抹掉。所以先比对 sha256：

| 版本 | 官方 `fe/lib/doris-fe.jar` sha256 |
|---|---|
| 2.1.6-rc04 | `6005bb99e7b13200ff192252d2a07d32a82fd3bf3a23c4cfe1078a33f7e7c937` |
| 2.1.2-rc04 | `608177785eaab4bc3aea3cf9757341aead84d2c29129531acbbf2e489b3a8337` |

这次 2.1.6 集群线上 jar 的 sha256 和官方一致。

**2）补丁 jar 要逐个条目比对，确认只差一个 class。**`jar uf` 会重新压缩，jar 的文件大小会变，所以不能看大小，要比对每个条目的 CRC：

```python
import zipfile
a = zipfile.ZipFile('official/doris-fe.jar')
b = zipfile.ZipFile('patched/doris-fe.jar')
da = {i.filename: i.CRC for i in a.infolist()}
db = {i.filename: i.CRC for i in b.infolist()}
print(len(da), len(db), set(da) ^ set(db), [k for k in da if k in db and da[k] != db[k]])
```

实测结果：

- 2.1.6：`7354 7354 set() ['org/apache/doris/common/Log4jConfig.class']`
- 2.1.2：`6955 6955 set() ['org/apache/doris/common/Log4jConfig.class']`

**3）替换和回滚（由人来操作，每个 FE 都要做）：**

```bash
cd ${DORIS_HOME}/fe
sha256sum lib/doris-fe.jar                 # 必须等于上表中对应版本的官方值
./bin/stop_fe.sh
cp -p lib/doris-fe.jar ./backup/doris-fe.jar.orig   # 备份到 lib 目录以外
cp ./patch/doris-fe.jar lib/doris-fe.jar
echo "audit_log_filter_chinese = strip" >> conf/fe.conf   # 可选，默认就是 strip
./bin/start_fe.sh --daemon

# 回滚
./bin/stop_fe.sh && cp -p ./backup/doris-fe.jar.orig lib/doris-fe.jar && ./bin/start_fe.sh --daemon
```

## 七、免换 jar 的方案：`-Dlogging.config` 及其局限

用户最初问的是：有没有不换 jar 的办法？Doris 没有现成的配置项能控制审计 pattern。不过读 HttpServer 源码可以发现，`logging.config` 是作为 Spring 的**默认属性**（优先级最低）传进去的。因此可以在 fe.conf 的 `JAVA_OPTS`（JDK 17 对应 `JAVA_OPTS_FOR_JDK_17`）最前面加上：

```bash
-Dlogging.config=${DORIS_HOME}/fe/custom/log4j2-audit-strip.xml
```

让 Spring Boot 加载一份放在 conf 目录之外、不会被覆盖的静态 log4j2 配置。在 2.1.6 上用**官方原版 jar** 实测，审计日志里的中文确实被删掉了，但局限也很明显：

1. **运行时一改日志级别就失效。**通过 `/rest/v1/log` 修改日志级别会触发 `reconfig()`，切回代码生成的配置，审计日志马上又出现中文，直到下次重启。这一点已实测复现。
2. 静态 XML 不再跟随 fe.conf 里 `sys_log_*` / `audit_log_*` 的变化，需要手工维护。
3. 从 FE 启动到 HTTP 服务起来之前，用的仍是原配置。这段时间还没有查询，所以基本不影响审计日志。

这个方案在 2.1.2 上只读了源码（机制相同），没有实测。考虑到第 1 条，最终选择了换 jar。

## 八、风险与边界

- **`__internal_schema.audit_log` 表不受影响。**内置审计插件 AuditLoader 写表走的是另一条路径，不经过这个 appender，表里仍是原始 SQL（含中文）（见官方文档 [审计日志](https://doris.apache.org/zh-CN/docs/3.x/admin-manual/audit-plugin/)）。如果表里也要去中文，需要另外处理。fe.log、fe.warn.log 同样不受影响（见 [FE 日志管理](https://doris.apache.org/zh-CN/docs/3.x/admin-manual/log-management/fe-log/)）。
- **drop 模式有合规风险**：含中文的查询会从 fe.audit.log 里完全消失。一般建议用 strip。
- **strip 会改变 SQL 文本**：日志里的 SQL 可能没法直接复制执行。SqlHash 仍按原始 SQL 计算。
- **jar 和版本强绑定**：2.1.6 和 2.1.2 的 jar 不能混用；升级 Doris 后必须基于新版本源码重新打补丁。
- 测试只覆盖了 JDK 8 运行时和单 FE。多 FE 时每个 FE 都要替换。

## 九、小结：用 AI 做私有化补丁的几条经验

1. **给足上下文，尤其是准确版本号。**`SHOW FRONTENDS` 的 Version、线上生成的配置文件、线上 jar 的 sha256，比任何口头描述都可靠。这次走过的唯一一段弯路，就是版本号拿错了。
2. **让 AI 自证一致性。**不要只要一个“编译好的 jar”，而是要它证明：未改动源码编译出的 class 与官方逐字节一致，补丁 jar 与官方 jar 只差目标 class。这两条证据成立，替换的风险就很可控。
3. **最小改动。**只改生成器一个类；配置项由补丁自己读取，不碰 `Config.java`；生成文件的结构保持不变；配置非法时 FE 拒绝启动，不让过滤悄悄失效。
4. **先找免改代码的方案，并把局限讲清楚。**`-Dlogging.config` 能用，但一次运行时 reconfig 就会失效。这类边界一定要实测，不能只靠推断。
5. **人来做最后一步。**停 FE、备份、替换、回滚这些生产操作，由人按 README 执行；AI 负责准备好材料，并给出可核对的证据。

## 参考

- apache/doris `Log4jConfig.java`：[2.1.6-rc04](https://github.com/apache/doris/blob/2.1.6-rc04/fe/fe-core/src/main/java/org/apache/doris/common/Log4jConfig.java) ｜ [2.1.2-rc04](https://github.com/apache/doris/blob/2.1.2-rc04/fe/fe-core/src/main/java/org/apache/doris/common/Log4jConfig.java)
- apache/doris `HttpServer.java`（`logging.config`）：[2.1.6-rc04](https://github.com/apache/doris/blob/2.1.6-rc04/fe/fe-core/src/main/java/org/apache/doris/httpv2/HttpServer.java)
- Apache Doris 文档：[审计日志](https://doris.apache.org/zh-CN/docs/3.x/admin-manual/audit-plugin/) ｜ [FE 日志管理](https://doris.apache.org/zh-CN/docs/3.x/admin-manual/log-management/fe-log/)（2.1 文档已归档，链接为当前版本）｜ [下载](https://doris.apache.org/download)
- Log4j2：[Pattern Layout（%replace）](https://logging.apache.org/log4j/2.x/manual/pattern-layout.html) ｜ [Filters（RegexFilter）](https://logging.apache.org/log4j/2.x/manual/filters.html)
- [JVM 规范第 4 章：class 文件格式（major version）](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)
