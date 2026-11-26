# Story Engine (Fabric, Minecraft 1.21.11)

A small mod that plays scripted, text-based stories in Minecraft: chat dialogue, title cards,
action-bar messages, pauses, and server commands (for example VexBot's `/vexbot spawn`).
Stories are plain JSON files, so you can add new ones without touching any Java.

## Build it

1. Install **JDK 21** and **Gradle** (or copy the `gradlew` files from the official Fabric example mod).
2. In this folder run: `gradle wrapper --gradle-version 9.2.1` (only needed once, skip if you copied gradlew).
3. Run `./gradlew build` (Windows: `gradlew.bat build`).
4. Your mod is in `build/libs/story-engine-0.1.0.jar`. Put it in your `mods` folder with **Fabric API**.
5. To test inside your dev setup instead: `./gradlew runClient`.

## Play a story

- `/story list` shows stories
- `/story play example` starts one (tab-complete works)
- `/story stop` stops yours
- `/story reload` re-reads the files after you edit them

Stories live in `config/storyengine/stories/` inside your game folder.
The first launch copies `example.json` there.

## Story file format

```json
{
  "name": "My Story",
  "author": "Me",
  "steps": [
    { "type": "title", "title": "Chapter 1", "subtitle": "The Beginning" },
    { "type": "say", "speaker": "Narrator", "text": "Hello, {player}!" },
    { "type": "wait", "seconds": 2 },
    { "type": "command", "command": "vexbot spawn" }
  ]
}
```

| type | uses | default pause after |
|------|------|---------------------|
| `say` | `speaker`, `text` | 3s |
| `actionbar` | `text` | 3s |
| `title` | `title`, `subtitle` | 4s |
| `command` | `command` (no leading `/`, `{player}` = your name) | 0s |
| `wait` | `seconds` | 1s |
| `kit` | `id` (a name from the story's `kits`) | 0s |
| `fight` | `id` (a name from the story's `fights`) | 0s |

Any step can set `"seconds"` to change the pause after it.
Check VexBot's own page for its current commands (`/vexbot spawn`, `givekit`, teams, and so on).

## The journal (press J)

Press **J** in game to open the journal: pick a character, pick one of their episodes, press Play.
Each character's story is split into **arcs** (Wemmbu has eight, from the Empire Arc to the Kingdoms Arc), and each arc holds episodes. The mod then pastes the episode's build far away in the overworld, teleports you into it, and plays
the episode's story, which gives kits and spawns enemies scene by scene. `/story leave` takes you
back to where you were and removes the enemies. (`/story episode <id>` does the same as the journal.)

Characters, arcs and episodes are listed in `config/storyengine/journal.json`. A character lists its arcs by `"character"`, and an episode joins an arc with `"arc": "<arc id>"`. Empty arcs show "No episodes here yet". If you already ran an older version, delete `config/storyengine/journal.json` once so the new one is copied in. Each playable episode has:

```json
{
  "id": "forgotten_dimension",
  "title": "I Escaped Minecraft's Forgotten Dimension",
  "arc": "wemmbu_fake_identity",
  "description": "Shown in the journal.",
  "playable": true,
  "story": "forgotten_dimension",
  "build": { "schematic": "underworld.schem", "origin": [30000, 100, 30000], "spawn": [20, 2, 20], "pasteAir": true },
  "startKit": "", "startFight": ""
}
```

- `story` is the name of a file in `stories/` (without `.json`). Its kits and fights are used by the episode.
- `startKit` / `startFight` are optional: a kit or fight from that story, run the moment you arrive.
- Episodes with `"playable": false` show as "coming soon" in the journal.
- After editing `journal.json` or a story, run `/story reload`.

### Builds (.schem)

Make the build in a creative world with WorldEdit, then `//copy` and `//schem save underworld`.
Copy the `.schem` file into `config/storyengine/schematics/`. Sponge schematic versions 2 and 3 work.

- `origin` is where the schematic's lowest corner goes (default far away at 30000, 100, 30000).
  Give each episode a different origin so builds never overlap.
- `spawn` is where you appear, counted from the origin. Leave it `null` for the middle of the build.
- `pasteAir: true` also pastes the schematic's air, which clears terrain inside the build area.
- If the schematic file is missing, a plain stone-brick arena is used so the episode is still playable.
- Only blocks are pasted. Chest contents, sign text and other block entities are not.
- Big builds are pasted a few thousand blocks per tick, with a progress message, so the game doesn't freeze.

## Kits and fights

A story file can define `kits` and `fights`. Each one is a named list of normal Minecraft commands
(no leading slash, `{player}` = the player's name). A `kit` step gives the gear; a `fight` step
spawns the enemies. The story keeps playing while the player fights.

```json
"kits":   { "starter": [ "give {player} minecraft:golden_apple 4" ] },
"fights": { "ambush":  [ "execute at {player} run summon minecraft:vindicator ~4 ~ ~0 {Tags:[\"storyengine_fight\"]}" ] },
"steps":  [ { "type": "kit", "id": "starter" }, { "type": "fight", "id": "ambush" } ]
```

`forgotten_dimension.json` already has 9 kits and 3 fights placed at the matching scenes
(for example armored Wither Skeletons at the fortress, Soul Keepers in chapters 5 and 13).
Soul Keepers are armored Vindicators. To make a fight easier or harder, change the counts,
enchantments or armor in the `fights` section. Enemies carry the tag `storyengine_fight`, so
`/kill @e[tag=storyengine_fight]` clears leftovers.

If a kit or fight does nothing, check the game log: a command with wrong syntax prints an error there.

## How the code is organised (read in this order)

1. `Story.java`: the shape of a story file. `Journal.java` is the same idea for the journal.
2. `StoryLibrary.java`: finds and loads the JSON files.
3. `StoryRunner.java`: the timer-based player that performs each step (no sleeping; it counts game ticks).
4. `StoryEngineMod.java`: the entry point that registers the `/story` command, the tick hook and the journal messages.
5. `EpisodeManager.java`, `Schematic.java`, `SchematicPlacer.java`, `MiniNbt.java`: starting an episode, reading `.schem` files and pasting them.
6. `client/` and `net/`: the J key, the journal screen and the two messages between the player and the server.

## Notes

- Nothing here was compiled or run yet. Minecraft's code names change between versions, so if
  the build complains about a method name, send me the error and I'll fix it.
- The commands are open to all players. Add a permission check before using this on a public server.
- Only use story scripts you have permission to use, and credit the original creators.
