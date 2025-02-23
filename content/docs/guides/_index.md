---
title: Guides
weight: 2
---

{{% guides.inline %}}

{{ range where .Site.RegularPages ".Parent.RelPermalink" "/docs/guides/" }}
<h5><a href="{{.Permalink}}">{{ .Title }}</a></h5>
<p style="margin: 0;">{{ .Description }}</p>
{{ end }}

{{% /guides.inline %}}
