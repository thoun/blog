---
title: Classes, notifications and bugs!
author: Guillaume Benny
date: 2021-11-25 8:00:00 -0500
categories: [Tips]
tags: [back,class,notification]
pin: true
---

When creating a game, you will probably want to create classes so that your
code doesn't become a mess. Something like this:

```php
class Card
{
    public $cardId;
    public $locationId;
    public $playerId;
    // ... And lots of other members...
}
```

If you need to move a card in the user interface, you can send the card
directly in a notification:

```php
$card = // A function that returns an instance of the Card class
$card->moveToSomewhere(); // Changes $locationId=1 and saves the change in the database

$this->notifyAllPlayers(
    'move_card',
    clienttranslate('Card is moving somewhere!'),
    [
        'card' => $card,
    ]
);

```

This works because the card instance is transformed in an associative array: the
keys are the public members and the values are the values of those members. So
in javascript it will look like this:

```javascript
notif_MoveCard(notif) {
    // Valid members:
    // notif.arg.card.cardId
    // notif.arg.card.locationId
    // notif.arg.card.playerId
}
```

This works well but there is a bug lurking in there. While implementing the
Solo mode for [The Isle of Cats](https://boardgamearena.com/gamepanel?game=theisleofcats),
I suddendly had to do more than one thing with the same card in the same action. This
happens since the 'opponent' is automated. So the code looked something like this:

```php
$card = // A function that returns an instance of the Card class
$card->moveToSomewhere(); // Changes $locationId=1 and saves the change in the database

$this->notifyAllPlayers(
    'move_card',
    clienttranslate('Card is moving somewhere!'),
    [
        'card' => $card,
    ]
);

$card->moveToSomewhereElse(); // Changes $locationId=2 and saves the change in the database

$this->notifyAllPlayers(
    'move_card',
    clienttranslate('Card is moving somewhere else!'),
    [
        'card' => $card,
    ]
);

```

You will receive two notifications in javascript, but both will have `locationId == 2`.
This happens because the same instance of the class is stored with each notification
but it's not transformed in an associative array until the end of the processing on
the server, which may span multiple state transitions.
This bug can be very hard to spot when the notifications are far apart. To
prevent this, you can force the class to become an array so that you get an
immediate copy of the data:

```php
$this->notifyAllPlayers(
    'move_card',
    clienttranslate('Card is moving somewhere!'),
    [
        'card' => (array)$card, // Here the (array) cast is added
    ]
);

```

Since this is easy to forget and does not work if your members are instances of
other classes (this is not a deep copy), I've created more general functions:

```php

private static function toNotifArray($array_or_value)
{
    if (is_array($array_or_value)) {
        return array_map(function ($v) {
            return toNotifArray($v);
        }, $array_or_value);
    }
    if (is_object($array_or_value)) {
        return toNotifArray((array)$array_or_value);
    }
    return $array_or_value;
}

public function gameNotifyAllPlayers($notifType, $notifLog, $notifArgs)
{
    $this->notifyAllPlayers($notifType, $notifLog, self::toNotifArray($notifArgs));
}

public function gameNotifyPlayer($playerId, $notifType, $notifLog, $notifArgs)
{
    $this->notifyPlayer($playerId, $notifType, $notifLog, self::toNotifArray($notifArgs));
}
```

So this works with no surprises:


```php
$this->gameNotifyAllPlayers(
    'move_card',
    clienttranslate('Card is moving somewhere!'),
    [
        'card' => $card,
    ]
);
```
