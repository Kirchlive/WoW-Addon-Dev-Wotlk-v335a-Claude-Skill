---
name: wow-addon-wotlk-335
description: Develop, port, and debug World of Warcraft addons for the WotLK 3.3.5a (Interface 30300, Lua 5.1) client and Classic+ private servers built on it — including Project Epoch, Warmane, Ascension, and Atlantiss. Use this skill whenever the user asks about WoW addon development on a 3.3.5/3.3.5a/WotLK private server, mentions Project Epoch, references files in an `Interface/AddOns/` folder, asks to write or fix a `.toc` file with `Interface: 30300`, debugs Lua errors from a 3.3.5 client, ports an addon from Vanilla 1.12 / Retail / TBC to WotLK 3.3.5, or works with libraries like Ace3, pfQuest-wotlk, ElvUI-WotLK, or any addon in the WotLK 3.3.5 ecosystem — even if they don't explicitly say "WotLK" or "3.3.5".
---

# WoW Addon Development — WotLK 3.3.5a

A skill for writing, porting, and debugging World of Warcraft addons targeting the **3.3.5a client (build 12340, Interface 30300, Lua 5.1)** and the Classic+ private servers built on it.

## Environment facts (memorize these)

| Property | Value |
|---|---|
| Client patch | 3.3.5a |
| Build | 12340 |
| Interface version (`## Interface:` in TOC) | `30300` |
| Lua version | **5.1** (specifically 5.1.5; **no 5.2+ features**) |
| FrameXML source mirror | `wowgaming/3.3.5-interface-files` on GitHub |
| Engine extension library | `FrostAtom/awesome_wotlk` (DLL, optional) |

If the user is on a Classic+ server (Project Epoch, Ascension, etc.), the client is still 3.3.5a 12340 — server-side changes do not affect API surface. Server-specific quirks live in `references/server-specific/`.

---

## When to read which reference

Skill bodies stay small on purpose. Read only the references that the current task actually needs.

| User intent / signal | Read these references |
|---|---|
| **Writing a new addon from scratch** | `references/api-reference.md`, `assets/minimal-addon/` |
| **Porting from Vanilla 1.12 → WotLK** | `references/porting-guide.md`, `references/lua-51-compatibility.md` |
| **Porting from Retail/TBC → WotLK** | `references/porting-guide.md`, `references/api-reference.md` |
| **Debugging Lua errors from a 3.3.5 client** | `references/lua-51-compatibility.md`, `references/ace3-on-335.md` |
| **TOC file issues, addon won't load** | `scripts/validate_toc.py` (run it), then `references/api-reference.md` § TOC |
| **Working with Ace3, AceGUI, AceDB on 3.3.5** | `references/ace3-on-335.md` |
| **Frame/UI work, secure templates, taint** | `references/frame-xml-cookbook.md` |
| **Need APIs that don't exist in 3.3.5 (C_NamePlate, etc.)** | `references/external-tools.md` § FrostAtom/awesome_wotlk |
| **Project Epoch-specific question** (custom seals, EpochFixes, pfQuest-epoch) | `references/server-specific/project-epoch.md` |
| **MPQ/BLP asset extraction or conversion** | `references/external-tools.md` § BLP/MPQ |

If the user's task spans multiple categories, read the references in order — `lua-51-compatibility.md` and `porting-guide.md` are the most frequently relevant.

---

## Non-negotiable rules for generated code

These are the rules that, when broken, produce code that *looks* correct but silently breaks at runtime on a 3.3.5 client. Apply them to every Lua file you write or modify.

### Lua syntax — no 5.2+ features

The 3.3.5 client embeds **Lua 5.1.5**. The following 5.2+ constructs do not parse and will cause `PLAYER_LOGIN`-time crashes:

- `goto label` / `::label::` — Lua 5.2+
- `//` integer division — Lua 5.3+
- Bitwise operators `&` `|` `~` `<<` `>>` — Lua 5.3+
- `_ENV` upvalue — Lua 5.2+ (use `setfenv`/`getfenv` instead)
- `\z` string escape — Lua 5.2+

For the full list of replacements, read `references/lua-51-compatibility.md`.

### `xpcall` does NOT support extra arguments

This is the single most common 5.1-vs-modern bug:

```lua
-- BROKEN on 3.3.5: extra args silently dropped, func runs with self = nil
xpcall(func, handler, arg1, arg2)

-- CORRECT: pcall + manual error handler
local ok, err = pcall(func, arg1, arg2)
if not ok then handler(err) end
```

This breaks AceGUI-3.0 v36+, AceConfig-3.0, and any modern Ace3 release. When porting Ace3-based addons, scan for variadic `xpcall` first — `scripts/scan_xpcall.py` does this automatically.

### Modern API replacements (high-frequency subset)

Full table in `references/porting-guide.md`. Most-frequently-needed replacements:

| Modern API (Legion+) | 3.3.5 replacement |
|---|---|
| `C_Timer.After(delay, fn)` | `CreateFrame("Frame")` + one-shot `OnUpdate` (snippet in porting-guide.md) |
| `C_Timer.NewTicker(int, fn)` | persistent `OnUpdate` with elapsed accumulator |
| `Settings.RegisterCanvasLayoutCategory` | `InterfaceOptions_AddCategory(panel)` |
| `Settings.RegisterAddOnCategory` | `InterfaceOptions_AddCategory(panel)` |
| `C_AddOns.GetAddOnMetadata` | `GetAddOnMetadata` (no `C_` prefix) |
| `MenuUtil` / `Menu` / `CreateAnchor` | `UIDropDownMenu` + `UIDropDownMenu_AddButton` |
| `texture:SetAtlas(name)` | not available; `if texture.SetAtlas then ... end` guard |
| `texture:SetColorTexture(r,g,b,a)` | `texture:SetTexture(r, g, b, a)` |
| `frame:SetMask(path)` | not available; guard with capability check |
| `WOW_PROJECT_ID`, `WOW_PROJECT_MAINLINE` | not defined; nil-guard usage |
| `[AllowLoadGameType xxx]` in TOC | not supported; remove conditionals |

### TOC file format (3.3.5)

Minimum viable TOC — copy from `assets/minimal-addon/MinimalAddon.toc`:

```
## Interface: 30300
## Title: My Addon
## Notes: Description
## Author: YourName
## Version: 1.0.0
## SavedVariables: MyAddonDB
## SavedVariablesPerCharacter: MyAddonCharDB

main.lua
```

Run `python scripts/validate_toc.py path/to/Addon.toc` to catch invalid Interface versions, missing required fields, and 5.2+ TOC directives.

### Hooking Blizzard frames — use `hooksecurefunc`, never raw replace

```lua
-- BROKEN: raw replacement taints the frame and breaks protected actions
local original = QuestLog_Update
function QuestLog_Update() original() ; my_logic() end

-- CORRECT: post-hook, taint-safe
hooksecurefunc("QuestLog_Update", function() my_logic() end)
```

This was the lesson learned by pfQuest-wotlk and is the root cause of about half of all "I can't cast spells in a raid frame" bug reports on Project Epoch.

---

## Validation workflow

Before declaring an addon "done", run these in order. They take seconds and catch the bugs that take hours to diagnose in-game.

```bash
# 1. TOC sanity
python scripts/validate_toc.py path/to/Addon/Addon.toc

# 2. Lua 5.1 syntax check (catches 5.2+ features)
python scripts/lint_lua51.py path/to/Addon/

# 3. xpcall scan (only relevant for Ace3-based or ported addons)
python scripts/scan_xpcall.py path/to/Addon/
```

If all three pass, the addon will at least *load* on a 3.3.5 client. Runtime behavior still needs in-game testing.

---

## External resources

For deep API lookups that go beyond what `references/api-reference.md` covers, prefer these external sources in this order:

1. **MilkyWay Codex MCP server** — if installed, gives token-efficient programmatic lookup of every 3.3.5a Lua API function, event, widget, CVar, and combat-log event. See `references/external-tools.md` for installation.
2. **`wowgaming/3.3.5-interface-files`** — full Blizzard FrameXML Lua/XML source mirror. When you need to know "how does Blizzard implement X", grep this repo locally.
3. **wowpedia.fandom.com** — canonical API documentation, but always verify against the FrameXML mirror because wowpedia includes Retail-only APIs without flagging them.

Never bundle copyrighted Blizzard PDFs, the official "WoW Programming" book, or extracted MPQ assets into addons or repos this skill produces — link to authoritative sources instead.

---

## Server-specific notes

This skill is generic for the 3.3.5a client. Server-specific behavior (custom items, custom spells, server-injected globals, custom realmlist quirks) lives in `references/server-specific/`. Currently covered:

- `references/server-specific/project-epoch.md` — Project Epoch (Vanilla content + TBC talents + custom seals/items, custom Ascension Launcher, EpochFixes patterns)

If the user is on a server not covered here, fall back to the generic 3.3.5a guidance and warn them that server-specific globals or custom items may behave differently.
