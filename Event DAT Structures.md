# Event DAT Structures

This file contains information related to the structure of the event / cutscene DAT files.

# Structure / Layout

```cpp
typedef signed char         int8_t;
typedef short               int16_t;
typedef int                 int32_t;
typedef unsigned char       uint8_t;
typedef unsigned short      uint16_t;
typedef unsigned int        uint32_t;
typedef unsigned int        uintptr_t;

struct eventheader_t
{
    uint32_t    BlockCount;                 // Event block count.
    uint32_t    BlockSizes[BlockCount];     // Event block size table.
};

struct eventblock_t
{
    uint32_t    Actornumber;                // Entity Server Id
    uint32_t    TagCount;                   // Event count.
    uint16_t    TagOffset[TagCount];        // Event offset table.
    uint16_t    EvectExecNum[TagCount];     // Event id table.
    uint32_t    ImedCount;                  // Event immediate data count.
    uint32_t    ImidData[ImedCount];        // Event immediate data table.
    uint32_t    EventDataSize;              // Event data size.

    // Event data. (4 byte aligned.)
    if (EventDataSize % 4 == 0)
        uint8_t EventData[EventDataSize];
    else
        uint8_t EventData[EventDataSize + (4 - EventDataSize % 4)];
};

struct eventdat_t
{
    eventheader_t Header <optimize=false>;
    eventblock_t  Blocks[Header.BlockCount] <optimize=false, comment=IdToStr>;
};

/**
 * Converts a block actor number to a string for the comment row.
 *
 * @param {eventblock_t&} block - The event block.
 * @return {string} The entity id string.
 */
string IdToStr(eventblock_t& block)
{
    string s;
    SPrintf(s, "Entity: %08X", block.Actornumber);
    return s;
}

eventdat_t dat;
```

**Notes:**

  * _Structures are designed to be used with 010 Editor and do not reflect valid C/C++ code._
  * _Names of structure members are based on the information dumped from the publicly available PS2 beta discs DWARF information._

# Structure Information

Event DAT files contain event / cutscene information for an entire zone. When the game is compiled, all scripts for each zone are compiled down and bundled into per-zone files that contain all information related to a given zones events. When this is done, each event script is processed and packaged into separate blocks that make up the full event file.

The `eventdat_t` object holds the two main structures.

The `eventheader_t` object holds basic header information that defines the data blocks within the file. `BlockCount` tells the client how many blocks are within the file, while `BlockSizes` is an array holding the size of each block in the file as the block sizes are not static.

The `eventblock_t` object holds the information specific to a given entities event information. Including zone/player specific events which is given the id `0x7FFFFFFF` within the event VM. Below is an explaination of each field in this structure.

## `Actornumber`

The entity server id that the given event block belongs to. 

  * `0x7FFFFFFF` represents player/zone events not directly connected to any specific entity in the zone.

_This field may also be referred to as `EntityServerId` to align to other reversed information._

## `TagCount`

The number of events the block contains.

_This field may also be referred to as `EventCount` to align to other reversed information._

## `TagOffset`

The array of event offsets where each event starts within the `EventData` byte code.

_This field may also be referred to as `EventOffsets` to align to other reversed information._

## `EvectExecNum`

The array of event ids.

_This field may also be referred to as `EventIds` to align to other reversed information._

## `ImedCount`

The number of immediate data entries the block contains.

_This field may also be referred to as `ReferenceCount` to align to other reversed information._

## `ImidData`

The table of immediate data entries that the event can reference. This is extra data that can be used with various opcodes that will not fit into the byte code and is instead referenced by index into this table. (This can be things like other event ids, item ids, string ids for DAT string lookups, etc.)

_This field may also be referred to as `ReferenceData` to align to other reversed information._

## `EventDataSize`

The raw size of the event byte code data.

## `EventData`

The table of byte code data. This holds all of the byte code for every event this block contains. The data is 4 byte aligned, however the `EventDataSize` holds the true unaligned size.

The byte code data is stored in the order the scripts were compiled and bundled into the `TagOffset` table.

_You can determine the size of each events byte code by using the `TagOffset` list as a start and stop point for each entry, with the last entries size being the rest of the remaining data._

---

_This data is current as of Feb. 28, 2022 retail client._