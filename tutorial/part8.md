# Part 8 - Items and Inventory

Who doesn't love finding a useful item to save you in a pinch? Our @ is in search of riches and treasure after all. We're about to add health potions and an inventory UI to manage them. I'm not gonna lie. This part is gonna be a beast. So brew a fresh cup of your stimulant of choice and buckle up.

## Adding health potions to the dungeon

Let's begin by simply adding health potions to our map. They won't do anything yet - but you gotta start somewhere.

First add a new component to `./src/state/components.js` called `IsPickup`.

```diff
export class IsOpaque extends Component {}

+export class IsPickup extends Component {}

export class IsRevealed extends Component {}
```

This component will actually do some things later but for now it's just a flag to know that an entity with it can be picked up.

Register it in `./src/state/ecs.js`:

```diff
import {
  ...
  IsOpaque
+  IsPickup,
  IsRevealed
  ...
} from "./components";
```

```diff
ecs.registerComponent(IsOpaque);
+ecs.registerComponent(IsPickup);
ecs.registerComponent(IsRevealed);
```

Next we'll create a couple of new prefabs - the generic base `Item` and our more specific `HealthPotion` in `./src/state/prefabs.js`.

```javascript
export const Item = {
  name: 'Item',
  components: [
    { type: 'Appearance' },
    { type: 'Description' },
    { type: 'Layer300' },
    { type: 'IsPickup' },
  ],
};

export const HealthPotion = {
  name: 'HealthPotion',
  inherit: ['Item'],
  components: [
    {
      type: 'Appearance',
      properties: { char: '!', color: '#DAA520' },
    },
    {
      type: 'Description',
      properties: { name: 'health potion' },
    },
  ],
};
```

Notice we place our Item prefab on layer300. This is our 'items' layer and will ensure they render above the floor but below players and monsters.

Register these as well, taking care to register the base `Item` prefab before the more specific `HealthPotion`. Remember that prefabs must be registered before they can be inherited from.

```diff
import {
  Being,
+  Item,
  Tile,
+  HealthPotion,
  Goblin,
  Player,
  Wall,
  Floor,
} from "./prefabs";
```

```diff
// register "base" prefabs first!
ecs.registerPrefab(Being);
+ecs.registerPrefab(Item);

+ecs.registerPrefab(HealthPotion);
ecs.registerPrefab(Wall);
```

And finally let's actually add them to the map in `./src/index.js`:

```diff
times(5, () => {
  const tile = sample(openTiles);
  world.createPrefab("Goblin").add(Position, { x: tile.x, y: tile.y });
});

+times(5, () => {
+  const tile = sample(openTiles);
+  world.createPrefab("HealthPotion").add(Position, { x: tile.x, y: tile.y });
+});
```

I found it useful during this part to render a ton of potions - maybe 50 - and no goblins. This makes testing a lot easier as you don't have to explore the dungeon to find a potion or worry about getting killed on the way.

Run the game! You should see some potions lying on the dungeon floor (!)

## Adding an inventory

Of course our @ needs a place to store the things it picks up. Time to add an inventory!

Yet again, we'll start by adding a new component. In `./src/state/components.js` add an `Inventory`. It will only need a list property for storing an array of item entities.

```javascript
export class Inventory extends Component {
  static properties = {
    list: [],
  };
}
```

As always, register it in `./src/state.ecs.js`.

```diff
import {
  ...
  Health,
+  Inventory,
  IsBlocking,
  ...
} from "./components";
```

```diff
ecs.registerComponent(Health);
+ecs.registerComponent(Inventory);
ecs.registerComponent(IsBlocking);
```

We also need to add it to our player prefab in `./src/state/prefabs.js`:

```diff
export const Player = {
  name: "Player",
  inherit: ["Being"],
  components: [
    {
      type: "Appearance",
      properties: { char: "@", color: "#FFF" },
    },
    {
      type: "Description",
      properties: { name: "You" },
    },
    { type: "Health", properties: { current: 20, max: 20 } },
+   { type: "Inventory" },
  ],
};
```

If you run the game now, you should be able to click on your @ and inspect it's components. There should be an empty inventory!

## Managing our inventory with (g)Get and (d)Drop

We'll start in our components file with adding a couple events to the `Inventory` component. We will need an `onPickUp` and an `onDrop` event. These will handle adding and removing items to our inventory list as well as removing the item from the dungeon floor on pickup and putting down on drop.

In `./src/state/components.js` add `onPickUp` and `onDrop` events to the Inventory components. Both of these events will expect some entity as a payload.

```diff
export class Inventory extends Component {
  static properties = {
    list: [],
  };

+  onPickUp(evt) {
+    this.list.push(evt.data);
+
+    if (evt.data.position) {
+      evt.data.remove(evt.data.position);
+    }
+  }
+
+  onDrop(evt) {
+    remove(this.list, (x) => x.id === evt.data.id);
+    evt.data.add(Position, this.entity.position);
+  }
}
```

We already take advantage of the `onAttached` lifecycle method in our `Position` component. This event fires automatically when the add method is called on an entity and we use it to populate our entitiesAtLocation cache.

We can also take advantage of the `onDetached` lifecyle method to remove entities from the cache like so:

```diff
export class Position extends Component {
  static properties = { x: 0, y: 0 };

  onAttached() {
    const locId = `${this.entity.position.x},${this.entity.position.y}`;
    addCacheSet("entitiesAtLocation", locId, this.entity.id);
  }

+  onDetached() {
+    const locId = `${this.x},${this.y}`;
+    deleteCacheSet("entitiesAtLocation", locId, this.entity.id);
+  }
}
```

Now that we have our eventing setup we just need to create keybindings to fire them with the correct payloads.

We will want to add some logging when things picked up and dropped so import the addLog function at the top of `./src/index.js`.

```diff
-import world from "./state/ecs";
+import world, { addLog } from "./state/ecs";
```

In the same file add the keybindings for (g)Get and (d)Drop

```diff
if (userInput === "ArrowLeft") {
  player.add(Move, { x: -1, y: 0 });
}
+
+if (userInput === "g") {
+  let pickupFound = false;
+  readCacheSet("entitiesAtLocation", toLocId(player.position)).forEach(
+    (eId) => {
+      const entity = world.getEntity(eId);
+      if (entity.isPickup) {
+        pickupFound = true;
+        player.fireEvent("pick-up", entity);
+        addLog(`You pickup a ${entity.description.name}`);
+      }
+    }
+  );
+  if (!pickupFound) {
+    addLog("There is nothing to pick up here");
+  }
+}
+
+if (userInput === "d") {
+  if (player.inventory.list.length) {
+    player.fireEvent("drop", player.inventory.list[0]);
+  }
+}

userInput = null;
```

We're doing a few things here.

First we create our keybinding for `g` and immediately create a flag to track whether or not a pickup was found.

```javascript
if (userInput === "g") {
  let pickupFound = false;
```

After that we read from our entitiesAtLocation cache and iterate through everything we find. If it has an `isPickup` component we set our pickupFound flag to true and fire our pick-up event passing it the entity. Then we add a message to our adventure log describing what was picked up.

```javascript
readCacheSet('entitiesAtLocation', toLocId(player.position)).forEach((eId) => {
  const entity = world.getEntity(eId);
  if (entity.isPickup) {
    pickupFound = true;
    player.fireEvent('pick-up', entity);
    addLog(`You pickup a ${entity.description.name}`);
  }
});
```

And lastly we check the flag - if after iterating through all the entities at our location we still haven't found anything to pickup - let the player know in the adventure log.

```javascript
if (!pickupFound) {
  addLog('There is nothing to pick up here');
}
```

Next we add a keybinding for `d`. For now we just check if there is anything in the inventory and if so drop the first item. We'll add a UI next so we can actually select the item to drop but this will get us started.

```javascript
if (userInput === 'd') {
  if (player.inventory.list.length) {
    addLog(`You drop a ${player.inventory.list[0].description.name}`);
    player.fireEvent('drop', player.inventory.list[0]);
  }
}
```

Now you can walk around the map and hit the `g` key to pickup potions and the `d` key to put them somewhere else!

## Inventory UI

We really need a UI to display the inventory so we can choose what items to drop.

We'll start in `./src/lib/canvas.js` with settings for where our inventory will display on our grid and another utility function for drawing rectangles.

In the grid object add another setting for our inventory like so:

```javascript
export const grid = {
  ...
  inventory: {
    width: 37,
    height: 28,
    x: 21,
    y: 4,
  },
};
```

And then import rectangle from our grid library and use it in a new function that will draw rectangles on our grid like this:

```javascript
import { rectangle } from './grid';
```

```javascript
export const drawRect = (x, y, width, height, color) => {
  const rect = rectangle({ x, y, width, height });

  Object.values(rect.tiles).forEach((position) => {
    drawBackground({ color, position });
  });
};
```

Now in our render system we need to render the inventory itself. We don't want to render it all the time, so we'll add a new concept called gameState. We'll only render our inventory when our game is in the 'INVENTORY' gameState.

In `./src/systems/render.js` import the new drawRect function as well as a couple of variables from `./src/index.js` that we haven't created yet.

```diff
import {
  clearCanvas,
  drawCell,
+  drawRect,
  drawText,
  grid,
  pxToCell,
} from "../lib/canvas";
import { toLocId } from "../lib/grid";
import { readCacheSet } from "../state/cache";
+import { gameState, selectedInventoryIndex } from "../index";
```

And then at the bottom of our render function add another conditional to display our inventory as an overlay if we're in the INVENTORY gameState:

```javascript
if (gameState === 'INVENTORY') {
  // translucent to obscure the game map
  drawRect(0, 0, grid.width, grid.height, 'rgba(0,0,0,0.65)');

  drawText({
    text: 'INVENTORY',
    background: 'black',
    color: 'white',
    x: grid.inventory.x,
    y: grid.inventory.y,
  });

  if (player.inventory.list.length) {
    player.inventory.list.forEach((entity, idx) => {
      drawText({
        text: `${idx === selectedInventoryIndex ? '*' : ' '}${
          entity.description.name
        }`,
        background: 'black',
        color: 'white',
        x: grid.inventory.x,
        y: grid.inventory.y + 3 + idx,
      });
    });
  } else {
    drawText({
      text: '-empty-',
      background: 'black',
      color: '#666',
      x: grid.inventory.x,
      y: grid.inventory.y + 3,
    });
  }
}
```

This is pretty straight forward. We render a large rectangle grid of translucent cells to obscure the map, a heading `INVENTORY`, and then the inventory items themselves with a `*` before the current selected item. Finally we add a message if the inventory is empty.

Time to deal with those undefined variables we imported from `./src/index.js`.

Just above our keybinding add `gameState` and `selectInventoryIndex`:

```diff
let userInput = null;
let playerTurn = true;
+export let gameState = "GAME";
+export let selectedInventoryIndex = 0;

document.addEventListener("keydown", (ev) => {
  userInput = ev.key;
});
```

We need to be able to handle keybindings differently in the Inventory than we do when playing the game. This concept of a gameState allows us to do just that.

Update the update function for our game loop to look like this:

```javascript
const update = () => {
  if (player.isDead) {
    return;
  }

  if (playerTurn && userInput && gameState === 'INVENTORY') {
    processUserInput();
    render(player);
    playerTurn = true;
  }

  if (playerTurn && userInput && gameState === 'GAME') {
    processUserInput();
    movement();
    fov(player);
    render(player);

    if (gameState === 'GAME') {
      playerTurn = false;
    }
  }

  if (!playerTurn) {
    ai(player);
    movement();
    fov(player);
    render(player);

    playerTurn = true;
  }
};
```

Notice how we run different systems in each game state and also that in the INVENTORY game state we always set playerTurn to true.

We need to update our processUserInput function to handle different gameStates as well. It should now look like this:

```javascript
const processUserInput = () => {
  if (gameState === 'GAME') {
    if (userInput === 'ArrowUp') {
      player.add(Move, { x: 0, y: -1 });
    }
    if (userInput === 'ArrowRight') {
      player.add(Move, { x: 1, y: 0 });
    }
    if (userInput === 'ArrowDown') {
      player.add(Move, { x: 0, y: 1 });
    }
    if (userInput === 'ArrowLeft') {
      player.add(Move, { x: -1, y: 0 });
    }

    if (userInput === 'g') {
      let pickupFound = false;
      readCacheSet('entitiesAtLocation', toLocId(player.position)).forEach(
        (eId) => {
          const entity = world.getEntity(eId);
          if (entity.isPickup) {
            pickupFound = true;
            player.fireEvent('pick-up', entity);
            addLog(`You pickup a ${entity.description.name}`);
          }
        }
      );
      if (!pickupFound) {
        addLog('There is nothing to pick up here');
      }
    }

    if (userInput === 'i') {
      gameState = 'INVENTORY';
    }

    userInput = null;
  }

  if (gameState === 'INVENTORY') {
    if (userInput === 'i' || userInput === 'Escape') {
      gameState = 'GAME';
    }

    if (userInput === 'ArrowUp') {
      selectedInventoryIndex -= 1;
      if (selectedInventoryIndex < 0) selectedInventoryIndex = 0;
    }

    if (userInput === 'ArrowDown') {
      selectedInventoryIndex += 1;
      if (selectedInventoryIndex > player.inventory.list.length - 1)
        selectedInventoryIndex = player.inventory.list.length - 1;
    }

    if (userInput === 'd') {
      if (player.inventory.list.length) {
        addLog(`You drop a ${player.inventory.list[0].description.name}`);
        player.fireEvent('drop', player.inventory.list[0]);
      }
    }

    userInput = null;
  }
};
```

You'll notice we now check for the gameState before processing inputs. We also added a new keybinding (i)Inventory that sets the gameState and acts as a toggle for the menu.

Give it a shot! Run the game and check your inventory to see the empty state - then pick up some things and use the arrows to select different items.

## Consuming a potion

In part 6 we used a simple `take-damage` event to reduce the health of a target. This approach was quick and easy but not very flexible. We don't really have a concept for different types of damage or any means whatsoever to mix and match the various kinds of damage we might want to add. For instance, how would we create a fire and ice sword?

Let's try and do something more flexible for potions. Imagine we wanted to build a crafting system where a player could combine different herbs to create a potion. We would expect each herb to have one or many effects that could combine to create a potion that delivered those effects to the player on consumption. We won't be building a crafting system but we will attempt to build something generic enough to handle one.

The approach we'll take is to create an effects system. `Effect` components will store the various effects an entity has. An `ActiveEffects` component will be used by our effects system to make any necessary calculations and update relevant components. Don't worry, it's not as complicated as it sounds.

Let's start as always in `./src/state/components.js`. Add the two components we just spoke about, `Effect` and `ActiveEffects`

```javascript
export class ActiveEffects extends Component {
  static allowMultiple = true;
  static properties = { component: '', delta: '' };
}

export class Effects extends Component {
  static allowMultiple = true;
  static properties = { component: '', delta: '' };
}
```

Up to now our entities have only supported a single component of any given type. The line `static allowMultiple = true;` tells Geotic we want to allow multiple components of this type on an entity. Each `Effect` or `ActiveEffect` contains two properties - a component name and a delta. This will allow us to easily create potions that effect different components on an entity like health and or power. We can also play with the delta (the net change in the component value) to create a health potion with a delta of 5 or a poison with a delta of -5.

But before we get ahead of ourselves let's register our new components in `./src/state/ecs.js`:

```diff
import {
+  ActiveEffects,
  Ai,
  Appearance,
  Description,
  Defense,
+  Effects,
  Health,
```

```diff
// all Components must be `registered` by the engine
+ecs.registerComponent(ActiveEffects);
ecs.registerComponent(Ai);
ecs.registerComponent(Appearance);
ecs.registerComponent(Description);
ecs.registerComponent(Defense);
+ecs.registerComponent(Effects);
ecs.registerComponent(Health);
```

We already have a HealthPotion prefab but it doesn't have any effects yet. Let's add one now in `src/state/prefabs`:

```diff
export const HealthPotion = {
  name: "HealthPotion",
  inherit: ["Item"],
  components: [
    {
      type: "Appearance",
      properties: { char: "!", color: "#DAA520" },
    },
    {
      type: "Description",
      properties: { name: "health potion" },
    },
+    {
+      type: "Effects",
+      properties: { component: "health", delta: 5 },
+    },
  ],
};
```

Now for our effects system. Add a new file `effect.js` at `./src/systems/effects.js` it should look like this:

```javascript
import world from '../state/ecs';
const { ActiveEffects } = require('../state/components');

const activeEffectsEntities = world.createQuery({
  all: [ActiveEffects],
});

export const effects = () => {
  activeEffectsEntities.get().forEach((entity) => {
    entity.activeEffects.forEach((c) => {
      if (entity[c.component]) {
        entity[c.component].current += c.delta;

        if (entity[c.component].current > entity[c.component].max) {
          entity[c.component].current = entity[c.component].max;
        }
      }

      c.destroy();
    });
  });
};
```

It's actually a pretty simple component. We have a query to find Entities with ActiveEffects and we iterate over any we find. The activeEffects components is an array (because we allow multiple) so we loop over it and for each active effect we find - check if the entity has the relevant component and if so calculate the delta and update it.

Finally we remove the activeEffect.

Now that we have a generic effects system in place it's time to create a means to consume a potion so it can take effect!

Add another keybinding (c)Consume in `./src/index.js` right before (d)Drop like this:

```javascript
if (userInput === 'c') {
  const entity = player.inventory.list[selectedInventoryIndex];

  if (entity) {
    if (entity.has('Effects')) {
      // clone all effects and add to self
      entity
        .get('Effects')
        .forEach((x) => player.add('ActiveEffects', { ...x.serialize() }));
    }

    addLog(`You consume a ${entity.description.name}`);
    player.inventory.list = player.inventory.list.filter(
      (item) => item.id !== entity.id
    );

    if (selectedInventoryIndex > player.inventory.list.length - 1)
      selectedInventoryIndex = player.inventory.list.length - 1;
  }
}
```

Consume does a few things. First it gets the currently selected item in your inventory. If it has effects components, it clones each of them and adds them to the player as ActiveEffects. A helpful message is logged to and the item is destroyed. Remember all the way at the beginning when we set the items property on the inventory component to an "<EntityArray>"? Geotic keeps track of those entities so when we destroy one it is automatically removed from the inventory list. Pretty nice :)

All that's left is to call the new effects system itself.

Import the new system in `./src/index.js`:

```javascript
import { effects } from './systems/effects';
```

And then call it in the update function like this:

```diff
const update = () => {
  if (player.isDead) {
    return;
  }

  if (playerTurn && userInput && gameState === "INVENTORY") {
    processUserInput();
+   effects();
    render(player);
    playerTurn = true;
  }

  if (playerTurn && userInput && gameState === "GAME") {
    processUserInput();
+   effects();
    movement();
    fov(player);
    render(player);

    if (gameState === "GAME") {
      playerTurn = false;
    }
  }

  if (!playerTurn) {
    ai(player);
+   effects();
    movement();
    fov(player);
    render(player);

    playerTurn = true;
  }
};
```

Add goblins back to your game if you removed them earlier and a give it a go! Find a potion, get punched by a goblin, and quaff a potion - your health bar should refill!

There is but one more small addition - let's add the keybindings to our inventory screen so our player knows how to consume or drop a potion.

In `./src/systems/render.js` draw another line of text just below 'INVENTORY'

```diff
drawText({
  text: "INVENTORY",
  background: "black",
  color: "white",
  x: grid.inventory.x,
  y: grid.inventory.y,
});
+
+drawText({
+  text: "(c)Consume (d)Drop",
+  background: "black",
+  color: "#666",
+  x: grid.inventory.x,
+  y: grid.inventory.y + 1,
+});
```

That was a long ride! Hopefully you can see you how powerful a generic system like this can be. Can you make a poison potion or one that combines multiple effects?

We'll take this concept even further in [part 9](https://github.com/luetkemj/jsrlt/blob/master/tutorial/part9.md) when we take on ranged scrolls and targeting!
