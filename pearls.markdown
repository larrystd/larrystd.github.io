---
layout: article
title: Programming Pearls
---

Programming Pearls. 编程如若创造珍珠，数据结构为模板，算法则为细凿。介绍些paper、alogrithm、tech books等。

{% for pearls in site.data.pearls %}
### [{{pearls.title}}]({{pearls.url}})
🔗: [{{ pearls.tag }}](/tags.html#{{ pearls.tag }})  
📝: {{ pearls.note }}
<hr style="width:100%"/>
{% endfor %}
