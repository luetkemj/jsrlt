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
