# Part 13 - Gearing up

This is the final part of the tutorial. Most roguelikes share a commonality between each other, which is gear.
Making your character stroker over time and help him traverse the dungeon you set up. You

By far not an easy feat and I thought it would be a good opportunity to show you various things you can do to enhance the players experience.

There will need to be some clean up as well, but we are mainly implementing new features.

So let's get right on to it.

We'll start in our `components.js` file.

Our player should be able to equip items, but currently everything is stored in the inventory.
Equipment, unlike consumable items, should remain on the player permanently and one way to achieve that is with equipment slots.

```js
export class EquipmentSlot extends Component {
  static allowMultiple = true;
  static keyProperty = "name";

  static properties = {
    name: "",
    itemId: this.itemId,
  };

  get item() {
    return this.world.getEntity(this.itemId);
  }

  set item(entity) {
    return (this.itemId = entity.id);
  }

  onEquip(evt) {
    // TODO
  }

  onUnequip(evt) {
    //T ODO
  }
}
```

This probably reminds you a little bit of the inventory we implemented earlier.
The difference here is that these slots don't store items, but reference them.

For this we create a getter and setter method. The getter return the entity from the world object and the setter associates the entity id to the slot. Fairly simple.

As you remember, geotic allows us to simple use `on` in conjunction with a word to create an event.

We leave the two events, `onEquip` and `onUnequip` for later, as we need to implement more things before we can work on those.

Our items will need to have some additional properties too.

First we add three new components:

```js
export class Dropped extends Component {}

export class IsEquippable extends Component {}
export class IsEquipped extends Component {}
```

That makes it easy to identify what is a consumable, what is equipped, as well as if we dropped an item.

In game design you should always ask the question why you should do something.
Why equip items?

The answer is pretty straight forward. You want to get some sort of benefit or effect from these items.

This also resolves what we should do next. Implement effects for our equipment.
Those effects share some similarities with the ActiveEffects we implemented a while ago.

```js
export class EquipmentEffect extends Component {
  static allowMultiple = true;
  static properties = {
    component: "",
    delta: "",
  };
}
```

Here we simply want to be able to tell which component is influenced and what values they influence.
The allowMultiple property will let us set more than one effect on an item, which is pretty nice indeed.

Perfect time to change how we described our `Stat` components.
Instead of max, we should set it to base.

```diff
export class Defense extends Component {
-  static properties = { max: 1, current: 1 };
+  static properties = { base: 1, current: 1 };
}


```

```diff
export class Health extends Component {
-  static properties = { max: 10, current: 10 };
+  static properties = { base: 10, current: 10 };

  onTakeDamage(evt) {
    this.current -= evt.data.amount;

    if (this.current <= 0) {
      if (this.entity.has(Ai)) {
        this.entity.remove(this.entity.ai);
      }
```

```diff
export class Power extends Component {
-  static properties = { max: 5, current: 5 };
+  static properties = { base: 5, current: 5 };
}
```

Awesome. That almost concludes our work in the `components.js` file.

Now we can go back to the `EquipmentSlot` and finish implementing our events.

```diff
  set item(entity) {
    return (this.itemId = entity.id);
  }

  onEquip(evt) {
-    // TODO
+    if (!evt.data.equipmentEffect) return;
+    evt.data.equipmentEffect.forEach((effect) => {
+      if (effect.component === 'health') {
+        this.entity[effect.component].base += effect.delta;
+      }
+      this.entity[effect.component].current += effect.delta;
+    });
  }

  onUnequip(evt) {
-    // TODO
+    if (!evt.data.equipmentEffect) return;
+    evt.data.equipmentEffect.forEach((effect) => {
+      if (effect.component === 'health') {
+        this.entity[effect.component].base -= effect.delta;
+      }
+      this.entity[effect.component].current -= effect.delta;
+    });
  }
}
```

This one is also fairly simple.
Same as with our inventory, we pass the event data to our method.
We quickly check if there is the `equipmentEffect` property on the item and if not, simply return.

If there are effects on the item, we iterate through them and check if the component is 'health'. I opted for health to be an influence on both the base and the current, meaning when you equip an item with health, it will add them to your current health as well.

The base is used to show empty hearts in our rendering.

Finally we add the effect delta to our components.
A minus value as delta will substract, so we don't have to worry about a separate implementation for that.

You could add additional checks, like if values go below 0, but I think that's a bit overkill for us now. If you equip an item that brings your life below 0 your character simply dies. MUAAHAHAAA.

Jokes aside. The `unEquip` method will just reverse what we do in the `equip` method.

Great. Now one last thing and we leave the components and move on.

```js
export class Slot extends Component {
  static properties = { name: "" };
}
```

This component will live on our item, to tell where it needs to go later on.

As always, register the components in the `ecs.js`. I will do that later, when we are finished with the prefabs.

Which brings us to the `prefabs.js`.

Start with implementing our base.

```diff
export const Item = {
  name: 'Item',
  components: [
    { type: 'Appearance' },
    { type: 'Description' },
    { type: 'Layer300' },
    { type: 'IsPickup' },
  ],
};

+export const Gear = {
+  name: 'Gear',
+  inherit: ['Item'],
+  components: [{ type: 'IsEquippable' }],
+};
```

We will create a new type called `Gear` and inherit from item, since that what the gear is at the core. Adding the component of `IsEquippable` will be the major difference.

Now onwards to the actual equipment. My implementation is very standard, but you can be as fancy and creative as you want.

```js
export const Weapon = {
  name: "Weapon",
  inherit: ["Gear"],
  components: [
    { type: "Appearance", properties: { char: "⚔", color: "#0066ff" } },

    {
      type: "Description",
      properties: { name: "weapon" },
    },
    {
      type: "Slot",
      properties: {
        name: "weapon",
      },
    },
  ],
};

export const Armor = {
  name: "Armor",
  inherit: ["Gear"],
  components: [
    { type: "Appearance", properties: { char: "🛡", color: "#ff99ff" } },

    {
      type: "Description",
      properties: { name: "armor piece" },
    },
  ],
};
```

Those are our two base layouts. We have a Weapon and Armor.
Notice that the Weapon has the `Slot` component attached we implemented earlier.

Reason for that is, that I use `Weapon` as the actual item to implement, while `Armor` is a base type.

Let's continue.

```js
export const Chest = {
  name: "Chest",
  inherit: ["Armor"],
  components: [
    {
      type: "Description",
      properties: { name: "chest piece" },
    },
    {
      type: "Slot",
      properties: {
        name: "chest",
      },
    },
  ],
};

export const Helmet = {
  name: "Helmet",
  inherit: ["Armor"],
  components: [
    {
      type: "Description",
      properties: { name: "helmet" },
    },
    {
      type: "Slot",
      properties: {
        name: "head",
      },
    },
  ],
};

export const Shield = {
  name: "Shield",
  inherit: ["Armor"],
  components: [
    {
      type: "Description",
      properties: { name: "shield" },
    },
    {
      type: "Slot",
      properties: {
        name: "shield",
      },
    },
  ],
};

export const Boots = {
  name: "Boots",
  inherit: ["Armor"],
  components: [
    {
      type: "Description",
      properties: { name: "boots" },
    },
    {
      type: "Slot",
      properties: {
        name: "legs",
      },
    },
  ],
};
```

Here are all the other armor pieces. They each have a different slot and names and as you can see, you could implement SOOO much more. You could have items that have implicit modifiers, such as resistances, regenerating hp and so on. Composition > inheritance =)

As mentioned, I kept it fairly simple for this bit, but only because I want to show you how to create randomized items a little bit further on.

But just for good measure, here is an implementation that shows how the `EquipmentEffect` would work

```js
export const Cloak = {
  name: "Cloak",
  inherit: ["Armor"],
  components: [
    {
      type: "Description",
      properties: { name: "cloak" },
    },
    {
      type: "Slot",
      properties: {
        name: "chest",
      },
    },
    {
      type: "EquipmentEffect",
      properties: {
        component: "Defense",
        delta: 1,
      },
    },
  ],
};
```

Now before we move on and register all these prefabs, we need to make small adjustments to our `Beings`.

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
-    { type: 'Power', properties: { max: 2, current: 2 } },
-    { type: 'Health', properties: { max: 7, current: 7 } },
+    { type: 'Power', properties: { base: 2, current: 2 } },
+    { type: 'Health', properties: { base: 7, current: 7 } },
  ],
};

export const GoblinWarrior = {
  name: 'GoblinWarrior',
  inherit: ['Being'],
  components: [
+    { type: 'Ai' },
    {
      type: 'Appearance',
      properties: { char: 'w', color: 'green' },
    },
    {
      type: 'Description',
      properties: { name: 'goblin warrior' },
    },
-    { type: 'Defense', properties: { max: 2, current: 2 } },
-    { type: 'Power', properties: { max: 4, current: 4 } },
+    { type: 'Defense', properties: { base: 2, current: 2 } },
+    { type: 'Power', properties: { base: 4, current: 4 } },
  ],
};
```

If you forgot to attach the `Ai` component to the `GoblinWarrior` like I did, this is the perfect time to remedy this issue.

Speaking of perfect timing. Our Components and prefabs are growing quite rapidly.
You should think about how to refactor those into more manageable components.

Here is a little example of what you could do.
Create a new folder `components` at `./src/state/components` and inside create a file called `index.js`.

### ./src/state/components/index.js

```js
export { Ai } from "./Ai";
export { Animate } from "./Animation";
export { ActiveEffects, Effects } from "./Effects";
export { Inventory } from "./Inventory";
export { Layer100, Layer300, Layer400 } from "./Layers";
export { Appearance, Description } from "./Misc";
export { Move } from "./Move";
export { Position } from "./Position";
```

Here you define the exports for all the components.

With that done, you can move your components in their own idividual files grouped however you like. I will demonstrate that on the `Effects` file.

### ./src/state/components/Effects.js

```js
import { Component } from "geotic";

const effectProps = {
  component: "",
  delta: "",
  animate: { char: "", color: "" },
  events: [], // { name: "", args: {} },
  addComponents: [], // { name: '', properties: {} }
  duration: 0, // in turns
};
export class ActiveEffects extends Component {
  static allowMultiple = true;
  static properties = effectProps;
}

export class Effects extends Component {
  static allowMultiple = true;
  static properties = effectProps;
}
```

When moving the components, the only major thing to remember is to import geotic, specifically the `Component`.

This is completely up to you when and if you want to do it. I just mention it, because I personally like working in a well organized code base.

Where were we? Right, registering our components and prefabs in the `ecs.js`.

```diff
import { Engine } from 'geotic';
import {
  ActiveEffects,
  Ai,
  Animate,
  Appearance,
  Defense,
  Description,
+  Dropped
  Effects,
+  EquipmentEffect,
+  EquipmentSlot,
  Health,
  Inventory,
  IsBlocking,
  IsDead,
+  IsEquippable,
+  IsEquipped,
  IsInFov,
  IsOpaque,
  IsPickup,
  IsRevealed,
  Layer100,
  Layer300,
  Layer400,
  Move,
  Paralyzed,
  Position,
  Power,
  RequiresTarget,
  +Slot,
  Target,
  TargetingItem,
} from './components';
import {
+  Armor,
  Being,
+  Boots,
+  Chest,
  Floor,
+  Gear,
  Goblin,
  GoblinWarrior,
  HealthPotion,
+  Helmet,
  Item,
  Player,
  ScrollFireball,
  ScrollLightning,
  ScrollParalyze,
+  Shield,
  StairsDown,
  StairsUp,
  Tile,
  Wall,
+  Weapon,
} from './prefabs';
```

```diff
ecs.registerComponent(ActiveEffects);
ecs.registerComponent(Animate);
ecs.registerComponent(Ai);
ecs.registerComponent(Appearance);
ecs.registerComponent(Description);
ecs.registerComponent(Defense);
+ecs.registerComponent(Dropped);
ecs.registerComponent(Effects);
+ecs.registerComponent(EquipmentSlot);
+ecs.registerComponent(EquipmentEffect);
ecs.registerComponent(Health);
ecs.registerComponent(Inventory);
ecs.registerComponent(IsBlocking);
ecs.registerComponent(IsDead);
+ecs.registerComponent(IsEquippable);
+ecs.registerComponent(IsEquipped);
ecs.registerComponent(IsInFov);
ecs.registerComponent(IsOpaque);
ecs.registerComponent(IsPickup);
ecs.registerComponent(IsRevealed);
ecs.registerComponent(Layer100);
ecs.registerComponent(Layer300);
ecs.registerComponent(Layer400);
ecs.registerComponent(Move);
ecs.registerComponent(Paralyzed);
ecs.registerComponent(Position);
ecs.registerComponent(Power);
ecs.registerComponent(RequiresTarget);
+ecs.registerComponent(Slot);
ecs.registerComponent(Target);
ecs.registerComponent(TargetingItem);
```

```diff
ecs.registerPrefab(Tile);
ecs.registerPrefab(Being);
ecs.registerPrefab(Item);
+ecs.registerPrefab(Gear);

// equipment
+ecs.registerPrefab(Armor);
+ecs.registerPrefab(Boots);
+ecs.registerPrefab(Chest);
+ecs.registerPrefab(Helmet);
+ecs.registerPrefab(Shield);
+ecs.registerPrefab(Weapon);

// enemies
ecs.registerPrefab(Goblin);
ecs.registerPrefab(GoblinWarrior);
```

Registered all your components and prefabs? Great.
So far all the changes and implementations we made don't affect our game.

From now on we will get into the meaty part.

First let's add them into the game by adding them to the `level_entities.js` weighting table.

```diff
export const ITEM_WEIGHT = {
  0: [
    { name: 'HealthPotion', weight: 35 },
+    { name: 'Weapon', weight: 30 },
+    { name: 'Shield', weight: 30 },
+    { name: 'Boots', weight: 30 },
+    { name: 'Helmet', weight: 30 },
+    { name: 'Chest', weight: 30 },
  ],
  2: [{ name: 'ScrollParalyze', weight: 10 }],
  4: [{ name: 'ScrollLightning', weight: 25 }],
  6: [{ name: 'ScrollFireball', weight: 25 }],
};
```

As mentioned when we created the weighting system, the weight numbers we give are completely arbitrary and up to you. You can adjust the numbers however you desire.

Let's save all this and test it out.

You might get an error, because we change the Health, Defense and Power components from 'max' to 'base.

In `render.js` change the 'max' to 'base'.

```diff
drawText({
-    text: '♥'.repeat(player.health.max),
+    text: '♥'.repeat(player.health.base),
    background: 'black',
    color: '#333',
    x: grid.playerHud.x,
    y: grid.playerHud.y + 1,
  });

-  const hp = player.health.current / player.health.max;
+  const hp = player.health.current / player.health.base;
```

You should be able to see the new items being generated and you can pick them up.
We should work on equipping the items.

For that we move into the `index.js` file and add a new keyboard shortcut.

```diff
import {
  ActiveEffects,
  Ai,
  Effects,
+  EquipmentSlot,
+  IsEquipped,
  IsInFov,
  Move,
  Position,
  Target,
  TargetingItem,
} from './state/components';
```

```js
if (userInput === "e") {
  const entity = player.inventory.inventoryItems[selectedInventoryIndex];

  if (entity) {
    if (entity.isEquippable) {
      if (!entity.has(IsEquipped)) {
        if (player.equipmentSlot?.[entity.slot.name]) {
          const previousEquipped = world.getEntity(
            player.equipmentSlot?.[entity.slot.name].itemId
          );
          previousEquipped.remove(previousEquipped.isEquipped);

          player.fireEvent("unequip", previousEquipped);
          player.remove(player.equipmentSlot[previousEquipped.slot.name]);
          addLog(`You unequip ${previousEquipped.description.name}`);
        }
        player.add(EquipmentSlot, {
          name: entity.slot.name,
          itemId: entity.id,
        });
        entity.add(IsEquipped);

        player.fireEvent("equip", entity);
        addLog(`You equip ${entity.description.name}`);
      } else {
        entity.remove(entity.isEquipped);

        player.fireEvent("unequip", entity);
        player.equipmentSlot[entity.slot.name].destroy();
        addLog(`You unequip ${entity.description.name}`);
      }
    }

    selectedInventoryIndex = 0;

    gameState = "GAME";
  }
}
```

It shares a lot of functionality with the inventory system, so it should be quite easy to understand.

```js
if (entity.isEquippable) {
  if (!entity.has(IsEquipped)) {
    if (player.equipmentSlot?.[entity.slot.name]) {
```

Here I only want to discuss one operator that we haven't used before if I recall correctly.
It's the `?.` after the property `equipmentSlot`.

It's called the optional chaining operator. You are probably familiar with the `.` chaining operator, which allows us to access the properties of an object.

The `optional chaining operator` permits reading a value in a nested object, without validating the chain.
It will short-circuit the expression and return undefined.

Pretty handy if you ask me.

Continuing with the logic:

```js
const previousEquipped = world.getEntity(
  player.equipmentSlot?.[entity.slot.name].itemId
);
previousEquipped.remove(previousEquipped.isEquipped);

player.fireEvent("unequip", previousEquipped);
player.remove(player.equipmentSlot[previousEquipped.slot.name]);
addLog(`You unequip ${previousEquipped.description.name}`);
```

We check what the previously equipped item and remove the `isEquipped` property.
This basically unequips the item, but still didn't trigger our `unequip` event.

And exactly that is what we do next. We trigger the `unequip` event removing all the effects the item provided our character and finish it up by removing the equipment slot from the player. We only want to remove the particular slot, not the property `equipmentSlot`.

Once that is done, we will equip our selected item.

```js
player.add(EquipmentSlot, {
    name: entity.slot.name,
    itemId: entity.id,
  });
  entity.add(IsEquipped);

  player.fireEvent('equip', entity);
  addLog(`You equip ${entity.description.name}`);
}
```

Here we add the proper equipment slot and give it a proper itemId.
Adding the `IsEquipped` property and firing the event `equip` concludes the part when you equip an item.

Now there is the case when you want to unequip an item, without equipping a new one.

```js
 else {
  entity.remove(entity.isEquipped);

  player.fireEvent('unequip', entity);
  player.equipmentSlot[entity.slot.name].destroy();
  addLog(`You unequip ${entity.description.name}`);
}
```

This is where this `else` case comes into play. Here we can destroy the equipment slot, because we don't need to add anything else.

We also want to add and remove the `Dropped` component when we drop or pick up an item respectively.

```diff
      readCacheSet('entitiesAtLocation', toLocId(player.position)).forEach(
        (eId) => {
          const entity = world.getEntity(eId);
          if (entity.isPickup) {
            pickupFound = true;
            player.fireEvent('pick-up', entity);
            addLog(`You pickup a ${entity.description.name}`);
+            if (entity.has(Dropped)) {
+              entity.remove(entity.dropped)
+            }
          }
        }
```

```diff
    if (userInput === 'd') {
      if (player.inventory.inventoryItems.length) {
        const entity = player.inventory.inventoryItems[selectedInventoryIndex];
        addLog(`You drop a ${entity.description.name}`);
+        entity.add(Dropped)
        player.fireEvent('drop', entity);
      }
    }
```

With this finished up, go ahead and equip your items.
While not being able to see it from the inventory, you should see the message popping up in the log.

As a player it would be good to be able to tell which items are equipped and which aren't.
For this, let's move into our `render.js`.

There is a lot do to in here.
First, let's expand our player hud to include the rest of our stats.

Add the following code below the Depth indicator

```diff
  drawText({
    text: `Depth: ${Math.abs(readCache('z'))}`,
    background: 'black',
    color: '#666',
    x: grid.playerHud.x,
    y: grid.playerHud.y + 2,
  });

+  drawText({
+    text: `Power: ${player.power.current}`,
+    background: 'black',
+    color: '#DDD',
+    x: grid.playerHud.x,
+    y: grid.playerHud.y + 4,
+  });
+
+  drawText({
+    text: `Defense: ${player.defense.current}`,
+    background: 'black',
+    color: '#DDD',
+    x: grid.playerHud.x,
+    y: grid.playerHud.y + 5,
+  });
```

When you save and check it out in the game, you should see the values for our Power and Defense showing up.

Next we will add a description for our `equip` shortcut and also mark the items that are equipped.

```diff
drawText({
-    text: '(c)Consume (d)Drop',
+    text: '(c)Consume (d)Drop (e)Equip',
    background: 'black',
    color: '#666',
    x: grid.inventory.x,
    y: grid.inventory.y + 1,
  });

  if (player.inventory.inventoryItems.length) {
    player.inventory.inventoryItems.forEach((item, index) => {
      drawText({
-        text: `${index === selectedInventoryIndex ? '*' : ' '}${
-          item.description.name
-        }`,
+        text: `${index === selectedInventoryIndex ? '*' : ' '}${
+          item.description.name
+        }${item.isEquipped ? '[e]' : ''}`,
        background: 'black',
        color: 'white',
        x: grid.inventory.x,
        y: grid.inventory.y + 3 + index,
      });
    });
```

If you equip an item now, there should be a `[e]` next to it, indicating that the item is equipped.

Awesome. Now we have a working equipment system and as promised let's create some items with randomly generated properties.

Create a new file `affix.js` at `./src/systems/affix.js` and make it look like this:

```js
export const WEAPON_PREFIXES = {
  glinting: {
    component: "power",
    delta: 2,
  },
  polished: {
    component: "power",
    delta: 3,
  },
  tempered: {
    component: "power",
    delta: 5,
  },
};

export const ARMOR_PREFIXES = {
  healthy: {
    component: "health",
    delta: 1,
  },
  stalwart: {
    component: "health",
    delta: 2,
  },
  virile: {
    component: "health",
    delta: 3,
  },
};

export const ARMOR_SUFFIXES = {
  laquered: {
    component: "defense",
    delta: 1,
  },
  fortified: {
    component: "defense",
    delta: 2,
  },
  plated: {
    component: "defense",
    delta: 3,
  },
};
```

This is a very, very, very simplified version of an affix table.
I think it displays the overall concept very well and with simplicity comes ease of understanding.

As you can see Weapons and Armor don't share the same prefixes. Some rolls might be able to occur on any item, and you can try to explore implementing that by yourself with this base knowledge.

The generation of items occurs in our `dungeon.js` so that's where we head to next.

Let's get the imports our of the way first.

```diff
import { readCache } from '../state/cache';
+import {
+  Dropped
+  EquipmentEffect,
+  IsEquippable,
+  IsPickup,
+  Position,
+} from '../state/components';
import world from '../state/ecs';
+import {
+  ARMOR_PREFIXES,
+  ARMOR_SUFFIXES,
+  WEAPON_PREFIXES,
+} from '../systems/affix';
import {
  getEntityChancesForLevel,
  getHighestMatch,
  getWeightedValue,
} from '../utils/misc';
```

In order to generate our affixes we will create 3 new functions.

```js
/**
 * generates affixes and associates them to existing equippable items on the floor
 */
const generateAffixes = () => {
  const equippableItems = world
    .createQuery({ all: [IsEquippable, IsPickup] })
    .get();

  equippableItems.forEach((item) => {
    if (item.slot.name === "weapon") {
      addAffix(item, WEAPON_PREFIXES);
      return;
    }

    // suffix
    addAffix(item, ARMOR_SUFFIXES);

    // prefix

    addAffix(item, ARMOR_PREFIXES);
  });
};

/**
 * adds affixes to an item based on an object
 * affixes structure:
 * {
 *    healthy: { component: 'health', delta: 1,},
 *    stalwart: {component: 'health', delta: 2,},
 *    virile: { component: 'health', delta: 3,},
 * };
 *
 * @param {entity} item
 * @param {Object} affixes
 */
const addAffix = (item, affixes) => {
  times(getWeightedNumber(false), () => {
    const affix = Object.keys(affixes)[getWeightedNumber()];
    item.add(EquipmentEffect, affixes[affix]);
    item.description.name = `${affix} ${item.description.name}`;
  });
};

/**
 * Generate a weighted number
 * if isAffix is true, it generates a number between 0 and 2
 * this number is used to generate the specific affix.
 *
 * if isAffix is false, it generates a number between 0 and 1
 * this number is used to define how many affixes are generated
 *
 * @param {Boolean} isAffix
 * @returns Number
 */
const getWeightedNumber = (isAffix = true) => {
  const rng = Math.floor(Math.random() * 20 + 1);

  if (!isAffix) {
    if (rng <= 13) return 0;
    return 1;
  }

  if (rng <= 10) return 0;
  if (rng <= 16) return 1;
  return 2;
};
```

This time we start from the bottom and work our way up.

```js
const getWeightedNumber = (isAffix = true) => {
  const rng = Math.floor(Math.random() * 20 + 1);

  if (!isAffix) {
    if (rng <= 13) return 0;
    return 1;
  }

  if (rng <= 10) return 0;
  if (rng <= 16) return 1;
  return 2;
};
```

Here we simply want to generate a weighted number, in order to specify rarity.
You could use a similar technique we have used for the weighting of monsters and items, but I wanted to show you a very simplistic way as well.

In short, if isAffix is true we return a weighted number between 0 and 2, and if isAffix is false we return a weighted number between 0 and 1.

The 0 and 1 are used for deciding if the item rolls an affix or not.
Between 0 and 2 are used to determine which affix it chooses.

We will use these generated numbers in the next function.

```js
const addAffix = (item, affixes) => {
  times(getWeightedNumber(false), () => {
    const affix = Object.keys(affixes)[getWeightedNumber()];
    item.add(EquipmentEffect, affixes[affix]);
    item.description.name = `${affix} ${item.description.name}`;
  });
};
```

`addAffix` accepts two parameters. The first is the item entity and the second is our previously created affix object. You only pass in the object for the item type you are creating.

As you can see, the `times` function is called either 0 or 1 times.
After that has been decided we add the randomly selected affix as an EquipmentEffect to our item.

Bringing it all together is the last function.

```js
const generateAffixes = () => {
  const equippableItems = world
    .createQuery({ all: [IsEquippable, IsPickup], none: [Dropped] })
    .get();

  equippableItems.forEach((item) => {
    if (item.slot.name === "weapon") {
      addAffix(item, WEAPON_PREFIXES);
      return;
    }

    // suffix
    addAffix(item, ARMOR_SUFFIXES);

    // prefix

    addAffix(item, ARMOR_PREFIXES);
  });
};
```

You don't want to add affixes to all items continuously. Because we generate items and also affixes whenever a level is created, you need to make sure to only add affixes to items on the ground which aren't dropped by the player. If we wouldn't do this, our items would get new affixes each time we change a level.

Finally, after you generated the items, we want to add the affixes.

```diff
  generateEntities(currentLevel, MAX_ITEMS_BY_FLOOR, ITEM_WEIGHT, dungeon);
+  generateAffixes();

  generateEntities(
    currentLevel,
    MAX_MONSTERS_BY_FLOOR,
    MONSTER_WEIGHT,
    dungeon
  );
```

And this is how you can create affixes to your Items. Of course you can do way more, like adding base stats to items, which you can then add to the player as well, but this is not included in this tutorial =).

We have done a lot. We implemented equipment slots, generated new items with affixes and handle equipping items. All in all this concludes this tutorial, however there is one last thing.

Maybe you noticed or maybe you didn't. But when we created our grid, we left a lot of room on the left side. Wouldn't this be the perfect place to display our equipment slots?

And what if you could hover over them and a tooltip appears, describing the item?

What are we waiting for, let's implement that!

Open the `canvas.js` file and right where you see the player hud in our grid, we want to make some changes.

```diff
  playerHud: {
    width: 20,
-    height: 34,
+    height: 6,
    x: 0,
    y: 0,
  },

+  playerEquipment: {
+    width: 20,
+    height: 13,
+    x: 0,
+    y: 7,
+    head: { x: 5, y: 7 },
+    weapon: { x: 1, y: 11 },
+    chest: { x: 5, y: 11 },
+    shield: { x: 9, y: 11 },
+    legs: { x: 5, y: 15 },
+  },
+
+  equipmentInfo: {
+    width: 20,
+    height: 7,
+    x: 0,
+    y: 20,
+  },
```

That will be how we layout our equipment and the info panel.
This is also the first time, that we created nested grids.

I was wondering about the way we display our items. We could just display it by text, but why not implement some images.

Canvas can handle SVGs and there is tons of great svg icons for weapons and armor.
One advantage of svg files is that they are easily displayed as code.
Another advantage is scalability. The way we set up our canvas, the SVGs will scale nicely, which wouldn't necessarily be the case for PNGs or JPGs.

Inside our `./src/lib/` folder create a file called `equipment.js`.
I already prepared some images, but feel free to switch out the paths and use different images.

```js
export const Sword =
  '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" fill="#color"><g transform="scale(-1 -1) translate(-512, -512)"><path d="M24.68 24.68c-3.535 3.537-5.85 9.779-5.85 16.264 0 4.39 1.123 8.6 2.905 12.003l23.41-7.803 7.802-23.409c-3.403-1.782-7.612-2.904-12.003-2.904-6.485 0-12.727 2.314-16.263 5.85zm17.133 40.545L84.49 105.82c2.94-4.483 5.96-8.317 9.486-11.843 3.526-3.525 7.36-6.546 11.843-9.486L65.226 41.814l-5.854 17.558zm64.892 41.48c-3.067 3.067-5.818 6.763-8.872 11.806l77.446 73.667c2.645-3.307 5.214-6.216 7.948-8.95 2.735-2.735 5.644-5.304 8.951-7.949l-73.667-77.446c-5.043 3.054-8.739 5.805-11.806 8.872zm88.941 88.94c-9.114 9.115-17.08 22.447-35.67 50.598l11.092 11.092c34.16-51.62 34.647-52.106 86.267-86.267l-11.092-11.092c-28.15 18.59-41.483 26.556-50.597 35.67zm24.042 24.043c-3.998 3.997-7.577 8.54-11.858 14.661l242.865 237.584 42.474 21.236-21.236-42.474L234.349 207.83c-6.12 4.281-10.664 7.86-14.661 11.858z"/></g></svg>';

export const Helmet =
  '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" fill="#color"><g><path d="M240.028 26v221.481L257.065 256l17.037-8.519V26h-34.074zM222.99 60.074c-80.22 0-136.297 56.077-136.297 136.296h119.26l17.037 17.037V60.074zm66.018 0v153.333l17.037-17.037h119.26c0-80.219-56.077-136.296-136.297-136.296zM69.657 213.407v34.074h50.047l-33.01-34.074H69.657zm41.528 0l34.074 34.074h34.074l-34.074-34.074h-34.074zm58.565 0l34.074 34.074h19.167v-8.518l-25.556-25.556H169.75zm144.815 0l-25.556 25.556v8.518h19.167l34.074-34.074h-27.685zm52.176 0l-34.074 34.074h34.074l34.074-34.074H366.74zm58.565 0l-33.01 34.074h50.047v-34.074h-17.037zM86.694 264.52v34.074l120.325 60.694 5.68-36.497-100.449-41.234-8.519-17.037H86.694zm321.575 0l-8.519 17.037-100.448 41.234 5.68 36.497 120.324-60.694v-34.074h-17.037zm-168.241 2.13L222.99 366.74l34.074 17.037 34.074-17.037-17.037-100.093-17.037 8.519-17.037-8.519zM78.176 314.564l-46.852 41.528v59.63l61.76-93.704-14.908-7.454zm355.648 0l-14.907 7.454 61.759 93.703v-59.63l-46.852-41.527zm-324.768 15.972L40.907 432.759l64.954 44.722 58.565-119.259-55.37-27.685zm293.888 0l-55.37 27.685 58.565 119.26 64.954-44.723-68.149-102.222zm-222.546 35.139L120.768 486h89.445l12.778-51.111h25.555v-36.204l-68.148-33.01zm151.204 0l-68.148 33.01v36.203h25.555L301.787 486h89.444l-59.63-120.324z"/></g></svg>';

export const Legs =
  '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" fill="#color"><g><path d="M128 22.781c-11.101 10.941-19.822 27.6-26.076 41.203 6.044 20.063 11.083 40.869 27.539 54.926 18.862-14.015 27.05-33.752 35.187-56.351C154.631 51.155 144.412 34.368 128 22.78zm256 0c-16.412 11.587-26.631 28.374-36.65 39.778 8.137 22.599 16.325 42.336 35.187 56.351 16.456-14.057 21.495-34.863 27.54-54.926C403.821 50.381 395.1 33.722 384 22.781zM222.23 46.104c-11.546 2.749-24.948 7.229-37.04 12.68-8.622 28.9-21.924 55.363-45.965 74.734l16.55 177.107-19.933-8.438-14.61-167.787c-16.163-16.006-28.001-43.023-38.39-71.285-3.545-2.304-7.083-4.15-10.621-5.424 6.237 82.926 25.341 186.732 47.006 274.592 2.544-1.159 5.746-2.4 8.724-3.459 29.464 7.318 56.995 29.357 81.848 53.067C192 272 256 160 222.23 46.104zm67.54 0C256 160 320 272 302.2 381.89c24.853-23.71 52.384-45.75 81.848-53.067 2.978 1.06 6.18 2.3 8.724 3.46 21.665-87.86 40.77-191.667 47.006-274.593-3.538 1.274-7.076 3.12-10.62 5.424-10.39 28.262-22.228 55.28-38.391 71.285l-14.61 167.787-19.933 8.438 16.55-177.107c-24.04-19.37-37.343-45.834-45.964-74.735-12.093-5.45-25.495-9.93-37.041-12.68zM129.004 347.83c-13.31 5.672-27.915 18.355-33.014 34.666 23.725 4.679 52.808 18.407 75.524 40.389l3.947 26.867 33.467-12.074-1.33-29.082c-19.75-28.701-51.073-52.92-78.594-60.766zm253.992 0c-27.52 7.846-58.843 32.065-78.594 60.766l-1.33 29.082 33.467 12.074 3.947-26.867c22.716-21.982 51.8-35.71 75.524-40.389-5.099-16.311-19.704-28.994-33.014-34.666zM90.69 399.703l-52.257 39.272c-10.312 15.251-12.923 32.609-8.657 47.158 52.559 9.293 88.252-3.287 129.043-25.838l-4.275-29.084c-14.703-15.135-33.665-26.354-63.854-31.508zm330.622 0c-30.189 5.154-49.151 16.373-63.854 31.508l-4.275 29.084c40.791 22.55 76.484 35.131 129.043 25.838 4.266-14.55 1.655-31.907-8.657-47.158l-52.257-39.272z"/></g></svg>';

export const Chest =
  '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" fill="#color"><g><path d="M162 35.75l-94.49 27.1c-12.05 6.3-23.47 23.9-31.01 46.35-6.07 18.2-9.62 38.9-10.93 58.3L136.7 112zm188 .1L375.4 112l111 55.6c-1.3-19.3-4.9-40.2-10.9-58.3-5.7-17.05-13.6-31.35-22.5-40.05-2.7-2.8-5.5-4.9-8.4-6.4zm-172.9 11.5l-25.7 77.45-92.9 46.4 14.08 53.5 88.82 44.4 94.6-15.9 94.6 15.9 88.8-44.4 14.1-53.5-92.8-46.4-25.8-77.35h-10.5l-59.3 73.95-.1 61.1h-18.1l.1-61-59.3-74.15zM78.65 247.7l22.05 83.9 146.2-43.8v-14.7l-88.4 14.7zm354.75 0l-80 40.1-88.4-14.7v14.7l146.3 43.8zm-186.5 58.7l-31.6 9.6-35.1 70.2 66.7-33.3zm18.1 0v46.5l66.9 33.4-35.2-70.3zM191.7 323l-86.4 26 25.3 96.3zm128.6.1l61.1 122.1 25.3-96.2zm-55.3 50l.1 43.2 100.7 37.8-20.4-40.8zm-18.1 0l-80.2 40.1-20.5 40.9L247 416.3zm.1 62.4l-81.6 30.6 81.6 10.2zm18.1 0v40.7l81.7-10.2z"/></g></svg>';

export const Shield =
  '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" fill="#color"><g><path d="M129.656 21.188L37.936 79.78c3.54 26.805 8.915 53.547 16.127 80.126L240.72 39.594l-19.282-12.5c-31.28-.885-62.204-2.842-91.782-5.907zm253.47.625c-40.51 3.975-83.496 5.938-126.47 5.843l204.625 132.72c7.108-25.89 12.487-51.92 16.095-78.032l-94.25-60.53zM257.937 50.75L59.468 178.656c8.025 26.32 17.865 52.456 29.532 78.313l243.25-158-74.313-48.22zm91.468 59.344l-74.562 48.437 151.28 98.782c11.714-25.803 21.592-51.91 29.688-78.187l-106.406-69.03zm-91.687 59.562L97 274.062c12.202 25.17 26.14 50.064 41.844 74.563l196.094-128.53-77.22-50.44zM352 231.22l-77.53 50.843 101.405 67.187c15.822-24.6 29.895-49.584 42.22-74.875L352 231.22zm-94.53 61.968l-108.345 71.03c13.564 20.062 28.326 39.847 44.28 59.313l132.032-85.28-67.968-45.063zm84.967 56.312L274.5 393.406l47.03 30.375c15.845-19.342 30.513-38.993 44.033-58.936L342.438 349.5zm-84.968 54.875L205.5 437.97c16.233 18.933 33.614 37.54 52.156 55.78 18.385-18.152 35.637-36.678 51.78-55.53l-52.092-33.626.125-.22z"/></g></svg>';
```

```js
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" fill="#color"><g transform="scale(-1 -1) translate(-512, -512)">
```

Let's quickly take a look on what is done.
We use a regular svg image with a viewBox of `0 0 512 512`. That's a little bit overkill, you could definitely get away with `0 0 64 64`, but since they are not very complex svg images, it's fine.

The first image wasn't in the correct orientation, so I simply flipped it around.

For the `fill` we will pass the color value in a function we create a little later.

Actually, this is exactly what we should do now.

In our `canvas.js` file we want to create a new function.

```js
export const drawImage = ({
  x,
  y,
  width,
  height,
  image,
  color = "#FFFFFF",
}) => {
  const img = new Image();

  const coloredImage = image.replace(/#color/g, color);

  img.src =
    "data:image/svg+xml;charset=utf-8," + encodeURIComponent(coloredImage);

  img.onload = () => {
    ctx.drawImage(
      img,
      x * cellWidth + cellWidth / 2,
      y * cellHeight + cellHeight / 2,
      width * cellWidth + cellWidth / 2,
      height * cellHeight
    );
  };
};
```

This function accepts x, y, width and height, our image (which is code) and color, set to be #FFFFFF by default.

First we create a new image object. This will contain our image after we have done some work with it.

Next we want to replace the color value with our color. This could be a regular named color, a hex color or rgb color. Your choice =)

As the image source we define a string consisting of `data:image/svg+xml;charset=utf-8,` and append our image code encoded as URI component.
This is important for our image to work.

Now, images are kind of tricky. See, when you load request an image, it has to be loaded by the system. This is why you want to have very small images, to reduce the footprint.

Before the image is loaded, we can't actually draw it. If we attempt to draw it before it's loaded, we get an error and in the worst case our game crashes.

So that's why we wrap the drawImage method inside an onload.
The `drawImage` method accepts a slew of arguments.
There is 3 variations.

```js
ctx.drawImage(image, dx, dy);
ctx.drawImage(image, dx, dy, dWidth, dHeight);
ctx.drawImage(image, sx, sy, sWidth, sHeight, dx, dy, dWidth, dHeight);
```

The first variant lets you place an image ad a destination (d), without any adjustments to the image.
In the second variant, you can define height and width in addition to the destination.

If you need to specify a part of the image you want to draw, you use the third variant.
Here you can use sx and sy to select a point and use sWidth and sHeight to create an Area. It's kind of like the snipping tool in Windows.

For our purposes, we need the second one because we want to scale our image.
Specifically we want to scale and align the image according to our grid.

We can now move into the `render.js` and implement renders for the equipment and tooltips.

Start by importing a few things.

```diff
import {
  clearCanvas,
  drawCell,
+  drawImage,
  drawRect,
  drawText,
  grid,
  pxToCell,
} from '../lib/canvas';
+import { Chest, Helmet, Legs, Shield, Sword } from '../lib/equipment';
import { toLocId } from '../lib/grid';
import { readCache, readCacheSet } from '../state/cache';
import {
  Appearance,
+  EquipmentEffect,
+  Inventory,
  IsInFov,
  IsRevealed,
  Layer100,
  Layer300,
  Layer400,
  Position,
} from '../state/components';
```

This is a little longer code section, so you might need to scroll a little to read the explanation.

I will skip over the `clearX` functions, since we have used them so often.

```js
const clearPlayerEquipment = () => {
  clearCanvas(
    grid.playerEquipment.x,
    grid.playerEquipment.y,
    grid.playerEquipment.width + 1,
    grid.playerEquipment.height
  );
};

const renderPlayerEquipment = (player) => {
  clearPlayerEquipment();
  let equipmentSlots = [
    { head: Helmet },
    { weapon: Sword },
    { chest: Chest },
    { shield: Shield },
    { legs: Legs },
  ];

  equipmentSlots.forEach((slot) => {
    const [name, image] = Object.entries(slot)[0];
    drawImage({
      x: grid.playerEquipment[name].x,
      y: grid.playerEquipment[name].y,
      width: 3,
      height: 3,
      image: image,
      color: player.equipmentSlot?.[name] ? "#FFFFFF" : "#111111",
    });
  });
};

const clearEquipmentInfo = () => {
  clearCanvas(
    grid.equipmentInfo.x,
    grid.equipmentInfo.y,
    grid.equipmentInfo.width + 1,
    grid.equipmentInfo.height
  );
};

const renderEquipmentInfo = (item) => {
  clearEquipmentInfo();

  drawText({
    text: `${item.appearance.char} ${item.description.name}`,
    background: `${item.appearance.background}`,
    color: `#DDD`,
    x: grid.equipmentInfo.x,
    y: grid.equipmentInfo.y,
  });

  if (item.has(EquipmentEffect)) {
    item.equipmentEffect.forEach((effect, index) => {
      drawText({
        text: `${effect.component}: +${effect.delta}`,
        background: "black",
        color: "#DDD",
        x: grid.equipmentInfo.x,
        y: grid.equipmentInfo.y + index + 1,
      });
    });
  }
};

const hoverEquipment = (x, y) => {
  let equipmentSlots = [
    { head: Helmet },
    { weapon: Sword },
    { chest: Chest },
    { shield: Shield },
    { legs: Legs },
  ];

  equipmentSlots.forEach((slot) => {
    const [name] = Object.entries(slot)[0];
    if (
      x >= grid.playerEquipment[name].x &&
      x <= grid.playerEquipment[name].x + 3 &&
      y >= grid.playerEquipment[name].y &&
      y <= grid.playerEquipment[name].y + 3
    ) {
      const query = world.createQuery({
        all: [Inventory],
      });

      let player = query.get()[0];
      if (player.equipmentSlot?.[name]) {
        const item = world.getEntity(player.equipmentSlot?.[name].itemId);
        renderEquipmentInfo(item);
      }
    }
  });
};
```

Okay, this is nothing really special, but I still wanted to show you how we can implement the equipment slots without repeating ourselves too much.

Important: this takes the player as an argument

```js
const renderPlayerEquipment = (player) => {
  clearPlayerEquipment();
  let equipmentSlots = [
    { head: Helmet },
    { weapon: Sword },
    { chest: Chest },
    { shield: Shield },
    { legs: Legs },
  ];

  equipmentSlots.forEach((slot) => {
    const [name, image] = Object.entries(slot)[0];
    drawImage({
      x: grid.playerEquipment[name].x,
      y: grid.playerEquipment[name].y,
      width: 3,
      height: 3,
      image: image,
      color: player.equipmentSlot?.[name] ? "#FFFFFF" : "#111111",
    });
  });
};
```

We define our slots in an array of objects. Every object uses the slot name as the key and the image as the value.

Then we can iterate over the array and assign the Object key and value to variables.

When we call our drawImage function, we set x and y according to the x and y coordinates of the respective slot name.
Width and height are based on our grid, not on actual size in px.

And lastly as the color we use either #FFFFFF if the slot exists on the player object, using the `optional chaining operator` as previously shown, or use #111111 if the slot doesn't exist.

This allows us to show the difference, when the item is equipped or unequipped.

Moving on to the tooltip section.
Most things in here are the same as for other render functions we have implemented before.

```js
const renderEquipmentInfo = (item) => {
  clearEquipmentInfo();

  drawText({
    text: `${item.appearance.char} ${item.description.name}`,
    background: `${item.appearance.background}`,
    color: `#DDD`,
    x: grid.equipmentInfo.x,
    y: grid.equipmentInfo.y,
  });

  if (item.has(EquipmentEffect)) {
    item.equipmentEffect.forEach((effect, index) => {
      drawText({
        text: `${effect.component}: +${effect.delta}`,
        background: "black",
        color: "#DDD",
        x: grid.equipmentInfo.x,
        y: grid.equipmentInfo.y + index + 1,
      });
    });
  }
};
```

The main attractor here is that we loop over the Equipment effects and show which effects are on the equipped item.

The hoverEquipment function contains similar code to our renderPlayerEquipment.
Here we accept x and y coordinates.

```js
const hoverEquipment = (x, y) => {
  let equipmentSlots = [
    { head: Helmet },
    { weapon: Sword },
    { chest: Chest },
    { shield: Shield },
    { legs: Legs },
  ];

  equipmentSlots.forEach((slot) => {
    const [name] = Object.entries(slot)[0];
    if (
      x >= grid.playerEquipment[name].x &&
      x <= grid.playerEquipment[name].x + 3 &&
      y >= grid.playerEquipment[name].y &&
      y <= grid.playerEquipment[name].y + 3
    ) {
      const query = world.createQuery({
        all: [Inventory],
      });

      let player = query.get()[0];
      if (player.equipmentSlot?.[name]) {
        const item = world.getEntity(player.equipmentSlot?.[name].itemId);
        renderEquipmentInfo(item);
      }
    }
  });
};
```

As in the renderPlayerEquipment function, we also use the equipment slot array.
You could simply create a global variable for that.

Now we get to the interesting bit.
For each slot, we want to check if the x and y coordinates are inside the dimensions of each slot.
We only want to display the tooltip for the specific item and only if we hover over that specific section.

If we are hovering over the slot, we need to get the player.
In order to reduce strain, I opted for generating the query inside the hover function, if we are hovering over a slot.

Reason for that is, that we call the hoverEquipment function on mouseMove, and we don't want to create queries every time we move the mouse.

Our player is the only entity that has an Inventory, so it's easy to use that in order to query. You could also add another component like `IsPlayer` and then query that to make it less ambiguous.

Then we check if the player has the specified slot and get the entity based of the itemId.

Lastly we need to call the renderPlayerEquipment and hoverEquipment functions.

```diff
export const render = (player) => {
  renderMap();
  renderPlayerHud(player);
+  renderPlayerEquipment(player);
  renderMessageLog();
  renderMenu();

  if (gameState === 'INVENTORY') {
    renderInventory(player);
  }
};
```

```diff
const canvas = document.querySelector('canvas');
canvas.onmousemove = throttle((e) => {
  if (gameState === 'GAME') {
    const [x, y] = pxToCell(e);

+    if (
+      x >= grid.playerEquipment.x &&
+      y >= grid.playerEquipment.y &&
+      x <= grid.playerEquipment.x + grid.playerEquipment.width &&
+      y <= grid.playerEquipment.y + grid.playerEquipment.height
+    ) {
+      clearEquipmentInfo();
+      hoverEquipment(x, y);
+    }

    renderMap();
    renderInfoBar({ x, y, z: readCache('z') });
  }

  if (gameState === 'TARGETING') {
    const [x, y] = pxToCell(e);
    renderMap();
    renderTargeting({ x, y, z: readCache('z') });
  }
}, 100);
```

Here we simply check if the cursor is over the playerEquipment area.
This is the area where all the different equipment slots reside.
You also notice that we call `clearEquipmentInfo` info from there.
This is simply to make sure once we move over an item, the previous item is cleared properly.

As a final step we need to fix a small bug. If you have used the same symbols as me for the Weapon and Armor you might notice, that it displays as `??`. This has to do with how we are splitting the strings.

Let's do some small clean up work and then fix this bug.
There isn't much explanation needed, so I will simply show you the code.

```diff
const renderInfoBar = (mPos) => {
  clearInfoBar();

  const { x, y, z } = mPos;
  const locId = toLocId({ x, y, z });

  const esAtLoc = readCacheSet('entitiesAtLocation', locId) || [];
  const entitiesAtLoc = [...esAtLoc];

  clearInfoBar();

  if (entitiesAtLoc) {
    if (entitiesAtLoc.some((eId) => world.getEntity(eId).isRevealed)) {
      drawCell({
        appearance: {
          char: '',
          background: 'rgba(255,255,255, 0.5)',
        },
        position: { x, y, z },
      });
    }

    entitiesAtLoc
      .filter((eId) => {
        const entity = world.getEntity(eId);
        return (
          layer100Entities.matches(entity) ||
          layer300Entities.matches(entity) ||
          layer400Entities.matches(entity)
        );
      })
      .forEach((eId) => {
        const entity = world.getEntity(eId);
        clearInfoBar();

+        const isPlural =
+          [...entity.description.name][
+            [...entity.description.name].length - 1
+          ] === 's'
+            ? true
+            : false;
+
+        const entityText = `${!isPlural ? 'a ' : ''}${
+          entity.description.name
+        }(${entity.appearance.char})`;

        if (entity.isInFov) {
          drawText({
+            text: `You see ${entityText} here.`,
            x: grid.infoBar.x,
            y: grid.infoBar.y,
            color: 'white',
            background: 'black',
          });
        } else {
          drawText({
+            text: `You remember seeing ${entityText} here.`,
            x: grid.infoBar.x,
            y: grid.infoBar.y,
            color: 'white',
            background: 'black',
          });
        }
      });
  }
};
```

The bug is actually in the `canvas.js` file.

```diff
export const drawText = (template) => {
  const textToRender = template.text;

-  textToRender.split("").forEach((char, index) => {
+  [...textToRender].forEach((char, index) => {
    const options = { ...template };
    const character = {
      appearance: {
        char,
        background: options.background,
        color: options.color,
      },
      position: {
        x: index + options.x,
        y: options.y,
      },
    };

    delete options.x;
    delete options.y;
    drawCell(character, options);
  });
};
```

Instead of using split for each character, we can use the spread operator.
The spread operator will preserve our symbols, since they are not strictly Ascii.

That's it. Now you have a working equipment section that changes based on weather an item is equipped or not. When you hover over it, you can see the associated stats.

And with that, the tutorial is officially over. Now you can use everything you learned to create your own game.

I really hope you enjoyed it.

Great thanks to @luetkemj for starting this project.
Thanks to @ddmills for putting so much work into geotic.

Thanks to YOU for reading this tutorial.
