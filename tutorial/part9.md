# Ranged Scrolls and Targeting

In this part we will be adding ranged scrolls and targeting. This will require some refactoring of old systems and the addition of some new ones. We will also introduce animations to make the destruction from our ranged and area of effect scrolls more obvious. We have a lot to do so let's get into it!

## Refactoring the render system

In preparation for adding targeting we need to refactor the existing render system. Why? To make for a better user experience when targeting we should add a background color on the targeted cell. Because we only want to do this when targeting we need to be able to render things differently based on the gameSate. Our current render system is not modular enough to easily allow for that. Cool? Let's go!

The first thing we'll do to make things more modular is update our clearCanvas function. We want to be able to clear a specific section of the canvas instead of the entire thing every time.

In `./src/lib/canvas.js` change the existing clearCanvas function to this:

```javascript
export const clearCanvas = (x, y, w, h) => {
  const posX = x * cellWidth;
  const posY = y * cellHeight;

  const width = cellWidth * w;
  const height = cellHeight * h;

  ctx.clearRect(posX, posY, width, height);
};
```

Next we'll create functions to render and clear each section of our UI. The goal is to be able to do something like this:

```
if (gameState === 'GAME') {
  renderMap()
  renderPlayerHud()
  renderMessageLog()
}
```

Rather than go through the refactor bit by bit, I'll just give you the final result. It really is just wrapping the existing logic to render each section of the UI in their own functions and then calling them individually. Go ahead and paste the following into `./src/systems/render.js` overwriting everything that was already there.

```javascript
import { throttle } from "lodash";
import ecs from "../state/ecs";
import {
  Appearance,
  IsInFov,
  IsRevealed,
  Position,
  Layer100,
  Layer300,
  Layer400,
} from "../state/components";
import { messageLog } from "../state/ecs";
import {
  clearCanvas,
  drawCell,
  drawRect,
  drawText,
  grid,
  pxToCell,
} from "../lib/canvas";
import { toLocId } from "../lib/grid";
import { readCacheSet } from "../state/cache";
import { gameState, selectedInventoryIndex } from "../index";

const layer100Entities = ecs.createQuery({
  all: [Position, Appearance, Layer100],
  any: [IsInFov, IsRevealed],
});

const layer300Entities = ecs.createQuery({
  all: [Position, Appearance, Layer300],
  any: [IsInFov, IsRevealed],
});

const layer400Entities = ecs.createQuery({
  all: [Position, Appearance, Layer400, IsInFov],
});

const clearMap = () => {
  clearCanvas(grid.map.x - 1, grid.map.y, grid.map.width + 1, grid.map.height);
};

const renderMap = () => {
  clearMap();

  layer100Entities.get().forEach((entity) => {
    if (entity.isInFov) {
      drawCell(entity);
    } else {
      drawCell(entity, { color: "#333" });
    }
  });

  layer300Entities.get().forEach((entity) => {
    if (entity.isInFov) {
      drawCell(entity);
    } else {
      drawCell(entity, { color: "#333" });
    }
  });

  layer400Entities.get().forEach((entity) => {
    if (entity.isInFov) {
      drawCell(entity);
    } else {
      drawCell(entity, { color: "#100" });
    }
  });
};

const clearPlayerHud = () => {
  clearCanvas(
    grid.playerHud.x,
    grid.playerHud.y,
    grid.playerHud.width + 1,
    grid.playerHud.height
  );
};

const renderPlayerHud = (player) => {
  clearPlayerHud();

  drawText({
    text: `${player.appearance.char} ${player.description.name}`,
    background: `${player.appearance.background}`,
    color: `${player.appearance.color}`,
    x: grid.playerHud.x,
    y: grid.playerHud.y,
  });

  drawText({
    text: "♥".repeat(grid.playerHud.width),
    background: "black",
    color: "#333",
    x: grid.playerHud.x,
    y: grid.playerHud.y + 1,
  });

  const hp = player.health.current / player.health.max;

  if (hp > 0) {
    drawText({
      text: "♥".repeat(hp * grid.playerHud.width),
      background: "black",
      color: "red",
      x: grid.playerHud.x,
      y: grid.playerHud.y + 1,
    });
  }
};

const clearMessageLog = () => {
  clearCanvas(
    grid.messageLog.x,
    grid.messageLog.y,
    grid.messageLog.width + 1,
    grid.messageLog.height
  );
};

const renderMessageLog = () => {
  clearMessageLog();

  drawText({
    text: messageLog[2],
    background: "#000",
    color: "#666",
    x: grid.messageLog.x,
    y: grid.messageLog.y,
  });

  drawText({
    text: messageLog[1],
    background: "#000",
    color: "#aaa",
    x: grid.messageLog.x,
    y: grid.messageLog.y + 1,
  });

  drawText({
    text: messageLog[0],
    background: "#000",
    color: "#fff",
    x: grid.messageLog.x,
    y: grid.messageLog.y + 2,
  });
};

const clearInfoBar = () => {
  drawText({
    text: ` `.repeat(grid.infoBar.width),
    x: grid.infoBar.x,
    y: grid.infoBar.y,
    background: "black",
  });
};

const renderInfoBar = (mPos) => {
  clearInfoBar();

  const { x, y } = mPos;
  const locId = toLocId({ x, y });

  const esAtLoc = readCacheSet("entitiesAtLocation", locId) || [];
  const entitiesAtLoc = [...esAtLoc];

  clearInfoBar();

  if (entitiesAtLoc) {
    entitiesAtLoc
      .filter((eId) => {
        const entity = ecs.getEntity(eId);
        return (
          layer100Entities.isMatch(entity) ||
          layer300Entities.isMatch(entity) ||
          layer400Entities.isMatch(entity)
        );
      })
      .forEach((eId) => {
        const entity = ecs.getEntity(eId);
        clearInfoBar();

        if (entity.isInFov) {
          drawText({
            text: `You see a ${entity.description.name}(${entity.appearance.char}) here.`,
            x: grid.infoBar.x,
            y: grid.infoBar.y,
            color: "white",
            background: "black",
          });
        } else {
          drawText({
            text: `You remember seeing a ${entity.description.name}(${entity.appearance.char}) here.`,
            x: grid.infoBar.x,
            y: grid.infoBar.y,
            color: "white",
            background: "black",
          });
        }
      });
  }
};

const renderInventory = (player) => {
  clearInfoBar();
  // translucent to obscure the game map
  drawRect(0, 0, grid.width, grid.height, "rgba(0,0,0,0.65)");

  drawText({
    text: "INVENTORY",
    background: "black",
    color: "white",
    x: grid.inventory.x,
    y: grid.inventory.y,
  });

  drawText({
    text: "(c)Consume (d)Drop",
    background: "black",
    color: "#666",
    x: grid.inventory.x,
    y: grid.inventory.y + 1,
  });

  if (player.inventory.list.length) {
    player.inventory.list.forEach((entity, idx) => {
      drawText({
        text: `${idx === selectedInventoryIndex ? "*" : " "}${
          entity.description.name
        }`,
        background: "black",
        color: "white",
        x: grid.inventory.x,
        y: grid.inventory.y + 3 + idx,
      });
    });
  } else {
    drawText({
      text: "-empty-",
      background: "black",
      color: "#666",
      x: grid.inventory.x,
      y: grid.inventory.y + 3,
    });
  }
};

export const render = (player) => {
  renderMap();
  renderPlayerHud(player);
  renderMessageLog();

  if (gameState === "INVENTORY") {
    renderInventory(player);
  }
};

const canvas = document.querySelector("#canvas");
canvas.onmousemove = throttle((e) => {
  if (gameState === "GAME") {
    const [x, y] = pxToCell(e);
    renderMap();
    renderInfoBar({ x, y });
  }
}, 50);
```

Everything still working? Exactly the same as it was before? Good, let's add one small bit of additional functionality to our render system. When we mouse over the map to inspect things, it would be nice to add a background color to the cell we are hovering. This is similar to what we will want when actually targeting a bit later.

Import `some` from lodash at the top of the file:

```diff
-import { throttle } from "lodash";
+import { some, throttle } from "lodash";
```

And make this change to the renderInfoBar function:

```diff
const renderInfoBar = (mPos) => {
  clearInfoBar();

  const { x, y } = mPos;
  const locId = toLocId({ x, y });

  const esAtLoc = readCacheSet("entitiesAtLocation", locId) || [];
  const entitiesAtLoc = [...esAtLoc];

  clearInfoBar();

  if (entitiesAtLoc) {
+    if (some(entitiesAtLoc, (eId) => ecs.getEntity(eId).isRevealed)) {
+      drawCell({
+        appearance: {
+          char: "",
+          background: "rgba(255, 255, 255, 0.5)",
+        },
+        position: { x, y },
+      });
+    }

    entitiesAtLoc
      .filter((eId) => {
```

Our render system is fully refactored and we get a nice little bit of extra UI pizzazz we couldn't do previously.

## Targeting UI

With all that out of the way we can now focus on the UI for targeting. We will need another gameState and we will add a (temporary) keybinding to toggle it on and off.

In `./src/index.js` add a new keybinding (z) that will toggle the gameState from `GAME` to `TARGETING`

```diff
const processUserInput = () => {
  if (gameState === "GAME") {
    ...
    if (userInput === "i") {
      gameState = "INVENTORY";
    }

+    if (userInput === "z") {
+      gameState = "TARGETING";
+    }

    userInput = null;
  }

+  if (gameState === "TARGETING") {
+    if (userInput === "z" || userInput === "Escape") {
+      gameState = "GAME";
+    }
+
+    userInput = null;
+  }

  if (gameState === "INVENTORY") {
    if (userInput === "i" || userInput === "Escape") {
      gameState = "GAME";
    }
```

We also need to add another conditional to our update function to call a limited set of systems when in the `TARGETING` game state.

```diff
const update = () => {
  if (player.isDead) {
    return;
  }

+  if (playerTurn && userInput && gameState === "TARGETING") {
+    processUserInput();
+    render(player);
+    playerTurn = true;
+  }

  if (playerTurn && userInput && gameState === "INVENTORY") {
    processUserInput();
    effects();
    render(player);
    playerTurn = true;
  }
```

Next in `./src/systems/render.js` we need a new function `renderTargeting`. It can go anywhere in the group of `renderSomething` functions we created in the refactor earlier.

```javascript
const renderTargeting = (mPos) => {
  const { x, y } = mPos;
  const locId = toLocId({ x, y });

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
        position: { x, y },
      });
    }
  }
};
```

And finally we need to call it onmousemove but only if the game is in the `TARGETING` state.

```diff
canvas.onmousemove = throttle((e) => {
  if (gameState === "GAME") {
    const [x, y] = pxToCell(e);
    renderMap();
    renderInfoBar({ x, y });
  }

+  if (gameState === "TARGETING") {
+    const [x, y] = pxToCell(e);
+    renderMap();
+    renderTargeting({ x, y });
+  }
}, 50);
```

## Animations

With all this focus on the UI it seems like a good time to add animations. Without a visual cue it will hard to tell if our ranged spells actually worked. And as a bonus we'll be able to use it for our existing attacks, quaffing health potions, whatever you want really.

To create animations we will as always, start with a component. The component will hold information about how that animation looks and behaves and a system will use that information to actually render the animation.

In `./src/state/components.js` add an `Animate` component like this:

```javascript
export class Animate extends Component {
  static allowMultiple = true;
  static properties = {
    startTime: null,
    duration: 250,
    char: "",
    color: "",
  };

  onSetStartTime(evt) {
    this.startTime = evt.data.time;
    evt.handle();
  }
}
```

You'll notice the onSetStartTime event. This will be used by our system to calculate the current frame and determine when the animation is complete.

While we're in here, add an animate property to the Effects and ActiveEffects components. This will allow us to add animations to our effects too!

```diff
export class ActiveEffects extends Component {
  static allowMultiple = true;
-  static properties = { component: "", delta: "" };
+  static properties = {
+    component: "",
+    delta: "",
+    animate: { char: "", color: "" },
+  };
}

export class Effects extends Component {
  static allowMultiple = true;
-  static properties = { component: "", delta: "" };
+  static properties = {
+    component: "",
+    delta: "",
+    animate: { char: "", color: "" },
+  };
}
```

We should go ahead and register the new component in `./src/state/ecs.js`

```diff
import {
  ActiveEffects,
  Ai,
+  Animate,
  Appearance,
  Description,
  Defense,
```

```diff
// all Components must be `registered` by the engine
ecs.registerComponent(ActiveEffects);
+ecs.registerComponent(Animate);
ecs.registerComponent(Ai);
```

On to the animation system itself. Create a new file at `./src/systems/animation.js` that looks like this:

```javascript
import { last } from "lodash";
import ecs from "../state/ecs";
import { clearCanvas, drawCell } from "../lib/canvas";
const { Animate, IsInFov } = require("../state/components");
import { gameState } from "../index";

const animatingEntities = ecs.createQuery({
  all: [Animate],
});

const hexToRgb = (hex) => {
  var result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
  return result
    ? {
        r: parseInt(result[1], 16),
        g: parseInt(result[2], 16),
        b: parseInt(result[3], 16),
      }
    : {};
};

export const animation = () => {
  if (gameState !== "GAME") {
    return;
  }

  animatingEntities.get().forEach((entity) => {
    const animate = last(entity.animate);

    const { r = 255, g = 255, b = 255 } = hexToRgb(animate.color);

    const time = new Date();
    // set animation startTime
    if (!animate.startTime) {
      entity.fireEvent("set-start-time", { time });
    }
    const frameTime = time - animate.startTime;
    // end animation when complete
    if (frameTime > animate.duration) {
      return entity.remove("Animate");
    }
    const framePercent = frameTime / animate.duration;
    // do the animation
    // clear the cell first
    clearCanvas(entity.position.x, entity.position.y, 1, 1);

    // redraw the existing entity
    drawCell(entity);

    // draw the animation over top
    drawCell({
      appearance: {
        char: animate.char || entity.appearance.char,
        color: `rgba(${r}, ${g}, ${b}, ${1 - framePercent})`,
        background: "transparent",
      },
      position: entity.position,
    });
  });
};
```

What's going on in here? The animation function iteself boils down to a few simple steps.

Step 1: We only want to run animations in the game state and because our entities can have multiple animations we always choose the last one.

```javascript
export const animation = () => {
  if (gameState !== "GAME") {
    return;
  }

  animatingEntities.get().forEach((entity) => {
    const animate = last(entity.animate);
```

Step 2: Calculate rgb values so we can create an rgba css string. This is how we will animate the alpha (opacity).

```javascript
const { r = 255, g = 255, b = 255 } = hexToRgb(animate.color);
```

Step 3: Set the frameTime and check if our animation has completed.

```javascript
const time = new Date();
// set animation startTime
if (!animate.startTime) {
  entity.fireEvent("set-start-time", { time });
}
const frameTime = time - animate.startTime;
// end animation when complete
if (frameTime > animate.duration) {
  return entity.remove("Animate");
}
```

Step 4: Calculate a percentage to use for our alpha channel based on how complete our animation is.

```javascript
const framePercent = frameTime / animate.duration;
```

Step 5 Render the frame.

```javascript
    // do the animation
    // clear the cell first
    clearCanvas(entity.position.x, entity.position.y, 1, 1);

    // redraw the existing entity
    drawCell(entity);

    // draw the animation over top
    drawCell({
      appearance: {
        char: animate.char || entity.appearance.char,
        color: `rgba(${r}, ${g}, ${b}, ${1 - framePercent})`,
        background: "transparent",
      },
      position: entity.position,
    });
  });
};
```

Now that our system is in place let's call it from `./src/index.js`.

```diff
import { ai } from "./systems/ai";
+import { animation } from "./systems/animation";
import { effects } from "./systems/effects";
```

```diff
const update = () => {
+  animation();

  if (player.isDead) {
    return;
  }
```

To complete the integration with our effects system we just need to add Animate components from active effects to entities in the effects system. In `./src/systems/effects.js` add this line:

```diff
export const effects = () => {
  activeEffectsEntities.get().forEach((entity) => {
    entity.activeEffects.forEach((c) => {
      if (entity[c.component]) {
        entity[c.component].current += c.delta;

        if (entity[c.component].current > entity[c.component].max) {
          entity[c.component].current = entity[c.component].max;
        }
      }
+      entity.add("Animate", { ...c.animate });

      c.remove();
    });
  });
};
```

Now we can test one out by adding an animate property to the effects of our existing health potions in `./src/state/prefabs.js`

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
    {
      type: "Effects",
-      properties: { component: "health", delta: 5 },
+      properties: {
+        component: "health",
+        delta: 5,
+        animate: { color: "#ff0000", char: "♥" },
+      },
+    },
  ],
};
```

Test it out!

You'll notice that our animations only run when in the game state - so we don't see the animation from the potion until after closing the inventory. To get a more immediate response we can set the gameState back to 'GAME' after consuming a potion. Do that in `./src/index.js`.

```diff
addLog(`You consume a ${entity.description.name}`);
    entity.destroy();

    if (selectedInventoryIndex > player.inventory.list.length - 1)
      selectedInventoryIndex = player.inventory.list.length - 1;

+    gameState = "GAME";
  }
}
```

Animations can now easily be added to any entities by simply attaching an animate component.

## LIGHTNING SCROLL!

With all of that out of the way we are finally able to create out first scroll! We'll start with a lightning scroll that will target a random nearby enemy.

In `./src/state/prefabs` add the following prefab:

```javascript
export const ScrollLightning = {
  name: "ScrollLightning",
  inherit: ["Item"],
  components: [
    {
      type: "Appearance",
      properties: { char: "♪", color: "#DAA520" },
    },
    {
      type: "Description",
      properties: { name: "scroll of lightning" },
    },
    {
      type: "Effects",
      properties: {
        animate: { color: "#F7FF00" },
        events: [
          {
            name: "take-damage",
            args: { amount: 25 },
          },
        ],
      },
    },
    { type: "RequiresTarget", properties: { acquired: "RANDOM" } },
  ],
};
```

Our scroll is very similar to the `HealthPotion` prefab with Appearance, Description, and Effects components. It does have one extra - `RequiresTarget`. That component will be used to determine the scrolls target as well as the means of acquiring it. In this case it'll be a random target.

In `src/state/components.js` go ahead and add the `RequiresTarget` component.

```javascript
export class RequiresTarget extends Component {
  static properties = {
    acquired: "RANDOM",
  };
}
```

And register both the new prefab and component in `./src/state/ecs.js`

```diff
  Position,
  Power,
+  RequiresTarget,
} from "./components";
```

```diff
  Wall,
  Floor,
+  ScrollLightning,
} from "./prefabs";
```

```diff
ecs.registerComponent(Power);
+ecs.registerComponent(RequiresTarget);

// register "base" prefabs first!
ecs.registerPrefab(Tile);
```

```diff
ecs.registerPrefab(Goblin);
ecs.registerPrefab(Player);
+ecs.registerPrefab(ScrollLightning);
```

The other interesting thing about our lightning scroll is in it's Effects component. It has and events property we will use to fire arbitrary events on an entity.

Add that to the Events and ActiveEvents components in `./src/state/components.js`.

```diff
export class ActiveEffects extends Component {
  static allowMultiple = true;
  static properties = {
    component: "",
    delta: "",
    animate: { char: "", color: "" },
+    events: [], // { name: "", args: {} },
  };
}

export class Effects extends Component {
  static allowMultiple = true;
  static properties = {
    component: "",
    delta: "",
    animate: { char: "", color: "" },
+    events: [], // { name: "", args: {} },
  };
}
```

We will of course need to update the effects system to actually fire any events. In `./src/systems/effects.js` add:

```diff
    entity[c.component].current = entity[c.component].max;
  }
}

+if (c.events.length) {
+  c.events.forEach((event) => entity.fireEvent(event.name, event.args));
+}

entity.add("Animate", { ...c.animate });

c.remove();
```

Now for a tiny refactor. In `./src/systems/movement.js` we have a kill function. Let's move that logic to the Health component's onTakeDamage event instead. This change allows our attacks and scrolls to be a bit more encapsulated. They don't have to know the health of their target or if it even survived.

In `./src/state/components.js` update the health component's onTakeDamage event:

```diff
onTakeDamage(evt) {
  this.current -= evt.data.amount;

+  if (this.current <= 0) {
+    this.entity.appearance.char = "%";
+    this.entity.remove("Ai");
+    this.entity.remove("IsBlocking");
+    this.entity.add("IsDead");
+    this.entity.remove("Layer400");
+    this.entity.add("Layer300");
+  }

  evt.handle();
}
```

And remove that logic from the movement system in `./src/systems/movement.js`:

```diff
  if (target.health.current <= 0) {
-    kill(target);

    return addLog(
      `${entity.description.name} kicked a ${target.description.name} for ${damage} damage and killed it!`
    );
```

```diff
-const kill = (entity) => {
-  entity.appearance.char = "%";
-  entity.remove("Ai");
-  entity.remove("IsBlocking");
-  entity.add("IsDead");
-  entity.remove("Layer400");
-  entity.add("Layer300");
-};
```

For our lighning scroll we will be targeting a random nearby enemy. To accomplish this we need to be able to specify a target so we can transfer effects to it from a scroll. Then our effects system can just do it's thing.

First we need two more components, `Target` and `TargetingItem`. Go ahead and add them in `./src/state/components.js`.

```javascript
export class Target extends Component {
  static properties = { locId: "" };
}

export class TargetingItem extends Component {
  static properties = { item: "<Entity>" };
}
```

And register them in `./src/state/ecs.js`

```diff
  Power,
  RequiresTarget,
+  Target,
+  TargetingItem,
} from "./components";
```

```diff
ecs.registerComponent(Power);
ecs.registerComponent(RequiresTarget);
+ecs.registerComponent(Target);
+ecs.registerComponent(TargetingItem);
```

And we will of course need a targeting system. Add a new file at `./src/systems/targeting.js`

```javascript
import ecs, { addLog } from "../state/ecs";
import { readCacheSet } from "../state/cache";

import { Target, TargetingItem } from "../state/components";

const targetingEntities = ecs.createQuery({
  all: [Target, TargetingItem],
});

export const targeting = () => {
  targetingEntities.get().forEach((entity) => {
    const { item } = entity.targetingItem;

    if (item && item.has("Effects")) {
      const targets = readCacheSet("entitiesAtLocation", entity.target.locId);

      targets.forEach((eId) => {
        const target = ecs.getEntity(eId);
        item
          .get("Effects")
          .forEach((x) => target.add("ActiveEffects", { ...x.serialize() }));
      });

      entity.remove("Target");
      entity.remove("TargetingItem");

      addLog(`You use a ${item.description.name}`);

      item.destroy();
    }
  });
};
```

All this system does is transfer Effects from the TargetingItem to the Target. Pretty simple.

With everything in place we just have to add scrolls to the map, and pick a target at random!

Start by adding scrolls to the dungeon floor in `./src/index.js`.

```diff
times(10, () => {
  const tile = sample(openTiles);
  ecs.createPrefab("HealthPotion").add(Position, { x: tile.x, y: tile.y });
});

+times(10, () => {
+  const tile = sample(openTiles);
+  ecs.createPrefab("ScrollLightning").add(Position, { x: tile.x, y: tile.y });
+});
```

Next we need to import our targeting system and create a query to get enemies in our field of vision.

```diff
import { fov } from "./systems/fov";
import { movement } from "./systems/movement";
import { render } from "./systems/render";
+import { targeting } from "./systems/targeting";
import ecs, { addLog } from "./state/ecs";
-import { Move, Position } from "./state/components";
+import { IsInFov, Move, Position, Ai } from "./state/components";
+
+const enemiesInFOV = ecs.createQuery({ all: [IsInFov, Ai] });
```

Next when an item is consumed we need to get a target if it requires one.

```diff
      if (entity) {
-        if (entity.has("Effects")) {
+        if (entity.requiresTarget) {
+          // get a target that is NOT the player
+          const target = sample([...enemiesInFOV.get()]);
+
+          if (target) {
+            player.add("TargetingItem", { item: entity });
+            player.add("Target", { locId: toLocId(target.position) });
+          } else {
+            addLog(`The scroll disintegrates uselessly in your hand`);
+            entity.destroy();
+          }
+        } else if (entity.has("Effects")) {
          // clone all effects and add to self
          entity
            .get("Effects")
            .forEach((x) => player.add("ActiveEffects", { ...x.serialize() }));
-        }

-        addLog(`You consume a ${entity.description.name}`);
-        entity.destroy();
+          addLog(`You consume a ${entity.description.name}`);
+          entity.destroy();
+        }

        if (selectedInventoryIndex > player.inventory.list.length - 1)
          selectedInventoryIndex = player.inventory.list.length - 1;

        gameState = "GAME";
      }
```

And finally, call the targeting system.

```diff
  if (playerTurn && userInput && gameState === "INVENTORY") {
    processUserInput();
+    targeting();
    effects();
    render(player);
    playerTurn = true;
```

Did you get all that? Is it working? That was a LOT. We still have 2 more scrolls to go but with so much in place already they should be a breeze!

## Paralyze scroll
