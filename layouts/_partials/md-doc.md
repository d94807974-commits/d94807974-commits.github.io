{{- $base := strings.TrimSuffix "/" site.BaseURL -}}
{{- $body := .RawContent -}}
{{- $body = replaceRE `\]\(/` (printf "](%s/" $base) $body -}}
{{- $body = replaceRE `(src|href)="/` (printf `${1}="%s/` $base) $body -}}
---
title: {{ .Title | jsonify }}
url: {{ .Permalink }}
markdown_url: {{ with .OutputFormats.Get "markdown" }}{{ .Permalink }}{{ end }}
{{- if not .Date.IsZero }}
date: {{ .Date.Format "2006-01-02T15:04:05-07:00" }}
lastmod: {{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}
{{- end }}
author: {{ site.Params.author | jsonify }}
language: zh-TW
{{- with .Params.description }}
description: {{ . | jsonify }}
{{- end }}
summary: {{ (.Params.summary | default .Summary | plainify | htmlUnescape) | jsonify }}
{{- with .Params.tags }}
tags: {{ . | jsonify }}
{{- end }}
{{- with .Params.categories }}
categories: {{ . | jsonify }}
{{- end }}
---

# {{ .Title }}

{{ $body }}
