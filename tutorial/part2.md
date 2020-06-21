# Part 2 - The generic Entity, the render functions, and the map

For this tutorial we will be using an ECS architecture. Yes, this is probably overkill for the size of our project but I have found it to be the easiest architecture to reason about once my games reach anything beyond basic complexity. We won't be writing our own ECS engine, we will instead be relying on the excellent geotic for that. But before we install anything and start refactoring our game we should probably talk a bit about what an ECS architecture is and why you would choose to use one.

Games are complicated!

My first 4 or 5 games used all sorts of ways to manage state. Every one of those games eventually fell apart when adding new features became too complex. I needed a simple way to manage that complexity.

With ECS, complexity arises from simplicity.

- yogi picture

Ok Yoda, not helpful. What the heck is it?

Thomas Biskup's 40 minute talk, [There be dragons: Entity Component Systems for Roguelikes](https://www.youtube.com/watch?v=fGLJC5UY2o4&feature=youtu.be) at the 2018 Rogulike Celebration is a great primer from someone much smarter than I.

Wikipedia a more formal explanation:

> ECS follows the composition over inheritance principle that allows greater flexibility in defining entities where every object in a game's scene is an entity (e.g. enemies, bullets, vehicles, etc.). Every entity consists of one or more components which contains data or state. Therefore, the behavior of an entity can be changed at runtime by systems that add, remove or mutate components. This eliminates the ambiguity problems of deep and wide inheritance hierarchies that are difficult to understand, maintain and extend. - [wikipedia](https://en.wikipedia.org/wiki/Entity_component_system)

But at it's core ECS is just a way to manage application state. State is stored in components, entities are collections of those components with a unique id, and systems run logic on those entities in order to add, remove, and mutate their components.

As our game grows in scope I think you will find that these 3 simple parts will help to manage the underlying complexity. Leaving us to follow our inspiration and just make a game!

Enough talk, let's install geotic, create our first entity and learn how to do things the ECS way!

---

As our game grows in complexity I think you will find that

I find that as the game grows in scale and complexity, the simple parts of ECS allow me to continue to reason about it. Complexity arises from simplicity.

It is composes of 3 simple idea constructs. The entity, the component, and the system.

Complexity arises from simplicity.

I find it's easier to reason about features addition tends to scale linearly

I find it's a lot easier to reason about and as the game becomes more complex, features do not become more

Our game is really only made up of two very basic elements that are absolutely required for it to function - state and render. Our render function is designed to render to the screen a snapshot of our current game state. As our game gets more complex so does the state of it. Different parts of states begin to interact with other bits of it based on the state of some other seemingly unrelated but...

At it's core our game is just application state and a function to render a representation of that state to the screen up 60 times a second. When we move our hero on the screen as we did in the last part we were just minipulating application state. We add weapons armor and health to our player all through application state. But once we introduce enemies who also have weapons, armor, and health we find that we may want to share how we implem.

The inherant challenge then to manage that state such that you can reason about it when bugs arise and add features to

Our game logic is written in javascript and rendered on an html canvas.

Entity, Component, Render System and the map

What is ECS?
Why ECS?
How to think the ECS way.

In an ECS you have a few ways to modify the

Refactor current project to use geotic
And make the map
