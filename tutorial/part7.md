#Part 7 - Creating the Interface

We're getting closer and closer to a playable game but before we add additional gameplay mechanics we need to take a step and focus on the UI. We're going to add 3 elements to our game to improve the interface. A HUD with player stats, an adventure log, and an info bar.

To start, open up `./src/lib/canvas.js` and add some new properties to the grid object. These settings detail the width, height and location of each of our new elements. When complete the grid object will look like this:

```javascript
export const grid = {
  width: 100,
  height: 34,

  map: {
    width: 79,
    height: 29,
    x: 21,
    y: 3,
  },

  messageLog: {
    width: 79,
    height: 3,
    x: 21,
    y: 0,
  },

  playerHud: {
    width: 20,
    height: 34,
    x: 0,
    y: 0,
  },

  infoBar: {
    width: 79,
    height: 3,
    x: 21,
    y: 32,
  },
};
```

Next we'll add a new function called `drawText`. This new function will accept a template that includes a text string and color and position options. It splits the string into characters and builds entity like objects it can pass to our existing `drawCell` function.

```javascript
export const drawText = (template) => {
  const textToRender = template.text;

  textToRender.split("").forEach((char, index) => {
    const character = {
      appearance: {
        char,
        background: template.background,
        color: template.color,
      },
      position: {
        x: index + template.x,
        y: template.y,
      },
    };

    delete template.x;
    delete template.y;

    drawCell(character, template);
  });
};
```

With the foundation all in place, we can add our HUD which for now will just display our player name and a health bar.

In our render system `./src/systems/render.js` we'll need to import our new function and the grid settings:

```diff
-import { clearCanvas, drawCell } from "../lib/canvas";
+import { clearCanvas, drawCell, drawText, grid } from "../lib/canvas";
```

We will also need some info from the player entity itself. As with other systems we can pass that in as an argument:

```diff
-export const render = () => {
+export const render = (player) => {
```

Remember to actually pass it in from `./src/index.js` wherever render is called. There should be three calls to the render function - pass in player to each like this:

```diff
-render();
+render(player);
```

Next we can use the new drawText function right after our layers have been rendered:

```diff
  layer400Entities.get().forEach((entity) => {
    if (entity.isInFov) {
      drawCell(entity);
    } else {
      drawCell(entity, { color: "#100" });
    }
  });

+  drawText({
+    text: "@ You",
+    background: "#000",
+    color: "white",
+    x: 0,
+    y: 0,
+  });
};
```

If you try our the game now you should see the player name in the top left of the screen!

Let's add a health bar just below it.

We're going to take advantage of canvas' simplistic nature. Remember it just renders sequentially what you want, where you want. With that in mind - let's render a grayed out version of our health bar below our color version that will become shorted as our health diminishes. This will in turn reveal the grayed out bar.

Render the gray version of our health bar

```javascript
drawText({
  text: "♥".repeat(grid.playerHud.width),
  background: "black",
  color: "#333",
  x: 0,
  y: 1,
});
```

Calculate player health as a percentage of max and generate a string of html heart characters of the calculated length:

```javascript
// player health bar
const hp = player.health.current / player.health.max;

if (hp > 0) {
  drawText({
    text: "♥".repeat(hp * grid.playerHud.width),
    background: "black",
    color: "red",
    x: 0,
    y: 1,
  });
}
```

Give it a go! Attack some goblins and you should start to see your health bar diminish!

Onwards to the adventure log!

Adventure logs play a crucial role in roguelikes. With such minimal graphics they give the player insight into the mechanics of each turn. We will start by logging the combat mechanics from the last part.

In `./src/state/ecs.js` we will add an array to store all our messages and a simple function to add new messages to it.

Notice we are using the unshift method on the array instead of the more commonly used push. This is because we want to put the messages at the beginning of our array instead of the end. The reason for this will become clear later.

```diff
ecs.registerPrefab(Player);

+export const messageLog = ["", "Welcome to Gobs 'O Goblins!", ""];
+export const addLog = (text) => {
+  messageLog.unshift(text);
+  console.log(messageLog);
+};

export default ecs;
```

Now in our movement system we can import our `addLog` function and change all of our console logs to addLogs.

```diff
-import ecs from "../state/ecs";
+import ecs, { addLog } from "../state/ecs";
import { addCacheSet, deleteCacheSet, readCacheSet } from "../state/cache";
import { grid } from "../lib/canvas";
import { Move } from "../state/components";
```

```diff
const attack = (entity, target) => {
  if (target.health.current <= 0) {
    kill(target);

-    return console.log(
+    return addLog(
      `${entity.description.name} kicked a ${target.description.name} for ${damage} damage and killed it!`
    );
  }

-  console.log(
+  addLog(
    `${entity.description.name} kicked a ${target.description.name} for ${damage} damage!`
  );
};
```

```diff
export const movement = () => {
if (target.has("Health") && target.has("Defense")) {
  attack(entity, target);
} else {
-  console.log(
+  addLog(
    `${entity.description.name} bump into a ${target.description.name}`
  );
}
```

Finally we need to actually render the log in our render system `./src/systems/render.js`

We are going to simply render the first three messages of our log. As new messages are added to the beginning, older ones will fall off. They will still be stored in the array - we just won't render them.

First import `messageLog` to our render system:

```javascript
import { messageLog } from "../state/ecs";
```

Then, right after our health bar add the following:

```javascript
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
```

You'll notice we rendered each line with a different color - this is provides a bit of a gradient to our logs giving our newest messages more emphasis.

Walk around - bump into stuff - test it out!

The last thing we will do is give the player the ability to mouse over the map and get some information about entities in any visible cell.

We need to import a few things to start:

```diff
+import { throttle } from "lodash";
```

```diff
-import { clearCanvas, drawCell, drawText, grid } from "../lib/canvas";
+import { clearCanvas, drawCell, drawText, grid, pxToCell } from "../lib/canvas";
+import { toLocId } from "../lib/grid";
+import { readCacheSet } from "../state/cache";
```

With that out of the way, add this to the very end of `./src/systems.render.js` **outside** of the render function itself:

```javascript
// info bar on mouseover
const clearInfoBar = () =>
  drawText({
    text: ` `.repeat(grid.infoBar.width),
    x: grid.infoBar.x,
    y: grid.infoBar.y,
    background: "black",
  });

const canvas = document.querySelector("#canvas");
canvas.onmousemove = throttle((e) => {
  const [x, y] = pxToCell(e);
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
}, 100);
```

There is a lot happening there - let's go through it.

First we have a utility function to clear the info bar that we'll use later. It doesn't actually clear the info bar so much as it draws over it.

```javascript
const clearInfoBar = () =>
  drawText({
    text: ` `.repeat(grid.infoBar.width),
    x: grid.infoBar.x,
    y: grid.infoBar.y,
    background: "black",
  });
```

Next we just grab a reference to the canvas dom element so we can add an onmousemove listener to it.

```javascript
const canvas = document.querySelector("#canvas");
```

We throttle the event here to prevent it from being called as fast as possible to help with performance. The code between the braces should only run once every 100ms while the mouse is moving over the canvas.

```javascript
canvas.onmousemove = throttle((e) => {
  ...
}, 100);
```

Once in the braces we use the mouse event to calculate our location on the grid which is used to access any entities at the current location. We also clear the info bar.

```javascript
const [x, y] = pxToCell(e);
const locId = toLocId({ x, y });

const esAtLoc = readCacheSet("entitiesAtLocation", locId) || [];
const entitiesAtLoc = [...esAtLoc];

clearInfoBar();
```

If there are any entities at the current location we use our layer queries to access only those that are visible. We then write a message for each entity - overwriting the previous to make sure only the top layer shows in the end. We also take care to write a subtly different message for entities we can currenlty see and those we have discovered.

```javascript
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
```

There is a lot of room here for customizing the messages to make this your own. But this should give you a head start towards a usable UI that gives the player needed insight into the mechanics of your game in a relatively pleasing manor.
