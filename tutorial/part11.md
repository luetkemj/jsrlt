# Part 11 - Delving into the Dungeon

Our dungeon lacks depth. There is afterall only a single floor to explore. In this part we will again refactor things - this time to support z levels, and to generate dungeon floors on the fly. We will have persistant randomly generated dungeon floors with stairs to link them. That's a lot to do so let's get started!

## Supporting z levels in the cache

Our cache will need to store the current z level and some details about each floor that has been created. This is how we will be able to generate persistent levels on the way down that we can revisit on the way back up.

In `./src/state/cache.js`:

```diff
+import { get, set } from "lodash";
+
export const cache = {
  entitiesAtLocation: {},
+  z: -1,
+  floors: {}, // { z: { stairsUp: '', stairsDown: '' } }
};

+export const addCache = (path, value) => {
+  set(cache, path, value);
+};
+
+export const readCache = (path) => get(cache, path);
+
export const addCacheSet = (name, key, value) => {
  if (cache[name][key]) {
    cache[name][key].add(value);
```

```diff
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
+    z: cache.z,
+    floors: cache.floors,
  };
};
```

```diff
export const deserializeCache = (data) => {
  cache.entitiesAtLocation = Object.keys(data.entitiesAtLocation).reduce(
    (acc, val) => {
      acc[val] = new Set(data.entitiesAtLocation[val]);
      return acc;
    },
    {}
  );

+  cache.z = data.z;
+  cache.floors = data.floors;
};

export const clearCache = () => {
  cache.entitiesAtLocation = {};
+  cache.z = 1;
  cache.floors = {};
};
```

Before moving on the next section, test your game and make sure everything is still working exactly as it was before.

## Refactor Z

Our cache now supports z levels but the rest of our game does not. That is going to require a pretty hefty refactor. While it would have been nice if we had anticipated this need from the get go - I would argue it's best to code for what you need at the time, not what you might want down the line. Refactoring is just a natural part of the gig. There are 10 files to adjust and for clarity, I have each file as a heading with the changes below it. There's no commentary for the rest of this section. Just grab a cup of coffee or tea and get on it.

### `./src/index.js`

```diff
import {
  clearCache,
  deserializeCache,
+  readCache,
  readCacheSet,
  serializeCache,
} from "./state/cache";
```

```diff
const loadGame = () => {
  const data = JSON.parse(localStorage.getItem("gameSaveData"));
  if (!data) {
    addLog("Failed to load - no saved games found");
    return;
  }

  for (let entity of ecs.entities.all) {
    entity.destroy();
  }
+  clearCache();
```

```diff
const dungeon = createDungeon({
  x: grid.map.x,
  y: grid.map.y,
+  z: readCache('z'),
  width: grid.map.width,
  height: grid.map.height,
});
```

```diff
  times(5, () => {
    const tile = sample(openTiles);
-    ecs.createPrefab("Goblin").add(Position, { x: tile.x, y: tile.y });
+    ecs.createPrefab("Goblin").add(Position, tile);
  });

  times(10, () => {
    const tile = sample(openTiles);
-    ecs.createPrefab("HealthPotion").add(Position, { x: tile.x, y: tile.y });
+    ecs.createPrefab("HealthPotion").add(Position, tile);
  });

  times(10, () => {
    const tile = sample(openTiles);
-    ecs.createPrefab("ScrollLightning").add(Position, { x: tile.x, y: tile.y });
+    ecs.createPrefab("ScrollLightning").add(Position, tile);
  });

  times(10, () => {
    const tile = sample(openTiles);
-    ecs.createPrefab("ScrollParalyze").add(Position, { x: tile.x, y: tile.y });
+    ecs.createPrefab("ScrollParalyze").add(Position, tile);
  });

  times(10, () => {
    const tile = sample(openTiles);
-    ecs.createPrefab("ScrollFireball").add(Position, { x: tile.x, y: tile.y });
+    ecs.createPrefab("ScrollFireball").add(Position, tile);
  });
```

```diff
if (gameState === "GAME") {
  if (userInput === "ArrowUp") {
-    player.add(Move, { x: 0, y: -1 });
+    player.add(Move, { x: 0, y: -1, z: readCache("z") });
  }
  if (userInput === "ArrowRight") {
-    player.add(Move, { x: 1, y: 0 });
+    player.add(Move, { x: 1, y: 0, z: readCache("z") });
  }
  if (userInput === "ArrowDown") {
-    player.add(Move, { x: 0, y: 1 });
+    player.add(Move, { x: 0, y: 1, z: readCache("z") });
  }
  if (userInput === "ArrowLeft") {
-    player.add(Move, { x: -1, y: 0 });
+    player.add(Move, { x: -1, y: 0, z: readCache("z") });
  }
```

```diff
canvas.onclick = (e) => {
  const [x, y] = pxToCell(e);
-  const locId = toLocId({ x, y });
+  const locId = toLocId({ x, y, z: readCache("z") });

  readCacheSet("entitiesAtLocation", locId).forEach((eId) => {
```

```diff
if (gameState === "TARGETING") {
  const entity = player.inventory.list[selectedInventoryIndex];
  if (entity.requiresTarget.aoeRange) {
-    const targets = circle({ x, y }, entity.requiresTarget.aoeRange);
+    const targets = circle({ x, y }, entity.requiresTarget.aoeRange).map(
+      (locId) => `${locId},${readCache("z")}`
+    );
    targets.forEach((locId) => player.add("Target", { locId }));
  } else {
    player.add("Target", { locId });
```

### `./src/lib/dungeon.js`

```diff
import { random, times } from "lodash";
import ecs from "../state/ecs";
import { rectangle, rectsIntersect } from "./grid";
import { Position } from "../state/components";

-function digHorizontalPassage(x1, x2, y) {
+function digHorizontalPassage(x1, x2, y, z) {
  const tiles = {};
  const start = Math.min(x1, x2);
  const end = Math.max(x1, x2) + 1;
  let x = start;

  while (x < end) {
-    tiles[`${x},${y}`] = { x, y, sprite: "FLOOR" };
+    tiles[`${x},${y},${z}`] = { x, y, z, sprite: "FLOOR" };
    x++;
  }

  return tiles;
}

-function digVerticalPassage(y1, y2, x) {
+function digVerticalPassage(y1, y2, x, z) {
  const tiles = {};
  const start = Math.min(y1, y2);
  const end = Math.max(y1, y2) + 1;
  let y = start;

  while (y < end) {
-    tiles[`${x},${y}`] = { x, y, sprite: "FLOOR" };
+    tiles[`${x},${y},${z}`] = { x, y, z, sprite: "FLOOR" };
    y++;
  }

  return tiles;
}

export const createDungeon = ({
  x,
  y,
+  z,
  width,
  height,
  minRoomSize = 6,
  maxRoomSize = 12,
  maxRoomCount = 30,
}) => {
  // fill the entire space with walls so we can dig it out later
  const dungeon = rectangle(
-    { x, y, width, height },
+    { x, y, z, width, height },
    {
      sprite: "WALL",
    }
  );

  const rooms = [];
  let roomTiles = {};

  times(maxRoomCount, () => {
    let rw = random(minRoomSize, maxRoomSize);
    let rh = random(minRoomSize, maxRoomSize);
    let rx = random(x, width + x - rw - 1);
    let ry = random(y, height + y - rh - 1);

    // create a candidate room
    const candidate = rectangle(
-      { x: rx, y: ry, width: rw, height: rh, hasWalls: true },
+      { x: rx, y: ry, z, width: rw, height: rh, hasWalls: true },
      { sprite: "FLOOR" }
    );

    // test if candidate is overlapping with any existing rooms
    if (!rooms.some((room) => rectsIntersect(room, candidate))) {
      rooms.push(candidate);
      roomTiles = { ...roomTiles, ...candidate.tiles };
    }
  });

  let prevRoom = null;
  let passageTiles;

  for (let room of rooms) {
    if (prevRoom) {
      const prev = prevRoom.center;
      const curr = room.center;

      passageTiles = {
        ...passageTiles,
-        ...digHorizontalPassage(prev.x, curr.x, curr.y),
-        ...digVerticalPassage(prev.y, curr.y, prev.x),
+        ...digHorizontalPassage(prev.x, curr.x, curr.y, z),
+        ...digVerticalPassage(prev.y, curr.y, prev.x, z),
      };
    }

    prevRoom = room;
  }

  dungeon.rooms = rooms;
  dungeon.tiles = { ...dungeon.tiles, ...roomTiles, ...passageTiles };

  // create tile entities
  Object.keys(dungeon.tiles).forEach((key) => {
    const tile = dungeon.tiles[key];

    if (tile.sprite === "WALL") {
-      ecs.createPrefab("Wall").add(Position, dungeon.tiles[key]);
+      ecs.createPrefab("Wall").add(Position, { ...dungeon.tiles[key], z });
    }

    if (tile.sprite === "FLOOR") {
-      ecs.createPrefab("Floor").add(Position, dungeon.tiles[key]);
+      ecs.createPrefab("Floor").add(Position, { ...dungeon.tiles[key], z });
    }
  });

  return dungeon;
};
```

### `./src/lib/fov.js`

```diff
  height,
  originX,
  originY,
+  originZ,
  radius
) {
  const visible = new Set();

  const blockingLocations = new Set();
-  opaqueEntities
-    .get()
-    .forEach((x) => blockingLocations.add(`${x.position.x},${x.position.y}`));
+  opaqueEntities.get().forEach((x) => {
+    if (x.position.z === originZ) {
+      blockingLocations.add(`${x.position.x},${x.position.y},${x.position.z}`);
+    }
+  });

  const isOpaque = (x, y) => {
-    const locId = `${x},${y}`;
+    const locId = `${x},${y},${originZ}`;
    return !!blockingLocations.has(locId);
  };
  const reveal = (x, y) => {
-    return visible.add(`${x},${y}`);
+    return visible.add(`${x},${y},${originZ}`);
  };

  function castShadows(originX, originY, row, start, end, transform, radius) {
```

### `./src/lib/grid.js`

```diff
-export const rectangle = ({ x, y, width, height, hasWalls }, tileProps) => {
+export const rectangle = ({ x, y, z, width, height, hasWalls }, tileProps) => {
  const tiles = {};

  const x1 = x;
  const x2 = x + width - 1;
  const y1 = y;
  const y2 = y + height - 1;
  if (hasWalls) {
    for (let yi = y1 + 1; yi <= y2 - 1; yi++) {
      for (let xi = x1 + 1; xi <= x2 - 1; xi++) {
-        tiles[`${xi},${yi}`] = { x: xi, y: yi, ...tileProps };
+        tiles[`${xi},${yi},${z}`] = { x: xi, y: yi, z, ...tileProps };
      }
    }
  } else {
    for (let yi = y1; yi <= y2; yi++) {
      for (let xi = x1; xi <= x2; xi++) {
-        tiles[`${xi},${yi}`] = { x: xi, y: yi, ...tileProps };
+        tiles[`${xi},${yi},${z}`] = { x: xi, y: yi, z, ...tileProps };
      }
    }
  }

  const center = {
    x: Math.round((x1 + x2) / 2),
    y: Math.round((y1 + y2) / 2),
+    z,
  };

  return { x1, x2, y1, y2, center, hasWalls, tiles };
};
```

```diff
export const idToCell = (id) => {
  const coords = id.split(",");
-  return { x: parseInt(coords[0], 10), y: parseInt(coords[1], 10) };
+  return { x: parseInt(coords[0], 10), y: parseInt(coords[1], 10), z: parseInt(coords[2], 10) };
};

-export const cellToId = ({ x, y }) => `${x},${y}`;
+export const cellToId = ({ x, y, z }) => `${x},${y},${z}`;
```

### `./src/lib/pathfinding.js`

```diff
import PF from "pathfinding";
import { some, times } from "lodash";
import ecs from "../state/ecs";
-import cache, { readCacheSet } from "../state/cache";
+import { readCache, readCacheSet } from "../state/cache";
import { toCell } from "./grid";
import { grid } from "./canvas";
```

```diff
export const aStar = (start, goal) => {
  const matrix = JSON.parse(JSON.stringify(baseMatrix));

-  const locIds = Object.keys(cache.entitiesAtLocation);
+  const locIds = Object.keys(readCache("entitiesAtLocation"));
  locIds.forEach((locId) => {
-    if (
-      some([...readCacheSet("entitiesAtLocation", locId)], (eId) => {
-        return ecs.getEntity(eId).isBlocking;
-      })
-    ) {
-      const cell = toCell(locId);
-
-      matrix[cell.y][cell.x] = 1;
+    const cell = toCell(locId);
+    if (cell.z === readCache("z")) {
+      if (
+        some([...readCacheSet("entitiesAtLocation", locId)], (eId) => {
+          return ecs.getEntity(eId).isBlocking;
+        })
+      ) {
+        matrix[cell.y][cell.x] = 1;
+      }
    }
  });
```

### `./src/state/components.js`

```diff
export class Layer400 extends Component {}

export class Move extends Component {
-  static properties = { x: 0, y: 0, relative: true };
+  static properties = { x: 0, y: 0, z: 0, relative: true };
}

export class Paralyzed extends Component {}

export class Position extends Component {
-  static properties = { x: 0, y: 0 };
+  static properties = { x: 0, y: 0, z: -1 };

  onAttached() {
-    const locId = `${this.entity.position.x},${this.entity.position.y}`;
+    const locId = `${this.entity.position.x},${this.entity.position.y},${this.entity.position.z}`;
    addCacheSet("entitiesAtLocation", locId, this.entity.id);
  }

  onDetached() {
-    const locId = `${this.x},${this.y}`;
+    const locId = `${this.x},${this.y},${this.z}`;
    deleteCacheSet("entitiesAtLocation", locId, this.entity.id);
  }
}
```

### `./src/systems/ai.js`

```diff
import ecs from "../state/ecs";
import { Ai, Description } from "../state/components";
import { aStar } from "../lib/pathfinding";
+import { readCache } from "../state/cache";

const aiEntities = ecs.createQuery({
  all: [Ai, Description],
});

const moveToTarget = (entity, target) => {
  const path = aStar(entity.position, target.position);
  if (path.length) {
    const newLoc = path[1];
-    entity.add("Move", { x: newLoc[0], y: newLoc[1], relative: false });
+    entity.add("Move", { x: newLoc[0], y: newLoc[1], z: readCache("z"), relative: false });
  }
};
```

### `./src/systems.fov.js`

```diff
import { grid } from "../lib/canvas";
import createFOV from "../lib/fov";
import { IsInFov, IsOpaque, IsRevealed } from "../state/components";
+import { readCache } from "../state/cache";
```

```diff
-  const FOV = createFOV(opaqueEntities, width, height, originX, originY, 10);
+  const FOV = createFOV(opaqueEntities, width, height, originX, originY, readCache("z"), 10);
```

### `./src/systems/movement.js`

```diff
let mx = entity.move.x;
let my = entity.move.y;
+let mz = entity.move.z;

if (entity.move.relative) {
  mx = entity.position.x + entity.move.x;
```

```diff
    // check for blockers
    const blockers = [];
    // read from cache
-    const entitiesAtLoc = readCacheSet("entitiesAtLocation", `${mx},${my}`);
+    const entitiesAtLoc = readCacheSet("entitiesAtLocation", `${mx},${my},${mz}`);

    for (const eId of entitiesAtLoc) {
```

```diff
entity.remove("Position");
-entity.add("Position", { x: mx, y: my });
+entity.add("Position", { x: mx, y: my, z: mz });

entity.remove(Move);
```

### `./src/systems/render.js`

```diff
import { toLocId } from "../lib/grid";
-import { readCacheSet } from "../state/cache";
+import { readCache, readCacheSet } from "../state/cache";
import { gameState, messageLog, selectedInventoryIndex } from "../index";
```

```diff
const renderInfoBar = (mPos) => {
  clearInfoBar();

-  const { x, y } = mPos;
-  const locId = toLocId({ x, y });
+  const { x, y, z } = mPos;
+  const locId = toLocId({ x, y, z });

  const esAtLoc = readCacheSet("entitiesAtLocation", locId) || [];
  const entitiesAtLoc = [...esAtLoc];

  clearInfoBar();

  if (entitiesAtLoc) {
    if (some(entitiesAtLoc, (eId) => ecs.getEntity(eId).isRevealed)) {
      drawCell({
        appearance: {
          char: "",
          background: "rgba(255, 255, 255, 0.5)",
        },
-        position: { x, y },
+        position: { x, y, z },
      });
    }
```

```diff
const renderTargeting = (mPos) => {
-  const { x, y } = mPos;
+  const { x, y, z } = mPos;
-  const locId = toLocId({ x, y });
+  const locId = toLocId({ x, y, z });

  const esAtLoc = readCacheSet("entitiesAtLocation", locId) || [];
  const entitiesAtLoc = [...esAtLoc];

  clearInfoBar();

  if (entitiesAtLoc) {
    if (some(entitiesAtLoc, (eId) => ecs.getEntity(eId).isRevealed)) {
      drawCell({
        appearance: {
          char: "",
          background: "rgba(74, 232, 218, 0.5)",
        },
-        position: { x, y },
+        position: { x, y, z },
      });
    }
  }
};
```

```diff
const canvas = document.querySelector("#canvas");
canvas.onmousemove = throttle((e) => {
  if (gameState === "GAME") {
    const [x, y] = pxToCell(e);
    renderMap();
-    renderInfoBar({ x, y });
+    renderInfoBar({ x, y, z: readCache("z") });
  }

  if (gameState === "TARGETING") {
    const [x, y] = pxToCell(e);
    renderMap();
-    renderInfoBar({ x, y });
+    renderInfoBar({ x, y, z: readCache("z") });
  }
}, 50);
```

Like any good refactor at the end everything should be working just like it was before! Run the game and make sure that's the case before moving on.

## Digging deeper

With the big refactor out of the way let's go ahead and make our stair prefabs. We'll need to add one for each direction in `./src/state/prefabs.js`

```javascript
export const StairsUp = {
  name: "StairsUp",
  inherit: ["Tile"],
  components: [
    {
      type: "Appearance",
      properties: { char: "<", color: "#AAA" },
    },
    {
      type: "Description",
      properties: { name: "set of stairs leading up" },
    },
  ],
};

export const StairsDown = {
  name: "StairsDown",
  inherit: ["Tile"],
  components: [
    {
      type: "Appearance",
      properties: { char: ">", color: "#AAA" },
    },
    {
      type: "Description",
      properties: { name: "set of stairs leading down" },
    },
  ],
};
```

Go ahead and register them in `./src/state/ecs.js`

```diff
  Wall,
  Floor,
+  StairsUp,
+  StairsDown,
} from "./prefabs";
```

```diff
ecs.registerPrefab(ScrollParalyze);
+ecs.registerPrefab(StairsUp);
+ecs.registerPrefab(StairsDown);

export default ecs;
```

We want to generate levels as they are needed - when a players descends to a new and deeper depth. Currently we only generate a single level in `addCache,`. We'll need to break that up into a few different functions that can be called as needed.

In `./src/index.js` we need to first import a couple functions we'll use in a bit.

```diff
import "./lib/canvas.js";
import { grid, pxToCell } from "./lib/canvas";
-import { toLocId, circle } from "./lib/grid";
+import { toCell, toLocId, circle } from "./lib/grid";
import {
+  addCache,
  clearCache,
  deserializeCache,
```

Next go ahead and delete the entire `initGame` function.

```diff
-const initGame = () => {
-  // init game map and player position
-  const dungeon = createDungeon({
-    x: grid.map.x,
-    y: grid.map.y,
-    z: readCache("z"),
-    width: grid.map.width,
-    height: grid.map.height,
-  });
-
-  player = ecs.createPrefab("Player");
-  player.add(Position, {
-    x: dungeon.rooms[0].center.x,
-    y: dungeon.rooms[0].center.y,
-  });
-
-  const openTiles = Object.values(dungeon.tiles).filter(
-    (x) => x.sprite === "FLOOR"
-  );
-
-  times(5, () => {
-    const tile = sample(openTiles);
-    ecs.createPrefab("Goblin").add(Position, tile);
-  });
-
-  times(10, () => {
-    const tile = sample(openTiles);
-    ecs.createPrefab("HealthPotion").add(Position, tile);
-  });
-
-  times(10, () => {
-    const tile = sample(openTiles);
-    ecs.createPrefab("ScrollLightning").add(Position, tile);
-  });
-
-  times(10, () => {
-    const tile = sample(openTiles);
-    ecs.createPrefab("ScrollParalyze").add(Position, tile);
-  });
-
-  times(10, () => {
-    const tile = sample(openTiles);
-    ecs.createPrefab("ScrollFireball").add(Position, tile);
-  });
-
-  fov(player);
-  render(player);
-};
```

In it's place add a new function `createDungeonLevel`:

```javascript
const createDungeonLevel = ({
  createStairsUp = true,
  createStairsDown = true,
} = {}) => {
  const dungeon = createDungeon({
    x: grid.map.x,
    y: grid.map.y,
    z: readCache("z"),
    width: grid.map.width,
    height: grid.map.height,
  });

  const openTiles = Object.values(dungeon.tiles).filter(
    (x) => x.sprite === "FLOOR"
  );

  times(5, () => {
    const tile = sample(openTiles);
    ecs.createPrefab("Goblin").add(Position, tile);
  });

  times(10, () => {
    const tile = sample(openTiles);
    ecs.createPrefab("HealthPotion").add(Position, tile);
  });

  times(10, () => {
    const tile = sample(openTiles);
    ecs.createPrefab("ScrollLightning").add(Position, tile);
  });

  times(10, () => {
    const tile = sample(openTiles);
    ecs.createPrefab("ScrollParalyze").add(Position, tile);
  });

  times(10, () => {
    const tile = sample(openTiles);
    ecs.createPrefab("ScrollFireball").add(Position, tile);
  });

  let stairsUp, stairsDown;

  if (createStairsUp) {
    times(1, () => {
      const tile = sample(openTiles);
      stairsUp = ecs.createPrefab("StairsUp");
      stairsUp.add(Position, tile);
    });
  }

  if (createStairsDown) {
    times(1, () => {
      const tile = sample(openTiles);
      stairsDown = ecs.createPrefab("StairsDown");
      stairsDown.add(Position, tile);
    });
  }

  return { dungeon, stairsUp, stairsDown };
};
```

This function now contains all the dungeon creating logic that was in `initGame`. In addition to that it can optionally add stairs - we won't want another downstairs on the last level or an upstairs on the first.

Next add another function called `goToDungeonLevel`

```javascript
const goToDungeonLevel = (level) => {
  const goingUp = readCache("z") < level;
  const floor = readCache("floors")[level];

  if (floor) {
    addCache("z", level);
    player.remove(Position);
    if (goingUp) {
      player.add(Position, toCell(floor.stairsDown));
    } else {
      player.add(Position, toCell(floor.stairsUp));
    }
  } else {
    addCache("z", level);
    const { stairsUp, stairsDown } = createDungeonLevel();

    addCache(`floors.${level}`, {
      stairsUp: toLocId(stairsUp.position),
      stairsDown: toLocId(stairsDown.position),
    });

    player.remove(Position);

    if (goingUp) {
      player.add(Position, toCell(stairsDown.position));
    } else {
      player.add(Position, toCell(stairsUp.position));
    }
  }

  fov(player);
  render(player);
};
```

This function is responsible for loading a level if it already exists or creating a new one if needed. At the top of the function we store two variables `goingUp` and `floor`. The variable `floor` is used to check if the floor exists or not. If it does we check `goingUp` to decide at which stairs to place the player. If `floor` does not exist we need to create one and add the stairs to cache. Finally we run the `fov` and `render` systems.

Now we can add back the `initGame` function, far smaller this time.

```javascript
const initGame = () => {
  const { stairsDown } = createDungeonLevel({ createStairsUp: false });

  player = ecs.createPrefab("Player");

  addCache(`floors.${-1}`, {
    stairsDown: toLocId(stairsDown.position),
  });

  player.add(Position, stairsDown.position);

  fov(player);
  render(player);
};
```

The `initGame` function is now only responsible for kicking off `createDungeonLevel`, creating the player and adding some things to cache.

With our code broken into reusable functions we can now add our keybindings in to call goToDungeonLevel and get our stairs working!

```diff
if (gameState === "GAME") {
+  if (userInput === ">") {
+    if (
+      toLocId(player.position) ==
+      readCache(`floors.${readCache("z")}.stairsDown`)
+    ) {
+      addLog("You descend deeper into the dungeon");
+      goToDungeonLevel(readCache("z") - 1);
+    } else {
+      addLog("There are no stairs to descend");
+    }
+  }
+
+  if (userInput === "<") {
+    if (
+      toLocId(player.position) == readCache(`floors.${readCache("z")}.stairs`)
+    ) {
+      addLog("You climb from the depths of the dungeon");
+      goToDungeonLevel(readCache("z") + 1);
+    } else {
+      addLog("There are no stairs to climb");
+    }
+  }

  if (userInput === "ArrowUp") {
    player.add(Move, { x: 0, y: -1, z: readCache("z") });
  }
```

Before you test it out - our gameloop interprets any keypress as an input. If the key is not bound to anything it acts as a skip of a turn. For this reason we need to ignore the `shift` key in order to be able to type `<` and `>` without skipping a turn and giving the baddies an extra go.

```diff
document.addEventListener("keydown", (ev) => {
-  userInput = ev.key;
+  if (ev.key !== "Shift") {
+    userInput = ev.key;
+  }
});
```

That's it! Your stairs should now be working and properly link levels together!

Whoops! We have a bug - the previous levels are still rendering...

Fortunately this is an easy fix. In `./src/systems/render.js` we need to check the z level of entities before rendering.

```diff
  clearMap();

  layer100Entities.get().forEach((entity) => {
+    if (entity.position.z !== readCache("z")) return;
+
    if (entity.isInFov) {
      drawCell(entity);
    } else {
```

```diff
  });

  layer300Entities.get().forEach((entity) => {
+    if (entity.position.z !== readCache("z")) return;
+
    if (entity.isInFov) {
      drawCell(entity);
    } else {
```

```diff
  });

  layer400Entities.get().forEach((entity) => {
+    if (entity.position.z !== readCache("z")) return;
+
    if (entity.isInFov) {
      drawCell(entity);
    } else {
```

## A bit of UI

Now that everything is working properly, it would be nice to see what level we're at. We should also enhance our "menu" to show the new key commands for using stairs.

In `./src/systems/render.js` call drawText again at the end of the function `renderPlayerHud` to add the current "depth".

```javascript
drawText({
  text: `Depth: ${Math.abs(readCache("z"))}`,
  background: "black",
  color: "#666",
  x: grid.playerHud.x,
  y: grid.playerHud.y + 2,
});
```

Next we need to add more keybindings to our "menu". Unfortunalely we're running out of space and can't fit the text required. We could solve this a number of different ways. We could make a proper menu like Inventory to give us a lot more real estate. We could remove the need for keybindings at all and have the stairs activate when a player bumps into the them. For this tutorial we're gonna take the cheap way out and just increase the height of our game by one. That will allow us to have 2 lines for our menu.

```diff
const renderMenu = () => {
  drawText({
-    text: `(n)New (s)Save (l)Load | (i)Inventory (g)Pickup (arrow keys)Move/Attack (mouse)Look/Target`,
+    text: `(i)Inventory (g)Pickup (arrow keys)Move/Attack (mouse)Look/Target (<)Stairs Up (>)Stairs Down`,
    background: "#000",
    color: "#666",
    x: grid.menu.x,
    y: grid.menu.y,
  });
+
+  drawText({
+    text: `(n)New (s)Save (l)Load`,
+    background: "#000",
+    color: "#666",
+    x: grid.menu.x,
+    y: grid.menu.y + 1,
+  });
};
```

And in `./src/lib/canvas.js`

```diff
export const grid = {
  width: 100,
-  height: 34,
+  height: 35,
```

In this part we made it through yet another big refactor. We now have stairs in our game leading to auto generated levels that are stored in cache so we can revisit them as we ascend the dungeon.

In the next part we'll start to balance things out a little bit. Enemies will be easier in the early levels and get harder and harder as you descend.
