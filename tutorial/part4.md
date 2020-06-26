# Part 4 - Field of view

In this part we will be implementing "Field of View". We'll need to calculate all the cells our @ can see and keep track of those that have already been discovered. Through this common mechanic we get the thrill of exploration!

Before we can get to do that, we have a couple of outstanding tasks to take care of.

Have you noticed how the floor renders on top of our @? That's because our game has no concept of layers and our tiles have no backgrounds. Let's take care of both of those right now.

Let's add backgrounds to our tiles first. In `./src/lib/canvas.js` add two additional functions:

```javascript
const drawBackground = ({ color, position }) => {
  if (color === "transparent") return;

  ctx.fillStyle = color;

  ctx.fillRect(
    position.x * cellWidth,
    position.y * cellHeight,
    cellWidth,
    cellHeight
  );
};

export const drawCell = (entity, options = {}) => {
  const char = options.char || entity.appearance.char;
  const background = options.background || entity.appearance.background;
  const color = options.color || entity.appearance.color;
  const position = entity.position;

  drawBackground({ color: background, position });
  drawChar({ char, color, position });
};
```

Instead of calling `drawChar` directly we can now call `drawCell` and pass it any entity with an appearance and a position component. Our new `drawCell` function calls `drawBackground` and `drawChar` for us. We can still get a transparent background if we need to by passing the color 'transparent'. We also have an options object for overrides if needed.

No just replace `drawChar` in `./src/systems/render.js` with `drawCell` and pass the entity in directly.

```diff
import ecs from "../state/ecs";
import { Appearance, Position } from "../state/components";
-import { clearCanvas, drawChar } from "../lib/canvas";
+import { clearCanvas, drawCell } from "../lib/canvas";

const renderableEntities = ecs.createQuery({
  all: [Position, Appearance],
});

export const render = () => {
  clearCanvas();

  renderableEntities.get().forEach((entity) => {
-    const { appearance, position } = entity;
-    const { char, color } = appearance;
-
-    drawChar({ char, color, position });
+    drawCell(entity);
  });
};
```

And add a default background to our Appearance component in `./src/state/components`

```diff
export class Appearance extends Component {
  static properties = {
    color: "#ff0077",
    char: "?",
+    background: "#000",
  };
}
```

Ahhhh... I feel so much better - that's been bugging me for a while now :)

Check out the game to make sure its running smooth... and our @ has gone missing.

Html canvas draws pixels where you tell it, when you tell it. We just happen to be telling it to draw our @ before we tell it to the floor - so it draws our @ and then draws the floor directly on top. We could try and reorder our entities but that would be a huge pain to keep track of. Let's do it with a couple new components instead.

Add these components to `./src/state/components.js`

```javascript
export class Layer100 extends Component {}

export class Layer300 extends Component {}

export class Layer400 extends Component {}
```

And register them in `./src/state/ecs.js`

```diff
import { Engine } from "geotic";
import {
  Appearance,
  IsBlocking,
+  Layer100,
+  Layer300,
+  Layer400,
  Move,
  Position,
} from "./components";

const ecs = new Engine();

// all Components must be `registered` by the engine
ecs.registerComponent(Appearance);
ecs.registerComponent(IsBlocking);
+ecs.registerComponent(Layer100);
+ecs.registerComponent(Layer300);
+ecs.registerComponent(Layer400);
ecs.registerComponent(Move);
ecs.registerComponent(Position);

export const player = ecs.createEntity();
player.add(Appearance, { char: "@", color: "#fff" });
player.add(Position);

export default ecs;
```

I like to number my layers in hundreds just in case I need to squeeze something in between. 100 is for the ground, 300 for things on the ground like items, and 400 is for the player.

Next we need to add the layer components to all our entities so we now where to render what. First in `./src/lib/dungeon.js` import the `Layer100` component and add it to the floors and walls.

```diff
import {
  Appearance,
  IsBlocking,
+  Layer100,
  Position,
} from "../state/components";
```

```diff
if (tile.sprite === "WALL") {
  const entity = ecs.createEntity();
  entity.add(Appearance, { char: "#", color: "#AAA" });
  entity.add(IsBlocking);
+  entity.add(Layer100);
  entity.add(Position, dungeon.tiles[key]);
}

if (tile.sprite === "FLOOR") {
  const entity = ecs.createEntity();
  entity.add(Appearance, { char: "•", color: "#555" });
+  entity.add(Layer100);
  entity.add(Position, dungeon.tiles[key]);
}
```

And then in `./src/state/ecs.js` we need to add `Layer400` to our player entity.

```diff
export const player = ecs.createEntity();
player.add(Appearance, { char: "@", color: "#fff" });
+player.add(Layer400);
player.add(Position);

export default ecs;
```

Almost there! We need to query for each layer component so we can render everything in the correct order.

Make `./src/systems/render.js` look like this:

```javascript
import ecs from "../state/ecs";
import {
  Appearance,
  Position,
  Layer100,
  Layer300,
  Layer400,
} from "../state/components";
import { clearCanvas, drawCell } from "../lib/canvas";

const layer100Entities = ecs.createQuery({
  all: [Position, Appearance, Layer100],
});

const layer300Entities = ecs.createQuery({
  all: [Position, Appearance, Layer300],
});

const layer400Entities = ecs.createQuery({
  all: [Position, Appearance, Layer400],
});

export const render = () => {
  clearCanvas();

  layer100Entities.get().forEach((entity) => {
    drawCell(entity);
  });

  layer300Entities.get().forEach((entity) => {
    drawCell(entity);
  });

  layer400Entities.get().forEach((entity) => {
    drawCell(entity);
  });
};
```

We've removed the `renderableEntities` query in favor of queries for each layer. Then we simply render each layer in order.

Our @ has returned to the dungeon and the floor properly is beneath it's feet!

Nice work! We're starting to build a solid foundation for our game. We just have one more thing before we can add our "Field of View". Remember in the last tutorial where I talked about how we weren't going to add an `entitiesAtLocation` cache? I lied. We're totally going to add one. Right now.

First let's add a new file to store our cache with a few helper functions for accessing it. Create a file in `./src/state` called `cache.js` at `./src/state/cache.js` and make it look like this:

```javascript
export const cache = {
  entitiesAtLocation: {},
};

export const addCacheSet = (name, key, value) => {
  if (cache[name][key]) {
    cache[name][key].add(value);
  } else {
    cache[name][key] = new Set();
    cache[name][key].add(value);
  }
};

export const deleteCacheSet = (name, key, value) => {
  if (cache[name][key] && cache[name][key].has(value)) {
    cache[name][key].delete(value);
  }
};

export const readCacheSet = (name, key, value) => {
  if (cache[name][key]) {
    if (value) {
      return cache[name][key].get(value);
    }

    return cache[name][key];
  }
};

export default cache;
```

We just set up an object to store our cache and create some helper functions for basic CRUD operations. Our entitiesAtLocation cache will be an object with locId keys. LocIds are just a stringified combination of the location x and y properities (e.g, '0,1'). The value at each key will be a [Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set) of entity ids. A Set has some advantages over an array in this case. Specifically it's easier to access specific values and it's faster.

The purpose of this cache is to track what entities are in each location. So to start we should add entities to the cache if they have `Position` component. Geotic provides some lifecycle methods that can help with this. In our components file at `./src/state/components.js` we can add entities to the cache when the Position component is attached to an entity!

```diff
import { Component } from "geotic";
+import { addCacheSet } from "./cache";
```

```diff
export class Position extends Component {
  static properties = { x: 0, y: 0 };

+  onAttached() {
+    const locId = `${this.entity.position.x},${this.entity.position.y}`;
+    addCacheSet("entitiesAtLocation", locId, this.entity.id);
+  }
}
```

Next we need to update our cache when an entity moves. The simplest way for us to do this right now is in our movement system at `./src/systems/movement.js`. After all the checks to determine if an entity is able to move and right before we update their position, we can update the cache like this:

```diff
import ecs from "../state/ecs";
+import { addCacheSet, deleteCacheSet } from "../state/cache";
import { grid } from "../lib/canvas";
import { Move } from "../state/components";
```

```diff
+deleteCacheSet(
+  "entitiesAtLocation",
+  `${entity.position.x},${entity.position.y}`,
+  entity.id
+);
+addCacheSet("entitiesAtLocation", `${mx},${my}`, entity.id);

entity.position.x = mx;
entity.position.y = my;
```

We simply delete the entity id at it's previous location in cache and add it to the new one.

Ok, now that our cache is all set up, let's use it! Still in `./src/systems/movement.js` replace the current check for blockers:

```javascript
// check for blockers
const blockers = [];
for (const e of ecs.entities.all) {
  if (e.position.x === mx && e.position.y === my && e.isBlocking) {
    blockers.push(e);
  }
}
if (blockers.length) {
  entity.remove(Move);
  return;
}
```

With out new one that uses cache:

```javascript
const blockers = [];
// read from cache
const entitiesAtLoc = readCacheSet("entitiesAtLocation", `${mx},${my}`);

for (const eId of entitiesAtLoc) {
  if (ecs.getEntity(eId).isBlocking) {
    blockers.push(eId);
  }
}
if (blockers.length) {
  entity.remove(Move);
  return;
}
```

The biggest change here is that we now only have to check the entities at our intended location if they are blocking. Previously we checked the location of every entity in the entire game to determine if it was both blocking AND in the place we wanted to go. This will become a common pattern ahead as we are going to need to know what entities are in a given location a lot moving forward.

---

We are all caught up and ready to add "Field of Vision"! We are going to be using my javascript implementation of an FOV algorithm I found online by Bob Nystrom (Munificent). He wrote the online roguelike [Hauberk](https://munificent.github.io/hauberk/) and a fantastic book called [Game Programming Patterns](http://gameprogrammingpatterns.com/). I'll be honest and say I don't remember exactly how this algorithm works. Hooking it up is the important bit for our purposes. If you want to understand it fully, there is an exhaustive [blog post about it here](http://journal.stuffwithstuff.com/2015/09/07/what-the-hero-sees/) where Bob explains far better than I could.

Ok, first let's go ahead and add a new file called `fov.js` to our lib directory at `./src/lib/fov.js`. Go ahead and paste this code into it:

```javascript
import { distance, idToCell } from "./grid";

const octantTransforms = [
  { xx: 1, xy: 0, yx: 0, yy: 1 },
  { xx: 1, xy: 0, yx: 0, yy: -1 },
  { xx: -1, xy: 0, yx: 0, yy: 1 },
  { xx: -1, xy: 0, yx: 0, yy: -1 },
  { xx: 0, xy: 1, yx: 1, yy: 0 },
  { xx: 0, xy: 1, yx: -1, yy: 0 },
  { xx: 0, xy: -1, yx: 1, yy: 0 },
  { xx: 0, xy: -1, yx: -1, yy: 0 },
];

// width: width of map (or visible map?)
// height: height of map (or visible map?)
export default function createFOV(
  opaqueEntities,
  width,
  height,
  originX,
  originY,
  radius
) {
  const visible = new Set();

  const blockingLocations = new Set();
  opaqueEntities
    .get()
    .forEach((x) => blockingLocations.add(`${x.position.x},${x.position.y}`));

  const isOpaque = (x, y) => {
    const locId = `${x},${y}`;
    return !!blockingLocations.has(locId);
  };
  const reveal = (x, y) => {
    return visible.add(`${x},${y}`);
  };

  function castShadows(originX, originY, row, start, end, transform, radius) {
    let newStart = 0;
    if (start < end) return;

    let blocked = false;

    for (let distance = row; distance < radius && !blocked; distance++) {
      let deltaY = -distance;
      for (let deltaX = -distance; deltaX <= 0; deltaX++) {
        let currentX = originX + deltaX * transform.xx + deltaY * transform.xy;
        let currentY = originY + deltaX * transform.yx + deltaY * transform.yy;

        let leftSlope = (deltaX - 0.5) / (deltaY + 0.5);
        let rightSlope = (deltaX + 0.5) / (deltaY - 0.5);

        if (
          !(
            currentX >= 0 &&
            currentY >= 0 &&
            currentX < width &&
            currentY < height
          ) ||
          start < rightSlope
        ) {
          continue;
        } else if (end > leftSlope) {
          break;
        }

        if (Math.sqrt(deltaX * deltaX + deltaY * deltaY) <= radius) {
          reveal(currentX, currentY);
        }

        if (blocked) {
          if (isOpaque(currentX, currentY)) {
            newStart = rightSlope;
            continue;
          } else {
            blocked = false;
            start = newStart;
          }
        } else {
          if (isOpaque(currentX, currentY) && distance < radius) {
            blocked = true;
            castShadows(
              originX,
              originY,
              distance + 1,
              start,
              leftSlope,
              transform,
              radius
            );
            newStart = rightSlope;
          }
        }
      }
    }
  }

  reveal(originX, originY);
  for (let octant of octantTransforms) {
    castShadows(originX, originY, 1, 1, 0, octant, radius);
  }

  return {
    fov: visible,
    distance: [...visible].reduce((acc, val) => {
      const cell = idToCell(val);
      acc[val] = distance({ x: originX, y: originY }, { x: cell.x, y: cell.y });
      return acc;
    }, {}),
  };
}
```

Now that we have our Field of Vision algorithm in place we need to wire it up. We'll start with the system this time. Create a new file, again called `fov.js` but this time the systems directory at `./src/systems/fov.js`. It should look like this:

```javascript
import { readCacheSet } from "../state/cache";
import ecs, { player } from "../state/ecs";
import { grid } from "../lib/canvas";
import createFOV from "../lib/fov";
import { IsInFov, IsOpaque, IsRevealed } from "../state/components";

const inFovEntities = ecs.createQuery({
  all: [IsInFov],
});

const opaqueEntities = ecs.createQuery({
  all: [IsOpaque],
});

export const fov = () => {
  const { width, height } = grid;

  const originX = player.position.x;
  const originY = player.position.y;

  const FOV = createFOV(opaqueEntities, width, height, originX, originY, 10);

  // clear out stale fov
  inFovEntities.get().forEach((x) => x.remove(IsInFov));

  FOV.fov.forEach((locId) => {
    const entitiesAtLoc = readCacheSet("entitiesAtLocation", locId);

    if (entitiesAtLoc) {
      entitiesAtLoc.forEach((eId) => {
        const entity = ecs.getEntity(eId);
        entity.add(IsInFov);

        if (!entity.has("IsRevealed")) {
          entity.add(IsRevealed);
        }
      });
    }
  });
};
```

You may notice the new components we imported at the top - we'll create those next. Let's go over what this system is doing first.

The first thing the system does is gather some data to pass into `createFOV`. The last argument to createFOV is the visual range of our hero. Adjust accordingly.

```javascript
export const fov = () => {
  const { width, height } = grid;

  const originX = player.position.x;
  const originY = player.position.y;

  const FOV = createFOV(opaqueEntities, width, height, originX, originY, 10);
```

Next remove the component `IsInFov` from all entities. This clears out all the state from the last turn ensuring that we always have the latest data. Our FOV algorithm returns an array of locations within our hero's field of view. We need to find all the entities at each location and add an IsInFov component. If an entity has never been revealed we will add an IsRevealed component as well. This is why we created a cache earlier. Having to iterate through every entity in the game for every tile in FOV every turn... ugh. That would be bad.

```javascript
  // clear out stale fov
  inFovEntities.get().forEach((x) => x.remove(IsInFov));

  FOV.fov.forEach((locId) => {
    const entitiesAtLoc = readCacheSet("entitiesAtLocation", locId);

    if (entitiesAtLoc) {
      entitiesAtLoc.forEach((eId) => {
        const entity = ecs.getEntity(eId);
        entity.add(IsInFov);

        if (!entity.has("IsRevealed")) {
          entity.add(IsRevealed);
        }
      });
    }
  });
};
```

We should probably create those components we're referencing. In `./src/state/components.js` add the following:

```javascript
export class IsInFov extends Component {}

export class IsOpaque extends Component {}

export class IsRevealed extends Component {}
```

And register them in `./src/state/ecs.js`

```diff
import {
  Appearance,
  IsBlocking,
+  IsInFov,
+  IsOpaque,
+  IsRevealed,
  Layer100,
  Layer400,
  Move,
  Position,
} from "./components";

const ecs = new Engine();

// all Components must be `registered` by the engine
ecs.registerComponent(Appearance);
ecs.registerComponent(IsBlocking);
+ecs.registerComponent(IsInFov);
+ecs.registerComponent(IsOpaque);
+ecs.registerComponent(IsRevealed);
ecs.registerComponent(Layer100);
ecs.registerComponent(Layer400);
ecs.registerComponent(Move);
ecs.registerComponent(Position);
```
