# Event VM Functions

This file contains information on the various functions that are used with the event virtual machine. The information here can be used to find, view and understand the various VM related functions in a tool such as IDA or Ghidra.

## `EventStartWait`

**Pattern:** `A0 ?? ?? ?? ?? C7 05 ?? ?? ?? ?? 00 00 00 00 84 C0 75 ?? B0 01 C3 E8 ?? ?? ?? ?? 83 F8 FF`

## `InitEvent2`

**Pattern:** `83 EC 14 53 55 56 8B 35 ?? ?? ?? ?? 57 C7 44 24 18 00 00 00 00 8B 06 83 C6 04 89 44 24 20 89 74 24 1C 8D 1C 85 00 00 00 00 C1 EB 02 85 C0 0F 8E`

## `XiAtelBuff::EventNew`

**Pattern:** `56 57 8B F9 8A 87 20 01 00 00 84 C0 78 5B 68 ?? ?? ?? ?? E8 ?? ?? ?? ?? 83 C4 04 85 C0 74`

## `XiEvent::XiEvent`

**Pattern:** `53 55 56 57 68 ?? ?? ?? ?? 8B F1 E8 ?? ?? ?? ?? 33 DB 83 C4 04 3B C3 74 ?? 8B C8 E8`

## `XiEvent::~XiEvent`

**Pattern:** `56 8B F1 57 8B 46 04 66 8B 0E 89 46 08 8B 86 5C 02 00 00 85 C0 66 89 4E 02`

## `XiEvent::XiEventInit`

**Pattern:** `53 55 56 8B F1 33 C0 33 DB 66 8B 46 02 57 8B 04 85 ?? ?? ?? ?? 3B C3 0F ?? ?? ?? ?? ?? 8B 8E 60`

## `XiEvent::EvenIdle`

**Pattern:** `56 8B F1 8B 46 08 85 C0 0F ?? ?? ?? ?? ?? 57 BA FF 00 00 00 33 C0 8D 7E 24 0F BF 0F 3B CA`

## `XiEvent::ExecProg`

**Pattern:** `0F BF 44 24 04 3D ?? ?? 00 00 0F ?? ?? ?? ?? ?? FF`

## `XiEvent::eventgetcode`

**Pattern:** `8B 51 20 33 C0 66 8B 81 56 02 00 00 8B 4C 24 04 03 C2 33 D2 8A 14 01 03 C8 33 C0 8A 41 01 C1 E0 08 03 C2 C2 04 00`

## `XiEvent::getworkofs`

**Pattern:** `8B 44 24 04 56 50 8B F1 E8 ?? ?? ?? ?? 03 44 24 0C 84 E4 0F ?? ?? ?? ?? ?? 3D 00 08 00 00 7D ?? 83 F8 50 7C ?? 50 68 ?? ?? ?? ?? E8` \
**Pattern:** `8B 44 24 04 6A 00 50 E8 04 00 00 00 C2 04 00` _(Forward wrapper.)_

## `XiEvent::getworkstrofs`

**Pattern:** `8B 44 24 04 56 50 8B F1 E8 ?? ?? ?? ?? 03 44 24 0C 84 E4 0F ?? ?? ?? ?? ?? 3D 00 08 00 00 7D ?? 83 F8 50 7C ?? 50 68 ?? ?? ?? ?? E8`

## `XiEvent::setworkofs`

**Pattern:** `8B 44 24 04 56 50 8B F1 E8 ?? ?? ?? ?? 03 44 24 10 84 E4 0F ?? ?? ?? ?? ?? 3D 00 08 00 00 7D ?? 83 F8 50 7C ?? 50 68 ?? ?? ?? ?? E8` \
**Pattern:** `8B 44 24 08 8B 54 24 04 6A 00 50 52 E8 0F 00 00 00 C2 08 00` _(Forward wrapper.)_

## `XiEvent::setworkstrofs`

**Pattern:** `8B 44 24 04 56 50 8B F1 E8 ?? ?? ?? ?? 03 44 24 10 84 E4 0F ?? ?? ?? ?? ?? 3D 00 08 00 00 7D ?? 83 F8 40 7C ?? 50 68 ?? ?? ?? ?? E8`

## `XiEvent::GetActorIndex`

**Pattern:** `8B 44 24 04 56 8D B0 40 00 00 80 83 FE 39`

## `XiEvent::GetReqLevel`

**Pattern:** `33 C0 8B 54 24 04 66 8B 81 58 02 00 00 56 C1 E0 05 0F BF 44 08 24 3B C2 7F ?? 33 C0 5E C2 04 00`

## `XiEvent::GetReqStatus`

**Pattern:** `33 C0 8B 54 24 04 66 8B 81 58 02 00 00 56 C1 E0 05 0F BE 44 08 3A 3B C2 75 ?? 33 C0 5E C2 04 00`

## `XiEvent::ReqSet`

**Pattern:** `55 56 57 8B F9 8B 4C 24 14 83 CE FF 33 C0 8D 57 3A 66 81 7A EA FF 00`

## `XiEvent::lookatone`

**Pattern:** `8B 54 24 04 83 EC 08 8D 44 24 0C 56 8B F1 8D 4C 24 04 50 51 52 8B CE E8 ?? ?? ?? ?? 3C 01`
