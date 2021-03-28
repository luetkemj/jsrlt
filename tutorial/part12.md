# Part 12 - Increasing Difficulty

Our dungeon is looking nice so far. We can generate rooms, levels, opponents and items.
However, it might be boring if we always fight monsters the same way.

Optimally our Dungeon will get progressively more difficult.

In order to achieve that, we can do multiple things.

1. Increase the number of monsters we face in each dungeon level.
2. change the number of items generated per level.
3. increase monster strength depending on level
4. Introduce some form of elite monsters.
5. Include traps that can harm our player.
6. Add new effects that can harm the player.

As you can see there is a lot we could do and there is much much more.

For this tutorial I suggest we start with 1. and 2. and then look how we can implement 3.
This will allow us an easier pathway, if we decide to add an experience / level-up system to the game.

Let's check out our current items and monsters
At the moment we are generating all our items and monsters in the `index.js` file when we create a dungeon level.

```js
times(5, () => {
  world.createPrefab("Goblin").add(Position, getOpenTiles(dungeon));
});

times(10, () => {
  world.createPrefab("HealthPotion").add(Position, getOpenTiles(dungeon));
});

times(10, () => {
  world.createPrefab("ScrollLightning").add(Position, getOpenTiles(dungeon));
});

times(10, () => {
  world.createPrefab("ScrollParalyze").add(Position, getOpenTiles(dungeon));
});

times(10, () => {
  world.createPrefab("ScrollFireball").add(Position, getOpenTiles(dungeon));
});
```

Our current set up isn't really supportive for what we want to do, so it's time to refactor a bit more.

The steps we want to take are something like this.

- Move Dungeon and Level creation from `index.js` to `dungeon.js`
- define on which levels monsters and items can spawn at and how many
- create a weighting system for items and monsters
- add more monsters

With that layed out, we can start by moving our dungeon generation into our `./src/lib/dungeon.js` file.
Cut the `getOpenTiles` as well as the `createDungeonLevel` functions and move them into the `dungeon.js` file.

Change the imports in the `index.js` file like so:

```diff
-import { createDungeon } from './lib/dungeon';
+import { createDungeonLevel, getOpenTiles } from './lib/dungeon';
```

After you moved the createDungeonLevel and getOpenTiles functions, sake sure that these are properly exported from `dungeon.js`.

```diff
-const getOpenTiles = (dungeon) => {
+export const getOpenTiles = (dungeon) => {
  const openTiles = Object.values(dungeon.tiles).filter(
    (x) => x.sprite === 'FLOOR'
  );
  return sample(openTiles);
};

-const createDungeonLevel = ({
+export const createDungeonLevel = ({
  createStairsUp = true,
  createStairsDown = true,
} = {}) => {
  const dungeon = createDungeon({
    x: grid.map.x,
    y: grid.map.y,
    z: readCache('z'),
    width: grid.map.width,
    height: grid.map.height,
  });
```

Now let's go ahead and create a file called `level_entities.js` at `./src/lib/level_entities.js`.

```js
export const MAX_ITEMS_BY_FLOOR = {
  // floor: items
  1: 2,
  5: 3,
  9: 5,
  14: 7,
  19: 10,
};

export const ITEM_WEIGHT = {
  // floor: [{name: ItemName, weight: number}]
  0: [{ name: "HealthPotion", weight: 35 }],
  2: [{ name: "ScrollParalyze", weight: 10 }],
  4: [{ name: "ScrollLightning", weight: 25 }],
  6: [{ name: "ScrollFireball", weight: 25 }],
};

export const MAX_MONSTERS_BY_FLOOR = {
  // floor: monsters
  1: 3,
  5: 6,
  9: 9,
  14: 12,
  19: 15,
};

export const MONSTER_WEIGHT = {
  // floor: [{name: MonsterName, weight: number}]
  0: [{ name: "Goblin", weight: 80 }],
  1: [{ name: "GoblinWarrior", weight: 15 }],
  5: [{ name: "GoblinWarrior", weight: 30 }],
  7: [{ name: "GoblinWarrior", weight: 60 }],
};
```

As you can see we define maximum numbers for items and monsters for certain floors as well as create some weighting.

All the numbers are completely arbitrary and you can adjust them to your liking.

For the weighting it's important to remember that those numbers don't need to add up to 100. All the weightings will be added up and entities are chosen via rng.
In case this doesn't make sense now, you will understand it when we implement the weighting system.

Lastly, you might have noticed that we have 'GoblinWarrior' in the monster weight. We haven't created this monster yet, so let's do that.

In `prefabs.js` we want to add the GoblinWarrior prefab.

```js
export const GoblinWarrior = {
  name: "GoblinWarrior",
  inherit: ["Being"],
  components: [
    {
      type: "Appearance",
      properties: { char: "w", color: "green" },
    },
    {
      type: "Description",
      properties: { name: "goblin warrior" },
    },
  ],
};
```

You will notice, that we inherit from our Being prefab. This means that our GoblinWarrior and our Goblin share the same base properties. The ECS allows us to easily overwrite these properties and adjust our entities to our liking.

Because it would be kind of weird to have the same monster without any differences, let's adjust our Goblin and GoblinWarrior, so they differ in strength.

```diff
export const Goblin = {
  name: 'Goblin',
  inherit: ['Being'],
  components: [
    { type: 'Ai' },
    {
      type: 'Appearance',
      properties: { char: 'g', color: 'green' },
    },
    {
      type: 'Description',
      properties: { name: 'goblin' },
    },
+    { type: 'Power', properties: { max: 2, current: 2 } },
+    { type: 'Health', properties: { max: 7, current: 7 } },
  ],
};

export const GoblinWarrior = {
  name: 'GoblinWarrior',
  inherit: ['Being'],
  components: [
    {
      type: 'Appearance',
      properties: { char: 'w', color: 'green' },
    },
    {
      type: 'Description',
      properties: { name: 'goblin warrior' },
    },
+    { type: 'Defense', properties: { max: 2, current: 2 } },
+    { type: 'Power', properties: { max: 4, current: 4 } },
  ],
};
```

Here we change the power and health of the Goblin to be a lower difficulty monster.
Our GoblinWarrior will have increased Defense and Power to provide a bigger challenge.

As usual don't forget to add the prefab to your `ecs.js` file.

```diff
import {
  Being,
  Floor,
  Goblin,
+  GoblinWarrior,
  HealthPotion,
  Item,
  Player,
  ScrollFireball,
  ScrollLightning,
  ScrollParalyze,
  StairsDown,
  StairsUp,
  Tile,
  Wall,
} from './prefabs';
```

```diff
ecs.registerPrefab(Goblin);
+ecs.registerPrefab(GoblinWarrior);
```

I would like to give our Dungeon a little bit more variation as well, depending on how deep we are.
Go ahead and create a file called `dungeon_layout.js` at `./src/lib/dungeon_layout.js`.

Make it look like this:

```js
import { grid } from "./canvas";

export const DUNGEON_LAYOUT = {
  // floor: {maxRoomCount: number, width: number, height: number}
  1: {
    maxRoomCount: 6,
    width: Math.round(grid.map.width * 0.6),
    height: Math.round(grid.map.height * 0.6),
  },
  5: {
    maxRoomCount: 12,
    width: Math.round(grid.map.width * 0.8),
    height: Math.round(grid.map.height * 0.8),
  },
  9: {
    maxRoomCount: 18,
    width: grid.map.width,
    height: grid.map.height,
  },
  14: {
    maxRoomCount: 24,
    width: grid.map.width,
    height: grid.map.height,
  },
  19: { maxRoomCount: 30, width: grid.map.width, height: grid.map.height },
};
```

Let's quickly go over this. Similar to the `level_entities.js` file, we created an object which contains the floors and each floor holds an object with maxRoomCount, width and height.

For the earlier levels, a smaller dungeon should be created, since there is less monsters and it would otherwise feel a little bit empty.

As you might recall, our createDungeon function accepts a lot of arguments.

```js
export const createDungeon = ({
  x,
  y,
  z,
  width,
  height,
  minRoomSize = 6,
  maxRoomSize = 12,
  maxRoomCount = 30,
})
```

When we pass arguments to this function, we can use the DUNGEON_LAYOUT properties to change the level based on which floor we are on.

Now the last thing I want to do before we finish up in the `dungeon.js` file is creating our utility functions.
It's not strictly necessary that you declare them outside of the `dungeon.js` file, however it cleans up our code and make it a little bit more maintainable for the future.

With that said, add the following code to your `misc.js` at `./src/utils/misc.js`.

```js
/**
 * returns highest match to a given needle, that doesn't exceed needle
 *
 * @param {Array} array
 * @param {Number} needle
 * @returns
 */
export const getHighestMatch = (array, needle) => {
  let highest_match;
  for (let floor of array) {
    if (floor > needle) {
      break;
    }
    highest_match = floor;
  }
  return highest_match;
};

/**
 * gets weighted chances for all entities based on the level provided
 * Object structure:
 *   level: [{entity, weight}, {entity, weight}] *
 *
 * @param {Object} weightedChances
 * @param {Number} level
 * @returns {Object}
 */
export const getEntityChancesForLevel = (weightedChances, level) => {
  const entityWeightedChances = {};

  for (const floor in weightedChances) {
    if (floor > level) {
      break;
    } else {
      weightedChances[floor].forEach((entity) => {
        entityWeightedChances[entity.name] = entity.weight;
      });
    }
  }
  return entityWeightedChances;
};

/**
 * gets weighted key based on values from an object
 * Object structure
 * data: {key: weighting}
 *
 * @param {object} data
 * @returns key
 */
export const getWeightedValue = (data) => {
  let total = 0;

  for (let key in data) {
    total += data[key];
  }

  let random = Math.random() * total;
  let key,
    part = 0;
  for (key in data) {
    part += data[key];
    if (random < part) {
      return key;
    }
  }

  // If by some floating-point annoyance we have
  // random >= total, just return the last id.
  return id;
};
```

A quick walkthrough on what we are doing here.

```js
export const getHighestMatch = (array, needle) => {
  let highest_match;
  for (let floor of array) {
    if (floor > needle) {
      break;
    }
    highest_match = floor;
  }
  return highest_match;
};
```

This is very simply. We pass an array and a needle to the function and get a return based on the match that doesn't exceed the needle.
The needle will be a number (in our case the current floor we are at) and the array will be where we search in (for us that will be one of the earlier defined Objects for weighting and dungeon layout).

Now comes the little bit meaty part.
In order for our weight chances to be calculated, we want to collect all entities for each floor.
We provide our object with the weighting (monsters or items) and the current floor we are on.

Then we will iterate over the object, considering every floor up to our current floor.

Then we will assign each entity to the entityWeightedChances object, with the entity name as key and the weight as value.
This will also allow us to have increasing or decreasing weighting based on the floor.

```js
export const getEntityChancesForLevel = (weightedChances, level) => {
  const entityWeightedChances = {};

  for (const floor in weightedChances) {
    if (floor > level) {
      break;
    } else {
      weightedChances[floor].forEach((entity) => {
        entityWeightedChances[entity.name] = entity.weight;
      });
    }
  }
  return entityWeightedChances;
};
```

Our returned object will look something like this, if we are at level 2.

```js
entityWeightedChances = {
  Goblin: 80,
  GoblinWarrior: 15,
};
```

Let's say we have this structure:

```js
export const MONSTER_WEIGHT = {
  0: [{ name: "Goblin", weight: 80 }],
  1: [{ name: "GoblinWarrior", weight: 15 }],
  5: [
    { name: "GoblinWarrior", weight: 30 },
    { name: "Goblin", weight: 0 },
  ],
  7: [{ name: "GoblinWarrior", weight: 60 }],
};
```

Since keys have to be unique, we would simply overwrite the weighting from level 1. Pretty neat, huh?

And this is how we produce the weighted value keys.
We pass the data argument in the structure our getEntityChancesForLevel function returned.

```js
export const getWeightedValue = (data) => {
  let total = 0;

  for (let key in data) {
    total += data[key];
  }

  let random = Math.random() * total;
  let key,
    part = 0;
  for (key in data) {
    part += data[key];
    if (random < part) {
      return key;
    }
  }

  // If by some floating-point annoyance we have
  // random >= total, just return the last id.
  return key;
};
```

First we add up all the weights and assign them to total.
Next we create a a random number, based on our total. Remember, that 0 is inclusive and the 1 is exclusive, so we will always get a value that is between 0 and total, but never equal to total.

The rest is pretty simple. We assign the weighting to part and compare it then to random.
If the random number is smaller than our weighting, then we will return the key.

And as mentioned in the comment, floating-point variance can create some issues, so we return the last key.

That's it for our weighted value system.

Alright, let's not waste any time and move straight into our `dungeon.js` file. We have a lot of things to do.

First let's take care of our imports.

```diff
-import { random, times } from 'lodash';
+import { random, sample, times } from 'lodash';
+import { readCache } from '../state/cache';
import { Position } from '../state/components';
import world from '../state/ecs';
+import { getHighestMatch, getEntityChancesForLevel, getWeightedValue } from '../utils/misc';
+import { grid } from './canvas';
+import { DUNGEON_LAYOUT } from './dungeon_layout';
import { rectangle, rectsIntersect } from './grid';
+import { ITEM_WEIGHT, MAX_ITEMS_BY_FLOOR, MAX_MONSTERS_BY_FLOOR, MONSTER_WEIGHT } from './level_entities';
```

Quite a lot of imports. You might already have imported some when moving the `createDungeonLevels` function, but I wanted to make sure, you have everything needed before we continue.

Now, in our `createDungeonLevels` function in `dungeon.js` we need to make some adjustments.

```diff
const createDungeonLevel = ({
  createStairsUp = true,
  createStairsDown = true,
} = {}) => {
+  const currentLevel = Math.abs(readCache('z'));
+  const dimensions =
+    DUNGEON_LAYOUT[getHighestMatch(Object.keys(DUNGEON_LAYOUT), currentLevel)];
  const dungeon = createDungeon({
    x: grid.map.x,
    y: grid.map.y,
    z: readCache('z'),
-    width: grid.map.width,
+    width: dimensions.width,
-    height: grid.map.height,
+    height: dimensions.height,
+    maxRoomCount: dimensions.maxRoomCount,
  });

  const openTiles = Object.values(dungeon.tiles).filter(
    (x) => x.sprite === 'FLOOR'
  );
```

Let's go over this as well, because it might look a bit confusing.
When we created our cache, we stored z as -1. This is because we are in the first underground level (below 0).
While this is great and makes perfect sense for traversing through floors, since we simply need to add 1 when ascending and subtract 1 when descending, it doesn't allow us to compare with our objects.

To alleviate this, we use the Math.abs function, which will always return a positive value.

In dimensions, we are using `getHighestMatch` to get the properties based on the level we are on. The return will always be below or equal to the floor we are on.

The values for the properties will be passed to createDungeon and we achieved dynamic floor generation based on parameters =)

We can now replace our previous item and monster generation with our dynamic generation:
First add two new functions to our `dungeon.js` file.

```js
const generateEntities = (level, maxEntitiesByFloor, entityWeight, dungeon) => {
  const amount =
    maxEntitiesByFloor[getHighestMatch(Object.keys(maxEntitiesByFloor), level)];
  getEntitiesAtRandom(entityWeight, amount, level, dungeon);
};

const getEntitiesAtRandom = (
  weightedChances,
  amountOfEntities,
  currentLevel,
  dungeon
) => {
  const entityWeightedChances = getEntityChancesForLevel(
    weightedChances,
    currentLevel
  );

  times(Math.floor(Math.random() * amountOfEntities + 1), () => {
    let prefab = getWeightedValue(entityWeightedChances);
    world.createPrefab(prefab).add(Position, getOpenTiles(dungeon));
  });
};
```

These two functions simply use all the logic we implements in our `misc.js` file earlier.
`getEntitiesAtRandom` is responsible for randomly generating and entities based on the floor we are on.

`generateEntities` passes all necessary information to getEntitiesAtRandom and removes some of the implementation detail.
Notice, that we pass the dungeon, to get our open tiles. This is very important, otherwise we can't add the position to the entities.

With that done, we can adjust the code like this:

```diff
  const openTiles = Object.values(dungeon.tiles).filter(
    (x) => x.sprite === 'FLOOR'
  );

-  times(5, () => {
-    world.createPrefab('Goblin').add(Position, getOpenTiles(dungeon));
-  });
-
-  times(10, () => {
-    world.createPrefab('HealthPotion').add(Position, getOpenTiles(dungeon));
-  });
-
-  times(10, () => {
-    world.createPrefab('ScrollLightning').add(Position, getOpenTiles(dungeon));
-  });
-
-  times(10, () => {
-    world.createPrefab('ScrollParalyze').add(Position, getOpenTiles(dungeon));
-  });
-
-  times(10, () => {
-    world.createPrefab('ScrollFireball').add(Position, getOpenTiles(dungeon));
-  });

+  generateEntities(currentLevel, MAX_ITEMS_BY_FLOOR, ITEM_WEIGHT, dungeon);
+  generateEntities(currentLevel, MAX_MONSTERS_BY_FLOOR, MONSTER_WEIGHT, dungeon);

  let stairsUp, stairsDown;
```

That looks a lot cleaner and is very open for extension.
We have done quite a lot this time. Refactoring our dungeon creation as well as our item and monster generation is no small task.

I really hope you enjoyed this one as well.

The next part will cover how we can gear up our player and make him stronger.
