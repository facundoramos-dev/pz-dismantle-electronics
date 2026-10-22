# Diagnosis

Every number here was measured on a running Build 42.21 dedicated server
(the `projectzomboid` image, Linux/amd64 under emulation), against the game
files in `server-files/`. The engine behaviour was read from the decompiled
classes in `server-files/java/projectzomboid.jar`.

## The symptom

Right clicking a pair of headphones or a cordless phone and choosing the
dismantle option: the option is there, the character plays the animation, the
required items are checked and the device moves between containers. Then
nothing comes out of it. The device is still in the inventory, and no scrap
was produced.

## What the game base defines

`server-files/media/scripts/generated/recipes/recipes_electrical.txt`:

```
craftRecipe DismantleElectronics
{
    timedAction = DismantleElectrical,
    time = 60,
    OnCreate = RecipeCodeOnCreate.dismantleFlashlight,
    Tags = InHandCraft;Electrical,
    category = Electrical,
    xpAward = Electricity:2,
    inputs
    {
        item 1 tags[base:screwdriver] mode:keep flags[NoBrokenItems],
        item 1 tags[base:camera;base:digital;base:miscelectronic;base:flashlight] mode:destroy flags[IsNotWorn;ItemCount;ResearchInput],
    }
    outputs
    {
        item 1 Base.ElectronicsScrap,
    }
}
```

and

```
craftRecipe DismantleMiscElectronics
{
    timedAction = DismantleElectrical,
    time = 60,
    OnCreate = RecipeCodeOnCreate.dismantleMiscElectronics,
    Tags = InHandCraft;Electrical,
    category = Electrical,
    xpAward = Electricity:2,
    inputs
    {
        item 1 tags[base:screwdriver] mode:keep flags[NoBrokenItems],
        item 1 [Base.CDplayer;Base.HomeAlarm;Base.Remote;Base.Speaker] flags[ItemCount;ResearchInput] mappers[itemType] mode:destroy,
    }
    outputs
    {
        item 1 Base.ElectronicsScrap,
        item 1 mapper:itemType,
    }
    itemMapper itemType
    {
        Base.Amplifier       = Base.Speaker,
        Base.ElectronicsScrap = Base.CDplayer,
        Base.MotionSensor    = Base.HomeAlarm,
        Base.Receiver        = Base.Remote,
    }
}
```

## Which recipe takes which device

Measured with `CraftRecipeManager.getValidInputScriptForItem()` on the running
server:

| Device | `DismantleElectronics` | `DismantleMiscElectronics` |
|---|---|---|
| `Base.Headphones` | accepted | rejected |
| `Base.CordlessPhone` | accepted | rejected |
| `Base.Earbuds` | accepted | rejected |
| `Base.VideoGame` | accepted | rejected |
| `Base.Microphone` | accepted | rejected |
| `Base.Calculator` | accepted | rejected |
| `Base.CDplayer` | rejected | accepted |
| `Base.Speaker` | rejected | accepted |
| `Base.Remote` | rejected | accepted |
| `Base.HomeAlarm` | rejected | accepted |

So the phones and headphones are handled by the tag based recipe, which also
swallows the video games, microphones and calculators, and whose `OnCreate` is
`RecipeCodeOnCreate.dismantleFlashlight`, the handler written for torches.

## The output mapper is the failure point

`zombie.entity.components.crafting.recipe.OutputMapper.getOutputItem()`:

```java
public Item getOutputItem(CraftRecipeData recipeData, boolean testManualInputs) {
    if (!this.entrees.isEmpty() && recipeData != null) {
        for (int i = 0; i < this.entrees.size(); ++i) {
            OutputEntree entree = this.entrees.get(i);
            if (entree.pattern.isEmpty()) continue;
            ...
            if (!isMatch) continue;
            return entree.result;
        }
    }
    if (this.defaultOutputEntree != null) { ... }
    return null;
}
```

There is no fallback: when no entry matches the item that was consumed, and
the mapper has no `default`, the method returns `null`. The other mappers in
the game base all define one, for example `DismantleBlade` ends with
`default = Base.HuntingKnifeBlade`. The electronics one does not.

A `null` output then breaks the recipe. `CraftRecipeData.createRecipeOutputs()`:

```java
created = this.createOutputItems(data.outputScript, this.toOutputItems, testOnly, data, character);
if (!created && this.allowOutputResources && ...) { ... }
if (created) continue;
failed = true;
```

and `createOutputItems()` bails out as soon as the output item is `null`:

```java
Item outputItem = output.getItem(this);
if (outputItem == null) {
    return false;
}
```

With `failed` set, `createRecipeOutputs()` returns `false`, `perform()` returns
`false`, and `ISHandcraftAction.performRecipe()` never reaches
`processDestroyAndUsedItems()`. That is the whole symptom: the animation
plays, the input is moved around, and the item is never consumed.

## The measurements

Run against the running server, resolving the outputs the same way the client
crafting UI does (`OutputMapper.getOutputItem(data, true)`):

```
DismantleElectronics      + Base.Headphones     -> out[0]=Base.ElectronicsScrap
DismantleElectronics      + Base.CordlessPhone  -> out[0]=Base.ElectronicsScrap
DismantleElectronics      + Base.Earbuds        -> out[0]=Base.ElectronicsScrap
DismantleMiscElectronics  + Base.CDplayer       -> out[0]=Base.ElectronicsScrap  out[1]=NULL
DismantleMiscElectronics  + Base.Speaker        -> out[0]=Base.ElectronicsScrap  out[1]=NULL
DismantleMiscElectronics  + Base.Remote         -> out[0]=Base.ElectronicsScrap  out[1]=NULL
DismantleMiscElectronics  + Base.HomeAlarm      -> out[0]=Base.ElectronicsScrap  out[1]=NULL
```

`out[1]=NULL` on every device of the mapper based recipe. The second output,
the salvaged part, never resolves.

The same measurement with the recipe this mod adds:

```
DismantlePortableElectronics + Base.Headphones    -> out[0]=Base.ElectronicsScrap  out[1]=Base.ElectronicsScrap
DismantlePortableElectronics + Base.CordlessPhone -> out[0]=Base.ElectronicsScrap  out[1]=Base.ElectronicsScrap
DismantlePortableElectronics + Base.Earbuds       -> out[0]=Base.ElectronicsScrap  out[1]=Base.ElectronicsScrap
DismantlePortableElectronics + Base.Pager         -> out[0]=Base.ElectronicsScrap  out[1]=Base.ElectronicsScrap
```

`canPerform()` returns `true` for all of them, and the mapper resolves on every
one, including the devices the table does not list, because of the `default`
entry.

## A mod cannot replace a vanilla recipe

`zombie.scripting.ScriptBucket.LoadScripts()` loads every script body that
shares a name into the same object, and `CraftRecipe.LoadIO()` appends to the
inputs and outputs already there:

```java
this.ioLines.add(script);
```

Redefining `DismantleMiscElectronics` from a mod was measured on the running
server: the recipe went from 2 inputs and 2 outputs to 5 inputs and 4
outputs, with the vanilla inputs and the new ones stacked together. The
`itemMapper` was reloaded as well, so the mapper table ended up with 8 entries
instead of 4.

That is why this mod ships a new recipe with its own name instead of fixing
the vanilla one in place.

## Other things that were checked and ruled out

- **The legacy recipe system is gone.** `ScriptManager.getAllRecipes()`
  returns 0 recipes in this build. The `DismantleHeadphones` and
  `DismantleCordlessPhone` strings that are still in
  `media/lua/shared/Translate/es/Recipes.json` are leftovers from B41 and
  point at recipes that no longer exist. That is also why the Spanish
  crafting menu says "Desarmar auriculares" and "Desmontar teléfono
  inalámbrico" for entries with no recipe behind them.
- **`ISDisassembleMenu`, the "Desmontar" menu of the world objects, is dead
  code.** It is only reachable from `ISMoveableSpriteProps.lua`, for moveable
  furniture, and never from the inventory. The option the player sees on a
  phone comes from the crafting menu.
- **The items are not missing from the world.** `ItemTracker.log` counts 87
  headphones, 51 cordless phones, 114 earbuds and 140 electronics scrap in
  the world.
- **The items are not drained.** Every one of them reports
  `getCurrentUses() = 1` and the type `base:normal`, so the consume path has
  the units it needs.
- **The client is not the problem.** The craft is executed by the server:
  `ISHandcraftAction.perform()` and `complete()` both call `performRecipe()`
  only when `isServer()`. A mod installed on the client but not on the server
  cannot change the outcome.
- **The server rejects clients that are missing a mod.** The server sends its
  mod list in `ConnectionDetails.writeMods()` and the client checks it in
  `ConnectToServerState`, disconnecting with `UI_OnConnectFailed_ModRequired`
  when a mod is missing. The check runs in one direction only, so a client
  may carry extra mods, but the server's copy of the recipes is the one that
  decides the result.
- **The anti-cheat is not involved.** No `AntiCheatPermission` or similar
  warning appears in the logs of the session where the problem was reported.
- **The `Cannot collect unprocessed, inventory item is null!` warning is
  unrelated.** It comes from `ItemDataList.getUnprocessed()` and fires while
  the crafting UI is being built, before any recipe runs. It appears 19 times
  in the session, in bursts of a few seconds, which is the crafting menu
  being opened and closed, not a recipe failing.
