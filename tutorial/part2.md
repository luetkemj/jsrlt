# Part 2 - The generic Entity, the render functions, and the map

For this tutorial we will be using an ECS architecture. A lot of folks will probably say this is overkill for the size of our project but I have found it to be the easiest architecture to reason about once games reach anything beyond basic complexity. If you choose to expand on this project after the tutorial you'll have a decent base to do so. We won't be writing our own ECS engine, we will instead be relying on the excellent [geotic](https://github.com/ddmills/geotic) for that. But before we install anything and start refactoring our game we should probably talk a bit about what an ECS architecture is and why you would choose to use one.

To state the obvious, games are complicated! My first 4 or 5 attempts experimented with all sorts of ways to manage state. Every one of those games started simple and eventually fell apart when adding new features became too complex.

You shouldn't have to write complex code to do complex things. With ECS, complexity arises from simplicity. Follow a few simple rules; get complex behavior.

For a detailed overview of ECS in practice Thomas Biskup (ADOM) gave a great talk at the 2018 Roguelike Celebration. [There be dragons: Entity Component Systems for Roguelikes](https://www.youtube.com/watch?v=fGLJC5UY2o4&feature=youtu.be)

For a formal and rather dry definition we can turn to wikipedia:

> ECS follows the composition over inheritance principle that allows greater flexibility in defining entities where every object in a game's scene is an entity (e.g. enemies, bullets, vehicles, etc.). Every entity consists of one or more components which contains data or state. Therefore, the behavior of an entity can be changed at runtime by systems that add, remove or mutate components. This eliminates the ambiguity problems of deep and wide inheritance hierarchies that are difficult to understand, maintain and extend. - [wikipedia](https://en.wikipedia.org/wiki/Entity_component_system)

At it's core ECS is just a way to manage your application state. State is stored in components, entities are collections of those components, and systems run logic on those entities in order to add, remove, and mutate their components.

As our game grows in scope I think you will find that these 3 simple rules will help to manage the underlying complexity of it all leaving us to follow our inspiration and just make a game!

Enough talk, let's install geotic, create our first entity and learn how to do things the ECS way!

---

Before we get into it, go ahead and install geotic and start the engine.

```bash
npm install geotic
```

Next let's create a new folder `./src/state` and add a file called `ecs.js` at `./src/state/ecs.js`. Now we can import the Engine from geotic and start it up.

```javascript
import { Engine } from "geotic";

const ecs = new Engine();

export default ecs;
```

Currently we store all of the state of our hero in the player object. To move around we directly mutate it's position values. Right now everything is very simple and easy to understand. Unfortunately this pattern won't scale very well if we were to add an NPC, a few monsters, maybe an item or two... mutating state directly like this very quickly becomes cumbersome, complicated, and prone to bugs that are hard to diagnose. Not to mention saving and loading... as our game scales up we would have to figure out how to build and rebuild state for every single piece.

We're going to refactor our game to do everything it already does - draw the @ symbol and move it around - the ECS way. At first it's going to seem like a lot of code. The benefit though is that as we add NPCs, monsters, and items, the complexity of our code won't explode.

To start let's look at our player object and see how we can translate it to components:

```javascript
const player = {
  char: "@",
  color: "white",
  position: {
    x: 0,
    y: 0,
  },
};
```

Components are just containers used to store bits of state. Our player object is only concerned with two things so far, appearance (char, color) and position. Let's to compoenents to track these bits of state, Appearance and Position. A generic components we will be able to use them not just for our player but also for goblins, items, walls, anything we can see and pin to a specific location!

First create a file to hold our components called `components.js` at `./src/state/components.js`. In a larger application you may want to create individual files for each component but this is fine for our purposes.

Make `./src/state/components.js` look like this:

```javascript
import { Component } from "geotic";

export class Appearance extends Component {
  static properties = {
    color: "#ff0077",
    char: "?",
  };
}

export class Position extends Component {
  static properties = { x: 0, y: 0 };
}
```

This is a pretty simple file so far. We just define two components that each set some default static properties. A lot of our components will be very small like these, a few might be a bit bigger but most will be even smaller. Components are _supposed_ to be simple!

Before we make our first Entity we need to remember to register our new components with geotic. We will do that in `./src/state/ecs.js`.

```diff
import { Engine } from "geotic";
+import { Appearance, Position } from "./components";

const ecs = new Engine();

+// all Components must be `registered` by the engine
+ecs.registerComponent(Appearance);
+ecs.registerComponent(Position);

export default ecs;
```

We're ready to make our first Entity!

Just below where we register our components in `./src/state/ecs.js` we can create an empty entity for our player and then add our components to it.

```diff
ecs.registerComponent(Position);

+const player = ecs.createEntity();
+player.add(Appearance, { char: "@", color: "#fff" });
+player.add(Position);

export default ecs;
```

The add method takes two arguments, a component, and a properties object to override any of the static defaults. You'll notice we don't pass a second argument when we add the Position component because the default properties are just fine.

We now have the (E)ntity and (C)omponent parts of ECS but what about the (S)ystem? We've seen how geotic provides classes for entities and components but it doesn't do that for systems. A system is just a function that iterates over a collection of entities. You can do whatever you want/need to in one but how you implement it is entirely up to you.

Our first system will render our player entity to the screen. Let's create a new folder `./src/systems` and add a file called `render.js` at `./src/systems/render.js`. We could have our systems iterate over every single entity in the game at every tick but as you can imagine that's gonna get pretty inefficient as we add more systems. We can instead narrow our focus with geotic queries. A query is an always up to date set of entities that have a specified set of components.

Make `./src/systems/render.js` look like this:

```javascript
import ecs from "../state/ecs";
import { Appearance, Position } from "../state/components";

const renderableEntities = ecs.createQuery({
  all: [Position, Appearance],
});
```

`renderableEntities` will keep track of all entities that contain both the Position _and_ Appearance components. Let's use this query to loop through all renderableEntities and log the result to our javascript console. Go ahead and add the actual system at the end of `./src/systems/render.js`.

```javascript
export const render = () => {
  renderableEntities.get().forEach((entity) => {
    console.log(entity);
  });
};
```

Next we need to actually call our render system. At the top of `./src/index.js` import the system like this:

```diff
import { clearCanvas, drawChar } from "./lib/canvas";
+import { render } from "./systems/render";

const player = {
```

Then at the end of the file we can call it like this:

```diff
  clearCanvas();
  drawChar(player);
+ render();
};
```

Start the game with `npm start` if it's not already running and open up your browser's javascript console. Now every time you hit a key you should see all renderable entities logged to the console. There's only the one so far but this sort of thing is a good habit to get into just to prove that things are working as expected.

Now that our ECS engine is firing on all cylinders it's time to finally make it do something useful. Let's make our render system actually render!

Instead of logging each entity from the system we can use our drawChar function to draw them instead. We need to add a few things to `./src/systems/render.js` to do that.

```diff
import ecs from "../state/ecs";
import { Appearance, Position } from "../state/components";
+import { clearCanvas, drawChar } from "../lib/canvas";

const renderableEntities = ecs.createQuery({
  all: [Position, Appearance],
});

export const render = () => {
+  clearCanvas()
+
  renderableEntities.get().forEach((entity) => {
-    console.log(entity);
+    const { appearance, position } = entity;
+    const { char, color } = appearance;

+    drawChar({ char, color, position });
  });
};
```

Next we can go clean up a couple things in `./src/index.js`. We can remove the player object now that we have are storing the player as an entity. And we also don't need to call drawChar or clearCanvas anymore.

```diff
import "./lib/canvas.js";
-import { clearCanvas, drawChar } from "./lib/canvas";
import { render } from "./systems/render";

-const player = {
-  char: "@",
-  color: "white",
-  position: {
-    x: 0,
-    y: 0,
-  },
-};

-drawChar(player);
+render();

let userInput = null;

document.addEventListener("keydown", (ev) => {
  userInput = ev.key;
  processUserInput();
});

const processUserInput = () => {
  if (userInput === "ArrowUp") {
    player.position.y -= 1;
  }
  if (userInput === "ArrowRight") {
    player.position.x += 1;
  }
  if (userInput === "ArrowDown") {
    player.position.y += 1;
  }
  if (userInput === "ArrowLeft") {
    player.position.x -= 1;
  }

- clearCanvas();
- drawChar(player);
  render();
};

```

Ok - if we try to run the game now we should see our @ symbol in the top left but it no longer moves. If you look in the javascriopt console, you'll see it's lit up with errors. We deleted our player object but still reference it in processUserInput. We need to think about how to process user input the ECS way.

Moving an entity from one position to another is fraught with peril. What if there is a wall, or a trap, or a monster, or the entity is paralyzed, or mind controlled... What we would like to have is a generic way to let our game know where an entity intends to move, and then let the our game resolve what actually happens. To do this we will be adding an additional component and system.

Lets start by adding another component to `./src/state/components`. The order here doesn't really matter, I just like to keep them in alphabetical. :P

```diff
import { Component } from "geotic";

export class Appearance extends Component {
  static properties = {
    color: "#ff0077",
    char: "?",
  };
}

+export class Move extends Component {
+  static properties = { x: 0, y: 0 }
+}

export class Position extends Component {
  static properties = { x: 0, y: 0 };
}

```

Don't forget to register it in `./src/state/ecs`!

```diff
import { Engine } from "geotic";
-import { Appearance, Position } from "./components";
+import { Appearance, Move, Position } from "./components";

const ecs = new Engine();

// all Components must be `registered` by the engine
ecs.registerComponent(Appearance);
+ecs.registerComponent(Move);
ecs.registerComponent(Position);

const player = ecs.createEntity();

player.add(Appearance, { char: "@", color: "#fff" });
player.add(Position);

export default ecs;
```

Alright! Now we need to add this component to the player entity in processUserInput. To do that we first need to export the player entity from './src/state/ecs`

```diff
ecs.registerComponent(Position);

-const player = ecs.createEntity();
+export const player = ecs.createEntity();

player.add(Appearance, { char: "@", color: "#fff" });
```

Now instead of directly manipulating the player entity we will add a Move component to it and handle the logic of actually moving in a system. Make these changes to `./src/index.js`.

```diff
import "./lib/canvas.js";
import { render } from "./systems/render";
+import { player } from "./state/ecs";
+import { Move } from "./state/components";

render();

let userInput = null;

document.addEventListener("keydown", (ev) => {
  userInput = ev.key;
  processUserInput();
});

const processUserInput = () => {
  if (userInput === "ArrowUp") {
-    player.position.y -= 1;
+    player.add(Move, { x: 0, y: -1 });
  }
  if (userInput === "ArrowRight") {
-    player.position.x += 1;
+    player.add(Move, { x: 1, y: 0 });
  }
  if (userInput === "ArrowDown") {
-    player.position.y += 1;
+    player.add(Move, { x: 0, y: 1 });
  }
  if (userInput === "ArrowLeft") {
-    player.position.x -= 1;
+    player.add(Move, { x: -1, y: 0 });
  }

  render();
};
```

Almost there - we just need add our system. Create a new file called `movement.js` at `./src/systems/movement.js`. It should look like this:

```javascript
import ecs from "../state/ecs";
import { Move } from "../state/components";

const movableEntities = ecs.createQuery({
  all: [Move],
});

export const movement = () => {
  movableEntities.get().forEach((entity) => {
    const mx = entity.position.x + entity.move.x;
    const my = entity.position.y + entity.move.y;

    // this is where we will run any checks to see if entity can move to new location

    entity.position.x = mx;
    entity.position.y = my;

    entity.remove(Move);
  });
};
```

Just like in our render system we create a query at the top so we only have to loop over entities that actually intend to move. We then calculate the actual position the entity is trying to enter and update the entity with that new position. You can see where we will eventually check for walls, traps, monsters, whatever.

The last thing we have to do is import our movement system and call it in './src/index.js'

```diff
import "./lib/canvas.js";
+import { movement } from "./systems/movement";
import { render } from "./systems/render";
import { player } from "./state/ecs";
import { Move } from "./state/components";

render();

let userInput = null;

document.addEventListener("keydown", (ev) => {
  userInput = ev.key;
  processUserInput();
});

const processUserInput = () => {
  if (userInput === "ArrowUp") {
    player.add(Move, { x: 0, y: -1 });
  }
  if (userInput === "ArrowRight") {
    player.add(Move, { x: 1, y: 0 });
  }
  if (userInput === "ArrowDown") {
    player.add(Move, { x: 0, y: 1 });
  }
  if (userInput === "ArrowLeft") {
    player.add(Move, { x: -1, y: 0 });
  }

+  movement();
  render();
};
```

We're finally right back where we started! Your @ can move again!

---

OK, that was a lot to get through for the same result I know, but it'll be worth in the end. We have one more thing to do before we're done with part 2. We need a map to walk on. This will be fun because we'll get to actually flex our systems a bit and use all that work we just did!

To start let's create another file in `./src/lib` called `grid.js` at `./src/lib/grid.js`. It going to contain a bunch of utility functions for dealing with math on a square grid. Most of the functions here are javascript implementations based on the pseudocode from [redblobgames](https://www.redblobgames.com/). I'm not going to go over any of the logic in this file. If you're curious how these functions work I highly encourage you to read the articles on redblobgames. In fact, just bookmark it now. **It is an amazing resource**.

OK, go ahead and just paste this into `./src/lib/grid.js`:

```javascript
import { grid } from "../lib/canvas";
import { sample } from "lodash";

export const CARDINAL = [
  { x: 0, y: -1 }, // N
  { x: 1, y: 0 }, // E
  { x: 0, y: 1 }, // S
  { x: -1, y: 0 }, // W
];

export const DIAGONAL = [
  { x: 1, y: -1 }, // NE
  { x: 1, y: 1 }, // SE
  { x: -1, y: 1 }, // SW
  { x: -1, y: -1 }, // NW
];

export const ALL = [...CARDINAL, ...DIAGONAL];

export const toCell = (cellOrId) => {
  let cell = cellOrId;
  if (typeof cell === "string") cell = idToCell(cell);

  return cell;
};

export const toLocId = (cellOrId) => {
  let locId = cellOrId;
  if (typeof locId !== "string") locId = cellToId(locId);

  return locId;
};

const insideCircle = (center, tile, radius) => {
  const dx = center.x - tile.x;
  const dy = center.y - tile.y;
  const distance_squared = dx * dx + dy * dy;
  return distance_squared <= radius * radius;
};

export const circle = (center, radius) => {
  const diameter = radius % 1 ? radius * 2 : radius * 2 + 1;
  const top = center.y - radius;
  const bottom = center.y + radius;
  const left = center.x - radius;
  const right = center.x + radius;

  const locsIds = [];

  for (let y = top; y <= bottom; y++) {
    for (let x = left; x <= right; x++) {
      const cx = Math.ceil(x);
      const cy = Math.ceil(y);
      if (insideCircle(center, { x: cx, y: cy }, radius)) {
        locsIds.push(`${cx},${cy}`);
      }
    }
  }

  return locsIds;
};

export const rectangle = ({ x, y, width, height, hasWalls }, tileProps) => {
  const tiles = {};

  const x1 = x;
  const x2 = x + width - 1;
  const y1 = y;
  const y2 = y + height - 1;
  if (hasWalls) {
    for (let yi = y1 + 1; yi <= y2 - 1; yi++) {
      for (let xi = x1 + 1; xi <= x2 - 1; xi++) {
        tiles[`${xi},${yi}`] = { x: xi, y: yi, ...tileProps };
      }
    }
  } else {
    for (let yi = y1; yi <= y2; yi++) {
      for (let xi = x1; xi <= x2; xi++) {
        tiles[`${xi},${yi}`] = { x: xi, y: yi, ...tileProps };
      }
    }
  }

  const center = {
    x: Math.round((x1 + x2) / 2),
    y: Math.round((y1 + y2) / 2),
  };

  return { x1, x2, y1, y2, center, hasWalls, tiles };
};

export const rectsIntersect = (rect1, rect2) => {
  return (
    rect1.x1 <= rect2.x2 &&
    rect1.x2 >= rect2.x1 &&
    rect1.y1 <= rect2.y2 &&
    rect1.y2 >= rect2.y1
  );
};

export const distance = (cell1, cell2) => {
  const x = Math.pow(cell2.x - cell1.x, 2);
  const y = Math.pow(cell2.y - cell1.y, 2);
  return Math.floor(Math.sqrt(x + y));
};

export const idToCell = (id) => {
  const coords = id.split(",");
  return { x: parseInt(coords[0], 10), y: parseInt(coords[1], 10) };
};

export const cellToId = ({ x, y }) => `${x},${y}`;

export const isOnMapEdge = (x, y) => {
  const { width, height, x: mapX, y: mapY } = grid.map;

  if (x === mapX) return true; // west edge
  if (y === mapY) return true; // north edge
  if (x === mapX + width - 1) return true; // east edge
  if (y === mapY + height - 1) return true; // south edge
  return false;
};

export const getNeighbors = ({ x, y }, direction = CARDINAL) => {
  const points = [];
  for (let dir of direction) {
    let candidate = {
      x: x + dir.x,
      y: y + dir.y,
    };
    if (
      candidate.x >= 0 &&
      candidate.x < grid.width &&
      candidate.y >= 0 &&
      candidate.y < grid.height
    ) {
      points.push(candidate);
    }
  }
  return points;
};

export const getNeighborIds = (cellOrId, direction = "CARDINAL") => {
  let cell = toCell(cellOrId);

  if (direction === "CARDINAL") {
    return getNeighbors(cell, CARDINAL).map(cellToId);
  }

  if (direction === "DIAGONAL") {
    return getNeighbors(cell, DIAGONAL).map(cellToId);
  }

  if (direction === "ALL") {
    return [
      ...getNeighbors(cell, CARDINAL).map(cellToId),
      ...getNeighbors(cell, DIAGONAL).map(cellToId),
    ];
  }
};

export const isNeighbor = (a, b) => {
  let posA = a;
  if (typeof posA === "string") {
    posA = idToCell(a);
  }

  let posB = b;
  if (typeof posB === "string") {
    posB = idToCell(b);
  }

  const { x: ax, y: ay } = posA;
  const { x: bx, y: by } = posB;

  if (
    (ax - bx === 1 && ay - by === 0) ||
    (ax - bx === 0 && ay - by === -1) ||
    (ax - bx === -1 && ay - by === 0) ||
    (ax - bx === 0 && ay - by === 1)
  ) {
    return true;
  }

  return false;
};

export const randomNeighbor = (startX, startY) => {
  const direction = sample(CARDINAL);
  const x = startX + direction.x;
  const y = startY + direction.y;
  return { x, y };
};

export const getNeighbor = (x, y, dir) => {
  const dirMap = { N: 0, E: 1, S: 2, W: 3 };
  const direction = CARDINAL[dirMap[dir]];
  return {
    x: x + direction.x,
    y: y + direction.y,
  };
};

export const getDirection = (a, b) => {
  const cellA = toCell(a);
  const cellB = toCell(b);

  const { x: ax, y: ay } = cellA;
  const { x: bx, y: by } = cellB;

  let dir;

  if (ax - bx === 1 && ay - by === 0) dir = "→";
  if (ax - bx === 0 && ay - by === -1) dir = "↑";
  if (ax - bx === -1 && ay - by === 0) dir = "←";
  if (ax - bx === 0 && ay - by === 1) dir = "↓";

  return dir;
};
```

Now that that's out of the way let's make a big rectangle to walk around on.

First add some dimensions for our map to the grid config in `./src/lib/canvas.js`

```diff
export const grid = {
  width: 100,
  height: 34,
+
+  map: {
+    width: 79,
+    height: 29,
+    x: 21,
+    y: 3,
+  },
};
```

Now create another file called `dungeon.js` in our lib folder at `./src/lib/dungeon.js` and make it look like this:

```javascript
import ecs from "../state/ecs";
import { rectangle } from "./grid";
import { grid } from "./canvas";

import { Appearance, Position } from "../state/components";

export const createDungeon = () => {
  const dungeon = rectangle(grid.map);
  Object.keys(dungeon.tiles).forEach((key) => {
    const tile = ecs.createEntity();
    tile.add(Appearance, { char: "•", color: "#555" });
    tile.add(Position, dungeon.tiles[key]);
  });

  return dungeon;
};
```

This createDungeon function will eventually create a dungeon but for now we'll take what we can get.

To start, we use the rectangle function from our grid library.

```javascript
const dungeon = rectangle(grid.map);
```

It generates a bunch of different stuff to kick off the start of our dungeon but among them is an object containing all the tile locations. That object is at `dungeon.tiles` and looks something like this:

```javascript
{
  '0,0': {x:0, y:0},
  '0,1': {x:0, y:1},
  '0,2': {x:0, y:2}
}
```

Next we use the builtin Object.keys method to iterate over the object and create an entity with Appearance and Position components for every single tile.

```javascript
Object.keys(dungeon.tiles).forEach((key) => {
  const tile = ecs.createEntity();
  tile.add(Appearance, { char: "•", color: "#555" });
  tile.add(Position, dungeon.tiles[key]);
});
```

Finally we need to call createDungeon as the game initializes. In `./src/index.js` make these changes right at the top.

```diff
import "./lib/canvas.js";
+import { createDungeon } from "./lib/dungeon";
import { movement } from "./systems/movement";
import { render } from "./systems/render";
import { player } from "./state/ecs";
import { Move } from "./state/components";

+// init game map and player position
+const dungeon = createDungeon();
+player.position.x = dungeon.center.x;
+player.position.y = dungeon.center.y;
+
render();
```

See how we create the "dungeon", then use the center location to set our player's intital starting position.

If you check out the game now you should see a large grid of dots. Notice how you didn't have to do anything extra to render those dots - the render system took care of it for you because each tile has the required components in the renderableEntities query. Cool!

One problem you may notice is that nothing stops the player from walking right off the edge of the map. We can handle that in our movement system. We just need to make a quick check that the goal location from an entities move component is within our map's boundaries. Go ahead and make the following changes to `./src/systems/movement`

```diff
import ecs from "../state/ecs";
+import { grid } from "../lib/canvas";
import { Move } from "../state/components";

const movableEntities = ecs.createQuery({
  all: [Move],
});

export const movement = () => {
  movableEntities.get().forEach((entity) => {
-    const mx = entity.position.x + entity.move.x;
-    const my = entity.position.y + entity.move.y;
+    let mx = entity.position.x + entity.move.x;
+    let my = entity.position.y + entity.move.y;

    // this is where we will run any checks to see if entity can move to new location
+    // observe map boundaries
+    mx = Math.min(grid.map.width + grid.map.x - 1, Math.max(21, mx));
+    my = Math.min(grid.map.height + grid.map.y - 1, Math.max(3, my));

    entity.position.x = mx;
    entity.position.y = my;

    entity.remove(Move);
  });
};
```

Try the game again - there is now an invisible boundary at the edge of the map!

Good job and congratulations for making it this far! That was a lot to get through.

In part 2 we learned what an ECS architecture is and why you might choose to use one. We created our first components, entities, and systems to render an "@" on the dungeon floor and move it around!

In [part 3](https://github.com/luetkemj/jsrlt/blob/master/tutorial/part3.md) we'll revisit createDungeon and build an actual environment to walk around in.

See you there!
