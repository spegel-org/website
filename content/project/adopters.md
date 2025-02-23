---
title: Adopters
type: docs
toc: false
weight: 2
---

<p>These organizations are embracing Spegel in production.</br> Thank you for being a part of our community! 🌟💖</p>
<p>To be included, add your project or company to the <a target="_blank" href="https://github.com/spegel-org/website/blob/main/data/adopters.yaml">adopters list</a>.</p>

<style>
.container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 16px;
  margin-top: 16px;
}

.item {
  border-color: #e5e7eb;
  border-width: 1px;
  background-color: #f3f4f6;
  border-radius: 8px;
  overflow: hidden;
  transition: box-shadow 0.3s ease-in-out;
}

.item:hover {
  box-shadow: 0 3px 6px rgba(0, 0, 0, 0.07);
}

.item-image {
  background-color: #ffffff;
  height: 150px;
  margin: 0;
  object-fit: contain;
  padding: 16px;
  width: 100%;
  border-radius: 0;
}
</style>

{{% adopters.inline %}}
{{ $adopters := .Site.Data.adopters }}
{{ $sections := slice "projects" "companies" }}
{{ range $sections }}
## {{ . | humanize }}

<div class="container">
{{ range $i, $adopter := index $adopters . }}
  <a href="{{ $adopter.link }}" target="_blank" style="text-decoration: none; color: inherit;">
  <div class="item">
    <image class="item-image" src="{{ $adopter.image }}"/>
    <div style="padding: 16px;"><b>{{ $adopter.name }}</b></div>
  </div>
  </a>
{{ end }}
</div>
{{ end }}
{{% /adopters.inline %}}
