# WoW Addon Development Wotlk v3.3.5a - Claude Skill

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) for developing, porting, and debugging World of Warcraft addons targeting the **3.3.5a client (build 12340, Interface 30300, Lua 5.1)** — the client base used by Project Epoch, Warmane, Ascension, Atlantiss, and other Classic+ private servers.

This is a generic skill for the WotLK 3.3.5a client; server-specific quirks (Project Epoch's custom seals/items, EpochFixes patterns, etc.) live in `references/server-specific/`.

## What this skill gives Claude Code

- **Routing** for which reference doc to read for any given addon-dev task (porting, debugging, new addon, etc.)
- **Lua 5.1 compatibility reference** covering both directions of porting:
  - Vanilla 1.12 (Lua 5.0) → WotLK 3.3.5 (Lua 5.1) — what's now available, what still needs rewriting
  - Modern/Retail (Lua 5.2+) → 3.3.5 — every 5.2+ feature that breaks the parser
- **Modern API → 3.3.5 replacement table** covering `C_Timer`, `C_AddOns`, `C_NamePlate`, `Settings.*`, `MenuUtil`, atlas textures, and more
- **API quick-reference** for the high-frequency 3.3.5 surface (frames, events, units, auras, CLEU, items, tooltips, slash commands, SavedVariables)
- **Frame/UI cookbook** covering tooltip ownership, taint avoidance, `hooksecurefunc` patterns, SecureHandlers, OnUpdate throttling
- **Ace3-on-3.3.5 specifics** including the canonical xpcall fix and library version recommendations
- **External tools guide** for the MilkyWay Codex MCP server, the FrameXML mirror, the awesome_wotlk DLL, and BLP/MPQ tooling
- **Three executable validators** that can be run by Claude Code (or by you) on any addon source tree:
  - `scripts/lint_lua51.py` — catches Lua 5.2+ syntax (`goto`, `//`, `&|<<>>`, `\z`, `_ENV`, `table.pack`, `bit32.*`)
  - `scripts/scan_xpcall.py` — catches variadic `xpcall` (the Ace3 v36+ break)
  - `scripts/validate_toc.py` — catches bad Interface versions, forbidden TOC directives, missing files, malformed SavedVariables
- **Two starter templates** under `assets/`: minimal addon (no dependencies) and Ace3-based addon

## File layout

```
wow-addon-wotlk-335/
├── SKILL.md                                      # Required by Claude Code; routing layer
├── README.md                                     # This file (humans only)
├── references/
│   ├── lua-51-compatibility.md                   # The killer doc; load for any porting/debugging task
│   ├── porting-guide.md                          # Modern → 3.3.5 + Vanilla → 3.3.5 mapping
│   ├── api-reference.md                          # 3.3.5 API quick reference
│   ├── frame-xml-cookbook.md                     # UI patterns, taint, hooks, SecureHandlers
│   ├── ace3-on-335.md                            # Ace3 specifics + xpcall fix
│   ├── external-tools.md                         # MCP server, awesome_wotlk DLL, FrameXML mirror, BLP/MPQ
│   └── server-specific/
│       └── project-epoch.md                      # Project Epoch quirks, EpochFixes patterns, DLL policy
├── scripts/
│   ├── lint_lua51.py                             # Lua 5.2+ syntax detector
│   ├── scan_xpcall.py                            # Variadic xpcall detector
│   └── validate_toc.py                           # TOC validator
└── assets/
    ├── minimal-addon/                            # Bare-bones addon template
    │   ├── MinimalAddon.toc
    │   └── main.lua
    └── ace3-addon/                               # Ace3-based addon template
        ├── Ace3Addon.toc
        ├── embeds.xml
        └── main.lua
```

## Installation (Claude Code)

### As a personal skill

Copy or symlink the `wow-addon-wotlk-335/` folder into:

```
~/.claude/skills/wow-addon-wotlk-335/
```

Restart your Claude Code session. The skill will appear in `available_skills` and trigger automatically when you ask about WoW 3.3.5 / WotLK / Project Epoch addon work.

### As a project skill

If you keep all your WoW addons in one parent directory, you can install the skill at the project level:

```
<your-addons-folder>/.claude/skills/wow-addon-wotlk-335/
```

This way Claude Code only loads the skill when you're working in that project tree.

### Verifying the install

Ask Claude Code something like:

> What's the Interface version for a WotLK addon, and write me a TOC for a new addon called "MyTestAddon".

Claude should produce a TOC with `## Interface: 30300` and offer to copy `assets/minimal-addon/` as a starting point.

## Using the validators directly

The Python scripts work standalone — you don't need Claude Code to run them.

```bash
# Lua 5.1 syntax check on an entire addon tree
python scripts/lint_lua51.py path/to/MyAddon/

# Find variadic xpcall (Ace3 v36+ break)
python scripts/scan_xpcall.py path/to/MyAddon/

# TOC sanity
python scripts/validate_toc.py path/to/MyAddon/MyAddon.toc
```

All three exit `0` on success, `1` if issues were found, `2` on invocation error. Suitable for CI hooks.

## What this skill does NOT include

By design, the following are NOT bundled — install / clone separately as your workflow needs them:

- **Blizzard's WoW Programming PDF** — copyrighted, not redistributable. Use wowpedia / `wowgaming/3.3.5-interface-files` instead.
- **Extracted MPQ assets** — copyrighted Blizzard content; extract from your own client.
- **Ace3 library code** — third-party; download from WoWAce / ElvUI-WotLK and embed under `Libs/` per the Ace3 template's `embeds.xml`.
- **MilkyWay Codex MCP server** — separate install via npx / your MCP client; see `references/external-tools.md`.
- **FrostAtom/awesome_wotlk DLL** — separate install; check your private server's ToS first (Project Epoch may treat this as client modification).

## Contributing

PRs welcome, especially for:

- **Server-specific reference files** under `references/server-specific/` (Warmane, Ascension, Atlantiss, Sirus)
- **Library version regressions**: if a new Ace3 / pfQuest-wotlk / etc. release is verified clean on 3.3.5, update the recommended-versions table in `references/ace3-on-335.md`
- **Validator coverage**: more Lua 5.2+ edge cases, more TOC quirks, additional fixture files under a `tests/` folder
- **Translations**: the references are English-only; FR/DE/ES translations would help non-English-speaking server communities

Run all three validators against a real addon tree before submitting — the skill should always pass its own checks.

## License

The skill content (markdown references, Python scripts, addon templates) is released under the **MIT License**. See `LICENSE`.

The skill links to and references third-party projects (Defcons/epoch-addons, FrostAtom/awesome_wotlk, wowgaming/3.3.5-interface-files, MilkyWay Codex, etc.); each retains its own license. No third-party code is bundled here.

"World of Warcraft" and related names are trademarks of Blizzard Entertainment. This skill is not affiliated with or endorsed by Blizzard.

## Acknowledgements

This skill draws on the work of:

- **NSO73/WoW-Vanilla-Addon-Resources** — the original Vanilla 1.12 / Turtle WoW addon dev toolkit whose structure inspired this skill
- **Defcons/epoch-addons** — the most comprehensive Project Epoch addon collection and the source of the Modern API → 3.3.5 replacement table and most of the EpochFixes patterns
- **wowgaming/3.3.5-interface-files** — the canonical FrameXML mirror
- **FrostAtom/awesome_wotlk** — engine extensions for missing modern APIs
- **suprsokr/MilkyWay Codex** — the MCP server for AI-assisted 3.3.5a API lookup
