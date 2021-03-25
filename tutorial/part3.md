# Part 3 - Generating a dungeon

In part 3 we get to tackle one of the most import aspects of building a roguelike: Creating a procedurally generated dungeon!

We're going to be using a few more functions from the grid library and as mentioned previously we're not going to go over that logic in detail. I'd like instead to go step by step through the broad strokes of dungeon generation. Hopefully by the end of this you will have a feel for where you might be able to tweak things here or there or even add entirely new pieces to make this dungeon your own.

Remember our `createDungeon` function from the last tutorial? Let's gut it so we can build it back up to actually make a dungeon!

Like any good dungeon we will be digging ours out of the solid rock. Let's start by filling in the map with walls.

In `./src/lib/dungeon.js` we can delete our import from `./canvas`

```diff
import world from "../state/ecs";
import { rectangle } from "./grid";
-import { grid } from "./canvas";
```

and replace the entire createDungeon function with this new version:

```javascript
export const createDungeon = ({ x, y, width, height }) => {
  // fill the entire space with walls so we can dig it out later
  const dungeon = rectangle(
    { x, y, width, height },
    {
      sprite: "WALL",
    }
  );

  // create tile entities
  Object.keys(dungeon.tiles).forEach((key) => {
    const tile = dungeon.tiles[key];

    if (tile.sprite === "WALL") {
      const entity = world.createEntity();
      entity.add(Appearance, { char: "#", color: "#555" });
      entity.add(Position, dungeon.tiles[key]);
    }
  });

  return dungeon;
};
```

So we're doing a couple things here. First we're now passing an options object into createDungeon. Next we create our rectangle like before but we pass in an extra argument. The rectangle function allows you to pass an additional object that will get merged with each tile it outputs. This allows us to add additional data for use when we create our entities.

Next as we iterate through our dungeon tiles we make use of that extra data. If a tile has a sprite property that is equal to "WALL" we create an entity and add the appropriate components.

Before we can test this we'll need to pass in an options object in `./src/index.js`. Go ahead and make these changes to that file:

```diff
import "./lib/canvas.js";
+import { grid } from "./lib/canvas";
import { createDungeon } from "./lib/dungeon";
import { movement } from "./systems/movement";
import { render } from "./systems/render";
import { player } from "./state/ecs";
import { Move } from "./state/components";

// init game map and player position
-const dungeon = createDungeon()
+const dungeon = createDungeon({
+  x: grid.map.x,
+  y: grid.map.y,
+  width: grid.map.width,
+  height: grid.map.height,
+});
player.position.x = dungeon.center.x;
player.position.y = dungeon.center.y;
```

Go ahead and run the game. You should see a big grid of walls. You may also notice that our @ can walk right through them like a ghost - we'll fix that later. For now, let make our first room!

Back in `./src/lib/dungeon`:

```diff
import world from "../state/ecs";
import { rectangle } from "./grid";

import { Appearance, Position } from "../state/components";

export const createDungeon = ({ x, y, width, height }) => {
  // fill the entire space with walls so we can dig it out later
  const dungeon = rectangle(
    { x, y, width, height },
    {
      sprite: "WALL",
    }
  );

+  const room = rectangle(
+    { x: 30, y: 10, width: 10, height: 10, hasWalls: true },
+    { sprite: "FLOOR" }
+  );
+
+  dungeon.tiles = { ...dungeon.tiles, ...room.tiles };

  // create tile entities
  Object.keys(dungeon.tiles).forEach((key) => {
    const tile = dungeon.tiles[key];

    if (tile.sprite === "WALL") {
      const entity = world.createEntity();
      entity.add(Appearance, { char: "#", color: "#AAA" });
      entity.add(Position, dungeon.tiles[key]);
    }

+    if (tile.sprite === "FLOOR") {
+      const entity = world.createEntity();
+      entity.add(Appearance, { char: "•", color: "#555" });
+      entity.add(Position, dungeon.tiles[key]);
+    }
  });

  return dungeon;
};
```

Let's go over these changes:

We use rectangle again to create a room within our map. This time setting the sprite property on our extra data to "FLOOR".

```javascript
const room = rectangle(
  { x: 30, y: 10, width: 10, height: 10, hasWalls: true },
  { sprite: "FLOOR" }
);
```

Next we merge our dungeon tiles with our room tiles. The `...` is called the [spread operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax) and allows us to clone dungeon.tiles and merge it with room.tiles in one statement.

```javascript
dungeon.tiles = { ...dungeon.tiles, ...room.tiles };
```

Finally we add another conditional statement to handle floor tiles.

```javascript
if (tile.sprite === "FLOOR") {
  const entity = world.createEntity();
  entity.add(Appearance, { char: "•", color: "#555" });
  entity.add(Position, dungeon.tiles[key]);
}
```

Your game should now have a room dug out of the rock!

Now let's put the room in a random location every time we start the game.

[Lodash](https://lodash.com/) is a great utility library for javascript that ends up in most of my projects at some point. It has a simple random number generator we'll use. You're welcome to use a more robust rng with support for seeds if you want but we won't be covering anything like that in this tutorial.

In your terminal of choice from your projects root directory go ahead and install lodash:

```bash
npm install lodash
```

Alright, lets go ahead and import the random function from lodash at the top of `./src/lib/dungeon.js`.

```diff
+import { random } from "lodash";
import world from "../state/ecs";
import { rectangle } from "./grid";
```

Next we need a couple more options for our dungeon. Add minRoomSize and maxRoomSize with some defaults to our options object.

```diff
export const createDungeon = ({
  x,
  y,
  width,
  height,
+  minRoomSize = 6,
+  maxRoomSize = 12,
}) => {
```

Next we're going to store some variables for random width, height, x, and y.

```diff
  const dungeon = rectangle(
    { x, y, width, height },
    {
      sprite: "WALL",
    }
  );

+  let rw = random(minRoomSize, maxRoomSize);
+  let rh = random(minRoomSize, maxRoomSize);
+  let rx = random(x, width + x - rw);
+  let ry = random(y, height + y - rh);

  const room = rectangle(
```

Lodash's random function returns a number between the inclusive lower and upper bounds. Our job here is to provide reasonable boundaries so our rooms are within the map.

Width (rw) and height (rh) are easy, we can just use the min and max room size options we just added. The x (rx) and y (ry) are a little bit tricker. We have to subtract our random width and height from the far right and bottom locations on our grid.

Finally we just need to pass these randomized values into the rectangle function for creating rooms.

```diff
const room = rectangle(
-  { x: 30, y: 10, width: 10, height: 10, hasWalls: true },
+  { x: rx, y: ry, width: rw, height: rh, hasWalls: true },
  { sprite: "FLOOR" }
);
```

Now your game should place a randomly sized room in a random location on your map. We're getting closer to a procedurally generated dungeon!

Of course we're gonna need more than one room to excite our players. Let's make some changes to our algorithm to add more.

We need to import a new function from lodash called `times`. Times is a great little utility for running another function multiple times. We also need to add another option for createDungeon - maxRoomCount with a default of 30.

We'll go over the diff in a second but with our new code to generate multiple rooms `./src/lib/dungeon.js` should now look like this:

```javascript
import { random, times } from "lodash";
import world from "../state/ecs";
import { rectangle, rectsIntersect } from "./grid";
import { Appearance, Position } from "../state/components";

export const createDungeon = ({
  x,
  y,
  width,
  height,
  minRoomSize = 6,
  maxRoomSize = 12,
  maxRoomCount = 30,
}) => {
  // fill the entire space with walls so we can dig it out later
  const dungeon = rectangle(
    { x, y, width, height },
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
      { x: rx, y: ry, width: rw, height: rh, hasWalls: true },
      { sprite: "FLOOR" }
    );

    // test if candidate is overlapping with any existing rooms
    if (!rooms.some((room) => rectsIntersect(room, candidate))) {
      rooms.push(candidate);
      roomTiles = { ...roomTiles, ...candidate.tiles };
    }
  });

  dungeon.tiles = { ...dungeon.tiles, ...roomTiles };

  // create tile entities
  Object.keys(dungeon.tiles).forEach((key) => {
    const tile = dungeon.tiles[key];

    if (tile.sprite === "WALL") {
      const entity = world.createEntity();
      entity.add(Appearance, { char: "#", color: "#AAA" });
      entity.add(Position, dungeon.tiles[key]);
    }

    if (tile.sprite === "FLOOR") {
      const entity = world.createEntity();
      entity.add(Appearance, { char: "•", color: "#555" });
      entity.add(Position, dungeon.tiles[key]);
    }
  });

  return dungeon;
};
```

The bulk of the diff is here:

```javascript
const rooms = [];
let roomTiles = {};

times(maxRoomCount, () => {
  let rw = random(minRoomSize, maxRoomSize);
  let rh = random(minRoomSize, maxRoomSize);
  let rx = random(x, width + x - rw - 1);
  let ry = random(y, height + y - rh - 1);

  // create a candidate room
  const candidate = rectangle(
    { x: rx, y: ry, width: rw, height: rh, hasWalls: true },
    { sprite: "FLOOR" }
  );

  // test if candidate is overlapping with any existing rooms
  if (!rooms.some((room) => rectsIntersect(room, candidate))) {
    rooms.push(candidate);
    roomTiles = { ...roomTiles, ...candidate.tiles };
  }
});

dungeon.tiles = { ...dungeon.tiles, ...roomTiles };
```

We use `times` to run a function a number of times equal to the maxRoomCount. This function creates a new room with random dimensions and saves it to the variable `candidate`. We then check `candidate` against an array of accepted rooms to make sure it doesn't intersect with any of them. If everything checks out we push it onto our array of accepted rooms and add it's tiles to the roomTiles object for merging with our dungeon.

Check out the game - there should now be multiple rooms of all sizes randomly sprinkled across the map!

All we need yet is passages connecting our rooms.

We'll add two new functions digHorizontalPassage and digVerticalPassage at the top of `./src/lib/dungeon` right after the imports.

```javascript
function digHorizontalPassage(x1, x2, y) {
  const tiles = {};
  const start = Math.min(x1, x2);
  const end = Math.max(x1, x2) + 1;
  let x = start;

  while (x < end) {
    tiles[`${x},${y}`] = { x, y, sprite: "FLOOR" };
    x++;
  }

  return tiles;
}

function digVerticalPassage(y1, y2, x) {
  const tiles = {};
  const start = Math.min(y1, y2);
  const end = Math.max(y1, y2) + 1;
  let y = start;

  while (y < end) {
    tiles[`${x},${y}`] = { x, y, sprite: "FLOOR" };
    y++;
  }

  return tiles;
}
```

These are very similar functions that just return straight passages of floor tiles. To use them we need to loop through all of our accepted rooms so that we can pair them up and draw both a vertical and horizontal lines connecting them.

Just above the line where we merge all of our tiles into dungeon.tiles add:

```javascript
let prevRoom = null;
let passageTiles;

for (let room of rooms) {
  if (prevRoom) {
    const prev = prevRoom.center;
    const curr = room.center;

    passageTiles = {
      ...passageTiles,
      ...digHorizontalPassage(prev.x, curr.x, curr.y),
      ...digVerticalPassage(prev.y, curr.y, prev.x),
    };
  }

  prevRoom = room;
}
```

And don't forget to add the passageTiles to your dungeon:

```diff
-dungeon.tiles = { ...dungeon.tiles, ...roomTiles };
+dungeon.tiles = { ...dungeon.tiles, ...roomTiles, ...passageTiles };
```

That's it! You should now have a procedurally generated dungeon! Check out your game - see if it worked.

if you have a hard time visualizing everything that's going in the algorithm like I do, check out [this live version](https://luetkemj.github.io/pcgdgns/) that animates each step. You can see the code attempt to place each room and dig the passages one leg at time. Just refresh the browser to see it build another dungeon.

Dungeon generation is the favorite part of a lot of developers. This is a part of the codebase where you can get a lot bang for your buck. Subtle changes can make huge differences in the landscape and really change how your game plays. In the live version above I added a drunken walk mining algorithm to generate a 'natural cave' in the middle of an artificial dungeon. What would happen if you don't test for overlapping rooms? Or allow 100 rooms, or make them all huge? Can you figure our how to tweak our code to fill a room with water?

There are so many possibilities here I almost forgot that our @ can still just walk wherever the heck it pleases. We have a little more work to do to make our dungeon 'playable'.

To make our game playable we need to do two things. Place our @ in the center of one of the generated rooms and make our walls blocking.

Let's got for the easy win and put our player in the center of one of the rooms.

In `./src/lib/dungeon.js` add our accepts rooms array to our dungeon object like this:

```diff
+  dungeon.rooms = rooms;
  dungeon.tiles = { ...dungeon.tiles, ...roomTiles, ...passageTiles };
```

Then in `./src/index.js` set the player position to the center of the first room in that array like this:

```diff
-player.position.x = dungeon.center.x;
-player.position.y = dungeon.center.y;
+player.position.x = dungeon.rooms[0].center.x;
+player.position.y = dungeon.rooms[0].center.y;
```

Simple enough! The @ will now always start in the center of a room.

Ok, now for the slightly more challenging piece. In order to prevent our "@" from walking through walls we need to add a new component on wall entities and then test that no entity with a blocking component exists in the location we want to move to from our movement system.

Let's start by adding another component to `./src/state/components`.

```javascript
export class IsBlocking extends Component {}
```

You will notice that this component doesn't have any properties like the others. We will only be checking if the component exists on an entity so we don't actually need any.

As always, don't forget to register the component in `./src/state/ecs`

```diff
import { Engine } from "geotic";
-import { Appearance, Move, Position } from "./components";
+import { Appearance, IsBlocking, Move, Position } from "./components";

const ecs = new Engine();

// all Components must be `registered` by the engine
ecs.registerComponent(Appearance);
+ecs.registerComponent(IsBlocking);
ecs.registerComponent(Move);
ecs.registerComponent(Position);

export const player = world.createEntity();
player.add(Appearance, { char: "@", color: "#fff" });
player.add(Position);

export default world;
```

Now head back to `./src/lib/dungeon` and where we will add the `IsBlocking` component to our wall entities.

First, import the component with the others:

```diff
-import { Appearance, Position } from "../state/components";
+import { Appearance, IsBlocking, Position } from "../state/components";
```

Then add it to wall entities

```diff
if (tile.sprite === "WALL") {
  const entity = world.createEntity();
  entity.add(Appearance, { char: "#", color: "#AAA" });
+  entity.add(IsBlocking);
  entity.add(Position, dungeon.tiles[key]);
}
```

Finally in movement we need to check if the location we want to move to has an entity with the blocking component. Right after the check to observe map boundaries and before we actually update position of our entity we can check for any other blocking entities.

```diff
    // this is where we will run any checks to see if entity can move to new location
    // observe map boundaries
    mx = Math.min(grid.map.width + grid.map.x - 1, Math.max(21, mx));
    my = Math.min(grid.map.height + grid.map.y - 1, Math.max(3, my));

+    // check for blockers
+    const blockers = [];
+    for (const e of world.getEntities()) {
+      if (e[1].position.x === mx && e[1].position.y === my && e[1].isBlocking) {
+        blockers.push(e);
+      }
+    }
+    if (blockers.length) {
+      entity.remove(entity.move);
+      return;
+    }

    entity.position.x = mx;
    entity.position.y = my;
```

All we're doing here is looping through all of the entities in the game testing for any that are both in the location we intend to move and contain the `IsBlocking` component. If we find any blockers they get added to the `blockers` array. Finally if any blockers were found we know we can't enter the new location and instead just remove the `Move` component and return.

Typically I build an entitiesAtLocation cache that tracks entity ids at each location. It looks something like this:

```javascript
{
  "0,0", ["entityid1"];
  "0,1", ["entityid2", "entityid4"];
  "0,2", ["entityid3"];
}
```

This is a lot faster as we only have to test the entities that are actually in the location we want to enter. But it's usually a mistake to over optimize early so for now, we're fine. Feel free to revisit this later if you need/want to.

---

Phew! Another part complete - this was another big one, congratulations on making it this far!

In part 3 we completed one of the most important and satisfying aspects of building a roguelike - procedural dungeon generation! You now have a basic algorithm ready to be expanded upon with newfound tools and knowledge. Have fun and make it your own!

In [part 4](https://github.com/luetkemj/jsrlt/blob/master/tutorial/part4.md) we'll get to tackle another typical feature in roguelikes - Field of Vision!

See you there!
