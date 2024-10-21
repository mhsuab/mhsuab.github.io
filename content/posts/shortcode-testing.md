+++
title = 'Shortcode Testing'
date = 2024-08-16T14:55:24-04:00
summary = 'Testing shortcode that is added to the theme'
draft = true
+++

## notice
{{< notice warning >}}
This is a warning notice. Be warned!
{{< /notice >}}

{{< noticeunk >}}
**warning** ???
{{< /noticeunk >}}

## mermaid
```mermaid
flowchart TD
  a --> b
  b --> c
  b --> d
  c --> e 
  d --> e
```

{{< mermaid >}}
graph LR;
A[Lemons]-->B[Lemonade];
B-->C[Profit];
{{< /mermaid >}}
