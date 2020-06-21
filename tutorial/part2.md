# Part 2 - The generic Entity, the render functions, and the map

For this tutorial we will be using an ECS architecture. A lot of folks will probably say this is overkill for the size of our project but I have found it to be the easiest architecture to reason about once games reach anything beyond basic complexity. If you choose to expand on this after the tutorial you'll have a decent base. We won't be writing our own ECS engine, we will instead be relying on the excellent [geotic](https://github.com/ddmills/geotic) for that. But before we install anything and start refactoring our game we should probably talk a bit about what an ECS architecture is and why you would choose to use one.

To state the obvious, games are complicated! My first 4 or 5 experimented with all sorts of ways to manage state. Every one of those games started simple and eventually fell apart when adding new features became too complex.

You shouldn't have to write complex code to do complex things. With ECS, complexity arises from simplicity. Follow a few simple rules; get complex behavior.

For a detailed overview of ECS in practice Thomas Biskup (ADOM) gave a great talk at the 2018 Roguelike Celebration. [There be dragons: Entity Component Systems for Roguelikes](https://www.youtube.com/watch?v=fGLJC5UY2o4&feature=youtu.be)

For a formal and rather dry definition we can turn to wikipedia:

> ECS follows the composition over inheritance principle that allows greater flexibility in defining entities where every object in a game's scene is an entity (e.g. enemies, bullets, vehicles, etc.). Every entity consists of one or more components which contains data or state. Therefore, the behavior of an entity can be changed at runtime by systems that add, remove or mutate components. This eliminates the ambiguity problems of deep and wide inheritance hierarchies that are difficult to understand, maintain and extend. - [wikipedia](https://en.wikipedia.org/wiki/Entity_component_system)

At it's core ECS is just a way to manage your application state. State is stored in components, entities are collections of those components, and systems run logic on those entities in order to add, remove, and mutate their components.

As our game grows in scope I think you will find that these 3 simple rules will help to manage the underlying complexity leaving us to follow our inspiration and just make a game!

Enough talk, let's install geotic, create our first entity and learn how to do things the ECS way!

---

Before we get into it, go ahead and install geotic and start the engine.

```bash
npm install geotic
```

Next let's create a new folder `./src/state` and add a file to it `./src/state/ecs.js`. At the top of `ecs.js` we need to import the Engine from geotic and start it up.

```javascript
import { Engine } from "geotic";

const ecs = new Engine();

export default ecs;
```

Currently we store all of the state of our hero in the player object. To move around we directly mutate the postion values. Right now everything is very simple and easy to understand. Unfortunately This pattern won't scale very well if we were to add an NPC, a few monsters, maybe an item or two... mutating state directly like this very quickly becomes cumbersome, complicated, and prone to bugs that are hard to diagnose.

We're going to refactor our game to do everything it already does - draw the @ symbol and move it around - the ECS way. At first it's going to seem like a lot of code. The benefit though is that as we add NPCs, monsters, items, the complexity of our code won't explode.

To start let's look at our player object:

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

It just stores the state of our player's appearance (char, color) and position. Remember, components are just generic objects to store bits of state. Let's go ahead then and make some generic components to store appearance and postion.

First create a file to hold our components `./src/state/components.js`. In a larger application you may want to create individual files for each component but this is fine for our purposes.

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
```

We're ready to make our first Entity!

Just below where we register our components in `./src/state/ecs.js` we can create an empty entity for our player.

```javascript
const player = ecs.createEntity();
```

Next we need to add components to our player.

```javascript
player.add(Appearance, { char: "@", color: "#fff" });
player.add(Position);
```

The add method takes two arguments, a component, and a properties object to override any of the static defaults. You'll notice we don't pass a second argument when we add the Position component because the default properties are just fine.

Finally, we need to export `ecs` so we can actually add it to our game. At this point `./src/state/ecs.js` should look like this:

```javascript
import { Engine } from "geotic";
import { Appearance, Position } from "./components";

const ecs = new Engine();

// all Components must be `registered` by the engine
ecs.registerComponent(Appearance);
ecs.registerComponent(Position);

const player = ecs.createEntity();

player.add(Appearance, { char: "@" });
player.add(Position);

export default ecs;
```

Ok let's add the engine to our game! In `./src/index.js` go ahead and import it like so:

```diff
import "./lib/canvas.js";
+import ecs from "./state/ecs";
import { clearCanvas, drawChar } from "./lib/canvas";
```

Cool! We now have the (E)ntity and (C)omponent parts of ECS but what about the (S)ystem? We've seen how geotic provides classes for entities and components but it doesn't do that for systems. A system is just a function that iterates over a collection of entities. You can do whatever you want/need to in one but how you implement them is up to you.

Our first system will render our player entity to the screen. Let's make a new folder called system and a file inside it called render.js at `./src/systems/render.js`. We could have our systems iterate over every single entity at every tick but as you can imagine that's pretty inefficient. We can narrow the focus of our systems with geotic queries. A query is an always up to date set of entities that meet a required set of parameters.

Make `./src/systems/render.js` look like this:

```javascript
import ecs from "../state/ecs";
import { Appearance, Position } from "../state/components";

const renderableEntities = ecs.createQuery({
  all: [Position, Appearance],
});
```

`renderableEntities` will keep track of all entities that contain both the Position _and_ Appearance components. Let's use this query to loop through all renderableEntities and log the result to our javascript console. Go ahead and add the actual system at the end `./src/systems/render.js`.

```javascript
export const render = () => {
  renderableEntities.get().forEach((entity) => {
    console.log(entity);
  });
};
```

Next we need to call our render system. At the top of `./src/index.js` import the system like this:

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

Start the game with `npm start` if it's not already running and open up your browser's javascript console. Now every time you kit a key you should see all renderableEntities logged to the console. There's only one so far but I like to do this to prove that things are working as I expect them to.

The javascript console is the first place to look when things aren't working properly. Errors, warnings and other information will appear here.

Now that we have all the parts of ECS setup, lets make them do something useful and actually draw the "@" symbol to the screen.
