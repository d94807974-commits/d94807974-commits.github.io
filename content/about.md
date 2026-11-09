---
title: "關於"
draft: false
---
你好，我是 Donald。這是我的個人部落格，基於 Hugo 和 GitHub Pages 搭建。

## AI 友善存取方式

本站的內容也提供給程式與 AI 工具直接讀取。以下入口都由 Hugo 在建置時自動產生，內容與網頁版同步：

| 入口 | 網址 | 內容 |
|---|---|---|
| 每頁 Markdown 版 | `<文章網址>index.md`，例如 [/about/index.md](/about/index.md) | 原始 Markdown 正文，加上標題、日期、摘要、標籤、原文網址等 front matter |
| llms.txt | [/llms.txt](/llms.txt) | 依 [llmstxt.org](https://llmstxt.org/) 格式的站點簡介與文章索引，連到各篇 Markdown 版 |
| llms-full.txt | [/llms-full.txt](/llms-full.txt) | 全部文章的 Markdown 全文，單一檔案 |
| RSS | [/index.xml](/index.xml) | 全文 RSS（`content:encoded`） |
| JSON Feed | [/feed.json](/feed.json) | JSON Feed 1.1，含全文 HTML 與 Markdown 版網址 |
| Sitemap | [/sitemap.xml](/sitemap.xml) | 全站頁面清單 |
| robots.txt | [/robots.txt](/robots.txt) | 允許爬取，並指向 sitemap 與 llms.txt |

每篇文章頁面的 `<head>` 有 `<link rel="alternate" type="text/markdown">` 指向 Markdown 版，並帶有 schema.org `BlogPosting` 結構化資料；頁首的「Markdown 版」連結也指向同一個檔案。每篇正文開頭的 `> [!VIEWPOINT] 核心觀點` 區塊是該文的立場摘要。

使用本站內容時，請註明出處與原文網址。
