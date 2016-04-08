---
layout: post
title:  "Phone / Tablet recognition"
date:   2016-04-08 08:29:36 +0100
categories: javascript ionic
---

### Just use the code below

```
this.isTablet = function () {
  if (navigator.userAgent.indexOf('Android') > -1) {
    if (navigator.userAgent.indexOf('mobile') === -1) {
      this.isTablet = true;
    }
    else {
      this.isTablet = false;
    }
  }

  if (window.innerHeight < window.innerWidth) {
    this.isTablet = window.innerHeight >= 768 ? true : false;
  } else {
    this.isTablet = window.innerWidth >= 768 ? true : false;
  }
}
```
