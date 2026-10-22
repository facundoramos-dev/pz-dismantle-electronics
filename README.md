# Dismantle Electronics Fix

A Project Zomboid (Build 42) mod that fixes taking apart the **portable
electronics**: cordless phones, headphones, earbuds and pagers.

The Build 42 game base splits these items across two recipes and leaves a hole
in between. Dismantling a phone or a pair of headphones plays the animation,
moves the device between containers, and then nothing comes out of it: the
device is never consumed and no scrap is produced.

- Mod id: `DismantleElectronicsFix` (the value that goes in a server `Mods=` list)
- Requires Build 42 or newer. No other mods needed.

## What it adds

A **Dismantle Portable Electronics** recipe, in the **Electrical** category. It
takes a screwdriver plus one of these, and gives back electronics scrap:

| Device | Workshop item |
|---|---|
| Cordless phone | `Base.CordlessPhone` |
| Pager | `Base.Pager` |
| Headphones | `Base.Headphones` |
| Earbuds | `Base.Earbuds` |

The recipe is safe to add or remove mid-game, and it needs nothing else. It
works in single player and on dedicated servers.

## Why the vanilla recipes fail

The game base defines two of them:

```
craftRecipe DismantleElectronics        // matches on the base:miscelectronic tag
{
    OnCreate = RecipeCodeOnCreate.dismantleFlashlight,
    inputs  { item 1 tags[base:screwdriver] mode:keep flags[NoBrokenItems],
              item 1 tags[base:camera;base:digital;base:miscelectronic;base:flashlight]
                      mode:destroy flags[IsNotWorn;ItemCount;ResearchInput] }
    outputs { item 1 Base.ElectronicsScrap }
}

craftRecipe DismantleMiscElectronics    // lists four devices by name
{
    inputs  { item 1 tags[base:screwdriver] mode:keep flags[NoBrokenItems],
              item 1 [Base.CDplayer;Base.HomeAlarm;Base.Remote;Base.Speaker]
                      flags[ItemCount;ResearchInput] mappers[itemType] mode:destroy }
    outputs { item 1 Base.ElectronicsScrap,
              item 1 mapper:itemType }
    itemMapper itemType
    {
        Base.Amplifier     = Base.Speaker,
        Base.ElectronicsScrap = Base.CDplayer,
        Base.MotionSensor  = Base.HomeAlarm,
        Base.Receiver      = Base.Remote,
    }
}
```

`DismantleElectronics` does match phones and headphones, through the
`base:miscelectronic` tag. It also matches video games, microphones,
calculators, hair dryers and bullhorns, and its `OnCreate` is
`RecipeCodeOnCreate.dismantleFlashlight`, the handler written for torches.

`DismantleMiscElectronics` salvages the part with an **output mapper**, and
that mapper is the fragile part. `OutputMapper.getOutputItem()` walks its
table looking for the entry whose pattern matches the item that was consumed,
and returns `null` when it finds none. A `null` output makes
`CraftRecipeData.createOutputItems()` give up, the recipe is marked as failed,
and `processDestroyAndUsedItems()` never runs: the device is never consumed and
no scrap is produced.

Measured on a running Build 42.21 server, the output of the second recipe
resolves to `null` for every device it lists, and the first recipe does not
cover the portable electronics properly. The full measurements are in
[`docs/diagnosis.md`](docs/diagnosis.md).

This mod adds the missing branch, with a mapper that has a `default` entry so
the output can never come back unresolved.

## Why a new recipe, and not a fixed one

A mod cannot replace a vanilla craft recipe. `ScriptBucket` loads every script
body that shares a name into the same object, and `CraftRecipe.LoadIO()` adds
the inputs and outputs to the ones already there instead of replacing them.
Redefining `DismantleMiscElectronics` from a mod piles a second set of inputs
and outputs on top of the vanilla ones.

So this mod ships a **new** recipe with its own name, and leaves the vanilla
ones alone.

## Installing it on a server

A server needs two lines in `server-data/Server/pzserver.ini`, and both are
needed:

```
WorkshopItems=...;WORKSHOP_ID
Mods=...;DismantleElectronicsFix
```

`WorkshopItems=` is the list of Workshop items the server downloads, and
`Mods=` is the list of mod ids it loads. The order in `Mods=` matters: the
frameworks go first and the translation mod goes last.

**The mod cannot be enabled before it is published.** A mod that is only in
the server's `mods/` folder has no Workshop id, so the server sends the
client a mod list it cannot fulfil: the client reports the mod as missing
and drops the connection. The same applies to any local mod, not just this
one.

## Layout

```
mods/DismantleElectronicsFix/42/    the mod published to the Workshop
docs/diagnosis.md                   the measurements the fix is based on
workshop/listing.json               listing data: title, descriptions per language
workshop/preview.png                cover image, 256x256
workshop/gallery/                   carousel images, 1280x720
```

The listing images and the upload itself are handled by
[pz-mod-toolbox](https://github.com/facundoramos-dev/pz-mod-toolbox).

## License

GPL-3.0, see [`LICENSE`](LICENSE).
