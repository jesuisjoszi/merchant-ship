# MerchantShip — integration docs

MerchantShip is a Hytale mod about harbours: abandoned ports you can claim with a charter, merchant ships that
sail in and trade, a notice board with offers and orders, and a quest board where the harbour hands out work.

This repository is not the mod. It is the public home of the parts other mods are meant to talk to, so you do
not have to go digging through a jar to find out what is callable and what it promises.

- **[CROSSMOD.md](CROSSMOD.md)** — the hooks, with examples: finding the nearest port, registering your own
  harbour contracts, getting a callback when one is filled, and calling all of it by reflection so MerchantShip
  stays an optional dependency for your mod.

The mod itself ships a plain-text copy of the same reference into its data folder on every server start
(`MerchantShip-API.txt`, next to `port.json`), and the admin panel repeats the short version under
Guide → *Dev option*.

## Getting the mod

MerchantShip is released on CurseForge. Bug reports, questions and "I would like a hook for X" are welcome as
issues here, or on Discord: <https://discord.gg/6cmUR94bhA>.

## Compatibility

MerchantShip plays nicely with StructureV2 (port sites avoid structures), DungeonBosses (avoids arenas),
Aetherhaven (town ports, and its towns keep their own building rights inside a port square) and MoreBandits
(pirate raids on voyages). All of those are optional — nothing breaks when they are missing.
