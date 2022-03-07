# Event VM Functions

This file contains information on the various functions that are used with the event virtual machine. The information here can be used to find, view and understand the various VM related functions in a tool such as IDA or Ghidra.

## `EventStartWait`

**Pattern:** `A0 ?? ?? ?? ?? C7 05 ?? ?? ?? ?? 00 00 00 00 84 C0 75 ?? B0 01 C3 E8 ?? ?? ?? ?? 83 F8 FF`

Prepares the event system. This function starts by ensuring the event data for the current zone is loaded this is done in steps to prevent deadlocks and has an internal task counter used to tell which state the system is currently in. The first steps are to load the two event DAT files (one holds the actual event data/blocks, the other holds the event messages/strings) which are done in two steps.

  * If the event data DAT is not loaded, it's loaded and the function returns, which will continue on the next tick.
  * If the event message DAT is not loaded, it's loaded and the function returns, which will continue on the next tick.
  * Once both files are loaded, then `InitEvent2` is called.
  * Once `InitEvent2` returns successfully, then all valid entities have `XiEvent::XiEventInit` called for their individual event instances, if present.
    * If no entities exist or have an event initialized, then the function returns while setting the event cancel flag to true.

## `InitEvent2`

**Pattern:** `83 EC 14 53 55 56 8B 35 ?? ?? ?? ?? 57 C7 44 24 18 00 00 00 00 8B 06 83 C6 04 89 44 24 20 89 74 24 1C 8D 1C 85 00 00 00 00 C1 EB 02 85 C0 0F 8E`

Prepares the entities that are involved with the current event attempting to run. This function starts by looping the event data blocks looking for and preparing all entities that are involved with the event that will take place. (Generally, this is your local player and the target entity.) The function has a few checks that make this happen when it first starts:

  * First is a call to `GetActorNum`, this function is used to obtain an entities server id and target index. Values for certain conditions are hardcoded.
    * `0x7FFFFFC0`, `0x7FFFFFF0`
      * Both of these return the local players information.
    * `0x7FFFFFC1`, `0x7FFFFFC2`, `0x7FFFFFC3`, `0x7FFFFFC4`, `0x7FFFFFC5`
      * These are intentionally overflowed upward by `+0x40`, resulting in the values: 1, 2, 3, 4, 5
      * These values are used to return the local players party members info (based on their index in the party, 0 is skipped for local player). (Party 0)
    * `0x7FFFFFC6`, `0x7FFFFFC7`, `0x7FFFFFC8`, `0x7FFFFFC9`, `0x7FFFFFCA`, `0x7FFFFFCB`
      * These are intentionally overflowed upward by `+0x44`, resulting in the values: 10, 11, 12, 13, 14, 15
      * These values are used to return the local players alliance party members info. (Party 1)
    * `0x7FFFFFCC`, `0x7FFFFFCD`, `0x7FFFFFCE`, `0x7FFFFFCF`, `0x7FFFFFD0`, `0x7FFFFFD1`
      * These are intentionally overflowed upward by `+0x48`, resulting in the values: 20, 21, 22, 23, 24, 25
      * These values are used to return the local players alliance party members info. (Party 2)
    * `0x7FFFFFF1`, `0x7FFFFFF2`, `0x7FFFFFF3`, `0x7FFFFFF4`, `0x7FFFFFF5`
      * These are used when referecing the local player party members. (Used for quests such as `Introduction to Teamwork`)
      * These values can cause an outgoing `0x0076` packet to be sent if party data is missing.
    * The default handler treats all other IDs passed to the function as a normal entity server id for NPCs.
      * The return values will be the same server id given, and the target id pulled from the server id via: `(serverId & 0x3FF)`
  * Next is a call to `NowEventChar`, this function is used to validate the entity has the given expected event id that is being loaded.
    * The function loops the current entities event block looking at the `EventIds` array.
      * If the expected id is found, or if an id of `0xFFFE` is found, the function exists successfully. (Returns 1.)
      * If the expected id is not found, the function fails. (Returns 0.)
  * Once a valid entity for the event is found, the next steps take place:
    * entity->Render.Flags0 |= 0x100;
    * `if ( (entity->Render.Flags1 & 0x1000) != 0 || !entity->UpdateMask )`
      * If either of these are true, the loop is broken and the client will attempt to send an outgoing `0x0016` packet to request the expected event entity.
    * `if ( (v4->Render.Flags0 & 0x200) == 0 )`
      * If this is true, the function hard-exits in failure.
    * `if ( (CliEventMode & 0x100) != 0 )`
      * If this is true, the client will call `XiActor::SetCastMagicID` function on the entity and consider it valid.
      * The further checks are skipped and go straight to the call to `XiAtelBuff::EventNew`.
    * `if ( entity->Status != 47 )`
      * If this is true, then additional checks take place. The entity is checked for:
        * If entity is sitting in a chair (entity->Status is 63 to 83).
          * If true the next check is skipped.
        * `entity->AnimationPlay` is checked for anything non-zero.
          * If true, then the client checks if the entity is within the current event playing.
            * If this is false, the function hard-exists in failure.
    * `if ( (entity->ServerId & 0xFF000000) == 0 )`
      * If this is true, the client will call `XiActor::SetCastMagicID` function on the entity and set another value inside of its `WarpPointer` to `0x20202020`
    * `if ( (v8->Render.Flags0 & 0x4000) != 0 || (v8->Render.Flags0 & 0x8000) != 0 )`
      * If this is true, further checks are skipped and go straight to the call to `XiAtelBuff::EventNew`.
    * `if ( XiActor::IsLockedStatus(entity->WarpPointer) )`
      * If this is true, the function hard-exists in failure.
  * Once the above has been checked, then the entity is considered valid. The following then happens:
    * `XiAtelBuff::EventNew` is called for this entity.
    * The entities `Movement.LastPosition` is copied into `Movement.Move`.
  * This is repeated for all valid entities for the event until the block count has been reached, then the function exits.

## `XiAtelBuff::EventNew`

**Pattern:** `56 57 8B F9 8A 87 20 01 00 00 84 C0 78 5B 68 ?? ?? ?? ?? E8 ?? ?? ?? ?? 83 C4 04 85 C0 74`

Creates the actual `xievent_t` object that is stored into the entities `EventPointer` field. This function is responsible for calling the `XiEvent::XiEvent` constructor.

## `XiEvent::XiEvent`

**Pattern:** `53 55 56 57 68 ?? ?? ?? ?? 8B F1 E8 ?? ?? ?? ?? 33 DB 83 C4 04 3B C3 74 ?? 8B C8 E8`

The XiEvent object constructor. This initializes the various inner variables of the object preparing it for usage. The constructor puts the object into an invalid initialized state via the `xievent_t::InitFlag` being set to 0. This is done intentionally and expects `XiEvent::XiEventInit` to be called before usage.

This function is just a basic constructor, but here's a quick rundown of what happens:

  * An instance of `xieventex_t` is allocated and stored into `xievent_t::ExtData[0]` and `xievent_t::ExtData[1]`.
  * The various properties of `xievent_t` are initialized to their various defaults.
  * Some of the loaded event DAT information for the event block executing is copied and stored into the members.
  * The parent entity is obtained and some adjustments are made to `Render.Flags0` and `Render.Flags3`, preparing the entity to be in a valid event state.
  * The event id list is iterated to find the proper id and then the entity is setup further to reset its animations by calling: `XiAtelBuff::IdleDefMotionReset`
  * The entities status is adjusted to be put into the event state, while backing up their previous status.
  * The entity animation state is reset by calling: `XiAtelBuff::IdleDefMotionReset`

## `XiEvent::~XiEvent`

**Pattern:** `56 8B F1 57 8B 46 04 66 8B 0E 89 46 08 8B 86 5C 02 00 00 85 C0 66 89 4E 02`

The XiEvent object destructor. This cleans up the various allocations and other client data that was created and adjusted during the use of the XiEvent object.

This function is fairly straight forward in cleaning things up, but here's a quick rundown of what happens:

  * The entity is obtained, if valid then it's event action is cancelled via: `XiAtelBuff::KillLastAction` 
  * The entity is tested for type `XiSkeletonActor::classXiSkeletonActor`
    * If matched, then the entity is told to stop moving its mouth via: `XiAtelBuff::StopMouth` and `XiSkeletonActor::DeleteResp`
    * The entities various flags and animation fields are reset from the event state.

## `XiEvent::XiEventInit`

**Pattern:** `53 55 56 8B F1 33 C0 33 DB 66 8B 46 02 57 8B 04 85 ?? ?? ?? ?? 3B C3 0F ?? ?? ?? ?? ?? 8B 8E 60`

The second initializer for the XiEvent object. This is used to finalize the initialization of the object and put it into a 'ready' state.

  * Obtains the entity attached to the event object.
  * Copies the entities `Movement.LocalPosition` coords into the events `xieventex_t::EventPos`. (X Y Z)
  * Copies the entities `Movement.LocalPosition` directions into the events `xieventex_t::EventDir`. (Yaw, pitch, roll.)
  * Copies the entities `AnimationSpeed` into the events `xieventex_t::MainSpeedBase`.
  * The entity is tested for type: `CXiSkeletonActor`
    * If valid, the entities `Render.Flags3` is adjusted and `NpcSpeechFrame` is reset to -1.
  * The entity type is tested for 0, 1, 2, 6, 7
    * If one of those types is true, then the entity is further tested for:
    * `if ((entity->Status == 47 || entity->Status is sitting in chair (63 to 83) || entity->Status == 48) && entity->Type == 1)`
      * `if (( entity->Render.Flags0 & 4 ) == 0)`
        * `Status` is copied into `EventStatus`
    * `else if (( entity->Render.Flags0 & 4 ) == 0)`
      * `entity->StatusEvent = 0;`
  * `else if ((entity->Render.Flags0 & 4) == 0)`
      * `entity->StatusEvent = 0;`
  * Further setup of the main event object is continued.
  * The initial ReqStack entry `ReqStack[0]` is populated with the main event information.
  * The running `ReqStack` instance `StackExecPointer` is copied into the `ExecPointer`.
  * The event id array is checked twice.
    * The first check is to attempt to find the expected event id that is set to run.
      * If this is found, the `ReqStack[0]` entry is updated with the index of the found matching event id in the array.
      * `ReqStack[0]` is also updated to begin running the given event. (`Priority` for this initial run is set to 16.)
      * If not found, the second check happens.
    * The second check is to attempt to find an event id entry of `0xFFFE`.
      * If this is found, the `ReqStack[0]` entry is updated with the index of the found matching event id in the array.
      * `ReqStack[0]` is also updated to begin running the given event. (`Priority` for this initial run is set to 16.)
      * If not found, the event object is still marked as initialized and returns without setting up the `ReqStack[0]` object any further.

## `XiEvent::EventIdle`

**Pattern:** `56 8B F1 8B 46 08 85 C0 0F ?? ?? ?? ?? ?? 57 BA FF 00 00 00 33 C0 8D 7E 24 0F BF 0F 3B CA`

Sets up the `ReqStack` that will be executed on the current tick then runs `XiEvent::ExecProg` in a loop until `RetFlag` is set.

When this function first starts, it ensures that an entity server id has been set. If not, then it returns 0. 

Next, it determines the `ReqStack` that has the 'highest' `Priority`. The one found most important to run has it's index set into `RunPos`. This looks like:

```cpp
    Priority = 255;
    index = 0;
    ReqStack = this->ReqStack;
    do
    {
      if ( (__int16)ReqStack->Priority <= Priority )
      {
        Priority = (__int16)ReqStack->Priority;
        this->RunPos = index;
      }
      ++index;
      ++ReqStack;
    }
    while ( index < 16 );
```

_`Priority` is handled in reverse. 255 is the default and considered the lowest priority._

Once the loop has finshed the following then happens:

  * `if ((!EventExecEnd && !CancelEvent) || UnknownFlag != 0) && Priority != 255)`
    * If true, then the event is ready to run and the following happens:
      * The entity of the event is obtained.
      * `entity->Render.Flags1 &= ~0x20000;`
      * The `ExecPointer` is set to the `ReqStack[RunPos].StackExecPointer` value.
      * `XiEvent::ExecProg` is called in a loop until `RetFlag` is set.
      * When the loop breaks, the current `ExecPointer` is stored into the `ReqStack[RunPos].StackExecPointer` for the next tick to continue the event.
    * If false, the function returns.

## `XiEvent::ExecProg`

**Pattern:** `0F BF 44 24 04 3D ?? ?? 00 00 0F ?? ?? ?? ?? ?? FF`

The main virtual machine handler. This is where all opcodes are handled as the byte code is being processed when the event is executing.

This function is a giant switch case that handles the various opcodes. The default case is for opcode: `0x0000`

The opcode `0x0000` handler looks like this:

```cpp
void __thiscall FUNC_XiEvent_OpCode_0x0000(xievent_t *this)
{
    this->ReqStack[this->RunPos].StackExecPointer = 0;
    this->ReqStack[this->RunPos].Priority = 255;
    this->ReqStack[this->RunPos].WaitTime = -1.0;
    this->ReqStack[this->RunPos].WhoServerId = 0;
    this->ReqStack[this->RunPos].TagNum = 0;
    this->RetFlag = 1;
}
```

Opcode `0x0000` is used to stop/reset the current `ReqStack` object. Since this handler sets `RetFlag` then the loop processing `XiEvent::ExecProg` will break. When this happens, `ExecPointer` is then stored into `ReqStack[RunPos].StackExecPointer`. However, this also sets the `Priority` to 255, which means on next tick, this stack is not considered valid and will fail the check done in `XiEvent::EventIdle` for `Priority`. 

## `XiEvent::eventgetcode`

**Pattern:** `8B 51 20 33 C0 66 8B 81 56 02 00 00 8B 4C 24 04 03 C2 33 D2 8A 14 01 03 C8 33 C0 8A 41 01 C1 E0 08 03 C2 C2 04 00`

Reads a two-byte value from the current `EventData`, starting index is based on the current `ExecPointer`.

This function looks like this:

```cpp
int __thiscall FUNC_XiEvent_eventgetcode(xievent_t *this, int index)
{
    const auto data = &this->EventData[this->ExecPointer];

    return data[index] + (data[index + 1] << 8);
}
```

## `XiEvent::eventgetcode2`

**Pattern:** `8B 51 20 33 C0 66 8B 81 56 02 00 00 8B 4C 24 04 03 C2 33 D2 8A 54 01 02 03 C8 33 C0 8A 41 03 C1`

Reads a four-byte value from the current `EventData`, starting index is based on the current `ExecPointer`.

This function looks like this:

```cpp
int __thiscall FUNC_XiEvent_eventgetcode2(xievent_t *this, int index)
{
    const auto data = &this->EventData[this->ExecPointer + index];

    return data[0] + ((data[1] + ((data[2] + (data[3] << 8)) << 8)) << 8);
}
```

## `XiEvent::getworkofs`

**Pattern:** `8B 44 24 04 6A 00 50 E8 04 00 00 00 C2 04 00` _(Forward wrapper.)_ \
**Pattern:** `8B 44 24 04 56 50 8B F1 E8 ?? ?? ?? ?? 03 44 24 0C 84 E4 0F ?? ?? ?? ?? ?? 3D 00 08 00 00 7D ?? 83 F8 50 7C ?? 50 68 ?? ?? ?? ?? E8`
  * **Note:** _This pattern is shared with `XiEvent::getworkstrofs`, there will be two results, this function is the first result!_

Reads a value (via `XiEvent::eventgetcode`) from the `EventData` then uses that value to handle another lookup. This can be used to do a number of various data read requests. When the function is called, it is passed the `XiEvent` object, and two additional params, one is an index, the other is an index adjustment. Since the values are designed and read using 2 byte values with `XiEvent::eventgetcode`, the 3rd parameter can be used to shift and extend the result of the initial lookup index. This looks like:

```cpp
int __thiscall FUNC_XiEvent_getworkofs(xievent_t *this, int index, int indexShift)
{
    const auto val = indexShift + FUNC_XiEvent_eventgetcode(this, index);

    // ...
```

After this happens, then `val` is used to determine what to look up based on it's value.

  * `if ((val & 0x8000) != 0)`
    * `val` is considered a `References` entry index and returns: `this->References[4 * (val & 0x7FFF)]`
  * `else if (val < 2048)`
    * `if (val >= 80)`
      * `val` is considered invalid, 0 is returned.
    * `else`
      * `val` is considered a `WorkLocal` index and returns: `this->ExtData[1]->WorkLocal[val]`
  * `else if (val < 4352)`
    * `if ((val - 4096) < 96 && (val - 4096) >= 0)`
      * `val` is considered a `Work_Zone` index and returns: `Work_Zone[val - 4096]` (`Work_Zone` is a shared global array in `FFXiMain.dll` for the zone to use for all events.)
    * `else`
      * `val` is considered invalid, 0 is returned.
  * `else if (val < 4608)`
    * `if ((val - 4352) < 96 && (val - 4352) >= 0)`
      * `val` is considered a `Work_Zone_Memorize` index and returns: `Work_Zone[val - 4352]` (`Work_Zone_Memorize` is a shared global array in `FFXiMain.dll` for the zone to use for all events.)
    * `else`
      * `val` is considered invalid, 0 is returned.
  * `else if (val < 6144)`
    * `if ((val - 5888) < 96 && (val - 5888) >= 0)`
      * `val` is considered a `Work_Zone_1700` index and returns: `Work_Zone_1700[val - 5888]` (`Work_Zone_1700` is a shared global array in `FFXiMain.dll` for the zone to use for all events.)
    * `else`
      * `val` is considered invalid, 0 is returned.
  * `else if (val < 32640)` _(Handled as a switch case of `val` if this if is true.)_
    * The `EntityTargetIndex[1]` is used to obtain the entity at that given index, if invalid, the return is 0. If valid, then:
      * `val == 0x7F00` - Returns: `this->ExtData[1]->EventPos[0] * 1000.0`
      * `val == 0x7F01` - Returns: `this->ExtData[1]->EventPos[1] * 1000.0`
      * `val == 0x7F02` - Returns: `this->ExtData[1]->EventPos[2] * 1000.0`
      * `val == 0x7F03` - Returns: `enDirCli(this->ExtData[1]->EventDir[1]) * 4096.0 * 0.15915963` _(enDirCli is used to convert a radian to a single byte value.)_
      * `val == 0x7F06` - Returns the players current job id if the event entity (`EntityTargetIndex[1]`) is the player, 0 otherwise.
      * `val == 0x7F07` - Returns the entities race.
      * `val == 0x7F08` - Returns the players current job level if the event entity (`EntityTargetIndex[1]`) is the player, 1 otherwise.
      * `val == 0x7F0A` - Returns the entities server id.
      * `val == 0x7F0B` - Returns: `(entity->Render.Flags01 >> 25) & 1`
      * `default` - Returns: 0
  * `else if (val >= 0x7FFF)`
    * `val` is invalid, return is 0.
  * `else` _(Handled as a switch case of `val` for player entity data.)_
    * `val == 0x7F80` - Returns: `entity->LocalPosition.LocalX * 1000.0` for the local player entity.
    * `val == 0x7F81` - Returns: `entity->LocalPosition.LocalZ * 1000.0` for the local player entity.
    * `val == 0x7F82` - Returns: `entity->LocalPosition.LocalY * 1000.0` for the local player entity.
    * `val == 0x7F83` - Returns: `enDirCli(entity->LocalPosition.Unknown0000) * 4096.0 * 0.15915963` for the local player entity. _(enDirCli is used to convert a radian to a single byte value.)_
    * `val == 0x7F86` - Returns the players current job id.
    * `val == 0x7F87` - Returns the players race.
    * `val == 0x7F88` - Returns the players current job level.
    * `val == 0x7F8A` - Returns the players server id.
    * `val == 0x7F8B` - Returns: `(entity->Render.Flags01 >> 25) & 1` for the local player entity.
    * `default` - Returns: 0

## `XiEvent::getworkstrofs`

**Pattern:** `8B 44 24 04 56 50 8B F1 E8 ?? ?? ?? ?? 03 44 24 0C 84 E4 0F ?? ?? ?? ?? ?? 3D 00 08 00 00 7D ?? 83 F8 50 7C ?? 50 68 ?? ?? ?? ?? E8` \
**Note:** _This pattern is shared with `XiEvent::getworkofs`, there will be two results, this function is the second result!_

Reads a string value (via `XiEvent::eventgetcode`) from the `EventData` then uses that value to handle another lookup. This can be used to do a number of various data read requests. When the function is called, it is passed the `XiEvent` object, and two additional params, one is an index, the other is an index adjustment. Since the values are designed and read using 2 byte values with `XiEvent::eventgetcode`, the 3rd parameter can be used to shift and extend the result of the initial lookup index. _(See `XiEvent::getworkofs` for reference of what that looks like.)_

The returned values from this function are multiple and stored into a global array instead of being directly returned. This function is designed to read 15 bytes (strings) and automatically enforces a null terminator at the 16th byte.

  * `if ((val & 0x8000) != 0)`
    * Returns the pointer to a global array of data. (Data is default to 0.)
  * `else if (val < 2048)`
    * `if (val >= 80)`
      * `val` is considered invalid, 0 is returned.
    * `else`
      * `val` is considered a `WorkLocal` index and returns: `&this->ExtData[1]->WorkLocal[val]`
  * `else if (val < 4352)`
    * `if ((val - 4096) >= 96 || (val - 4096) < 0)`
      * `val` is considered invalid, 0 is returned.
    * `else`
      * `val` is considered a `Work_Zone` index and returns: `&Work_Zone[val - 4096]`
  * `else if (val < 4608)`
    * `if ((val - 4352) >= 64 || (val - 4352) < 0)`
      * `val` is considered invalid, 0 is returned.
    * `else`
      * `val` is considered a `Work_Zone_Memorize` index and returns: `Work_Zone_Memorize[val - 4352]`
  * `else if (val >= 6144)`
    * `val` is considered invalid, pointer to a global array of data. (Data is default to 0.)
  * `else if (val < 6144)`
    * `if ((val - 5888) >= 32 && (val - 5888) < 0)`
      * `val` is considered invalid, 0 is returned.
    * `else`
      * `val` is considered a `Work_Zone_1700` index and returns: `Work_Zone_1700[val - 5888]`
  * `else`
      * `val` is considered invalid, 0 is returned.

When a valid `val` index occurs, the resulting data that is mentioned above is copied into a global array. The address to the start of this global array is what is actually returned.

That looks like this:

```cpp
    // Copies the data from ptr into the global array..
    dword_1047F23C = ptr[0];
    dword_1047F240 = ptr[1];
    dword_1047F244 = ptr[2];
    dword_1047F248 = ptr[3];

    // Sets the null terminator..
    *(uint8_t*)(((uint8_t*)dword_1047F248)[3] = 0;

    // Return the address  to the global array..
    return &dword_1047F23C;
```

_This is seen being used to rename an entity during an event._

## `XiEvent::setworkofs`

**Pattern:** `8B 44 24 04 56 50 8B F1 E8 ?? ?? ?? ?? 03 44 24 10 84 E4 0F ?? ?? ?? ?? ?? 3D 00 08 00 00 7D ?? 83 F8 50 7C ?? 50 68 ?? ?? ?? ?? E8` \
**Pattern:** `8B 44 24 08 8B 54 24 04 6A 00 50 52 E8 0F 00 00 00 C2 08 00` _(Forward wrapper.)_

Writes a value to one of the various buffers available to events. This function first reads a value (via `XiEvent::eventgetcode`) from the `EventData` then uses that value to handle another lookup. This can be used to do a number of various data write requests. When the function is called, it is passed the `XiEvent` object, and three additional params, first is an index, second is an index adjustment, and the third is the value to write. Since the values are designed and read/written using 2 byte values with `XiEvent::eventgetcode`, the 3rd parameter can be used to shift and extend the result of the initial lookup index. This looks like:

```cpp
void __thiscall FUNC_XiEvent_setworkofs(xievent_t *this, int index, int value, int indexShift)
{
    const auto val = indexShift + FUNC_XiEvent_eventgetcode(this, index);

    // ...
```

After this happens, then `val` is used to determine what to write the value to based on its value.

  * `if ((val & 0x8000) != 0)`
    * `val` is considered a `References` entry index and cannot be written to, function returns.
  * `else if (val < 2048)`
    * `if (val >= 80)`
      * `val` is considered invalid, function returns.
    * `else`
      * `val` is considered a `WorkLocal` index and writes: `this->ExtData[1]->WorkLocal[val] = value`
  * `else if (val < 4352)`
    * `if ((val - 4096) >= 96 || (val - 4096) < 0)`
      * `val` is considered invalid, function returns.
    * `else`
      * `val` is considered a `Work_Zone` index and writes: `Work_Zone[val - 4096] = value`
  * `else if (val < 4608)`
    * `if ((val - 4352) >= 64 || (val - 4352) < 0)`
      * `val` is considered invalid, function returns.
    * `else`
      * `val` is considered a `Work_Zone_Memorize` index and writes: `Work_Zone_Memorize[val - 4352] = value`
  * `else if (val < 6144)`
    * `if ((val - 5888) >= 32 || (val - 5888) < 0)`
      * `val` is considered invalid, function returns.
    * `else`
      * `val` is considered a `Work_Zone_1700` index and writes: `Work_Zone_Memorize[val - 5888] = value`
  * `else if (val > 32640)`
    * `val` is invalid, function returns.
  * `else`
    * `val` is then changed to: `val -= 32512` and handled as a switch case:
      * `val == 0` - Writes: `this->ExtData[1]->EventPos[0] = value * 0.001`
      * `val == 1` - Writes: `this->ExtData[1]->EventPos[1] = value * 0.001`
      * `val == 2` - Writes: `this->ExtData[1]->EventPos[2] = value * 0.001`
      * `val == 3` - Writes: `this->ExtData[1]->EventDir[1] = value * 6.283 * 0.00024414062`
      * `default`
        * `val` is invalid, function returns.

## `XiEvent::setworkstrofs`

**Pattern:** `8B 44 24 04 56 50 8B F1 E8 ?? ?? ?? ?? 03 44 24 10 84 E4 0F ?? ?? ?? ?? ?? 3D 00 08 00 00 7D ?? 83 F8 40 7C ?? 50 68 ?? ?? ?? ?? E8`

Writes a string value to one of the various buffers available to events. This function first reads a value (via `XiEvent::eventgetcode`) from the `EventData` then uses that value to handle another lookup. This can be used to do a number of various data write requests. When the function is called, it is passed the `XiEvent` object, and three additional params, first is an index, second is a pointer to the string data, and the third is the index adjustment. Since the value returned from `XiEvent::eventgetcode` is an index and limited to 2 bytes, the last parameter is used as an index shifter to extend the value beyond the uint16_t limit. This looks like:

```cpp
void __thiscall FUNC_XiEvent_setworkstrofs(xievent_t *this, int index, void* buffer, int indexShift)
{
    const auto val = indexShift + FUNC_XiEvent_eventgetcode(this, index);

    // ...
```

After this happens, then `val` is used to determine what to write the value to based on its value.

  * `if ((val & 0x8000) != 0)`
    * `val` is considered a `References` entry index and cannot be written to, function returns.
  * `else if (val < 2048)`
    * `if (val >= 64)`
      * `val` is considered invalid, function returns.
    * `else`
      * `val` is considered a `WorkLocal` index and writes: `std::memcpy(&this->ExtData[1]->WorkLocal[val], buffer, 16);`
  * `else if (val < 4352)`
    * `if ((val - 4096) >= 80 || (val - 4096) < 0)`
      * `val` is considered invalid, function returns.
    * `else`
      * `val` is considered a `Work_Zone` index and writes: `std::memcpy(&Work_Zone[val - 4096], buffer, 16);`
  * `else if (val < 4608)`
    * `if ((val - 4352) >= 48 || (val - 4352) < 0)`
      * `val` is considered invalid, function returns.
    * `else`
      * `val` is considered a `Work_Zone_Memorize` index and writes: `std::memcpy(&Work_Zone_Memorize[val - 4352], buffer, 16);`
  * `else if (val >= 6144)`
    * `val` is considered invalid, function returns.
  * `else`
    * `if ((val - 5888) >= 16 || (val - 5888) < 0)`
      * `val` is considered invalid, function returns.
    * `else`
      * `val` is considered a `Work_Zone_1700` index and writes: `std::memcpy(&Work_Zone_1700[val - 5888], buffer, 16);`

_**Note:** This function **DOES NOT** automatically insert a null terminator onto the string data written. It is expected to be there already from the data being written!_

## `XiEvent::GetActorIndex`

**Pattern:** `8B 44 24 04 56 8D B0 40 00 00 80 83 FE 39`

Returns an entities target index and server id based on a hard-coded lookup value. _(This works similar to the above information about `GetActorNum`.)_

This function prototype looks like this:

```cpp
bool __thiscall FUNC_XiEvent_GetActorIndex(xievent_t *this, int32_t lookupValue, uint32_t *serverId, uint16_t *targetIndex);
```

  * `0x7FFFFFC0`, `0x7FFFFFF0`, `0x7FFFFFF9`
    * Returns the local players entity information.
  * `0x7FFFFFC1`, `0x7FFFFFC2`, `0x7FFFFFC3`, `0x7FFFFFC4`, `0x7FFFFFC5`
    * These are intentionally overflowed upward by `+0x40`, resulting in the values: 1, 2, 3, 4, 5
    * These values are used to return the local players party members info (based on their index in the party, 0 is skipped for local player). (Party 0)
  * `0x7FFFFFC6`, `0x7FFFFFC7`, `0x7FFFFFC8`, `0x7FFFFFC9`, `0x7FFFFFCA`, `0x7FFFFFCB`
    * These are intentionally overflowed upward by `+0x44`, resulting in the values: 10, 11, 12, 13, 14, 15
    * These values are used to return the local players alliance party members info. (Party 1)  
  * `0x7FFFFFCC`, `0x7FFFFFCD`, `0x7FFFFFCE`, `0x7FFFFFCF`, `0x7FFFFFD0`, `0x7FFFFFD1`
    * These are intentionally overflowed upward by `+0x48`, resulting in the values: 20, 21, 22, 23, 24, 25
    * These values are used to return the local players alliance party members info. (Party 2)
  * `0x7FFFFFF1`, `0x7FFFFFF2`, `0x7FFFFFF3`, `0x7FFFFFF4`, `0x7FFFFFF5`
    * These are used when referecing the local player party members. (Used for quests such as `Introduction to Teamwork`)
    * These values can cause an outgoing `0x0076` packet to be sent if party data is missing.
  * `0x7FFFFFF8`
    * Returns the local event entity information. (`EntityServerId[1]` and `EntityTargetIndex[1]`)
  * The default handler tests if the value is a normal entity server id for NPCs. `if ((val & 0xFF000000) != 0)`
    * If true, the return values will be the value given as the server id and `val & 0x3FF` to get the target index.
    * If false, the return values default to the local event entity information. (`EntityServerId[1]` and `EntityTargetIndex[1]`)
    
When the return values are set (they are put back into the incoming parameters), the function then returns true or false based on if the entity is valid/exists, and if the server id matches what was determined.

## `XiEvent::GetReqLevel`

**Pattern:** `33 C0 8B 54 24 04 66 8B 81 58 02 00 00 56 C1 E0 05 0F BF 44 08 24 3B C2 7F ?? 33 C0 5E C2 04 00`

Determines if the given priority is more important than the current running `ReqStack[RunPos].Priority`. If not, the return is 0. If it is, then it is compared to every `Priority` in the `ReqStack`. If it is more important than all 16 entries, the return is 1.

This function seems to only be used with opcode: `0x002A` The priority passed to `XiEvent::GetReqLevel` to be checked is the byte directly after the opcode. (`EventData[ExecPointer + 1]`) If the return is 0, then the `RetFlag` is set, otherwise, the execution continues as normal.

This function looks like this:

```cpp
int __thiscall FUNC_XiEvent_GetReqLevel(xievent_t *this, int priority)
{
    if (this->ReqStack[this->RunPos].Priority > priority)
    {
        auto count = 0;
        for (auto i = this->ReqStack; i->Priority > priority; ++i)
        {
            if (++count >= 16)
                return 1;
        }
    }

    return 0;
}
```

## `XiEvent::GetReqStatus`

**Pattern:** `33 C0 8B 54 24 04 66 8B 81 58 02 00 00 56 C1 E0 05 0F BE 44 08 3A 3B C2 75 ?? 33 C0 5E C2 04 00`

Determines if the given event id (tagnum) is currently running or is queued to run in any `ReqStack` entries. 

  * Returns 0 if the current `ReqStack[RunPos].TagNum` equals the given event id.
  * Returns 1 if one of the non-currently running `ReqStack` objects contains the given event id.
  * Returns -1 if no `ReqStack` is found with the given event id.

This function looks like this:

```cpp
int __thiscall FUNC_XiEvent_GetReqStatus(xievent_t *this, int32_t eventId)
{
    if (this->ReqStack[this->RunPos].TagNum == eventId)
        return 0;

    auto count = 0;
    for (auto i = &this->ReqStack[0].TagNum; *i != eventId; i += 32)
    {
        if (++count >= 16)
            return -1;
    }

    return 1;
}
```

## `XiEvent::ReqSet`

**Pattern:** `55 56 57 8B F9 8B 4C 24 14 83 CE FF 33 C0 8D 57 3A 66 81 7A EA FF 00`

Attempts to set/update a `ReqStack` entry to new information.

  * Returns 0 if the given event id (tagnum) is already found in the `ReqStack` array.
  * Returns 1 if the function is successful and has updated a `ReqStack` entry.
  * Returns 2 if no available `ReqStack` index was found.

When the function is first called, it loops through all 16 of the `ReqStack` entries. If any entry has the matching `Tagnum` to the given event id, the function exits, returning 0. Any slot with a `Priority` of 255 is stored as the to-be-used index into the `ReqStack` array. (The last found available slot is used.)

When the loop has finished, if no slot was found to be usable, the function exits and returns 2.

If a slot is found, then the function updates the following:

  * `ReqStack[idx].Priority` - Set to a passed parameter value.
  * `ReqStack[idx].TagNum` - Set to a passed parameter value.
  * `ReqStack[idx].StackExecPointer` - Set to a `EventOffsets` value, indexed by a passed parameter value.
  * `ReqStack[idx].WhoServerId` - Set to a passed parameter value.

_**Note:** This does not directly alter the current `RunPos` value or any other `XiEvent` state value. That will be handled automatically on the next tick while the event is still in a valid state based on the `Priority` values._

This function is used with the following opcodes: `0x0027`, `0x0028`, `0x0029`

## `XiEvent::lookatone`

**Pattern:** `8B 54 24 04 83 EC 08 8D 44 24 0C 56 8B F1 8D 4C 24 04 50 51 52 8B CE E8 ?? ?? ?? ?? 3C 01`

Sets an entity to be in an interactive state with another entity. When called, the function will validate the two entity lookup ids read passed to it to obtain the target indexes of both entities. If both entities are valid, then the function will do the following:

  * `entity1->Render.Flags3` is adjusted.
  * `entity1->NpcSpeechFrame` = speechFrame passed as a parameter to the function.
  * `entity1->InteractionTargetIndex` = ent2->TargetIndex

This function is used with the following opcodes: `0x001E`, `0x004A`, `0x0079`

---

_This data is current as of Feb. 28, 2022 retail client._