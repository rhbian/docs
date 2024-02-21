---
title: "acme"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---


# 更新证书
使用root权限。
``` sh
# acme.sh --renew  -d dev.skasis.com --key-file   /etc/trojan/private.key --fullchain-file /etc/trojan/fullchain.cer
```

```
# rc-service nginx reload
```
