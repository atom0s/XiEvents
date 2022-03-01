# XiEvents

Reverse engineered information related to the various cutscene / event data and systems used with the MMORPG, Final Fantasy XI.

In this repository, you will find the following:

  * Information related to which DAT file is used for each zones events / cutscenes.
  * Information related to the event / cutscene DAT file structure.
  * Information related to the internal client event system / VM. (Virtual Machine)
  * Tools that can be used with the above information.
  * Any other related material to the event / cutscene DAT files.

# Event Data (Byte Code)

The reason this repository exists is because of the method in which Square Enix has implemented the event / cutscene system. Since FFXI was designed for PlayStation 2, there were heavy constraints on file storage space as well as available RAM. Leaving scripts as plain text is a waste of space as well as a heavier load to deal with memory/CPU wise when parsing and executing.

Because of these factors, the scripts are compiled down to a custom byte code and handled internally in the client with a custom virtual machine to deal with the byte code. When the scripts are made, they are most likely done in a custom plain text scripting language. However, when the game is compiled and prepared for release, these scripts are compiled down into single DAT files (per-zone) containing the various information related to the events for every NPC and other interactable in the zone.

The goal of this repository is to [fully] reverse the various parts of the event data, systems, etc. to allow for the following:

  * The ability to view event data. (Both in its understood byte code format, as well as in a reinterpreted script language we make from scratch.)
  * The ability to make edits to any existing event in the client.
  * The ability to make completely new events, with ease.

# LEGAL

This repository and its work is for educational purposes only. 

We (contributors) do not claim ownership of any copyright content related to, or associated with, Final Fantasy XI.

```
(c) 2002-2022 SQUARE ENIX CO., LTD. All Rights Reserved. Title Design by Yoshitaka Amano. 
FINAL FANTASY, TETRA MASTER and VANA'DIEL are registered trademarks of Square Enix Co., Ltd. SQUARE ENIX, 
PLAYONLINE and the PlayOnline logo are trademarks of Square Enix Co., Ltd.
```

Please note; the reverse engineering done by this repository and its contributors is entirely 'clean room'. We **DO NOT** have or use any leaked source code or other unpublished material. By contributing to this repository, you agree to the following:

  * You have not and are not employeed by Square Enix, in any capacity.
  * You do not and have never had any leaked material from Square Enix related to FFXI in any manner.
  * You do not and have never referenced any leaked or otherwise unreleased material from Square Enix related to FFXI in any manner.

We **DO NOT** claim ownership of any material or information gathered through the means of reverse engineering the client and its files for this purpose.
