#Part 6 - Doing (and taking) some damage

In the last part we setup the groundwork for combat. Now we get to actually implement it. To do that we will need to create some new components to hold data related to combat.

Yet again we're about to add more components.

Remember in the last section when we added the description component to all of our entities and discovered they were scattered all over the codebase? That's only gonna get worse the bigger our game gets. It's time for another refactor :)

Refactoring is a natural part of the creative process - so learn to love it!

Instead of creating entities on the fly we are going to use prefabs instead. Prefabs are great because you can create basic prefabs that other more complex prefabs can inherit from. We're going to create a base "Tile" prefabs that both "Wall" and "Floor" can inherit from. Then we'll do a base "Being" prefab that our Player and Goblin prefabs can inherit from.

In the end prefabs are just composable blueprints for entities. In geotic they are modeled after [this talk by Thomas Biskup](https://www.youtube.com/watch?v=fGLJC5UY2o4&t=1534s). Let's go ahead and create some so you can see how they consolidate our code and make it easier to maintain in the long run.

Create a new file called `prefabs.js` at `./src/state/prefab.js`. Make it look like this:

```javascript
// Base
export const Tile = {
  name: "Tile",
  components: [
    { type: "Appearance" },
    { type: "Description" },
    { type: "Layer100" },
  ],
};

export const Being = {
  name: "Being",
  components: [
    { type: "Appearance" },
    { type: "Description" },
    { type: "IsBlocking" },
    { type: "Layer400" },
  ],
};

// Complex
export const Wall = {
  name: "Wall",
  inherit: ["Tile"],
  components: [
    { type: "IsBlocking" },
    { type: "IsOpaque" },
    {
      type: "Appearance",
      properties: { char: "#", color: "#AAA" },
    },
    {
      type: "Description",
      properties: { name: "wall" },
    },
  ],
};

export const Floor = {
  name: "Floor",
  inherit: ["Tile"],
  components: [
    {
      type: "Appearance",
      properties: { char: "•", color: "#555" },
    },
    {
      type: "Description",
      properties: { name: "floor" },
    },
  ],
};

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
  ],
};

export const Goblin = {
  name: "Goblin",
  inherit: ["Being"],
  components: [
    { type: "Ai" },
    {
      type: "Appearance",
      properties: { char: "g", color: "green" },
    },
    {
      type: "Description",
      properties: { name: "goblin" },
    },
  ],
};
```

Because prefabs must be registered before they can be inherited I find it useful to keep my base prefabs and more complex ones separated. Speaking of registering prefabs - let's do that in `./src/state/ecs`

First we import them:

```diff
} from "./components";

+import { Being, Tile, Goblin, Player, Wall, Floor } from "./prefabs";

const ecs = new Engine();
```

And next we register them after our components - taking care to register your base prefabs first.

```diff
ecs.registerComponent(Move);
ecs.registerComponent(Position);

+// register "base" prefabs first!
+ecs.registerPrefab(Tile);
+ecs.registerPrefab(Being);
+
+ecs.registerPrefab(Wall);
+ecs.registerPrefab(Floor);
+ecs.registerPrefab(Goblin);
+ecs.registerPrefab(Player);
```

Now we can refactor our game to use prefabs instead of creating entities all over the place. Beginning in `./src/lib/dungeon.js` we can get rid of a bunch of imports and simpify things quite a bit.

```diff
import { rectangle, rectsIntersect } from "./grid";

import {
-  Appearance,
-  Description,
-  IsBlocking,
-  IsOpaque,
-  Layer100,
  Position,
} from "../state/components";

function digHorizontalPassage(x1, x2, y) {
```

```diff
if (tile.sprite === "WALL") {
-  const entity = ecs.createEntity();
-  entity.add(Appearance, { char: "#", color: "#AAA" });
-  entity.add(IsBlocking);
-  entity.add(IsOpaque);
-  entity.add(Position, dungeon.tiles[key]);
-  entity.add(Layer100);
-  entity.add(Description, { name: "wall" });
+  ecs.createPrefab("Wall").add(Position, dungeon.tiles[key]);
}

if (tile.sprite === "FLOOR") {
-  const entity = ecs.createEntity();
-  entity.add(Appearance, { char: "•", color: "#555" });
-  entity.add(Position, dungeon.tiles[key]);
-  entity.add(Layer100);
-  entity.add(Description, { name: "floor" });
+  ecs.createPrefab("Floor").add(Position, dungeon.tiles[key]);
}
```

Ahh... deleting code feels so nice - let's do more!

We won't need to create our player entity in `./src/state/ecs.js` anymore so we can clean up a bit:

```diff
ecs.registerPrefab(Goblin);
ecs.registerPrefab(Player);

-export const player = ecs.createEntity();
-player.add(Appearance, { char: "@", color: "#fff" });
-player.add(Layer400);
-player.add(Description, { name: "You" });

export default ecs;
```

Next in `./src/index.js`:

```diff
import { render } from "./systems/render";
-import ecs, { player } from "./state/ecs";
+import ecs from "./state/ecs";
import {
-  Ai,
-  Appearance,
-  Description,
-  IsBlocking,
-  Layer400,
  Move,
  Position,
} from "./state/components";

// init game map and player position
const dungeon = createDungeon({
  x: grid.map.x,
  y: grid.map.y,
  width: grid.map.width,
  height: grid.map.height,
});

+const player = ecs.createPrefab("Player");
player.add(Position, {
  x: dungeon.rooms[0].center.x,
  y: dungeon.rooms[0].center.y,
});
```

```diff
times(5, () => {
  const tile = sample(openTiles);
-
-  const goblin = ecs.createEntity();
-  goblin.add(Ai);
-  goblin.add(Appearance, { char: "g", color: "green" });
-  goblin.add(Description, { name: "goblin" });
-  goblin.add(IsBlocking);
-  goblin.add(Layer400);
-  goblin.add(Position, { x: tile.x, y: tile.y });
+  ecs.createPrefab("Goblin").add(Position, { x: tile.x, y: tile.y });
});
```

The last big of refactoring we need to do is in `./systems/fov.js`. We had been importing our player entity to use as the origin for our FOV. We're going to pass in a generic origin instead.

```diff
import { readCacheSet } from "../state/cache";
-import ecs, { player } from "../state/ecs";
+import ecs from "../state/ecs";
import { grid } from "../lib/canvas";
import createFOV from "../lib/fov";
import { IsInFov, IsOpaque, IsRevealed } from "../state/components";
```

```diff
const opaqueEntities = ecs.createQuery({
  all: [IsOpaque],
});

-export const fov = () => {
+export const fov = (origin) => {
  const { width, height } = grid;

-  const originX = player.position.x;
-  const originY = player.position.y;
+  const originX = origin.position.x;
+  const originY = origin.position.y;

  const FOV = createFOV(opaqueEntities, width, height, originX, originY, 10);
```

And finally we need to pass in the origin everywhere we call our fov system. Back to `./src/index.js`!

We call fov during initialization of the game:

```diff
-fov();
+fov(player);
render();

let userInput = null;
```

And in the loop:

```diff
const update = () => {
  if (playerTurn && userInput) {
    console.log("I am @, hear me roar.");
    processUserInput();
    movement();
-    fov();
+    fov(player);
    render();

    playerTurn = false;
  }

  if (!playerTurn) {
    ai();
    movement();
-    fov();
+    fov(player);
    render();

    playerTurn = true;
  }
};
```

With that refactor out of the way - it's time to tackle combat!

Our combat simulation will be pretty basic to start. We can start with health, power, and defense. Let's add those components now in `./src/state/components.js`:

```javascript
export class Defense extends Component {
  static properties = { max: 1, current: 1 };
}

export class Health extends Component {
  static properties = { max: 10, current: 10 };
}

export class Power extends Component {
  static properties = { max: 5, current: 5 };
}
```

And as always, we need to register them in `./src/state/ecs.js`:

```diff
import {
  Ai,
  Appearance,
  Description,
+  Defense,
+  Health,
  IsBlocking,
  IsInFov,
  IsOpaque,
  IsRevealed,
  Layer100,
  Layer400,
  Move,
  Position,
+  Power,
} from "./components";
```

```diff
// all Components must be `registered` by the engine
ecs.registerComponent(Ai);
ecs.registerComponent(Appearance);
ecs.registerComponent(Description);
+ecs.registerComponent(Defense);
+ecs.registerComponent(Health);
ecs.registerComponent(IsBlocking);
ecs.registerComponent(IsInFov);
ecs.registerComponent(IsOpaque);
ecs.registerComponent(IsRevealed);
ecs.registerComponent(Layer100);
ecs.registerComponent(Layer400);
ecs.registerComponent(Move);
ecs.registerComponent(Position);
+ecs.registerComponent(Power);

// register "primitives" first!
```

Because of our refactor earlier we can just add these new components to our base "Being" prefab! No more having to find every instantiated entity in the code base every time we want to add new components!

In `./src/state/prefabs` just make this change:

```diff
export const Being = {
  name: "Being",
  components: [
    { type: "Appearance" },
+    { type: "Defense" },
    { type: "Description" },
+    { type: "Health" },
    { type: "IsBlocking" },
    { type: "Layer400" },
+    { type: "Power" },
  ],
};
```

Voilà - our player and goblins all have Defense, Health and Power components!

How can we be so sure? Let's add a small bit of tooling to prove it. With it we'll be able to log any entity on the map with a click of the mouse!

We first need a function to translate a mouse click to a location of our grid. In `./src/lib/canvas.js` add this function to the end of the file to do just that:

```javascript
export const pxToCell = (ev) => {
  const bounds = canvas.getBoundingClientRect();
  const relativeX = ev.clientX - bounds.left;
  const relativeY = ev.clientY - bounds.top;
  const colPos = Math.trunc((relativeX / cellWidth) * pixelRatio);
  const rowPos = Math.trunc((relativeY / cellHeight) * pixelRatio);

  return [colPos, rowPos];
};
```

Then in `./src/index.js` we have to import a few things to use it:

```diff
-import { sample, times } from "lodash";
+import { get, sample, times } from "lodash";
import "./lib/canvas.js";
-import { grid } from "./lib/canvas";
+import { grid, pxToCell } from "./lib/canvas";
+import { toLocId } from "./lib/grid";
+import { readCacheSet } from "./state/cache";
import { createDungeon } from "./lib/dungeon";
```

Then at the bottom the file add:

```javascript
// Only do this during development
if (process.env.NODE_ENV === "development") {
  const canvas = document.querySelector("#canvas");

  canvas.onclick = (e) => {
    const [x, y] = pxToCell(e);
    const locId = toLocId({ x, y });

    readCacheSet("entitiesAtLocation", locId).forEach((eId) => {
      const entity = ecs.getEntity(eId);

      console.log(
        `${get(entity, "appearance.char", "?")} ${get(
          entity,
          "description.name",
          "?"
        )}`,
        entity.serialize()
      );
    });
  };
}
```

All this does is use our pxToCell function to calculate a locId from a mouse click. We can then get all the entities at that locId from our cache and log them to the console.

Run the game and click on your @ with the developer console open. The player entity (and the floor it's standing on) should print to the console. You can inspect this object to prove that all the expected components are there!

Finally, it's time to ATTACK!

In a strict ECS architecture we would accomplish this with more components and systems. Maybe a TakeDamage component that stores the amount of damage to be reduced. Then another system that would actually remove the damage. But strict adherance to principle is it's own can of worms. Let's add another tool to our toolbelts that can be used when a system might be a bit too heavy handed.

Introducing Events!

All an event does is send a message to all components on an entity. [This talk by Brian Bucklew](https://www.youtube.com/watch?v=4uxN5GqXcaA) explains why this might be useful - and also happens to be what inspired the event system in geotic that we are using. I struggled for a while trying to understand when to use an event and when to use a system but finally settled on only using events for simple things like component management on their parent entity. Once you need things like queries you're better off using a system. In the end you will have to decide for yourself based on the challenges your game presents.

In `./src/state/components.js` we need to add an event handler to our Health component:

```diff
export class Health extends Component {
  static properties = { max: 10, current: 10 };

+  onTakeDamage(evt) {
+    this.current -= evt.data.amount;
+    evt.handle();
+  }
}
```

We can fire the `take-damage` event on any entity. If that entity has a component that cares, it gets handled, else it's just ignored.

Let's use our new powers in `./src/systems/movement.js`. First create a new function, `attack` that will fire the event:

```diff
  all: [Move],
});

+const attack = (entity, target) => {
+  const damage = entity.power.current - target.defense.current;
+  target.fireEvent("take-damage", { amount: damage });
+  console.log(`You kicked a ${target.description.name} for ${damage} damage!`);
+};

export const movement = () => {
```

Next, instead of kicking, when encountering a blocker, we can attack instead!

```diff
if (blockers.length) {
-  blockers.forEach((eId) => {
-    const attacker =
-      (entity.description && entity.description.name) || "something";
-    const target =
-      (ecs.getEntity(eId).description &&
-        ecs.getEntity(eId).description.name) ||
-      "something";
-    console.log(`${attacker} kicked a ${target}!`);
-  });
+  blockers.forEach((eId) => {
+  const target = ecs.getEntity(eId);
+  if (target.has("Health") && target.has("Defense")) {
+    attack(entity, target);
+  } else {
+    console.log(
+      `${entity.description.name} bump into a ${target.description.name}`
+    );
+  }
  entity.remove(Move);
  return;
}
```

Try it out! Run the game with the developer console open - after attacking a goblin, click on it and inspect it's health component to see if you did the damage you expect.

Pretty cool!

Although you may have noticed something if you kicked your goblin more than a few times - it's health goes into the negative... Time to implement death.

Right after the attack function, add kill:

```javascript
const kill = (entity) => {
  entity.appearance.char = "%";
  entity.remove("Ai");
  entity.remove("IsBlocking");
  entity.remove("Layer400");
  entity.add("Layer300");
};
```

Nothing crazy happening here. Just changing the char to a corpse (%) and removing some components. Without "Ai" our dead goblin doesn't get a turn anymore. We also want to remove "IsBlocking" so and change the layer from the main layer our @ inhabits to the item layer.

Looks like I forgot to register the Layer300 component - do that now in `./src/state/ecs` if you also forgot :P

Finally, let's update our attack function to kill the target if it's health is at or below 0:

```javascript
const attack = (entity, target) => {
  const damage = entity.power.current - target.defense.current;
  target.fireEvent("take-damage", { amount: damage });

  if (target.health.current <= 0) {
    kill(target);

    return console.log(
      `You kicked a ${target.description.name} for ${damage} damage and killed it!`
    );
  }

  console.log(`You kicked a ${target.description.name} for ${damage} damage!`);
};
```

Yeah! These dirty goblins don't stand a chance!!!

Ok - this isn't really fair. They're completely defenseless. Let's actually give them a chance with some actual (albeit basic) AI!
