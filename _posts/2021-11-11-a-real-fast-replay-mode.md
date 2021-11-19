---
title: How to make a real fast replay mode
author: Timothée Pecatte
date: 2021-11-18 23:33:00 +0800
categories: [Tips]
tags: [replay,front]
pin: true
---

This is the story of how I went from this (~1min for 80 replayed moves) :
![Slow replay](/blog/assets/img/replay/replay_slow.gif){: width="575" height="184"}

to this (~8sec) :

![Replay on steroids](/blog/assets/img/replay/replay_fastest.gif){: width="575" height="184"}

Before diving into code, maybe you are asking yourself:  
![But why?](/blog/assets/img/replay/but_why.jpg){: width="200" height="136"}

Well, when you have a bug report at move 450 that needs more than 8min to reach, you start wondering why is fast mode not that fast :)

# Fast replay mode
Some players don't know that feature, but once you enter a replay, you have access to a fast replay mode under the "advanced settings" link:  
*(let's not get distracted by the fact that the link is translated while the menu is not...)*

![Enable advanced settings](/blog/assets/img/replay/enable_advanced.png){: .left width="241" height="64"}
![Advanced settings](/blog/assets/img/replay/advanced_features.png){: width="168" height="220"}


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
 - setting `instantaneousMode = true`, which is used in `slideTo` to reduce the duration to 1, and in various other framework transition function (fadeOut, addToStock, ...)
 - displaying the "loader_mask" element on top of everything else
 - reducing synchronous duration of notifications to 1

This sounds like a great idea, so why is it still so slow? Let's use the performance tool to see what is happening:
![Performance analysis](/blog/assets/img/replay/replay_perf.png){: width="960" height="736"}
*(you can click on a row to have more details about which part of the code is corresponding to that row)*

Here are the three main bottlenecks that might arises in any games:
 - some notifications seems to take as much time as in normal slow mode
 - there are bunch of setTimeout floating around
 - there are a lot of style recalculation

For experimenting while trying to solve these issues, it was convenient to have an helper function that I could switch on/off easily:
```js
isFastMode(){
  return this.instantaneousMode; // true / false
}
```
That way, returning true would allow to use the normal step-by-step replay while mimicking the fast mode.


# Improving notifications
The framework is making all notifications synch timing to 1sec, which sounds good but has two flaws:
 - if you are using non-framework animations, this can break you code as notifications will run one after the other without checking that the animation is over
 - if you are using dynamic synch timing (using `this.notifqueue.setSynchronousDuration`), then your notification will take the same time as in a normal slow replay

 So the first step was to make all the notifications fast-replay-compatible. For instance, we have this notification in Agricola when a player play a card that will zoom on the card, them zoom off to its location before resolving the notification, which is useless in fast mode. So I just add a bypass of the whole flow by checking the value of `isFastMode`:
 ```js
 notif_buyCard(n) {
   debug('Notif: buying a card', n);
   let card = n.args.card;
   let duration = 700;
   let waitingTime = 80000 / this._cardAnimationSpeed;

   // Create the card if needed, and compute initial location of sliding event
   let exists = $(card.id);
   let from = this.computeSlidingAnimationFrom(card, 'cards-wrapper-' + card.pId);

   if (this.isFastMode()) {
     this.notifqueue.setSynchronousDuration(0);
   } else {
     // Zoom on it, then zoom off
     this.zoomOnCard(card.id, { from, duration })
       .then(() => this.wait(waitingTime))
       .then(() => this.zoomOffCard({ duration }))
       .then(() => this.notifqueue.setSynchronousDuration(10));
   }
   ...
}
 ```

I don't like that much the fact that notification timing are hardcoded to 1 in this mode, so I made it more flexible by allowing the notifications to enforce their synch time if they need to:
```js
this._notifications = [
  ['revealActionCard', 1100],
  ['placeFarmer', null],
  ['growChildren', 1000],
  ...
];

setupNotifications() {
  console.log(this._notifications);
  this._notifications.forEach((notif) => {
    var functionName = 'notif_' + notif[0];

    let wrapper = (args) => {
      let timing = this[functionName](args);
      if (timing === undefined) {
        if (notif[1] === undefined) {
          console.error(
            "A notification don't have default timing and didn't send a timing as return value : " + notif[0],
          );
          return;
        }

        // Override default timing by 0 in case of fast replay mode
        timing = this.isFastMode() ? 0 : notif[1];
      }

      if (timing !== null) {
        this.notifqueue.setSynchronousDuration(timing);
      }
    };
  });
}
```

# Making the flow as asynch as possible
Now let's take care of these bunch of setTimeouts, which are coming from two main part in my case:
 - sliding animations which have in a setTimeout with duration 1 for the `onEnd` listener -_-'
 - counters????

## Improve slide function
All my slidings are usually the `attach: true` parameter of my `slide` functions which means: once the animation is over, attach the mobile element to the target and remove absolute positionning. What a waste of resources to do all that in fast-mode where it's basically the same as just moving directly the node around:
```js
slide(mobileElt, targetElt, options = {}) {
  ...

  // Mobile elt
  mobileElt = $(mobileElt);
  let mobile = mobileElt;
  // Target elt
  targetElt = $(targetElt);
  let targetId = targetElt;
  let newParent = config.attach ? targetId : $(mobile).parentNode;

  // Handle fast mode
  if (this.isFastMode() && (config.destroy || config.clearPos)) {
    if (config.destroy) dojo.destroy(mobile);
    else dojo.place(mobile, targetElt);

    return new Promise((resolve, reject) => {
      resolve();
    });
  }
  ...
}
```

## Counters
In Agricola, we are using a loooot of counters for scores, resources, ...  
![Resource counters](/blog/assets/img/replay/counters.png){: width="600" height="502"}

Inside notifications, we are using the `toValue` to have this nice animation while playing the game. But these animations are useless in fast mode!
But shouldn't the framework be handling this as for reducing the duration of slideTo function? Well it probably should, but it's not doing it....

So let's make our own by copying the existing one (and improve it to handle floating value as well).
Instead of creating a dojo component, I just went a dirty way by creating an object on the fly with corresponding methods, please don't puke while reading this:
```js
/**
 * Own counter implementation that works with fast mode replay
 */
createCounter(id, defaultValue = 0) {
  if (!$(id)) {
    console.error('Counter : element does not exist', id);
    return null;
  }

  let game = this;
  let o = {
    span: $(id),
    targetValue: 0,
    currentValue: 0,
    speed: 100,
    getValue: function () {
      return this.targetValue;
    },
    setValue: function (n) {
      this.currentValue = +n;
      this.targetValue = +n;
      this.span.innerHTML = +n;
    },
    toValue: function (n) {
      if (game.isFastMode()) {
        this.setValue(n);
        return;
      }

      this.targetValue = +n;
      if (this.currentValue != n) {
        this.span.classList.add('counter_in_progress');
        setTimeout(() => this.makeCounterProgress(), this.speed);
      }
    },
    incValue: function (n) {
      let m = +n;
      this.toValue(this.targetValue + m);
    },
    makeCounterProgress: function () {
      if (this.currentValue == this.targetValue) {
        setTimeout(() => this.span.classList.remove('counter_in_progress'), this.speed);
        return;
      }

      let step = Math.ceil(Math.abs(this.targetValue - this.currentValue) / 5);
      this.currentValue += (this.currentValue < this.targetValue ? 1 : -1) * step;
      this.span.innerHTML = this.currentValue;
      setTimeout(() => this.makeCounterProgress(), this.speed);
    },
  };
  o.setValue(defaultValue);
  return o;
},
```

This is the new speed after doing all these changes, pretty neat already:
![Nicer replay](/blog/assets/img/replay/replay_faster_counter.gif){: width="575" height="184"}

# Prevent the repaints
When looking to the perf tool, the main bottleneck seems to be style calculation, which is normal because we are moving a lot of stuff around.
> Wait, I'm not actually seeing everything moving around since it's hidden beside the loading mask, so the browser should do that silently, right ?

The answer is: that's depend what `hidden` means. If we look back at `setModeInstataneous`, we can see that the function is doing this change of style of the main container:
```js
dojo.style('leftright_page_wrapper', 'visibility', 'hidden');
```
Hidding an element with this css property will keep it's width/height so that means the browser has to recompute the styles to properly updates the width/height of this element, even if you can't see it!
The solution is pretty easy then, just use `display:none` instead! Actually, we will need a bit more work because the loading screen won't have the proper height now that the main container is not longer displayed, which can be easily fixed by doing a cleaner loading screen solution:
```js
constructor() {
  ...
  dojo.place('loader_mask', 'overall-content', 'before');
  dojo.style('loader_mask', {
    height: '100vh',
    position: 'fixed',
  });
},

// Overwrite this to make display:none
setModeInstataneous() {
  if (this.instantaneousMode == false) {
    this.instantaneousMode = true;
    dojo.style('leftright_page_wrapper', 'display', 'none');
    dojo.style('loader_mask', 'display', 'block');
    dojo.style('loader_mask', 'opacity', 1);
  }
},

// Overwrite this to make display:block after fast mode is off
unsetModeInstantaneous() {
  if (this.instantaneousMode) {
    this.instantaneousMode = false;
    dojo.style('leftright_page_wrapper', 'display', 'block');
    dojo.style('loader_mask', 'display', 'none');
  }
},
```

And now we finally get this much nicer speed:
![Even nicer replay](/blog/assets/img/replay/replay_faster_style_computation.gif){: width="575" height="184"}
