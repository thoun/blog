---
title: How to make a real fast replay mode
author: Timothée Pecatte
date: 2021-11-18 23:33:00 +0800
categories: [Tips]
tags: [replay,front]
---

This is the story of how I went from this (~1min for 80 replayed moves) :
![Slow replay](/blog/assets/img/replay/replay_slow.gif)

to this (~8sec) :

![Slow replay](/blog/assets/img/replay/replay_fastest.gif)


# Fast replay mode
Some players don't know that feature, but once you enter a replay, you have access to a fast replay mode under the "advanced settings" link:  
*(let's not get distracted by the fact that the link is translated while the menu is not...)*

![Enable advanced settings](/blog/assets/img/replay/enable_advanced.png){: .left}
![Advanced settings](/blog/assets/img/replay/advanced_features.png)


But what is this fast mode doing exactly? Diving into BGA framework's code, we can find this:
```js
setModeInstataneous: function () {
  if (this.instantaneousMode == false) {
    this.instantaneousMode = true;
    this.savedSynchronousNotif = dojo.clone(this.notifqueue.synchronous_notifs);
    dojo.style('leftright_page_wrapper', 'visibility', 'hidden');
    dojo.style('loader_mask', 'display', 'block');
    dojo.style('loader_mask', 'opacity', 1);
    for (var i in this.notifqueue.synchronous_notifs) {
      if (this.notifqueue.synchronous_notifs[i] != - 1) {
        this.notifqueue.synchronous_notifs[i] = 1;
      }
    }
  }
},
```
Clicking on this button has the following consequences:
 - setting `instantaneousMode = true`, which is used in `slideTo` to reduce the duration to 1
 - displaying the "loader_mask" element on top of everything else
 - reducing synchronous duration of notifications to 1
 
