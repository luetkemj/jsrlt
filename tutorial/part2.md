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
