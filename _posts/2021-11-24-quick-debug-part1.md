---
title: Quick debug (part I)
author: Guy Baudouin
date: 2021-11-24 17:30:00 +0800
categories: [Tips]
tags: [back,debug]
pin: true
---

For some games, you need to be able to test a particular case quickly. 
I will detail here how I do it for the game King of Tokyo, who got 66 different cards, and of course a lot more of possible combinations. 
Playing the game over and over, hoping to get the right combination to test, is impossible.
So I needed a way to reproduce particular cases, by forcing some cards where I want them to be.

First, I created a method I call "debugSetup" :
```php
    function debugSetup() {
      // we will setup the particular case here
    }
```

this is called at the end of the setupNewGame method. I created this method in a DebugTrait, where I will put other debug related methods.

Before putting debug code here, let's protect it to be sure we don't put debug code in production :
```php
    function debugSetup() {
        if ($this->getBgaEnvironment() != 'studio') { 
            return;
        } 

      // we will setup the particular case here
    }
```
This will prevent our debug code to be executed in production, in case we forgot to comment "debugSetup" call !

Then we will create methods to add cards in table or in player hand (I'm using Stock for cards)

```php
    private function debugSetCardInTable($cardType) {
        $this->cards->moveCard( $this->getCardFromDb(array_values($this->cards->getCardsOfType($cardType))[0])->id, 'table');
    }

    private function debugSetCardInHand($cardType, $playerId) {
        $card = $this->getCardFromDb(array_values($this->cards->getCardsOfType($cardType))[0]);
        $this->cards->moveCard($card->id, 'hand', $playerId);
        return $card;
    }
```
Then, to add some cards on player's hand or on table, I just do :
```php
    function debugSetup() {
        if ($this->getBgaEnvironment() != 'studio') { 
            return;
        } 

        $this->debugSetCardInTable(JET_FIGHTERS_CARD);
        $this->debugSetCardInHand(RAPID_HEALING_CARD, 2343492);
        $this->debugSetCardInHand(ACID_ATTACK_CARD, 2343493);
    }
```
"2343492" being my player0 id, "2343493" for player1.

Starting the game with this, will result, in studio, with cards already in hand/table ! Easier to debug them.

But we can go a little further. For example, Rapid Healing card can only be used if player got some energy. This is stored as an int column in player table. Let's add some new debug methods !
```php
    private function debugSetPlayerEnergy($playerId, $energy) {
        self::DbQuery("UPDATE player SET `player_energy` = $energy where `player_id` = $playerId");
    }

    private function debugSetEnergy($energy) {
        self::DbQuery("UPDATE player SET `player_energy` = $energy");
    }
```

I can now use `$this->debugSetEnergy(5);` to set to all players, or `$this->debugSetPlayerEnergy(2343492, 5);` to set to only one player.

I made the same for health and points. Ready to test Rapid Healing !

Yet... even if I start a 2 player game, sometimes player0 will start, sometimes player1 ! I don't want to lose the time of a player's turn before testing, so I add `$this->gamestate->changeActivePlayer(2343493);` at the end of "debugSetup" to force the first player.

Another interesting tip is to force the next card to the deck. You can do it by changing order in deck with a high value (more than the card number will do the trick)
```php
self::DbQuery("UPDATE card SET `card_location_arg` = card_location_arg + 1000 where `card_type` = ".ZOMBIE_CARD);
``` 

That's it for part 1 ! I hope it will give you some new tips to debug quickly your game !
