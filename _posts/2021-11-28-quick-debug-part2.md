---
title: Quick debug (part 2)
author: Guy Baudouin
date: 2021-11-28 12:30:00 +0800
categories: [Tips]
tags: [front,debug,replay]
pin: true
---

The post is a follow-up to https://bga-devs.github.io/blog/posts/quick-debug-part1/ and will use the debug methods similar to what was explained on part 1.

For this post, I share my experience on game Azul. It can have a lot of tiles on board, and when you want to copy of a game based on a replay, it can be tedious to write tile positions one by one to reproduce the replay configuration.
So, the idea was to generate those debug lines based on the replay.

As it is similar to debug methods discussed on part 1, I'll just share debug method signatures I have on Azul :
```php
    private function debugSetWallTile(int $playerId, int $line, int $column, int $color) { 
        // set a tile on $line/$column coordinate
    }
    private function debugSetLineTiles(int $playerId, int $line, int $number, int $color) {
        // set $number tiles of the same color on $line
    }
```

For first bugs I filled `debugSetup()` manually with calls to these methods.
For some replays, I had to write more than 30 lines of debug, that's when I changed my mind and decided to write a little JS tool to "extract" tile positions from replay. So I write this little piece of JS :
```js
    function getDebugWall(playerId) {
        for (line = 1; line <= 5; line++) {
            for (column = 1; column <= 5; column++) {
                // every wall position is called a spot, wether it is filled by a tile or not
                const spotId = `player-table-${playerId}-wall-spot-${line}-${column}`;
                // if there is a tile, we will find a div of class "tile"
                const tile = Array.from(document.getElementById(spotId).children).find(elem => elem.classList.contains('tile'));
                if (tile) {
                    // color is the second class, and ends with a digit indicating color
                    const color = tile.classList[1].match(/(\d)/)[0];
                    // we generate a debug command
                    console.log(`$this->debugSetWallTile(${playerId}, ${line}, ${column}, ${color});\n`);
                }
            }
        }
    }
```

Then I go to a specific move on the replay, open debug console and paste method on it. Then I just call this from console :
```js
    // we call declared method just above with a player's id
    getDebugWall(84222058)
```
And console will give us the debug lines we can add to `debugSetup()` method.
```js
$this->debugSetWallTile(84222058, 1, 5, 2);
$this->debugSetWallTile(84222058, 2, 2, 3);
$this->debugSetWallTile(84222058, 2, 3, 4);
$this->debugSetWallTile(84222058, 3, 2, 2);
$this->debugSetWallTile(84222058, 3, 5, 5);
$this->debugSetWallTile(84222058, 4, 2, 1);
```

And we have our debug line, without doing boring repetitive and error-prone task of rewriting the game positions manually !
