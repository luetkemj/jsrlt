# Part 5 - Placing Enemies and kicking them (harmlessly)

We can finally walk around the dungeon - but it appears nobody's home. This tutorial will focus on placing monsters in the dungeon! And as the title implies, we'll kick them (harmlessly).

Let's start by adding some goblins to our dungeon. Our first challenge is to figure our where we can place our them. We don't want them stuck in walls or on top of the player or each other.

Currently our dungeon has walls and floors. Floors are non-blocking and usually empty. Lets get an array of all of the floor locations for a basic start.

In `./src/index.js` make this change:

```diff
player.add(Position, {
  y: dungeon.rooms[0].center.y,
});

+const openTiles = Object.values(dungeon.tiles).filter(
+  (x) => x.sprite === "FLOOR"
+);

fov();
```

We're just collecting all the tiles from our dungeon algorithm with floor sprites. Good enough to start. Now that we have some locations let's just jump right in and make some goblins!

```diff
const openTiles = Object.values(dungeon.tiles).filter(
  (x) => x.sprite === "FLOOR"
);

+times(5, () => {
+  const tile = sample(openTiles);
+
+  const goblin = world.createEntity();
+  goblin.add(Appearance, { char: "g", color: "green" });
+  goblin.add(Layer400);
+  goblin.add(Position, { x: tile.x, y: tile.y });
+});

fov();
```

We're using a couple function from lodash again - times, we've seen. Sample is just a quick way to grab a random element from an array.

We need to import a few things at the top of this file before moving on:

```diff
+import { sample, times } from "lodash";
import "./lib/canvas.js";
import { grid } from "./lib/canvas";
import { createDungeon } from "./lib/dungeon";
import { fov } from "./systems/fov";
import { movement } from "./systems/movement";
import { render } from "./systems/render";
-import { player } from "./state/ecs";
+import world, { player } from "./state/ecs";
import {
+  Appearance,
+  Layer400,
  Move,
  Position,
} from "./state/components";
```

Try it out and see if you can find the goblins!

Our dungeon now has some inhabitants! Although you may have noticed we can walk right through them as if they were ghosts and ghosts are terrible for kicking. Add the `IsBlocking` component to make them corporeal.

```diff
times(5, () => {
  const tile = sample(openTiles);

  const goblin = world.createEntity();
  goblin.add(Appearance, { char: "g", color: "green" });
+  goblin.add(IsBlocking);
  goblin.add(Layer400);
  goblin.add(Position, { x: tile.x, y: tile.y });
});
```

And import it:

```diff
import {
  Appearance,
+  IsBlocking,
  Layer400,
  Move,
  Position,
} from "./state/components";
```

Now for the kicking part. In `./src/systems/movement.js` when blockers exist - let's log to console.

```diff
if (blockers.length) {
+  console.log('Kick!')

  entity.remove(entity.move);
  return;
}
```

Ok - let's improve on this a bit by adding a new component that will let us log exactly who kicked what. In `./src/state/components` add a new component:

```javascript
export class Description extends Component {
  static properties = { name: "No Name" };
}
```

We need to register our new component in `./src/index.js` but while we're there let's add it to our player entity as well.

```diff
import { Engine } from "geotic";
import { cache } from "./cache";
import {
  Appearance,
+ Description,
  IsBlocking,
  IsInFov,
  IsOpaque,
  IsRevealed,
  Layer100,
  Layer400,
  Move,
  Position,
} from "./components";

const ecs = new Engine();

// all Components must be `registered` by the engine
ecs.registerComponent(Appearance);
+ecs.registerComponent(Description);
ecs.registerComponent(IsBlocking);
ecs.registerComponent(IsInFov);
ecs.registerComponent(IsOpaque);
ecs.registerComponent(IsRevealed);
ecs.registerComponent(Layer100);
ecs.registerComponent(Layer400);
ecs.registerComponent(Move);
ecs.registerComponent(Position);

export const player = world.createEntity();
player.add(Appearance, { char: "@", color: "#fff" });
player.add(Layer400);
+player.add(Description, { name: 'You' })

export default world;
```

We need to import the component add it to our goblins in `./src/index.js`:

```diff
import {
  Appearance,
+  Description
  IsBlocking,
  Layer400,
  Move,
  Position,
} from "./state/components";
```

```diff
times(100, () => {
  const tile = sample(openTiles);

  const goblin = world.createEntity();
  goblin.add(Appearance, { char: "g", color: "green" });
+  goblin.add(Description, { name: "goblin" });
  goblin.add(IsBlocking);
  goblin.add(Layer400);
  goblin.add(Position, { x: tile.x, y: tile.y });
});
```

And we should also import and add it to our dungeon walls and floors in `./src/lib/dungeon.js`

```diff
import {
  Appearance,
+  Description,
  IsBlocking,
  IsOpaque,
  Layer100,
  Position,
} from "../state/components";
```

```diff
if (tile.sprite === "WALL") {
  const entity = world.createEntity();
  entity.add(Appearance, { char: "#", color: "#AAA" });
+  entity.add(Description, { name: "wall" });
  entity.add(IsBlocking);
  entity.add(IsOpaque);
  entity.add(Position, dungeon.tiles[key]);
  entity.add(Layer100);
}

if (tile.sprite === "FLOOR") {
  const entity = world.createEntity();
  entity.add(Appearance, { char: "•", color: "#555" });
+  entity.add(Description, { name: "floor" });
  entity.add(Position, dungeon.tiles[key]);
  entity.add(Layer100);
}
```

Oof - our components are entity definitions are really getting spread out... there's gotta be a better to add new components across entities. There is - we'll refactor that later. For now, let's push forward and modify `./src/systems/movement.js`. Replace `console.log('Kick!')` with:

```javascript
blockers.forEach((eId) => {
  const attacker =
    (entity.description && entity.description.name) || "something";
  const target =
    (world.getEntity(eId).description &&
      world.getEntity(eId).description.name) ||
    "something";
  console.log(`${attacker} kicked a ${target}!`);
});
```

Check our your game! Kick stuff! Look in the console - it says "You kicked a goblin!" Pretty cool.

But let's be honest - it's not really that fun to kick a goblin if it can't kick back right? We should probably give them a turn...

You know what we don't have yet? A game loop! We don't need anything complicated but if we had a game loop we could give the goblins a turn to kick!

We need a way to track whose turn it is. Add a variable `playerTurn` just below userInput.

```diff
render();

let userInput = null;
+let playerTurn = true;

document.addEventListener("keydown", (ev) => {
```

Our gameLoop will call `processUserInput` from now on. So we can delete it from our keydown event listener.

```diff
document.addEventListener("keydown", (ev) => {
  userInput = ev.key;
-  processUserInput();
});
```

The game loop will also handle running our systems so we can delete them from processUserInput. We also need to clear userInput after it's been processed for our loop to work correctly. We can just set it null at the end.

```diff
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

-  movement();
-  fov();
-  render();
+   userInput = null;
};
```

Our game loop is pretty simple, looks like this, and will live in `./src/index.js` at the very bottom:

```javascript
const gameLoop = () => {
  update();
  requestAnimationFrame(gameLoop);
};

requestAnimationFrame(gameLoop);
```

You can read more about [requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame) on MDN. For our purposes, it's just a nice simple way to get our gameLoop going in the browser.

Inside out gameLoop we call `update()`. Let's add that and add it right above the gameLoop.

```javascript
const update = () => {
  if (playerTurn && userInput) {
    processUserInput();
    movement();
    fov();
    render();

    playerTurn = false;
  }

  if (!playerTurn) {
    movement();
    fov();
    render();

    playerTurn = true;
  }
};
```

Every time this function runs we test if it's the player's turn AND they have pushed a key `userInput`. If both of those are true, we run our player systems and set playerTurn to false. If playerTurn is false, we run our monster systems giving them a turn.

If it's the player's turn but they haven't push a key yet, the loop just goes back around without doing anything. It's a no-op.

We can test this by adding an "ai" system. It should only run on the monster's turn and for now will just give them something to think about.

Our ai system is super simple for now. Create a new file called `ai.js` at `./src/systems/ai.js` and make it look like this:

```javascript
import world from "../state/ecs";
import { Ai, Description } from "../state/components";

const aiEntities = world.createQuery({
  all: [Ai, Description],
});

export const ai = () => {
  aiEntities.get().forEach((entity) => {
    console.log(
      `${entity.description.name} ${entity.id} ponders it's existence.`
    );
  });
};
```

We create a query to get our entities with `[Ai, Description]` components. Then we just iterate over them and console a simple statement so it's obvious each of the goblins got a turn.

Now we need to make the Ai component and register it.

In `./src/state/components.js`

```javascript
export class Ai extends Component {}
```

And `./src/state/ecs`

```diff
import { Engine } from "geotic";
import { cache } from "./cache";
import {
+  Ai,
  Appearance,
  Description,
  IsBlocking,
```

```diff
// all Components must be `registered` by the engine
+ecs.registerComponent(Ai);
ecs.registerComponent(Appearance);
```

Finally we need to add the Ai component to our goblins and run the system in `./src/index.js`.

First we import the system and the component:

```diff
import "./lib/canvas.js";
import { grid } from "./lib/canvas";
import { createDungeon } from "./lib/dungeon";
+import { ai } from "./systems/ai";
import { fov } from "./systems/fov";
import { movement } from "./systems/movement";
import { render } from "./systems/render";
import world, { player } from "./state/ecs";
import {
+  Ai,
  Appearance,
  Description,
  IsBlocking,
```

Then we add the component to our goblins:

```diff
  const goblin = world.createEntity();
+  goblin.add(Ai);
  goblin.add(Appearance, { char: "g", color: "green" });
  goblin.add(Description, { name: "goblin" });
  goblin.add(IsBlocking);
  goblin.add(Layer400);
  goblin.add(Position, { x: tile.x, y: tile.y });
  goblin.add(Description, { name: "goblin" });
});
```

And finally we call the Ai system on the monsters turn - NOT on the player's turn.

```diff
const update = () => {
  if (playerTurn && userInput) {
+   console.log('I am @, hear me roar.')
    processUserInput();
    movement();
    fov();
    render();

    playerTurn = false;
  }

  if (!playerTurn) {
+    ai();
    movement();
    fov();
    render();

    playerTurn = true;
  }
};
```

Play the game again - with the console open and check out the logs. The player gets a go - and then the goblins. Plenty of time for kicking on both sides!

---

In part 5 we finally added some goblins to our game! So far all they do is stand around and think about things while we kick them (harmlessly). To give them a shot at us we also (finally) added a game loop!

In [part 6](https://github.com/luetkemj/jsrlt/blob/master/tutorial/part6.md) we look at doing (and taking) some damage when we add combat! See ya there :)
