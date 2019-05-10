title: adqwdq
author: betteryaogang
date: 2019-05-10 14:18:02
tags:
---
```
    public void init() {
        try {
            qqwry = new QQWry(Paths.get(qqWryPath));
            System.out.println("========================qqwry load success============================" + qqWryPath);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

