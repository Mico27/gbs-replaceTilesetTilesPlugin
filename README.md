# gbs-replaceTilesetTilesPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that exposes tile-level VRAM writes to scripts. It adds two events for replacing a range of background tile bitmaps in VRAM directly from a tileset asset — one that picks the tileset at build time, and an extended version that takes the tileset as a runtime pointer, so scripts can choose the source tileset dynamically.

Both events can target VRAM bank 0 or bank 1 on Game Boy Color hardware.

![image](https://github.com/user-attachments/assets/4400d11b-e663-4163-b840-da48ab1783ee)

![image](https://github.com/user-attachments/assets/c0ba1ef1-faac-4e95-9ae7-e18cf8e226b3)

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Events Reference](#events-reference)
5. [Memory Footprint](#memory-footprint)
6. [Bank 0 (HOME) Usage](#bank-0-home-usage)
7. [Changelog](#changelog)

---

## Concepts

### Background VRAM tile slots

The Game Boy stores background tile bitmaps in VRAM as 256 numbered slots (indices 0–255), with a second set of 256 slots in VRAM bank 1 on Game Boy Color. The tilemap references those slots by index, so changing the bitmap at a given slot instantly changes **every** on-screen tile that uses that index.

### Replace vs. reload

This plugin writes new bitmap data into existing VRAM slots. It does not change the tilemap — only the pixels stored at the target slots. That is useful for:

- swapping tile graphics at runtime without a scene change: animated tiles, day/night, damage states;
- loading tile variants from different tilesets into the same VRAM slots so the existing tilemap keeps working.

### Build-time vs. runtime tileset selection

**Replace Tileset Tiles** picks the source tileset from the project's asset list when the project is built. Only the target VRAM index and the source offset can vary at runtime.

**Replace Tileset Tiles Ex** takes the tileset as a runtime pointer — the bank number and address can come from script variables — so scripts can choose which tileset to use based on game state. It requires the caller to already know the bank and pointer of the desired tileset, which normally come from another plugin event that returns them.

<img width="537" height="841" alt="image" src="https://github.com/user-attachments/assets/b1005f4c-b45d-429d-9c3f-a52ab5e97ab3" />

---

## Project Setup

1. Copy the plugin folder into your GB Studio project's `plugins/` directory. No additional configuration, engine fields, or compatibility variants are required.

### Using Replace Tileset Tiles

1. Add a **Replace Tileset Tiles** event to any script.
2. Select the source **Tileset** asset.
3. Set **Target Tile Index** — the VRAM slot where writing starts (0–255). On Game Boy Color, add 2048 (`0x0800`) to target VRAM bank 1 instead of bank 0.
4. Set **Source Offset Tile Index** — the first tile within the tileset to read from; 0 starts at the beginning.
5. Set **Length** — the number of consecutive tiles to copy.

### Using Replace Tileset Tiles Ex

1. Before calling the event, store the tileset's bank number and pointer in two script variables.
2. Add a **Replace Tileset Tiles Ex** event and set **Tileset bank** and **Tileset Pointer** to those variables.
3. Set **Target Tile Index**, **Source Offset Tile Index** and **Length** — all of which can be variables or expressions here.

---

## Size Limits and Restrictions

### Background VRAM only

Both events write to background VRAM. Sprite (actor and projectile) tiles are not affected.

### Length is fixed in the standard event

In **Replace Tileset Tiles**, **Length** is a constant chosen when the project is built. In **Replace Tileset Tiles Ex** it is a runtime value and can come from a variable.

### Selecting the VRAM bank on Game Boy Color

Add 2048 (`0x0800`) to the target tile index to write into VRAM bank 1; without it, writes go to bank 0. The low 8 bits are the slot number within the chosen bank. On DMG hardware the extra bit is ignored and all writes go to the single VRAM bank.

### Source offset is a tile index, not a byte offset

**Source Offset Tile Index** counts tiles: index 0 is the first tile in the tileset, index 1 the second, and so on.

### The source tileset must have enough tiles

There is no bounds checking. If the offset plus the length exceeds the number of tiles in the tileset, the read runs past the end of the data and produces garbage graphics.

### Changes persist until overwritten or the scene changes

Written tile data stays in VRAM until a scene change reloads all tile data from ROM, or another write replaces the same slots. Saving and loading does not preserve VRAM tile data — reapply any runtime replacements after a load.

### No engine files modified

The plugin only adds a new engine source file, so it has no compatibility conflicts with other engine plugins.

---

## Events Reference

Both events appear under **Scene → Tiles** in the script editor.

---

### Replace Tileset Tiles

**`EVENT_REPLACE_TILESET_TILES`**

Replaces a contiguous range of background VRAM tile slots with pixel data from a project tileset asset.

| Field | Description |
|---|---|
| Tileset | The source tileset to read tile bitmaps from, chosen when the project is built. |
| Target Tile Index | VRAM slot index where the write starts (0–255). Add 2048 to target VRAM bank 1 on Game Boy Color. |
| Source Offset Tile Index | First tile within the tileset to copy; 0 is the beginning of the tileset. |
| Length | Number of consecutive tiles to copy. A constant, not a variable. |

---

### Replace Tileset Tiles Ex

**`EVENT_REPLACE_TILESET_TILES_EX`**

Extended variant that takes the source tileset as a runtime pointer, so every parameter — including the tile count — can be a variable or expression.

| Field | Description |
|---|---|
| Tileset bank | ROM bank number of the tileset. |
| Tileset Pointer | Memory address of the tileset. |
| Target Tile Index | VRAM slot index where the write starts (0–255). Add 2048 to target VRAM bank 1 on Game Boy Color. |
| Source Offset Tile Index | First tile within the tileset to copy. |
| Length | Number of consecutive tiles to copy. |

---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine (per-file SDCC compile with GB Studio's build flags, default engine settings). Values are the plugin's *delta* versus the stock engine; DMG build, with CGB noted where it differs. ROM cost lands in banked ROM (GB Studio's autobanker spreads it across switchable banks); using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks.

| | Cost |
|---|---|
| WRAM | +0 bytes |
| ROM | +140 bytes (DMG) / +131 bytes (CGB) |

- **WRAM:** no change.
- **Engine WRAM headroom:** the stock GB Studio 4.3.0 engine leaves about **854 bytes** of WRAM free (usable engine WRAM is 7,776 bytes at 0xC0A0–0xDF00; the stock engine uses 6,922 bytes). With this plugin installed roughly **854 bytes** remain. This figure does not depend on how many global variables your project defines: the script memory array has a fixed size of VM_HEAP_SIZE + (VM_MAX_CONTEXTS × VM_CONTEXT_STACK_SIZE) words — 768 + 16 × 64 = 1,792 words (3,584 bytes) with stock engine settings.
- **SRAM:** not used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB non-switchable ROM bank that the GB Studio engine core,
the interrupt handlers and the GBDK runtime all share. Banked ROM is cheap
(add another bank), bank 0 is not, so it is usually the first thing a project
runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **0** |
| Bank 0 free with this plugin installed | **1,451** of 16,384 (91% used) |

**This plugin costs nothing in bank 0.** All of its code lives in a switchable
ROM bank; nothing it adds is resident in bank 0.

<details><summary>How this was measured</summary>

GB Studio 4.3.2, DMG target, default engine settings. Each module's bank 0
contribution is the `A _HOME size` record that SDCC writes into its `.rel`
object, summed over the engine sources this plugin provides. Stock sizes come
from building projects whose only plugin ships no engine C, so every module in
them is the untouched engine; two such builds were compared and agreed on all
73 shared modules.

The "free" figure is a stock project with this plugin and nothing else. Your
own number will differ: other plugins, and any engine settings that change what
the core compiles, move it independently of this plugin.

</details>
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version
bumps, patch regeneration, packaging fixes and documentation edits are omitted.

### 2026-02-03

- New `replaceTilesetTilesEx` event, allowing the tileset far pointer to be passed through variables.

### 2025-04-23

- Initial release.
- Refactored to use the GBVM replace-tiles command.
