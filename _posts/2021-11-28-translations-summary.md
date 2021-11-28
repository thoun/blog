---
title: Translations - a recap
author: Timothée Pecatte
date: 2021-11-28 18:00:00 +0800
categories: [Guide]
tags: [back,front,translation]
pin: true
---

As I usually say on the discord server for BGA developpers, translations are easy and hard.
Hard because translations seems to be one of the most recurring issue developpers are facing on BGA, even for experienced developpers.
And easy because we will see that the basic mechanisms can be summarized in a couple principles.

**This small guide is not here to replace documentation, please make sure to also check this page:**
[Studio doc for Translations](https://en.doc.boardgamearena.com/Translations)



Main principles
---------------
Here are the two main principles:
  - `clienttranslate` on the backend is doing **nothing**, expect marking the string to add it to the translation system
  - `_(...)` will translate the string given **only if this exact string was marked at some point**

These principles are already enough to understand why this example is not working:
```php
$a = 2;
$msg = clienttranslate("The dice value is ${a}")
```
The parsing script will mark the string `The dice value is ${a}` as translatable, but will be prompted to translate `The dice value is 2` which is a different string, so it will fail.
So how to deal with dynamic strings like this one with the framework then? Time to talk about the famous `format_string_recursive` that frighten new devs!


format_string_recursive: a first look
------------------------------------
Let's actually have a look at some part of the code of this function, augmented with comments:
```js
format_string_recursive: function (log, args) {
  var text = '';
  if (log != '') {
    var log = this.clienttranslate_string(log); // TRANSLATION HAPPENS HERE !!
    if (log === null) {
      // THE RED BANNER YOU GET IN CASE OF MISSING TRANSLATION
      this.showMessage('Missing translation for `' + log + '`', 'error');
      console.error('Missing translation for `' + log + '`', 'error');
      return '';
    }

    [1] // that's for later
    [2] // and for even later

    try {
      // THIS IS WERE SUBSTITUTION TAKES PLACE
      text = dojo.string.substitute(log, args);
    } catch (e) {
      ... // HANDLE ERRORS
      console.error('Invalid or missing substitution argument for log message: ' + _884, 'error');
      text = log;
    }
  }
  return text;
}
```
This function takes two arguments: the message and the variable that needs to be substituted. It translates the log using `clienttranslate_string`, which is basically calling `_()` on it, and then replace variable occurences with their value. So instead of receiving only a message, the framework always deal with pairs `(message, arguments)` to handle these more complex cases:
 - for notifications, you give first the message, then the arguments, so you would do:
    ```php
    self::notifyAllPlayers('rollDice', clienttranslate('The dice value is ${a}'), ['a' => 2])
    ```

 - for state descriptions, the framework fetch the message from `states.inc.php`, so you would put `'descriptionmyturn' => clienttranslate("You must put the number ${a} on your grid")`. And then the arguments are just the general args of the states so you would need to add `'a' => 2` in your state args for instance.

 - you can also use this for your own UI elements in front, such as buttons:
  ```js
  this.addActionButton('mybutton', this.format_string_recursive(_('Take ${x} wood'), {x : 4}), null, false, 'blue')
  ```


Translatable arguments
----------------------

This is fine until you want for instance a state description that looks like `"You must build on ${type}"`.
Sure, you can put `'descriptionmyturn' => clienttranslate('You must build on ${type}')` and then adds to your state args `'type' => $type`, but how to get the framework to also translate this string (assuming you defined `$type` with `clienttranslate` before)?
Well the `format_string_recursive` can already handle that since the `[1]` omission in the function code looks like this:
```js
// CHECK TRANSLATABLE ARGUMENTS
if (typeof args.i18n != 'undefined') {
  for (i in args.i18n) {
    let key = args.i18n[i];
    args[key] = this.clienttranslate_string(args[key]);
  }
}
```
So, as explained in the doc, you only need to add an `i18n` entry specifying which arguments should also be translated before being substituted in the message. Notice that this is true for any use case of `format_string_recursive`, so it works with notifications, but also with state args and any user calls in frontend, such as:
```js
this.addActionButton('mybutton', this.format_string_recursive(
  _('Take ${x} ${res}'),
  {
    i18n: ['res']
    x : 4,
    res: _('wood'),
  }),
...);
```



Recursivity kicks in
---------------------
Now let's say I want to notify the fact that some players earned a bunch of resources : `3 woods, 2 stones, 1 food`.
My notification message will be something like `clienttranslate('${player_name} gains ${res}')` but since translation is made only in the client, what value should I give to `$res` in my args to make the framework properly handle that?
Let's look at the `[2]` omission in the function:
```js
// RECURSIVE FORMATING
for (key in args) {
  if ((key != 'i18n') && ((typeof args[key]) == 'object')) {
    if (args[key] !== null) {
      if ((typeof args[key].log != 'undefined') && (typeof args[key].args != 'undefined')) {
        args[key] = this.format_string_recursive(args[key].log, args[key].args);
      }
    }
  }
}
```
That's the part that justify the function name, and makes it so powerful. So far we were only dealing with strings/number as arguments that were substituted in the message, but if you actually gives an object, the function will apply itself on it, looking at the `log` and `args` entries of that object.
So our initial issue could be solved by sending this inside the args of the notification:
```php
[
  'res' => [
    'log' => clienttranslate('${nRes1} ${res1}, ${nRes1} ${res1}, ${nRes3} ${res3}'),
    'args' => [
      'i18n' => ['res1', 'res2', 'res3'], // CRUCIAL to have the resource types correctly translated
      'nRes1' => 3,
      'res1' => clienttranslate('woods'), // probably defined elsewhere in a real game, such as material.inc.php
      'nRes2' => 2,
      'res2' => clienttranslate('stones'),
      'nRes3' => 1,
      'res3' => clienttranslate('food')
    ]
  ]
]
```

This works, but it's not very dynamic: what if I have 4 resources ? 5 ? Only 2?
We can notice that the `clienttranslate` is not useful here and we can then generate the log dynamically instead
```php
$resources = [WOOD => 3, STONE => 2, FOOD => 1];
$logs = [];
$args = [];
$i = 0;
foreach($resources as $type => $amount){
  $logs[] = '${nRes'. $i. '} ${res'. $i .'}';
  $args['nRes' . $i] = $amount;
  $args['res' . $i] = RESOURCE_NAME[$type]; // constants holding clienttranslated names of resources
  $args['i18n'][] = 'res' . $i;
  $i++;
}

self::notifyAllPlayers('gainResources', clienttranslate('${player_name} gains ${res}'), [
  'res' => [
    'log' => implode(',', $logs),
    'args' => $args
  ]
]);
```


More recursivity
-----------------

The good thing about recursivity is that you can go as deep as you want, it will just works the same.
So you could do some state description like that for instance:
```php
[
  'log' => clienttranslate('${action1} then ${action2}'),
  'args' => [
    'action1' => [
      'log' => clienttranslate('take ${x} ${type}'),
      'args' => [
        'x' => 2,
        'i18n' => ['type'],
        'type' => clienttranslate('wood')
      ],
    ],

    'action2' => [
      'log' => clienttranslate('move ${army} ${x} squares'),
      'args' => [
        'x' => 3,
        'army' => [
          'log' => clienttranslate('${n1} ${unit1}, ${n2} ${unit2}'),
          'args' => [
            'i18n' => ['unit1', 'unit2'],
            'n1' => 3,
            'uni1' => clienttranslate('soldiers'),
            'n2' => 2,
            'unit2' => clienttranslate('archers')
          ]
        ]
      ]
    ]
  ]
]
```

And that's why I'm saying that translation is easy: you can handle any crazy complex cases with always the same pattern, by always keeping in mind these two simple principles :).
