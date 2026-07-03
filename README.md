# gbs-replaceTilesetTilesPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that exposes tile-level VRAM writes to scripts. It adds two events for replacing a range of background tile bitmaps in VRAM directly from a tileset asset — one using a compile-time-selected tileset via the standard GBVM `VM_REPLACE_TILE` instruction, and one extended version that accepts the tileset as a runtime far pointer (bank + address variables), enabling dynamic tileset selection at runtime.

Both events support targeting VRAM bank 0 or bank 1 on CGB hardware, making them compatible with dual-bank CGB tile data.

![image](https://github.com/user-attachments/assets/4400d11b-e663-4163-b840-da48ab1783ee)

![image](https://github.com/user-attachments/assets/c0ba1ef1-faac-4e95-9ae7-e18cf8e226b3)

<img width="537" height="841" alt="image" src="https://github.com/user-attachments/assets/b1005f4c-b45d-429d-9c3f-a52ab5e97ab3" />

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [How to Use](#how-to-use)
4. [Technicalities and Restrictions](#technicalities-and-restrictions)
5. [Events Reference](#events-reference)
6. [Inner Workings](#inner-workings)
7. [Memory Footprint](#memory-footprint)

---

## Concepts

### Background VRAM Tile Slots

The Game Boy stores background tile bitmaps in VRAM starting at address `0x8000`. Each tile occupies 16 bytes (8×8 pixels at 2 bits per pixel). There are 256 tile slots in bank 0 (indices 0–255), and on CGB hardware an additional 256 slots in VRAM bank 1 (also indexed 0–255 but addressed through `VBK_REG`). The tilemap references these slots by index; changing the bitmap data at a given slot index instantly changes every on-screen tile that uses that index.

### Replace vs. Reload

This plugin writes new bitmap data into existing VRAM tile slots. It does not change the tilemap (which tile indices appear where on screen) — only the pixel data stored at the target slots. This is useful for:

- Swapping tile graphics at runtime without a scene change (animated tiles, day/night, damage states).
- Loading tile variants from different tilesets into the same VRAM slots so that the existing tilemap continues to work.

### Compile-time vs. Runtime Tileset Selection

**Replace Tileset Tiles** selects the source tileset from the project's asset list at compile time. The tileset symbol and bank are baked into the compiled script using the standard `VM_REPLACE_TILE` GBVM instruction. Only the target VRAM index, the source offset, and the tile count can be determined at runtime.

**Replace Tileset Tiles Ex** passes the tileset as a far pointer at runtime — the bank number and memory address can come from script variables. This allows scripts to choose which tileset to use dynamically (e.g. based on a game variable), but requires the caller to know the bank and pointer of the desired tileset.

---

## Project Setup

1. Copy the plugin folder into your GB Studio project's `plugins/` directory.
2. No additional configuration, engine fields, or compatibility variants are required.

---

## How to Use

### Replace Tileset Tiles (compile-time tileset)

1. Add a **Replace Tileset Tiles** event to any script.
2. Select the source **Tileset** asset from the project's tileset list.
3. Set **Target Tile Index** — the VRAM slot number where writing will start (0–255). On CGB, add `0x0800` (2048) to the value to target VRAM bank 1 instead of bank 0.
4. Set **Source Offset Tile Index** — the index of the first tile within the selected tileset to read from. Use 0 to start from the beginning of the tileset.
5. Set **Length** — the number of consecutive tiles to copy.

### Replace Tileset Tiles Ex (runtime far pointer)

1. Before calling the event, store the tileset bank number and pointer in two script variables. These values are typically obtained from another plugin event that returns far pointers (such as a submapping or scene data lookup).
2. Add a **Replace Tileset Tiles Ex** event.
3. Set **Tileset bank** and **Tileset Pointer** to the variables holding the far pointer.
4. Set **Target Tile Index**, **Source Offset Tile Index**, and **Length** as runtime value expressions.

---

## Technicalities and Restrictions

### Affects Background (BKG) VRAM Only

Both events write to background VRAM (`SetBankedBkgData`). Sprite VRAM (OBJ tiles) is not affected. Only tiles visible via the background tilemap and window layer are modified.

### Length Is Compile-Time in the Standard Event

In **Replace Tileset Tiles**, the `Length` field is a compile-time constant number. In **Replace Tileset Tiles Ex**, the length is a runtime value expression and can come from a variable.

### CGB VRAM Bank Selection via Bit 11

On CGB, bit 11 of the target tile index (`0x0800`) selects the VRAM bank:
- `0x0000`–`0x00FF` (bit 11 = 0) → writes to VRAM bank 0
- `0x0800`–`0x08FF` (bit 11 = 1) → writes to VRAM bank 1

The lower 8 bits (`idx_target_tile & 0xFF`) are used as the tile slot number within the selected bank. This convention matches the CGB attribute tile bank bit. On DMG hardware, bit 11 is ignored and all writes go to the single VRAM bank.

### Source Offset Is a Tile Index, Not a Byte Offset

**Source Offset Tile Index** (`idx_start_tile`) is counted in tiles. Internally it is multiplied by 16 to get the byte offset into the tileset's tile data array. Index 0 is the first tile in the tileset, index 1 is the second, and so on.

### The Source Tileset Must Have Enough Tiles

The plugin does not perform bounds checking. If `idx_start_tile + tile_length` exceeds the number of tiles in the tileset, the read will run past the end of the tileset data and produce garbage tile graphics.

### Changes Are Persistent Until Overwritten or VRAM Is Reloaded

Written tile data persists in VRAM until the scene is changed (which reloads all tile data from ROM) or until another write replaces the same slots. Saving and loading does not restore or preserve VRAM tile data — any runtime replacements must be reapplied after a load.

### No Engine Files Modified

This plugin only adds a new engine source file (`replace_tiles_ex.c`). No existing GB Studio engine files are patched. There are no `engineAlt` variants.

---

## Events Reference

### Replace Tileset Tiles

**Event ID:** `EVENT_REPLACE_TILESET_TILES`  
**Group:** Scene → Tiles

Replaces a contiguous range of background VRAM tile slots with pixel data from a project tileset asset, using the built-in GBVM `VM_REPLACE_TILE` instruction.

| Field | Type | Description |
|---|---|---|
| Tileset | Tileset asset picker | The source tileset to read tile bitmaps from. Resolved at compile time. |
| Target Tile Index | Value expression | VRAM slot index where the write starts (0–255). On CGB, OR with `0x0800` to target VRAM bank 1. |
| Source Offset Tile Index | Value expression | First tile within the tileset to copy (0 = beginning of tileset). |
| Length | Number (compile-time) | Number of consecutive tiles to copy. |

---

### Replace Tileset Tiles Ex

**Event ID:** `EVENT_REPLACE_TILESET_TILES_EX`  
**Group:** Scene → Tiles

Extended variant that accepts the source tileset as a runtime far pointer. All parameters — including tile count — are runtime value expressions, allowing fully dynamic tile replacement.

| Field | Type | Description |
|---|---|---|
| Tileset bank | Value expression | ROM bank number of the tileset. |
| Tileset Pointer | Value expression | Memory address (pointer) of the `tileset_t` struct. |
| Target Tile Index | Value expression | VRAM slot index where the write starts (0–255). On CGB, OR with `0x0800` to target VRAM bank 1. |
| Source Offset Tile Index | Value expression | First tile within the tileset to copy. |
| Length | Value expression | Number of consecutive tiles to copy. |

**Notes:**
- The tileset bank and pointer are typically obtained by reading them from a scene or tileset data structure via another plugin or a GBVM memory read.
- All five parameters are pushed onto the VM stack before calling the native function.

---

## Inner Workings

### Standard Event: `VM_REPLACE_TILE`

**Replace Tileset Tiles** uses the GB Studio compiler helper `_replaceTile`, which emits the built-in `VM_REPLACE_TILE` GBVM bytecode instruction. This instruction is already part of the GB Studio engine and handles bank-switching and VRAM writes internally. The plugin simply provides a script event UI that wraps it with a tileset picker and runtime index/offset fields.

The compile function pushes `idx_target_tile` and `idx_start_tile` as stack arguments, then calls `_replaceTile` with the resolved tileset symbol, the stack references to the two index values, and the compile-time length:

```js
_stackPushScriptValue(input.idx_target_tile);
_stackPushScriptValue(input.idx_start_tile);
_replaceTile(".ARG1", tileset.symbol, ".ARG0", input.tile_length);
_stackPop(2);
```

### Extended Native Function: `replace_tiles_ex`

The Ex variant calls a custom native function `replace_tiles_ex` added by the plugin:

```c
void replace_tiles_ex(SCRIPT_CTX * THIS) OLDCALL BANKED {
    uint8_t  tile_length    = *(uint8_t *) VM_REF_TO_PTR(FN_ARG0);
    int16_t  idx_start_tile = *(int16_t *) VM_REF_TO_PTR(FN_ARG1);
    int16_t  idx_target_tile= *(int16_t *) VM_REF_TO_PTR(FN_ARG2);
    uint8_t  tileset_bank   = *(uint8_t *) VM_REF_TO_PTR(FN_ARG3);
    const tileset_t * tileset = *(tileset_t **) VM_REF_TO_PTR(FN_ARG4);
#ifdef CGB
    if (_is_CGB) VBK_REG = (idx_target_tile & 0x0800) ? 1 : 0;
#endif
    SetBankedBkgData(
        (UBYTE)(idx_target_tile),
        tile_length,
        tileset->tiles + (idx_start_tile << 4),
        tileset_bank
    );
#ifdef CGB
    VBK_REG = 0;
#endif
}
```

**Step by step:**

1. **Arguments are read** from the VM stack in right-to-left push order: `tile_length` from `FN_ARG0` (pushed last), down to `tileset` from `FN_ARG4` (pushed first).

2. **CGB VRAM bank selection** — bit 11 of `idx_target_tile` is tested. If set, `VBK_REG` is set to 1 to point at VRAM bank 1; otherwise bank 0 is used. On DMG builds this block is compiled out.

3. **`SetBankedBkgData`** — GBDK's banked tile-data write function. It takes:
   - `(UBYTE)idx_target_tile` — destination tile slot (lower 8 bits, 0–255)
   - `tile_length` — number of tiles to write
   - `tileset->tiles + (idx_start_tile << 4)` — source pointer: `idx_start_tile` tiles into the tileset's data array (`<< 4` converts tile index to byte offset since each tile is 16 bytes)
   - `tileset_bank` — the ROM bank to switch to when reading tile data

4. **`VBK_REG` is reset to 0** after the write so the rest of the engine continues to read from VRAM bank 0 as expected.

### Stack Argument Order

The JS compile function pushes arguments in this order:

```js
_stackPushScriptValue(input.tileset_ptr);    // FN_ARG4 (first pushed = deepest)
_stackPushScriptValue(input.tileset_bank);   // FN_ARG3
_stackPushScriptValue(input.idx_target_tile);// FN_ARG2
_stackPushScriptValue(input.idx_start_tile); // FN_ARG1
_stackPushScriptValue(input.tile_length);    // FN_ARG0 (last pushed = top of stack)
```

The GBVM calling convention numbers arguments from the top of the stack downward, so `FN_ARG0` is the most-recently-pushed value.

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
