# Event VM Structures

This file contains information related to the structures of the internal client event / cutscene VM system.

# Structure / Layout

```cpp
typedef signed char         int8_t;
typedef short               int16_t;
typedef int                 int32_t;
typedef unsigned char       uint8_t;
typedef unsigned short      uint16_t;
typedef unsigned int        uint32_t;
typedef unsigned int        uintptr_t;

/**
 * PS2: REQSTACK
 * Size: 0x20
 */
struct reqstack_t
{
    uint16_t        Priority;               // [Priorty]
    uint16_t        StackExecPointer;       // [StackExecPointer]
    float           MovePosition[4];        // [movepos]
    uint16_t        MoveTime;               // [movetime]
    uint8_t         TagNum;                 // [Tagnum]
    uint8_t         ReqFlag;                // [reqflag]
    float           WaitTime;               // [WaitTime]
    uint32_t        WhoServerId;            // [who]
};

/**
 * PS2: N/A (This data is part of XiEvent on PS2, on PC it seems to be an extended struct.)
 * Size: 0x188
 */
struct xieventex_t
{
    uint32_t        WorkLocal[80];              // [Work_Local]
    float           EventPos[4];                // [Eventpos]
    float           EventDir[4];                // [Eventdir]
    uint8_t         ReadEventMotionResFlag[2];  // [ReadEventMotionResFlag]
    uint8_t         FadeFlag;                   // [FadeFlag]
    uint8_t         Unknown0000;                // [???]
    uint16_t        MouthIndex;                 // [MouthIndex]
    uint16_t        Unknown0001;                // [???]
    float           MainSpeed;                  // [MainSpeed]
    float           MainSpeedBase;              // [MainSpeedBase]
    uint32_t        NowColor;                   // [NowColor]
    uint8_t         EndAlpha;                   // [EndAlpha]
    uint8_t         Unknown0002;                // [???]
    uint16_t        AlphaTime;                  // [AlphaTime]
    float           NowAlpha;                   // [nowAlpha]
    float           OfsAlpha;                   // [ofsAlpha]
    uint32_t        Unknown0003;                // [???]
    uint32_t        Unknown0004;                // [???]
};

/**
 * PS2: XiEvent
 * Size: 0x268
 */
struct xievent_t
{
    uint16_t        EntityTargetIndex[2];   // [ActorIndex]
    uint32_t        EntityServerId[2];      // [Actornumber]
    uint16_t        EventCount;             // [TagCount]
    uint16_t        Unknown0000;            // [???]
    uintptr_t       EventOffsets;           // [TagOffset]
    uintptr_t       EventIds;               // [EvectExecNum]
    uint16_t        ReferenceCount;         // [ImedCount]
    uint16_t        Unknown0001;            // [???]
    uintptr_t       References;             // [ImidData]
    uintptr_t       EventData;              // [EventData]
    reqstack_t      ReqStack[16];           // [Req]
    uint16_t        Unknown0002;            // [???]
    uint16_t        Unknown0003;            // [???]
    uint8_t         Unknown0004[18];        // [???]
    uint8_t         Unknown0005;            // [???]
    uint8_t         Unknown0006[5];         // [???]
    uint32_t        Unknown0007;            // [???]
    uint16_t        JumpTableIndex;         // [GosubStackPtr]
    uint16_t        JumpTable[8];           // [GosubRetAdrs]
    uint16_t        ExecPointer;            // [ExecPointer]
    uint16_t        RunPos;                 // [RunPos]
    uint8_t         RetFlag;                // [RetFlag]
    uint8_t         InitFlag;               // [InitFlag]
    xieventex_t*    ExtData[2];             // [N/A]
    uint32_t        ExtDataModifiedCounter; // [N/A]
};

xievent_t event;
```

**Notes:**

  * _Structures are designed to be used with 010 Editor and do not reflect valid C/C++ code._
  * _Names of structure members are based on the information dumped from the publicly available PS2 beta discs DWARF information._
    * _Names within [ ] in the comments are the PS2 names of matching fields when possible._
  * _Structures are fully mapped to their proper sizes, confirmed as of: Feb. 28, 2022_

# Structure Information

The event information block `xievent_t` is a state object used to hold the various information and variables used in relation to an entities current event.

Every entity is given their own `xievent_t` block when they are within an event/cutscene state. This block is stored at offset: `+0xD4` of the entity object for every entity in the game. If the pointer is 0, then the entity is not currently running an event sequence.

For reference to this, see: <https://github.com/AshitaXI/Ashita-v4beta/blob/main/plugins/sdk/ffxi/entity.h#L103>

# Structure Information (reqstack_t)

## `Priority`

The level of priority this stack block has over the others for executing. 

When the event VM ticks, it checks the `xievent_t::ReqStack` array to find which active block has the highest priority to run. Once the stack block to run is decided, then the `xievent_t::RunPos` is set to the index within the `xievent_t::ReqStack` table. (Priorities are more important the lower they are, with 255 being the default.)

This is how the priority is determined:

```cpp
    Priority = 255;
    v4 = 0;
    ReqStack = this->ReqStack;
    do
    {
      if ( (__int16)ReqStack->Priority <= Priority )
      {
        Priority = (__int16)ReqStack->Priority;
        this->RunPos = v4;
      }
      ++v4;
      ++ReqStack;
    }
    while ( v4 < 16 );
```

## `StackExecPointer`

The execution index into the event data byte code for this stack entry. 

This is used to tell the event VM where to set/reset the `xievent_t::ExecPointer` when switching between stacks.

## `MovePosition`

The movement position coords used with this stack entry. This is updated and used when the entity is told to move somewhere.

## `MoveTime`

The movement delay used when an entity is being told to move with a specific opcode. (OpCode: `0x0031`)

## `TagNum`

The event id currently associated with this stack entry.

## `ReqFlag`

Flag that is set to either: 0, 1, 2

Used with opcodes: `0x0028`, `0x0029`

## `WaitTime`

The amount of time the event should delay before continuing execution. This is used to pause an event for any reason, such as waiting for an action to finish, waiting for suspense, etc. 

The opcode `0x006F` is used to 'tick' the timer down. When a `0x006F` is hit, the event is 'stuck' in a loop on the same opcode until the timer reaches below 0. Once it reaches below 0, the `ExecPointer` is then incremented and the event continues.

## `WhoServerId`

The entity server id who is attached to the stack block. 

When first initialized, the block sets this to the parent entity server id who owns the event bock. During an event, various opcodes can make use of this value as well as change it.

# Structure Information (xieventex_t)

This structure holds additional 'extended' data used for the event.

## `WorkLocal`

Buffer of temporary data that the event can make use of / reference for various purposes. Similar to the events immediate data, but this is mutable.

## `EventPos`

The position of the event entity.

## `EventDir`

The direction of the event entity.

## `ReadEventMotionResFlag`

Flags used with several opcodes: `0x005B`, `0x005F`, `0x0066`

Appears to be related to loading/reading and preparing movement/motion data for an entities skeleton.

## `FadeFlag`

Flag used with opcode: `0x006C`

Used to set if an entity should be faded in/out opacity and color wise. 

## `Unknown0000`

Unknown. Possibly just alignment/padding data.

## `MouthIndex`

The entity target index that is currently talking when a message is printed to chat during the event.

## `Unknown0001`

Unknown. Possibly just alignment/padding data.

## `MainSpeed`

Speed used to calculate how fast an entity should move during the event.

## `MainSpeedBase`

Speed used to calculate how fast an entity should move during the event. (Animation)

## `NowColor`

Color used with `FadeFlag` when adjusting an entities color or opacity.

## `EndAlpha`

Alpha used with `FadeFlag` when adjusting an entities color or opacity.

## `Unknown0002`

Unknown.

## `AlphaTime`

Time used to delay how fast an entity should fade in/out.

## `NowAlpha`

Alpha used with `FadeFlag` when adjusting an entities color or opacity.

## `OfsAlpha`

Alpha used with `FadeFlag` when adjusting an entities color or opacity.

## `Unknown0003`

Unknown, used as flag mask with opcode: `0x00A7`

Depending on the flags set, this can cause the client to send a dialog selection packet: `0x005B`

## `Unknown0004`

Unknown, used as a pointer with opcode: `0x00A7`

This value is set from incoming `0x00BF` packets as an instance entry reponse.

# Structure Information (xievent_t)

This structure holds the main data used for the event.

## `EntityTargetIndex`

The event entity target index.

When the event is first initialized, both entries are set to the events parent entity information.

The event VM only ever seems to make use of the 2nd entry of this array.

## `EntityServerId`

The event entity server id.

When the event is first initialized, both entries are set to the events parent entity information.

The event VM only ever seems to make use of the 2nd entry of this array.

## `EventCount`

The number of event entries.

## `Unknown0000`

Unknown. Possibly just alignment/padding data.

## `EventOffsets`

Pointer to the array of event offsets where each event starts within the `EventData` byte code.

## `EventIds`

Pointer to the array of event ids.

## `ReferenceCount`

The number of immediate data entries the block contains.

## `Unknown0001`

Unknown. Possibly just alignment/padding data.

## `References`

Pointer to the table of immediate data entries that the event can reference. (Static data loaded from the DAT file.)

## `EventData`

Pointer to the table holding the events byte code information.

## `ReqStack`

Table of mini-stack states used for the loaded events and sub-states.

## `Unknown0002`

Unknown. Default value is 255. Used with opcode: `0x007A`

This is set to data pulled from the opcode data following the `0x007A` opcode.

## `Unknown0003`

Unknown. Used with opcode: `0x007A`

This is set to an event offset pulled from the `EventOffsets` table.

## `Unknown0004`

Unknown. No usages of this block of data has been seen at this time.

## `Unknown0005`

Unknown. Used with opcode: `0x007A`

This is set to data pulled from the opcode data following the `0x007A` opcode.

## `Unknown0006`

Unknown. No usages of this block of data has been seen at this time.

## `Unknown0007`

Unknown. This is set to the `EntityServerId[1]` when used. But the purpose is unknown at this time.

## `JumpTableIndex`

The current index into the `JumpTable` stack.

## `JumpTable`

The jump stack that holds the previous event data positions where a jump took place as a marker of where to return when a retn (opcode `0x001B`) is executed.

Jumps are stored in a stack orientation. This means that the last pushed value is the first to pop out of the table when a return happens. 

Jumps work like a call. They are expected to be returned from unless the VM is told to return and hault.

## `ExecPointer`

The index into the `EventData` byte code data that the VM is currently executing.

## `RunPos`

The index into the `ReqStack` elements that the VM is currently executing.

## `RetFlag`

Flag used to tell the VM to 'soft' stop execution of the current event. This is used to allow the client to continue without deadlocking. Various opcodes set this flag to true when their work requires additional frames to complete, allowing other entities to tick their event VMs. 

Each tick of the client loops through the entire entity map and calls the following functions in order:

  * `main` -> `MainProg` -> `MainFlow` -> `MainIdle`
  * `AtelIdle` <-- Main loop tick to update all entities and other various information/tasks.
  * `XiAtelBuff::Idle` <-- Called by `AtelIdle` for every valid entity.
  * `XiEvent::EventIdle`
  * `XiEvent::ExecProg` <-- Main VM handler that processes the current opcode.

To prevent the game from deadlocking, the `RetFlag` is used to 'suspend' the current event until next tick.

When the flag is set the following occurs:

  - The loop calling `XiEvent::ExecProg` on the current entities event VM is broken.
  - The current `ExecPointer` is copied into `ReqStack[RunPos]->StackExecPointer` to continue execution on the next frame.

When the next tick starts and `XiEvent::EventIdle` is called for the entity again, the following occurs:

  - The event VM checks to ensure the event is valid and not cancelled.
  - The `ReqStack` blocks are looped through, checking for the block that is highest in priority.
  - If a valid `ReqStack` exists, the index to it is stored into `RunPos` and the `StackExecPointer` is stored into `ExecPointer`.
  - Event execution is continued and the cycle repeats.

## `InitFlag`

Flag that is set when `XiEvent::EventInit` has completed preparing the event information.

## `ExtData`

Pointers to the entities extended data structure (`xieventex_t`).

There are two copies of the same pointer here as the client makes use of index 0 for a hard-copy that is never changed, while index 1 holds a mutable copy that can be changed. 

During opcode `0x007A`, there are multiple cases where these values are made use of, as well as `ExtDataModifiedCounter`.

This appears to be used to allow entities to share their extended event information.

## `ExtDataModifiedCounter`

Counter/flag used to tell when `ExtData[1]` has been modified. When this happens, the client also makes use of `EntityTargetIndex` and `EntityServerId`. These are used in a similar fashion where index 0 of both of these are hard-copies, and index 1 is mutable.

---

_This data is current as of Feb. 28, 2022 retail client._