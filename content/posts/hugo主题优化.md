---
date : 2025-07-02T14:48:21+08:00
lastmod : 2026-01-09 16:24:40
title : 'Hugo主题优化'
description : "修改部分PaperMod主题样式" # 文章描述，与搜索优化相关
author : ["想上天的鱼"]
tags: ["hugo","主题","PaperMod"]
ShowToc: true
draft : false
---

# 1. 时间轴中文化


**2026-01-07 14:19:44**

在配置文件中添加此配置`defaultContentLanguage: zh`，即可解决，无需更改主题文件。

---
使用基本配置时存在一个问题，时间轴月份仍然为英文，可以修改`layouts/_default/archives.html` 中代码实现月份显示中文。[参考链接](https://cloud.tencent.com/developer/article/1956256)

代码如下:
    
```yaml
{{- if eq .Key "December" }}
{{ "12月" }}
{{- end }}
{{- if eq .Key "November" }}
{{ "11月" }}
{{- end }}
{{- if eq .Key "October" }}
{{ "10月" }}
{{- end }}
{{- if eq .Key "September" }}
{{ "9月" }}
{{- end }}
{{- if eq .Key "August" }}
{{ "8月" }}
{{- end }}
{{- if eq .Key "July" }}
{{ "7月" }}
{{- end }}
{{- if eq .Key "June" }}
{{ "6月" }}
{{- end }}
{{- if eq .Key "May" }}
{{ "5月" }}
{{- end }}
{{- if eq .Key "April" }}
{{ "4月" }}
{{- end }}
{{- if eq .Key "March" }}
{{ "3月" }}
{{- end }}
{{- if eq .Key "February" }}
{{ "2月" }}
{{- end }}
{{- if eq .Key "January" }}
{{ "1月" }}
{{- end }}
<!-- {{- .Key }} -->
```
    

# 2. 文章页面显示汉字

需要在`themes/PaperMod/layouts/_default/list.html`文件中判断`.Title`是`Posts`时显示为`文章`两字即可。因为开启了RSS订阅显示，需要在此文件中注释到RSS标志显示。

```html
<h1>
    <!-- {{ .Title }} -->
    {{- if eq .Title "Posts" }}
        <h1>📚{{ "文章" }}</h1>
    {{- end }}
    <!-- {{- if and (or (eq .Kind `term`) (eq .Kind `section`)) (.Param "ShowRssButtonInSectionTermList") }}
    {{- with .OutputFormats.Get "rss" }}
    <a href="{{ .RelPermalink }}" title="RSS" aria-label="RSS">
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
        stroke-linecap="round" stroke-linejoin="round" height="23">
        <path d="M4 11a9 9 0 0 1 9 9" />
        <path d="M4 4a16 16 0 0 1 16 16" />
        <circle cx="5" cy="19" r="1" />
        </svg>
    </a>
    {{- end }}
    {{- end }} -->
</h1>
```
    

# 3. 标签页面显示汉字

需要在`themes/PaperMod/layouts/_default/terms.html`文件中判断`.Title`是`Tags`时显示为`标签`两字即可。
    
```html
<header class="page-header">
    <!-- <h1>{{ .Title }}</h1> -->
    {{- if eq .Title "Tags" }}
        <h1>🔖{{ "标签" }}</h1>
    {{- end }}
    {{- if .Description }}
    <div class="post-description">
        {{ .Description }}
    </div>
    {{- end }}
</header>
{{- end }}
```
    

# 4. 文章增加修改日期

原主题并没有显示「修改时间」的功能，在 `layouts/partials/post_meta.html` 中加入以下内容即可：

```go
{{ if ne (.Lastmod.Format "2006-01-02") (.Date.Format "2006-01-02") }}
{{- $scratch.Add "meta" (slice (printf "Updated:&nbsp;%s" (.Lastmod.Format (.Site.Params.dateFormat | default "January 2, 2006")))) }}
{{- end -}}
```

然后在具体的帖子里加入 `lastmod` 即可显示修改日期。

# 5. 修改行内代码样式

在站点根目录创建文件：`assets/css/extended/custom.css`,然后写入下面的样式：
```css
.post-content :not(pre) > code {
  color: #d32f2f !important;        /* 红色文字 */
  background-color: #ffebee !important; /* 淡红背景 */
  padding: 0.2em 0.4em !important;
  border-radius: 4px !important;
  font-size: 0.95em !important;
  font-family: "Fira Code", "JetBrains Mono", Consolas, "Courier New", monospace !important;
}
```
`:not(pre) > code` 是关键：只作用于行内 code，不影响代码块。