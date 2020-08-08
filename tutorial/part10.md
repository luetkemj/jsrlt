# Part 10 - Saving and loading

Saving and loading is an essential feature in almost every game, but if you're not careful it can easily become extremely difficult to manage. Fortunately, geotic makes saving all of our entities super easy. The only real challenge we'll face here is saving and loading our cache but even that really isn't too big of a deal. Let's get to it!

## Saving

Geotic really does make saving super simple - so long as all your game state is within geotic you can do something as simple as this:

```javascript
const saveGame = () => {
  const data = ecs.serialize();
  localStorage.setItem("savegame", data);
};

const loadGame = () => {
  const data = localStorage.getItem("savegame");
  ecs.deserialize(data);
};
```

We have a few more things to keep track of so we won't be able to get away that easy. In addtion to saving our entities in geotic we need to save our cache, the message log, and the id of our player entity. Not bad really.

The first thing we'll do is move our message log from `./src/state/ecs.js` to `./src/index.js`. This is just a refactor to consolidate our code a bit. It's not strictly required for saving. But it will make things a bit easier for us moving forward.

In `./src/state/ecs.js` go ahead and delete the message log.

```diff
-export const messageLog = ["", "Welcome to Gobs 'O Goblins!", ""];
-export const addLog = (text) => {
-  messageLog.unshift(text);
-};
```

And then just put that code in `./src/index.js` and remove the import at the top.

```diff
import { targeting } from "./systems/targeting";
-import ecs, { addLog } from "./state/ecs";
+import ecs from "./state/ecs";
import { IsInFov, Move, Position, Ai } from "./state/components";

+export const messageLog = ["", "Welcome to Gobs 'O Goblins!", ""];
+export const addLog = (text) => {
+  messageLog.unshift(text);
+};
```

Now we just need to fix the imports in a few files that still expect our messageLog to be in the old location.

`./src/systems/targeting.js`

```diff
-import ecs, { addLog } from "../state/ecs";
+import ecs from "../state/ecs";
+import { addLog } from "../index";
import { readCacheSet } from "../state/cache";
```

`./src/systems/render.js`

```diff
-import { messageLog } from "../state/ecs";
-import { gameState, selectedInventoryIndex } from "../index";
+import { gameState, messageLog, selectedInventoryIndex } from "../index";
```

`./src/systems/movement.js`

```diff
-import ecs, { addLog } from "../state/ecs";
+import ecs from "../state/ecs";
+import { addLog } from "../index";
import { addCacheSet, deleteCacheSet, readCacheSet } from "../state/cache";
```

With that out of the way we really only have to worry about our cache before we can save. Our cache data structure uses Set objects. This makes it super fast but unfortunately you can't just stringify it. We'll have to write a couple small functions to manually serialize and deserialize our cache. If you add more keys to your cache in the future it will be up to you handle those here as well.

In `./src/state/cache.js` add the following functions:

```javascript
export const serializeCache = () => {
  const entitiesAtLocation = Object.keys(cache.entitiesAtLocation).reduce(
    (acc, val) => {
      acc[val] = [...cache.entitiesAtLocation[val]];
      return acc;
    },
    {}
  );

  return {
    entitiesAtLocation,
  };
};

export const deserializeCache = (data) => {
  cache.entitiesAtLocation = Object.keys(data.entitiesAtLocation).reduce(
    (acc, val) => {
      acc[val] = new Set(data.entitiesAtLocation[val]);
      return acc;
    },
    {}
  );
};

export const clearCache = () => {
  cache.entitiesAtLocation = {};
};
```

We can import `serializeCache` add a `saveGame` function in `./src/index.js`.

```diff
-import { readCacheSet } from "./state/cache";
+import { readCacheSet, serializeCache } from "./state/cache";
```

```javascript
const saveGame = () => {
  const gameSaveData = {
    ecs: ecs.serialize(),
    cache: serializeCache(),
    playerId: player.id,
    messageLog,
  };
  localStorage.setItem("gameSaveData", JSON.stringify(gameSaveData));
  addLog("Game saved");
};
```

Finally we just need add a keybinding:

```diff
const processUserInput = () => {
+  if (userInput === "s") {
+    saveGame();
+  }

  if (gameState === "GAME") {
```

Saving your game isn't much good if you can't load it so let's handle that next.

## Loading

We'll need to import our deserializeCache function this time:

```diff
-import { readCacheSet, serializeCache } from "./state/cache";
+import { deserializeCache, readCacheSet, serializeCache } from "./state/cache";
```

Our loadGame function is a little bit bigger:

```javascript
const loadGame = () => {
  const data = JSON.parse(localStorage.getItem("gameSaveData"));
  if (!data) {
    addLog("Failed to load - no saved games found");
    return;
  }

  for (let entity of ecs.entities.all) {
    entity.destroy();
  }

  ecs.deserialize(data.ecs);
  deserializeCache(data.cache);

  player = ecs.getEntity(data.playerId);

  userInput = null;
  playerTurn = true;
  gameState = "GAME";
  selectedInventoryIndex = 0;

  messageLog = data.messageLog;
  addLog("Game loaded");
};
```

Let's go over it:

We start by retrieving our save game data from localStorage and making a quick check to see if we actually go anything.

```javascript
const loadGame = () => {
  const data = JSON.parse(localStorage.getItem("gameSaveData"));
  if (!data) {
    addLog("Failed to load - no saved games found");
    return;
  }
```

Next we destroy all existing entities - best to start with a clean slate.

```javascript
for (let entity of ecs.entities.all) {
  entity.destroy();
}
```

Then we deserialize our ecs entities and our cache

```javascript
ecs.deserialize(data.ecs);
deserializeCache(data.cache);
```

With our entities all back in place we can use our store player id to get the player entity:

```javascript
player = ecs.getEntity(data.playerId);
```

And then just reset the other little bits we track to their intitial state:

```javascript
userInput = null;
playerTurn = true;
gameState = "GAME";
selectedInventoryIndex = 0;
```

And last, we restore our message log and add a new "Game Loaded" message.

```javascript
  messageLog = data.messageLog;
  addLog("Game loaded");
};
```

We do need to make two more adjustments - our `player` and `messageLog` variables are constants. In order to reset the values we need to initialize them with the `let` keywork instead.

```diff
-export const messageLog = ["", "Welcome to Gobs 'O Goblins!", ""];
+export let messageLog = ["", "Welcome to Gobs 'O Goblins!", ""];
```

```diff
-const player = ecs.createPrefab("Player");
+let player = ecs.createPrefab("Player");
```

And finally we need a keybinding:

```diff
+  if (userInput === "l") {
+    loadGame();
+  }

  if (userInput === "s") {
    saveGame();
  }
```

With all this saving and loading, what about starting a new game? Up to this point the only way to do that was by refreshing the browser - yuck!

## Starting a new game

Starting a new game is very similar to loading an existing one. When starting a new game we don't need to worry about deserializing anything. We just destroy the old entities, clear the cache, reset the bits we care about to their initial state and most importantly, rebuild the dungeon.

Our newGame function looks like this:

```javascript
const newGame = () => {
  for (let item of ecs.entities.all) {
    item.destroy();
  }
  clearCache();

  userInput = null;
  playerTurn = true;
  gameState = "GAME";
  selectedInventoryIndex = 0;

  messageLog = ["", "Welcome to Gobs 'O Goblins!", ""];

  initGame();
};
```

We need to import the `clearCache` function:

```diff
-import { deserializeCache, readCacheSet, serializeCache } from "./state/cache";
+import { clearCache, deserializeCache, readCacheSet, serializeCache } from "./state/cache";
```

And we also need to create the `initGame` function. We already have all the code for initializing the game, we just need to wrap it in a function that we can call when the app starts and when a new game is created.

The body of this function already exists in `./src/index.js` - start by wrapping the existing lines within a new function.

```diff
+const initGame = () => {
  // init game map and player position
  const dungeon = createDungeon({
    x: grid.map.x,
    y: grid.map.y,
    width: grid.map.width,
    height: grid.map.height,
  });

  let player = ecs.createPrefab("Player");
  player.add(Position, {
    x: dungeon.rooms[0].center.x,
    y: dungeon.rooms[0].center.y,
  });

  const openTiles = Object.values(dungeon.tiles).filter(
    (x) => x.sprite === "FLOOR"
  );

  times(5, () => {
    const tile = sample(openTiles);
    ecs.createPrefab("Goblin").add(Position, { x: tile.x, y: tile.y });
  });

  times(10, () => {
    const tile = sample(openTiles);
    ecs.createPrefab("HealthPotion").add(Position, { x: tile.x, y: tile.y });
  });

  times(10, () => {
    const tile = sample(openTiles);
    ecs.createPrefab("ScrollLightning").add(Position, { x: tile.x, y: tile.y });
  });

  times(10, () => {
    const tile = sample(openTiles);
    ecs.createPrefab("ScrollParalyze").add(Position, { x: tile.x, y: tile.y });
  });

  times(10, () => {
    const tile = sample(openTiles);
    ecs.createPrefab("ScrollFireball").add(Position, { x: tile.x, y: tile.y });
  });

  fov(player);
  render(player);
+};
```

The only modification we need to make is to initialize the `player` variable outside of the scope of the `initGame` function.

Within the initGame function change this line:

```diff
-let player = ecs.createPrefab("Player");
+player = ecs.createPrefab("Player");
```

And intitialize the player variable with the others:

```diff
+let player = {};
let userInput = null;
let playerTurn = true;
export let gameState = "GAME";
export let selectedInventoryIndex = 0;
```

Now just make sure to call `initGame` from outside the `newGame` function and add another keybinding:

```diff
+initGame();

document.addEventListener("keydown", (ev) => {
  userInput = ev.key;
});

const processUserInput = () => {
  if (userInput === "l") {
    loadGame();
  }

+  if (userInput === "n") {
+    newGame();
+  }

  if (userInput === "s") {
    saveGame();
  }
```

## A bug

You may have noticed that after creating a new game or loading an existing one, goblins sometimes stop pathing towards the player. There is a small bug in the pathfinding lib that we need to address.

In `./src/lib/pathfinding.js` make this change:

```diff
-  const matrix = [...baseMatrix];
+  const matrix = JSON.parse(JSON.stringify(baseMatrix));
```

Basically our matrix variable wasn't getting cleared properly and instead would combine the old map with the new one. Goblins could only path through areas that happened to be open in both the old and the new maps. Instead of using the spread operator which performs a shallow copy, we are nuking it from orbit and converting the entire thing to a string before parsing it back into an object.

## Menu

Our keybindings have been steadily growing. Let's add a "menu" to clue the player into how to actually play our game.

First we need to add a new menu property to our grid configuration in `./src/lib/canvas`

```diff
export const grid = {
+  menu: {
+    width: 100,
+    height: 1,
+    x: 0,
+    y: 33,
+  },
```

Then in `./src/systems/render.js` add a function called `renderMenu`. It will just render a single line with our keybindings to the bottom of the screen. No need for anything anymore complex.

```javascript
const renderMenu = () => {
  drawText({
    text: `(n)New (s)Save (l)Load | (i)Inventory (g)Pickup (arrow keys)Move/Attack (mouse)Look/Target`,
    background: "transparent",
    color: "#666",
    x: grid.menu.x,
    y: grid.menu.y,
  });
};
```

And call it from the `render` function:

```diff
export const render = (player) => {
  renderMap();
  renderPlayerHud(player);
  renderMessageLog();
+  renderMenu();

  if (gameState === "INVENTORY") {
```

## Game over

One final addition. You may have noticed that you can't create a new game after getting killed. This is because we don't process user input if the player is dead. Let's put the game into a new state `GAMEOVER` and processUserInput in `./src/index.js`.

```diff
if (player.isDead) {
+  if (gameState !== "GAMEOVER") {
+    addLog("You are dead.");
+    render(player);
+  }
+  gameState = "GAMEOVER";
+  processUserInput();
  return;
}
```

In this part we added saving, loading, and starting over to our game. We also added a small "menu" to inform the player about our growing set of keybindings. Finally we can now properly handle the game over. This game is really coming along!

In part 11 we delve deeper as we add more levels to the dungeon!
