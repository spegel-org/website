---
title: Resources
type: docs
weight: 2
---

We’re happy to share resources contributed by the community. We hope you find them useful!

{{% resources.inline %}}

{{ $resources := .Site.Data.resources }}

{{ range $year := seq (now.Format "2006") -1 2023 }}
## {{ $year }}

<ul>
{{ range $i, $v := $resources }}
{{ $itemYear := (time.AsTime $v.date).Format "2006" }}

{{ if eq $itemYear (printf "%d" $year) }}

<li>
<a target="_blank" href="{{ $v.url }}">
{{ $v.title }}
</a>
</li>

{{ end }}

{{ end }}
</ul>

{{ end }}


{{% /resources.inline %}}
