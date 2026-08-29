# gbs-replaceTilesetTilesPlugin

**Version 4.3.0. Requires GB Studio 4.3.0 or newer.**

Lets a script overwrite background tile graphics while the game runs, copying tiles straight out of
a tileset in your project. Use it for animated water, a day and night palette of tiles, a wall that
changes when it takes damage, or seasonal art on the same map.

Two events. One picks the source tileset when the project is built. The other takes the tileset
from variables, so a script can choose which art to load. Both can write to either tile bank on
Game Boy Color.

![image](https://github.com/user-attachments/assets/4400d11b-e663-4163-b840-da48ab1783ee)

![image](https://github.com/user-attachments/assets/c0ba1ef1-faac-4e95-9ae7-e18cf8e226b3)

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Events Reference](#events-reference)
5. [FAQ](#faq)
6. [Memory Footprint](#memory-footprint)
7. [Bank 0 (HOME) Usage](#bank-0-home-usage)
8. [Changelog](#changelog)

---

## Concepts

### Background tile slots

The Game Boy holds background tile graphics in 256 numbered slots, plus a second set of 256 on
Game Boy Color. The scene's background map refers to those slots by number, so changing the art in
one slot changes every tile on screen that uses it, all at once.

### What this changes

These events write new art into slots that already exist. The background map is untouched, so
whatever was drawn stays where it was and only its appearance changes. One write can turn every
water tile in the scene into the next frame of an animation.

### Choosing the tileset at build time or while running

**Replace Tileset Tiles** takes the source tileset from your project's asset list when the project
is built. The target slot and the source offset can still vary while the game runs.

**Replace Tileset Tiles Ex** takes the tileset location from variables, so a script can decide which
art to load based on what is happening in the game. The caller has to know the tileset's location,
which normally comes from another plugin's event that reports it.

<img width="537" height="841" alt="image" src="https://github.com/user-attachments/assets/b1005f4c-b45d-429d-9c3f-a52ab5e97ab3" />

---

## Project Setup

1. Copy the plugin folder into your project's `plugins` folder. There is nothing to configure.

### Using Replace Tileset Tiles

1. Add a **Replace Tileset Tiles** event to any script.
2. Pick the source **Tileset**.
3. Set **Target Tile Index**, the slot where writing starts, from 0 to 255. On Game Boy Color add
   2048 to write into the second tile bank.
4. Set **Source Offset Tile Index**, the first tile to read from the tileset. 0 is its first tile.
5. Set **Length**, how many tiles in a row to copy.

### Using Replace Tileset Tiles Ex

1. Put the tileset's bank and address into two variables first.
2. Add a **Replace Tileset Tiles Ex** event and point **Tileset bank** and **Tileset Pointer** at
   those variables.
3. Set **Target Tile Index**, **Source Offset Tile Index** and **Length**. All three accept
   variables and expressions here.

---

## Size Limits and Restrictions

### Background tiles only

Both events write background tiles. Actor and projectile sprites are unaffected.

### Length is fixed in the standard event

**Replace Tileset Tiles** takes a constant **Length**, chosen when the project is built. In
**Replace Tileset Tiles Ex** it can come from a variable.

### Picking the tile bank on Game Boy Color

Add 2048 to the target tile index to write into the second bank. Without it, writes go to the
first. The remaining part of the number is the slot within that bank. On original Game Boy hardware
the extra amount is ignored.

### Source offset counts tiles

**Source Offset Tile Index** counts tiles, so 0 is the first tile in the tileset, 1 the second, and
so on.

### The source tileset must be long enough

Nothing checks the range. When the offset plus the length runs past the end of the tileset, the
copy reads whatever follows it in the ROM and you get scrambled tiles.

### Changes last until something overwrites them

Written tiles stay until a scene change reloads everything from the ROM, or another write covers
the same slots. Saving does not record tile graphics, so reapply your changes after a load.

### No engine files are replaced

The plugin adds a new engine file and changes none of the existing ones, so it has no conflicts
with other engine plugins.

---

## Events Reference

Both events appear under **Scene**, in the **Tiles** section.

### Replace Tileset Tiles

Overwrites a run of background tile slots with art from a tileset in your project.

| Field | Description |
|---|---|
| Tileset | The source tileset, chosen when the project is built. |
| Target Tile Index | Slot where writing starts, 0 to 255. Add 2048 for the second tile bank on Game Boy Color. |
| Source Offset Tile Index | First tile to copy from the tileset. 0 is its first tile. |
| Length | How many tiles in a row to copy. A constant. |

### Replace Tileset Tiles Ex

The same job with the source tileset given by variables, so every field can be a variable or
expression, including the tile count.

| Field | Description |
|---|---|
| Tileset bank | ROM bank holding the tileset. |
| Tileset Pointer | Address of the tileset. |
| Target Tile Index | Slot where writing starts, 0 to 255. Add 2048 for the second tile bank on Game Boy Color. |
| Source Offset Tile Index | First tile to copy from the tileset. |
| Length | How many tiles in a row to copy. |

---

## FAQ

**How do I animate water without metatiles?**
Draw the animation frames as consecutive tiles in one tileset. Then run a loop that calls
**Replace Tileset Tiles** with a different **Source Offset Tile Index** each time, writing over the
water slots. Every water tile on screen changes together.

**How do I find the target tile index for a specific tile?**
Open the scene's tileset and count from the first tile, starting at 0. Watch out for repeats:
GB Studio drops duplicate tiles when it builds, which shifts the slots after them.

**My tiles turned into noise. What happened?**
The copy read past the end of the source tileset. Check that **Source Offset Tile Index** plus
**Length** stays inside the tileset's tile count.

**My changes disappeared after a scene change.**
That is expected. A scene change reloads all tile art from the ROM. Reapply the change in the new
scene's **On Init**, or in a render script from the TileRenderEvent plugin.

**My changes disappeared after loading a save.**
Saves do not record tile graphics. Reapply any changes after a load.

**Can I change one tile on the map without affecting others?**
No. These events change the art in a slot, so every tile using that slot changes. To alter a single
position, use **Set Background Tile** to point that position at a different slot.

**Does it work on Game Boy Color?**
Yes. Add 2048 to the target index to write into the second tile bank.

**What is Replace Tileset Tiles Ex for?**
Choosing the source art while the game runs. It needs the tileset's location in variables, which
another plugin event has to supply, so use the standard event unless you specifically need that.

**Does it interfere with other plugins?**
No. It adds a new engine file and replaces none of the stock ones.

---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine at default engine settings, report of
2026-08-13. Figures are the difference against a stock project. Each event you use also compiles a
few bytes of script into your project, on top of the fixed cost below.

| Budget | Cost |
|---|---|
| Bank 0 (HOME) | 0 bytes |
| WRAM | 0 bytes |
| Banked ROM | +140 bytes |

- **Bank 0:** nothing. Everything the plugin adds is compiled into a switchable ROM bank.
- **WRAM:** no change.
- **Banked ROM:** 140 bytes for the copying code.
- **Engine WRAM headroom:** a stock GB Studio 4.3.0 project leaves about **854 bytes** of WRAM
  free (the engine has 7,776 bytes to work with and uses 6,922 of them). With this plugin
  installed roughly **854 bytes** remain. Adding more global variables to your project does not
  change that figure, because script memory is a fixed 3,584 byte block at stock engine settings.
- **SRAM:** not used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB fixed ROM bank shared by the GB Studio engine core, the
interrupt handlers and the GBDK runtime. Extra banked ROM is cheap to add,
bank 0 is not, so bank 0 is usually the first thing a project runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **0** |

**This plugin costs nothing in bank 0.** Everything it adds is compiled into a
switchable ROM bank.
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version bumps, patch
regeneration, packaging fixes and documentation edits are omitted.

### 2026-02-03

- Added **Replace Tileset Tiles Ex**, which takes the tileset location from variables.

### 2025-04-23

- Initial release.
- Reworked to use the built-in tile replacement command.
