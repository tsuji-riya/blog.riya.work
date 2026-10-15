---
title: blog.riya.work
---

組み込み機器の解析や、U-Boot / Linux の移植についての作業メモを書いています。

## 記事一覧

```base
filters:
  and:
    - file.hasProperty("date")
formulas:
  記事: file.asLink(title)
views:
  - type: table
    name: 記事
    order:
      - formula.記事
      - date
    sort:
      - property: date
        direction: DESC
```
