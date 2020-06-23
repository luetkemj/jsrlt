# Part 1 - Drawing the ‘@’ symbol and moving it around

We're going to use html canvas to draw the `@` symbol - and the rest of our game.

To start we need to add a canvas element to our html. Go ahead and do that now in `./index.html`

```diff
  <body>
-    <p>Gobs O' Goblins</p>
+    <canvas id="canvas"></canvas>
    <div class="build">Build: <%= htmlWebpackPlugin.options.version %></div>
  </body>
```

We'll also need to add some styles so we can actually see what's going on.

```diff
<html>
  <head>
    <meta charset="utf-8" />
    <title><%= htmlWebpackPlugin.options.title %></title>
+    <style>
+      canvas {
+        background: #000;
+      }
+    </style>
  </head>
```

Alright, now that we can see our canvas - that little black rectangle in the top left - we can get to drawing our hero!

First, we need to create a new folder `./src/lib` to store the code for our canvas. We will be storing most of our miscellaneous logic in here. Next create a new file `./src/lib/canvas.js` and make it look like this:

```javascript
const pixelRatio = window.devicePixelRatio || 1;
const canvas = document.querySelector("#canvas");
const ctx = canvas.getContext("2d");

export const grid = {
  width: 100,
  height: 34,
};

const lineHeight = 1.2;

let calculatedFontSize = window.innerWidth / grid.width;
let cellWidth = calculatedFontSize * pixelRatio;
let cellHeight = calculatedFontSize * lineHeight * pixelRatio;
let fontSize = calculatedFontSize * pixelRatio;

canvas.style.cssText = `width: ${calculatedFontSize * grid.width}; height: ${
  calculatedFontSize * lineHeight * grid.height
}`;
canvas.width = cellWidth * grid.width;
canvas.height = cellHeight * grid.height;

ctx.font = `normal ${fontSize}px Arial`;
ctx.textAlign = "center";
ctx.textBaseline = "middle";

export const drawChar = ({ char, color, position }) => {
  ctx.fillStyle = color;
  ctx.fillText(
    char,
    position.x * cellWidth + cellWidth / 2,
    position.y * cellHeight + cellHeight / 2
  );
};
```

Ok. That's a lot. Let's a go over it real quick.

```javascript
const pixelRatio = window.devicePixelRatio || 1;
```

This gives us access to the pixel ratio of the user's machine and will make our game nice and crisp no matter the resolution of our user.

```javascript
const canvas = document.querySelector("#canvas");
const ctx = canvas.getContext("2d");
```

Here we are storing a reference to the canvas element in our html and then getting the context with the canvas API. You can read more about the canvas API [here](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API).

```javascript
export const grid = {
  width: 100,
  height: 34,
};
```

In this next bit we set the dimensions of our grid. It will be 100 characters wide and 34 high.

After that we tweak alot of settings in our canvas so that it will always fill the browser. You won't have to touch much of this again so it's not super important that you understand every line. But again, if you want to your can go read the [docs](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API).

Finally we have our function to actually draw something on the canvas!

```javascript
export const drawChar = ({ char, color, position }) => {
  ctx.fillStyle = color;
  ctx.fillText(
    char,
    position.x * cellWidth + cellWidth / 2,
    position.y * cellHeight + cellHeight / 2
  );
};
```

With it we can set the character, color, and position of any cell on our grid. Let's give it a go!

To do that we need to import our script to `./src/index.js`.

```diff
-console.log("Hello World!");
+import "./lib/canvas.js";
```

If you check out the game in your browser you should see that the canvas is much larger. All we have to do now is import the drawChar function and use it! Make `./src/index.js` look like this:

```diff
import "./lib/canvas.js";
+import { drawChar } from "./lib/canvas";
+
+drawChar({ char: "@", color: "#FFF", position: { x: 0, y: 0 } });
```

Our hero has arrived!

At this point you should see your "@" in the top left of the canvas. You can experiement with it's position, color, and character. We're a long ways from a game but still, pretty neat!

This is a good time to save your progress, push your changes to git, and even run deploy again to show the world the amazing progress you're making :)

See step 0 of this tutorial if you need a reminder save your changes and deploy.

---

Let's take a step back to tidy some things up before moving on to moving our hero around.

First we have some styling to do. Delete this entire block in `./index.html`

```diff
-    <style>
-      canvas {
-        background: #000;
-      }
-    </style>
```

And replace it with this:

```html
<link
  href="https://fonts.googleapis.com/css2?family=Fira+Code&display=swap"
  rel="stylesheet"
/>
<style>
  body {
    background-color: #000;
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 100vw;
    height: 100vh;
    margin: 0 0;
    position: relative;
  }

  canvas {
    background: #000;
    margin-top: 30px;
  }

  .build {
    color: #444;
    font-size: 12px;
    font-family: "Fira Code", monospace;
    text-align: left;
    width: 100%;
    position: absolute;
    bottom: 0;
  }
</style>
```

We're adding some styles to position our canvas properly, get rid of any extra spacing around it, and normalize the colors. You'll also notice a link to google fonts. I'll be using a font called Fira Code, you're welcome to pick something else if you'd like but I like this one.

We also need to update the font in `./src/lib/canvas`.

```diff
canvas.height = cellHeight * grid.height;

-ctx.font = `normal ${fontSize}px Arial`;
+ctx.font = `normal ${fontSize}px 'Fira Code'`;
ctx.textAlign = "center";
```

If you pick a different font you will need to change it in the style block above and canvas.js

---

Time to move!

If you played around with the draw function you may have noticed that we'll just have to adjust the x and y coordinates to move our "@" around.

First, let's make an object to store our player and then pass that to drawChar.

```diff
-drawChar({ char: "@", color: "#FFF", position: { x: 0, y: 0 } });
+const player = {
+  char: "@",
+  color: "white",
+  position: {
+    x: 0,
+    y: 0,
+  },
+};
+
+drawChar(player);
```

Nothing in the game has really changed but this refactor will help in just a minute.

Now let's add an event listener so we can do stuff when a user presses a key.

Add this at the bottom of the file:

```javascript
document.addEventListener("keydown", (ev) => {
  console.log(ev.key);
});
```

Now if you open your browser's developer console and start typing on the game window again you will see each key as you type logged to the console!

Let's use that to our advantage and move our hero when the arrow keys are pressed. Replace the event listener we just created with the follow:

```javascript
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

  drawChar(player);
};
```

Now, we store every keypress in `userInput` so we can process it with `processUserInput`. This function just checks what key was pressed and if it's an arrow key we update the player position and finally call drawChar again. This is where you could change the keybindings to wasd or ghjk or whatever you want, but let's test it first and see if our hero is moving!

...

Hmmmmm... looks like we may have a bug - or a snake! The problem is we aren't clearing the canvas between renders. Let's add a function to our canvas lib to do just that.

Add this to the bottom of `./src/lib/canvas.js`

```javascript
export const clearCanvas = () =>
  ctx.clearRect(0, 0, canvas.width, canvas.height);
```

Now we can import our new function to `./src/index.js` and call it before drawChar.

```diff
-import { drawChar } from "./lib/canvas";
+import { clearCanvas, drawChar } from "./lib/canvas";
```

```diff
  }

+ clearCanvas()
  drawChar(player);
};
```

WooHoo! We did it! You should be proud of yourself, this is big step on our way towards making a real game. You have gained experience and leveled up!

Go to [part 2](https://github.com/luetkemj/jsrlt/edit/master/tutorial/part1.md) where we do all again the ECS way!
