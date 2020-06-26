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

export const drawCell = (entity) => {
  const {
    appearance: { char, background, color },
    position,
  } = entity;

  drawBackground({ color: background, position });
  drawChar({ char, color, position });
};
```

Instead of calling `drawChar` directly we can now call `drawCell` and pass it any entity with an appearance and a position component. Our new `drawCell` function calls `drawBackground` and `drawChar` for us. We can still get a transparent background if we need to by passing the color 'transparent'.

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

Add a couple new components in `./src/state/components`

```javascript
export class Layer100 extends Component {}
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

Next we need to add layer components to all our entities. First in `./src/lib/dungeon.js` import the `Layer100` component and add it to the floors and walls.

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
  entity.add(Position, dungeon.tiles[key]);
+  entity.add(Layer100);
}

if (tile.sprite === "FLOOR") {
  const entity = ecs.createEntity();
  entity.add(Appearance, { char: "•", color: "#555" });
  entity.add(Position, dungeon.tiles[key]);
+  entity.add(Layer100);
}
```

And then in `./src/state/ecs.js` we need to add `Layer400` to our player entity.

```diff
export const player = ecs.createEntity();
player.add(Appearance, { char: "@", color: "#fff" });
player.add(Position);
+player.add(Layer400);

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

Nice work! We're starting to build a solid foundation for our game. We just have one more thing before we can add our "Field of View". Remember in the last tutorial where I talked about how we weren't going to add an `entitiesAtLocation` cache? I lied. We're totally going to add one. Right now. Don't worry, it's not hard - geotic has a couple tricks up it's sleeve to help us out :)

catch up
fix layers in render
add cache for location tracking

fov code
hookit up with geotic

http://journal.stuffwithstuff.com/2015/09/07/what-the-hero-sees/
