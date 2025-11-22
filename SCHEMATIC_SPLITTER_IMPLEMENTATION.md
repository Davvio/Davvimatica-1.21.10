# Litematica Schematic Splitter Implementation

## Overview

This document describes the implementation of an automatic schematic splitting feature for the Litematica mod. This feature allows large schematics to be automatically divided into smaller, manageable chunks with proper offset positioning and individual material lists.

---

## Table of Contents

1. [Features](#features)
2. [How It Works](#how-it-works)
3. [Implementation Details](#implementation-details)
4. [Files Modified](#files-modified)
5. [Files Created](#files-created)
6. [Configuration Options](#configuration-options)
7. [Usage Guide](#usage-guide)
8. [Technical Architecture](#technical-architecture)
9. [Material List Format](#material-list-format)
10. [Examples](#examples)

---

## Features

### Core Functionality

- **Automatic Splitting**: Automatically splits saved schematics into configurable NxNxN chunks (default 16x16x16)
- **Offset-Based Alignment**: Each chunk is saved with proper offset coordinates so all chunks align perfectly when placed at the same origin
- **Material Lists**: Generates individual material list text files for each chunk
- **Multi-Region Support**: Handles schematics with multiple named regions
- **Configurable**: Fully configurable through Litematica's config system

### Key Benefits

1. **Large Build Management**: Break massive builds into manageable pieces
2. **Precise Material Planning**: Know exactly what materials each chunk needs
3. **Progressive Building**: Build one chunk at a time with clear requirements
4. **Perfect Alignment**: All chunks automatically align when placed at same origin
5. **Resource Organization**: See total items, stacks, and chest requirements per chunk

---

## How It Works

### The Splitting Process

1. **Save Trigger**: When you save a schematic normally, the splitter activates (if enabled)
2. **Chunk Division**: Each region is divided into NxNxN cubes based on `splitChunkSize`
3. **Offset Calculation**: Each chunk's position is calculated relative to the original origin (0,0,0)
4. **File Generation**: Creates:
   - Chunk schematic files (`.litematic`)
   - Material list files (`.txt`) - if enabled
5. **Folder Organization**: All chunks saved to `{schematic_name}_chunks/` subfolder

### Offset-Based Alignment Explanation

**The Critical Feature**: Each chunk stores its absolute position relative to the original schematic's origin.

```
Original Schematic Origin: (0, 0, 0)

Chunk [0,0,0] → Saved with offset (0, 0, 0)
Chunk [1,0,0] → Saved with offset (16, 0, 0)
Chunk [0,1,0] → Saved with offset (0, 16, 0)
Chunk [1,1,1] → Saved with offset (16, 16, 16)
```

**Why This Matters**: When you place all chunks at the **same origin point** in Minecraft, they automatically reconstruct the original structure perfectly - no manual positioning required!

---

## Implementation Details

### Configuration System

**Location**: `fi.dy.masa.litematica.config.Configs.java`

Three new configuration options added to the `Generic` class:

```java
public static final ConfigBoolean AUTO_SPLIT_SCHEMATICS =
    new ConfigBoolean("autoSplitSchematics", false).apply(GENERIC_KEY);

public static final ConfigInteger SPLIT_CHUNK_SIZE =
    new ConfigInteger("splitChunkSize", 16, 1, 256).apply(GENERIC_KEY);

public static final ConfigBoolean SPLIT_GENERATE_MATERIAL_LISTS =
    new ConfigBoolean("splitGenerateMaterialLists", true).apply(GENERIC_KEY);
```

### Core Splitting Logic

**Location**: `fi.dy.masa.litematica.util.SchematicSplitter.java`

#### Main Method: `splitAndSaveSchematic()`

```java
public static boolean splitAndSaveSchematic(
    LitematicaSchematic originalSchematic,
    Path originalFile,
    String fileName)
```

**Process Flow**:
1. Check if splitting is enabled via config
2. Get chunk size from config
3. Create output directory `{name}_chunks/`
4. Iterate through all regions in schematic
5. Calculate chunk grid dimensions for each region
6. Create individual chunk schematics
7. Save each chunk with coordinate-based filename
8. Generate material lists (if enabled)

#### Chunk Creation: `createChunkSchematic()`

```java
private static LitematicaSchematic createChunkSchematic(
    LitematicaSchematic original,
    String regionName,
    BlockPos regionPos,
    BlockPos regionSize,
    int chunkX, int chunkY, int chunkZ,
    int chunkSize)
```

**Key Steps**:
1. Calculate chunk bounds within region
2. Compute offset position: `regionPos.add(startX, startY, startZ)`
3. Create `AreaSelection` with proper positioning
4. Create `Box` with offset coordinates
5. Set explicit origin to `BlockPos.ORIGIN` (critical for alignment!)
6. Use `LitematicaSchematic.createEmptySchematic()` to create structure
7. Copy block data from original container
8. Set metadata (author, description, timestamps)

**Block Copying Logic**:
```java
for (int y = 0; y < sizeY; y++)
{
    for (int z = 0; z < sizeZ; z++)
    {
        for (int x = 0; x < sizeX; x++)
        {
            BlockState state = originalContainer.get(startX + x, startY + y, startZ + z);
            if (state != null)
            {
                chunkContainer.set(x, y, z, state);
                blockCount++;
            }
        }
    }
}
```

### Material List Generation

**Location**: `fi.dy.masa.litematica.util.SchematicSplitter.java` (method: `generateMaterialList()`)

#### Integration with Litematica's Material System

Uses the existing `MaterialListUtils.createMaterialListFor()` which:
- Converts block states to required items
- Handles waterlogged blocks (adds water buckets)
- Processes multi-item blocks (beds, doors, etc.)
- Uses material cache for accurate conversions

#### Output Format

Generates human-readable text files with:
- Header with chunk coordinates and schematic name
- Summary statistics (total items, stacks, chest count)
- Sorted list of materials with counts
- Stack breakdown (e.g., "32 stacks + 15 items")
- Footer with generation attribution

**Calculation Examples**:
```java
totalStacks = (int) Math.ceil(count / 64.0);
chestCount = totalStacks / 54.0;  // 54 slots per double chest
stacks = count / 64;
remainder = count % 64;
```

### Integration Point

**Location**: `fi.dy.masa.litematica.scheduler.tasks.TaskSaveSchematic.java`

**Integration** at line 149 in the `onStop()` method:

```java
if (this.schematic.writeToFile(this.dir, this.fileName, this.overrideFile))
{
    if (this.printCompletionMessage)
    {
        InfoUtils.showGuiOrInGameMessage(MessageType.SUCCESS,
            "litematica.message.schematic_saved_as", this.fileName);
    }

    // After successful save, split the schematic if enabled
    java.nio.file.Path savedFile = this.dir.resolve(
        this.fileName.endsWith(LitematicaSchematic.FILE_EXTENSION)
            ? this.fileName
            : this.fileName + LitematicaSchematic.FILE_EXTENSION);
    SchematicSplitter.splitAndSaveSchematic(this.schematic, savedFile, this.fileName);
}
```

**Why This Location?**
- Executes after successful schematic save
- Has access to directory, filename, and schematic object
- Won't run if save failed
- Doesn't interfere with in-memory-only schematics

---

## Files Modified

### 1. `src/main/java/fi/dy/masa/litematica/config/Configs.java`

**Changes**:
- Added 3 new config options (lines 117-119)
- Added options to OPTIONS list (lines 187-188)
- Added chunk size to integer configs (line 203)

**Lines Modified**: ~10 lines added

---

### 2. `src/main/java/fi/dy/masa/litematica/scheduler/tasks/TaskSaveSchematic.java`

**Changes**:
- Added import: `fi.dy.masa.litematica.util.SchematicSplitter` (line 20)
- Added splitting call in `onStop()` method (lines 145-149)

**Lines Modified**: ~6 lines added

---

### 3. `src/main/resources/assets/litematica/lang/en_us.json`

**Changes**:
- Added config comments for all 3 options (lines 140-142)
- Added config names for all 3 options (lines 143-145)
- Added success message for split completion (line 1019)
- Added error message for split failure (line 1020)

**Lines Modified**: ~8 lines added

---

## Files Created

### 1. `src/main/java/fi/dy/masa/litematica/util/SchematicSplitter.java`

**Size**: ~450 lines
**Package**: `fi.dy.masa.litematica.util`

**Class Structure**:

```
SchematicSplitter
├── splitAndSaveSchematic()      // Main entry point
├── createChunkSchematic()       // Creates individual chunks
└── generateMaterialList()       // Generates material .txt files
```

**Imports**:
- Standard Java: `java.io.*`, `java.nio.file.*`, `java.util.*`
- Minecraft: `net.minecraft.block.BlockState`, `net.minecraft.nbt.NbtCompound`, `net.minecraft.util.math.*`
- Litematica Core: `fi.dy.masa.litematica.schematic.*`, `fi.dy.masa.litematica.selection.*`
- Litematica Materials: `fi.dy.masa.litematica.materials.*`
- Malilib: `fi.dy.masa.malilib.gui.Message.MessageType`, `fi.dy.masa.malilib.util.InfoUtils`

**Key Features**:
- Comprehensive error handling with try-catch blocks
- Detailed logging for debugging
- Null safety checks throughout
- Progress reporting via logger

---

## Configuration Options

### Settings Location

Access via Litematica's config menu:
1. Open Litematica menu (default: `M`)
2. Click "Configuration"
3. Navigate to "Generic" tab
4. Scroll to find the splitting options

### Configuration Reference

| Setting | Type | Default | Range | Description |
|---------|------|---------|-------|-------------|
| `autoSplitSchematics` | Boolean | `false` | true/false | Master switch for automatic splitting |
| `splitChunkSize` | Integer | `16` | 1-256 | Size of each chunk in blocks (NxNxN) |
| `splitGenerateMaterialLists` | Boolean | `true` | true/false | Generate material list .txt files |

### Recommended Settings

**For Large Builds (100+ blocks)**:
```
autoSplitSchematics: true
splitChunkSize: 16
splitGenerateMaterialLists: true
```

**For Massive Builds (500+ blocks)**:
```
autoSplitSchematics: true
splitChunkSize: 32
splitGenerateMaterialLists: true
```

**For Small Testing**:
```
autoSplitSchematics: true
splitChunkSize: 8
splitGenerateMaterialLists: true
```

**Disable Splitting**:
```
autoSplitSchematics: false
(other settings ignored)
```

---

## Usage Guide

### Basic Workflow

#### Step 1: Enable Splitting
1. Open Litematica config
2. Navigate to Generic settings
3. Enable `autoSplitSchematics`
4. Adjust `splitChunkSize` if needed (default 16 is good)
5. Keep `splitGenerateMaterialLists` enabled

#### Step 2: Save Your Schematic
1. Create area selection as normal
2. Save schematic with any name (e.g., "MyBuild")
3. Splitting happens automatically after save
4. Watch for success message

#### Step 3: Find Your Chunks
Navigate to your schematics directory:
```
.minecraft/schematics/
├── MyBuild.litematic                    (original)
└── MyBuild_chunks/                      (chunk folder)
    ├── MyBuild_main_x0_y0_z0.litematic
    ├── MyBuild_main_x0_y0_z0_materials.txt
    ├── MyBuild_main_x0_y0_z1.litematic
    ├── MyBuild_main_x0_y0_z1_materials.txt
    └── ... (more chunks)
```

#### Step 4: Build a Chunk
1. **Read material list**: Open `MyBuild_main_x0_y0_z0_materials.txt`
2. **Gather materials**: Collect items listed in the file
3. **Load chunk**: Load `MyBuild_main_x0_y0_z0.litematic` in Litematica
4. **Place schematic**: Position at your desired origin point
5. **Build**: Use Easy Place or manual building

#### Step 5: Build Next Chunk
1. Load `MyBuild_main_x0_y0_z1.litematic`
2. **Important**: Place at the **same origin** as previous chunk
3. The chunk will automatically align correctly!
4. Repeat for all chunks

### Advanced Usage

#### Custom Chunk Sizes

**Small chunks (8x8x8)**:
- Good for very detailed builds
- More files but easier to manage
- Faster material gathering per chunk

**Medium chunks (16x16x16)**:
- Default, balanced option
- Good for most builds

**Large chunks (32x32x32)**:
- Fewer total files
- Better for simple/repetitive structures
- Larger material requirements per chunk

#### Multi-Region Schematics

If your schematic has multiple regions (e.g., "house", "garden", "pool"):
- Each region is split independently
- Chunks named: `{name}_{region}_x{X}_y{Y}_z{Z}.litematic`
- Example: `MyBuild_house_x0_y0_z0.litematic`

---

## Technical Architecture

### Schematic Data Flow

```
Original Schematic
    ↓
[TaskSaveSchematic.onStop()]
    ↓
[SchematicSplitter.splitAndSaveSchematic()]
    ↓
For each region:
    ↓
    Calculate chunk grid (chunksX, chunksY, chunksZ)
    ↓
    For each chunk position:
        ↓
        [createChunkSchematic()]
        ├── Calculate bounds
        ├── Create AreaSelection
        ├── Set offset position
        ├── Create empty schematic
        ├── Copy blocks
        └── Return chunk schematic
        ↓
        Save chunk to file
        ↓
        [generateMaterialList()]
        ├── Create material list
        ├── Sort by item name
        ├── Calculate totals
        └── Write .txt file
```

### Key Data Structures

#### AreaSelection
```java
AreaSelection chunkArea = new AreaSelection();
chunkArea.setName(name);
chunkArea.addSubRegionBox(chunkBox, true);
chunkArea.setExplicitOrigin(BlockPos.ORIGIN);
```

#### Box (Region Definition)
```java
BlockPos pos1 = chunkOffset;  // Start position with offset
BlockPos pos2 = chunkOffset.add(sizeX - 1, sizeY - 1, sizeZ - 1);  // End position
Box chunkBox = new Box(pos1, pos2, regionName);
```

#### Block Container
```java
LitematicaBlockStateContainer container = new LitematicaBlockStateContainer(sizeX, sizeY, sizeZ);
// Copy blocks
container.set(x, y, z, blockState);
```

### Coordinate System

**Original Schematic**:
- Has origin at (0, 0, 0) or custom origin
- Regions positioned relative to origin
- Size defines bounding box

**Chunk Schematic**:
- Shares same origin (0, 0, 0) as original
- Region positioned at offset from origin
- Offset = original region position + chunk index × chunk size

**Example**:
```
Original: Region "main" at (10, 64, 10), size (50, 50, 50)
ChunkSize: 16

Chunk [0,0,0]: Region at (10, 64, 10), size (16, 16, 16)
Chunk [1,0,0]: Region at (26, 64, 10), size (16, 16, 16)
Chunk [2,0,0]: Region at (42, 64, 10), size (16, 16, 16)
Chunk [3,0,0]: Region at (58, 64, 10), size (2, 16, 16)  ← Partial chunk
```

---

## Material List Format

### File Structure

```
============================================================
Material List for Chunk [X, Y, Z]
Schematic: {schematic_name}
============================================================

Total Items: {total_count}
Total Stacks: {stack_count} ({chest_count} full double chests)

------------------------------------------------------------

{Item Name}                              : {count} items  ({stacks} stacks + {remainder})
{Item Name}                              : {count} items  ({stacks} stacks + {remainder})
...

============================================================
Generated by Litematica Schematic Splitter
============================================================
```

### Example Material List

```
============================================================
Material List for Chunk [0, 1, 0]
Schematic: MedievalCastle_chunk
============================================================

Total Items: 3,847
Total Stacks: 61 (1.1 full double chests)

------------------------------------------------------------

Cobblestone                              :  1,536 items  (24 stacks)
Oak Planks                               :  1,024 items  (16 stacks)
Stone Bricks                             :    768 items  (12 stacks)
Oak Stairs                               :    256 items  (4 stacks)
Glass Pane                               :    128 items  (2 stacks)
Torch                                    :     96 items  (1 stacks + 32)
Oak Door                                 :     32 items
Iron Bars                                :      7 items

============================================================
Generated by Litematica Schematic Splitter
============================================================
```

### Reading the Material List

**Header Section**:
- Chunk coordinates [X, Y, Z]
- Schematic name reference

**Summary Statistics**:
- **Total Items**: Sum of all materials needed
- **Total Stacks**: Number of 64-item stacks (rounded up)
- **Double Chests**: How many full double chests needed (54 slots each)

**Material Entries**:
- **Item Name**: Human-readable name (localized)
- **Count**: Total number with thousand separators
- **Stack Breakdown**: Full stacks + remainder
  - "24 stacks" = exactly 24 × 64 = 1,536 items
  - "1 stacks + 32" = 1 × 64 + 32 = 96 items

**Sorting**: Alphabetical by item translation key for consistency

---

## Examples

### Example 1: Small House (32×32×32)

**Original**: `SmallHouse.litematic` (single region "main")
**Chunk Size**: 16
**Result**: 2×2×2 = 8 chunks

**File Structure**:
```
SmallHouse.litematic
SmallHouse_chunks/
├── SmallHouse_main_x0_y0_z0.litematic (16×16×16)
├── SmallHouse_main_x0_y0_z0_materials.txt
├── SmallHouse_main_x0_y0_z1.litematic (16×16×16)
├── SmallHouse_main_x0_y0_z1_materials.txt
├── SmallHouse_main_x1_y0_z0.litematic (16×16×16)
├── SmallHouse_main_x1_y0_z0_materials.txt
├── SmallHouse_main_x1_y0_z1.litematic (16×16×16)
├── SmallHouse_main_x1_y0_z1_materials.txt
├── SmallHouse_main_x0_y1_z0.litematic (16×16×16)
├── SmallHouse_main_x0_y1_z0_materials.txt
├── SmallHouse_main_x0_y1_z1.litematic (16×16×16)
├── SmallHouse_main_x0_y1_z1_materials.txt
├── SmallHouse_main_x1_y1_z0.litematic (16×16×16)
├── SmallHouse_main_x1_y1_z0_materials.txt
├── SmallHouse_main_x1_y1_z1.litematic (16×16×16)
└── SmallHouse_main_x1_y1_z1_materials.txt
```

**Total Files**: 1 original + 16 chunk files = 17 files

---

### Example 2: Castle Complex (100×80×100)

**Original**: `Castle.litematic` (single region)
**Chunk Size**: 16
**Result**: 7×5×7 = 245 chunks

**Calculations**:
- X: ⌈100 ÷ 16⌉ = 7 chunks
- Y: ⌈80 ÷ 16⌉ = 5 chunks
- Z: ⌈100 ÷ 16⌉ = 7 chunks

**Partial Chunks**:
- Most chunks: 16×16×16 (4,096 blocks)
- Edge chunks (X): 4×16×16 (1,024 blocks)
- Edge chunks (Y): 16×16×16 (4,096 blocks) - no partial
- Edge chunks (Z): 16×16×4 (1,024 blocks)

**Total Files**: 1 original + 490 chunk files = 491 files

---

### Example 3: Multi-Region Build

**Original**: `TownSquare.litematic` with 3 regions:
- "fountain" (32×32×32)
- "plaza" (64×64×4)
- "statue" (16×48×16)

**Chunk Size**: 16

**Results**:
- Fountain: 2×2×2 = 8 chunks
- Plaza: 4×4×1 = 16 chunks
- Statue: 1×3×1 = 3 chunks
- **Total**: 27 chunks

**File Structure**:
```
TownSquare.litematic
TownSquare_chunks/
├── TownSquare_fountain_x0_y0_z0.litematic
├── TownSquare_fountain_x0_y0_z0_materials.txt
├── ... (8 fountain chunks)
├── TownSquare_plaza_x0_y0_z0.litematic
├── TownSquare_plaza_x0_y0_z0_materials.txt
├── ... (16 plaza chunks)
├── TownSquare_statue_x0_y0_z0.litematic
├── TownSquare_statue_x0_y0_z0_materials.txt
└── ... (3 statue chunks)
```

**Total Files**: 1 original + 54 chunk files = 55 files

---

### Example 4: Material List Breakdown

**Chunk**: Castle_main_x0_y0_z0 (16×16×16 = 4,096 block volume)

**Actual Contents**: ~3,200 solid blocks (rest is air)

**Material List**:
```
Total Items: 3,247
Total Stacks: 51 (0.9 full double chests)

Cobblestone                              :  1,024 items  (16 stacks)
Stone Bricks                             :    768 items  (12 stacks)
Oak Planks                               :    512 items  (8 stacks)
Stone Brick Stairs                       :    384 items  (6 stacks)
Glass Pane                               :    256 items  (4 stacks)
Torch                                    :    192 items  (3 stacks)
Oak Door                                 :     64 items  (1 stacks)
Iron Bars                                :     32 items
Crafting Table                           :      8 items
Chest                                    :      4 items
Furnace                                  :      2 items
Bed                                      :      1 items
```

**Gathering Strategy**:
1. Bring ~1 double chest of storage
2. Prioritize bulk items (cobblestone, bricks, planks)
3. Collect smaller quantities as needed
4. Single items can be crafted on-site

---

## Troubleshooting

### Common Issues

#### Issue: No chunks generated
**Cause**: `autoSplitSchematics` disabled
**Solution**: Enable in config

#### Issue: Too many chunk files
**Cause**: Chunk size too small for build size
**Solution**: Increase `splitChunkSize` (e.g., 32 instead of 16)

#### Issue: Chunks don't align in-game
**Cause**: Not placing at same origin
**Solution**: Use the same placement origin for all chunks

#### Issue: Material lists not generating
**Cause**: `splitGenerateMaterialLists` disabled
**Solution**: Enable in config

#### Issue: Wrong materials in list
**Cause**: Using modded blocks without material cache
**Solution**: This is expected; manually adjust as needed

### Debug Information

Check logs at `.minecraft/logs/latest.log`:

**Successful split**:
```
[INFO] Starting schematic split for 'MyBuild' into 16x16x16 chunks
[INFO] Region 'main' size 50x50x50 will create 4x4x4 chunks (64 total)
[INFO] Successfully split schematic into 64 chunks in 'MyBuild_chunks'
```

**Errors**:
```
[ERROR] Failed to write chunk: MyBuild_main_x0_y0_z0
[ERROR] Error creating chunk schematic at [0,0,0]: {exception}
```

---

## Future Enhancements

Potential improvements for future versions:

1. **Configurable Naming**: Custom chunk filename templates
2. **JSON Material Lists**: Machine-readable material data
3. **Combined Material List**: Single file with totals for all chunks
4. **Chunk Preview Images**: Thumbnail renders of each chunk
5. **NBT Data Preservation**: Better handling of tile entities and entities
6. **Progress Indicators**: Real-time progress during splitting
7. **Undo/Cleanup**: Command to delete chunk folders
8. **Smart Splitting**: Avoid splitting through important structures
9. **Material Optimization**: Suggest efficient gathering routes
10. **Chunk Dependencies**: Mark which chunks must be built first

---

## Version History

### Version 1.0 (Current)
- Initial implementation
- Automatic chunk splitting with offset positioning
- Material list generation per chunk
- Three configuration options
- Support for multi-region schematics
- Comprehensive error handling and logging

---

## Credits

**Implementation**: Claude (Anthropic)
**Mod**: Litematica by masa
**Minecraft Version**: 1.21.10
**Litematica Version**: 0.24.5

---

## License

This implementation follows the same license as the Litematica mod. All modifications are intended for personal use and contribution to the Litematica project.

---

## Support

For issues or questions:
1. Check this documentation
2. Review configuration settings
3. Check game logs for errors
4. Test with a small schematic first
5. Verify Litematica version compatibility

---

**End of Documentation**

Generated: 2025
Last Updated: Implementation Complete
