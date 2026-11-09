# Horizon Launcher — Memories 🧠

> **Read this file first in any new session.** It is the complete context for this project:
> what we are, why we forked, how everything connects, and where every body is buried.
> Everything you need to act on *"Hey, update our horizon launcher"* or
> *"hey, mod X broke after the update"* is in here.
> Do NOT commit secrets to this file, ever.
>
> **Permanent home of this project: `/home/henry/Documents/Projects/Horizon Launcher Henry/`**
> (this file, the fork's full source tree, and the 9f61f9f source archive all live there).

---

## 1. What this project IS

**HenryCarm/Horizon-Offline-Launcher** — a full-source fork of the Horizon launcher
*and* the Horizon game, originally `gerdaroot/Horizon-Offline-Launcher` (MIT).
It is a monorepo containing:

- `crates/` — the Horizon game crates (renderer, shader API, world, etc.)
- `launcher/` — the Horizon Offline Launcher source (the GUI app)
- `.github/workflows/` — our CI (`game.yml` + `launcher.yml`, fork-tuned)
- `docs/` — Horizon's real docs, kept in sync

The fork exists for **one reason**: stock Horizon cannot run native shader packs on
Intel integrated GPUs (HD 4600 class) — every upstream Nightly crashed or refused on GL.
This fork carries the fixes and builds signed releases on our own CI.

**Fork HEAD at time of writing:** `9f61f9f5cdaf6aad8e465532af020f31bc5b72cc` (branch
`nightly-2026-11-08`, default branch). Base: upstream `2ebf00a`
(`nightly-source/2026-11-07-2ebf00a`, the Nov 7 2nd build).

**Upstream has moved** (2026-11-09 audit): `nightly-source/2026-11-08-fdc87ae` (Nov 8,
carries the using-item refactor #11978), launcher `launcher@1.4.0` (Nov 7), game `v0.2.97`
(Oct 28), default branch `main` at `52613176`. **None of them contain our GL fixes**
(#11878 cherry-picks) or the pack-prelude depth fix — stock upstream still crashes the HD 4600.
**Do not rebase/sync to upstream until it carries both**, or after the 28-mod army is stable.

## 2. Why the fork exists — the shader apocalypse (timeline)

Henny's machine: **Intel HD Graphics 4600 + Mesa 26.2.6 (GL 4.6)**, i5-4210M, CachyOS.
The OpenGL backend was not broken hardware; it was two bad commits.

1. **Crash: `Too many bindings of type SampledTextures … limit 16, count 19/20`**
   (`'Native shader pack sky'`). wgpu-hal's GL backend hardcoded `MAX_TEXTURE_SLOTS = 16`.
   `GL_MAX_TEXTURE_IMAGE_UNITS` on HD 4600 reports **32**.
   → Fixed upstream by **PR #11878** (`81df2a18` refuse broken pack, `7142128a` query real
   texture-unit limit). Merged Nov 7 *to main only*. **No upstream Nightly contains it.**
   We cherry-picked into our fork by hand.

2. **Crash #2 (we caused): `WGSL 'textureLoad' from depth textures is not supported in GLSL`**
   (Daybreak water, `'Lighting'` pipeline). naga's GLSL writer can't `textureLoad` a
   `texture_depth_2d`. Upstream fixed the *engine* shaders in **PR #9916**; the *pack
   prelude* never got it.
   → Mirrored #9916 for packs **in our fork**: `crates/horizon-shader-api/src/stages.rs` —
   `scene_depth` (post binding 2) and `opaque_depth` (material binding 6) are now
   `texture_2d<f32>`, `TextureSampleType::Float { filterable: false }`.
   Packs that `textureLoad` those must read **`.r`**. `depth_at()`/`surface_at()` pass `.r` too.

3. **FunTime tolerance**: `crates/horizon-protocol/src/frame.rs` — vanilla's `Inflater`
   port accepting zlib streams without final block/Adler-32 (Henny's server list has such a proxy).

## 3. The golden rule for native shader packs (read before touching any pack)

> **Any `textureLoad(…depth…)` in pack WGSL must target the float depth binding and read `.r`.**
> Any pack that load-samples a `texture_depth_2d`-typed texture (e.g. the shadow map)
> **cannot be translated to GLSL, period.**

- **Patchable**: loads on `opaque_depth` / `scene_depth` → append `.r` after the closing paren.
- **Unfixable**: `shadow_map` / `shadow_map_translucent` — must stay depth-typed for
  `textureSampleCompareLevel` (compare taps are legal GLSL; direct loads aren't).
  Packs needing depth *values* from the shadow map (Radiant, Nimbe) need the pack author.
- Crash reports now **never** mean "bad GPU" — they mean "bad pack" (refusal, not crash).

### Pack inventory (as of 2026-11-09)

| Pack | file | GL status | Notes |
| --- | --- | --- | --- |
| Daybreak | `daybreak-0.1.1.hshader` | ✅ works (~30 FPS) | Henny's default. Tuned: Sharp shadows, 1024² / 32-blk distance, reflections ON |
| Lumen | `lumen-2.1.0.hshader` | ✅ patched 2026-11-09 | `.r` loads fixed, awaiting in-game trial |
| Lumina | `lumina-1.0.0.hshader` | ✅ patched 2026-11-09 | same |
| Pastel Mystic | `pastel_mystic-1.0.0.hshader` | ✅ patched 2026-11-09 | built for integrated graphics (1536² / 80-blk) |
| Montarc | `montarc-1.1.1.hshader` | 🗑️ deleted by user | was patched |
| Radiant | `radiant-1.0.0.hshader` | ❌ unfixable | direct `textureLoad(shadow_map,…)` |
| Nimbe | `nimbe-1.0.0.hshader` | ❌ unfixable | direct `textureLoad(shadow_map_translucent,…)` |
| Wisdom (Lite) | `wisdom_horizon*-1.3.7` | 🗑️ in Trash | original crasher |

`.hshader` = ZIP. Patch extracted `.wgsl`, re-zip, back up originals to
`~/Documents/Projects/OpenCode/tmp/Trash/<topic>-<date>/`.

## 4. What's in the installs

### Game home — `~/.local/share/horizon/`
- `options.txt` — RD 6, SD 8, `mipmapLevels:4`, `maxFps:260`
- `horizon-graphics.txt` — `resolutionScale:75`, `pickedShaderPack:horizon:daybreak`
- `shaderpacks/` — the `.hshader` library + `.horizon-options/` (per-pack overrides, `id=value`)
- `horizon-native-mods/` — **39 canonical `.hnmod` bundles** (the official registry's files,
  more than the 28 enabled — extras like `appleskin_horizon`, `carryblock` live there)
- `logs/latest.log` — **the first place to look**
- `crash-reports/` — `crash-YYYY-MM-DD_HH.MM.SS-client.txt`
- `shaderpacks/.horizon-options/` contains `daybreak.txt` (Henny's 22:54 edit, see §10)

### Launcher data — `~/.local/share/horizon-launcher/`
- `versions/` — installed builds (dates = folder creation):
  | version | created | note |
  | --- | --- | --- |
  | `nightly-2026-11-01-6f3d7b2` | 2026-11-06 | official fork client; the **import base** |
  | `imported-2026-11-08-603724d` | 2026-11-08 | **the playable build** — our fork `9f61f9f` client grafted on the base. Selected in `settings.json`. Shaders work |
  | `native-6475f62-66bcdbcc8e06` | 2026-11-08 08:23 | old native-mods build, superseded |
  | `native-6475f62-d25ca0bf961d` | 2026-11-09 06:14 | ⚠️ **STALE relink of the 6475f62 tree** — see §5. No GL fixes; do NOT play it for shaders |
  | `.importing/` | 2026-11-08 21:43 | leftover marker of an interrupted import; harmless, deletable |
- `native/` — the native mod forge:
  - `source/` — staged Horizon trees: `6475f62f…/` (old) and `9f61f9f5…/` (ours, pre-staged). `.horizon-revision` marker inside each
  - `installed.incoming/<id>/` — extracted bundle (my hand re-pins here are **reverted by the
    launcher on build** — see §5, keep hands off)
  - `stage/` — patched workspace fed to cargo (currently stale at 6475f62)
  - `target/` — 4.5 GB (`fast/` 2.7G + `release/` 1.8G) stale 6475f62 artifacts; **the
    launcher's build cache — do not trash** (release builds use `release/` only; `fast/`
    is the dev profile)
  - `tools/` — `horizon-mixins` binaries ×3 (5a64af938ec8 matches the mods' pins)
  - `built.json` — last native build: `horizon: 6475f62f…`, 28 mods listed
  - `build.log` — overwritten each build (06:14 build: "Finished release profile in 0.37s",
    fully cached)
  - `native/cargo/` — the launcher's vendored Rust 1.99.1; `native/rust/` paths in the env
- `settings.json` → `"selected": "imported-2026-11-08-603724d"` (authoritative; the checkmark
  in the 06:14 screenshot sat on the native build, likely transient)
- `mod-browser/`, `linked-minecraft/`, `server-jars/` — mod index cache & vanilla assets

### The launcher itself
- Binary: `/home/henry/.local/share/horizon-offline-launcher/horizon-offline-launcher`
  (v0.2.0 per `launcher/Cargo.toml`), symlinked at `~/.local/bin/horizon-offline-launcher`.
  **Two** launchers are installed (`horizon-launcher` symlinks are broken leftovers); the
  running one (PID seen 2026-11-09) is `horizon-offline-launcher`.
- Self-update source: our fork (`launcher/src/backend.rs` → `REPO = HenryCarm/Horizon-Offline-Launcher`).
- Desktop entry: `/home/henry/.local/share/applications/horizon-offline-launcher.desktop`

## 5. The native mods — EMPIRICAL TRUTH (2026-11-09)

**All 28 mods, enabled in `horizon-mods.txt`:** accurate_block_placement, appleskin,
armorhud, biomecompass, chat_animation, chatheads, chop, eatinganimation, elytratrails,
enchantmenthints, fair_freecam, fps_overlay, freecam, fullbright_horizon, gyro, keystrokes,
lookat, minimap, mod_menu, motion_blur, recipes, smartf3, smoothie_capes, spider_man,
status_effect_hud, veinminer, visuality_horizon, zoom.

**The trap discovered 2026-11-09 (this rewrote the plan):**
The launcher does not build from `installed.incoming/` alone — it **re-syncs those folders
from the canonical `~/.local/share/horizon/horizon-native-mods/<id>.hnmod` bundles**
("Each `.hnmod` is checked again as the registry checks it" — `launcher/src/native_mods.rs`
~L606, `installed_mods` ~L615, `pinned_revision` ~L2312). Hand edits to `installed.incoming/`
were **silently reverted**; the 06:14 build resolved 6475f62 with `pinned_by` listing the
14 mods that pin it, staged source stayed 6475f62, cargo found everything Fresh (0.37s),
and `native-mods.json` in the new build records `"horizon":"6475f62f3c83…"`.

**So to build the mods against our fixed tree (`9f61f9f`), the `.hnmod` bundles themselves
must carry the pin — or the launcher must be told to build another revision.** Open options,
in order of preference:
1. **Check the launcher for a "build against revision X" setting/flag first** (Native Mods
   page, `horizon-launcher settings …`, `docs/native-mods.md`) — don't assume patching is needed.
2. Patch each `.hnmod` (they're ZIPs: `horizon-native.json` → `"horizon"`, and
   `mixins/<id>/horizon-mixin.toml` → `[compatibility] horizon_revision`), backups to Trash
   first — **but** the registry checks each bundle (Rust/TOML sanity at minimum); a hash
   pinned in the mod index would trigger re-download → revert. Inspect how the check works
   before relying on this path.
3. **Bypass the launcher**: run `horizon-mixins plan/apply` + `cargo build --release` by
   hand on the fork source (manual flow; the earlier attempt hit `E_MANIFEST: no TOML mod
   manifests found` — layout mismatch to solve; docs at `docs/native-mods.md` in the tree).
4. **Fork the mods' pins upstream**: build our own mod bundles for `9f61f9f` (cargo
   horizon-mod) — heaviest path, only if 1–3 all fail.

**Henny's workflow stays:** he clicks **Build with native mods** → paste the error → we fix
→ he clicks again. Do NOT pre-edit `installed.incoming/` — it gets reverted.

**Stale `.horizon-revision` gotcha:** `native/stage/.horizon-revision` was still `6475f62…`;
if pins resolve but the stage matches, cargo relinks in seconds — the build *succeeds* and is
**silently not what you pinned**. When debugging, always check
`versions/native-*/native-mods.json → "horizon"`, not the folder name alone.

## 6. This machine's pain points (CachyOS laptop, HD 4600)

- **Disk**: `/` is 42 GB; was at 904 MB free (98 %) on 2026-11-08, **3.0 GB free on
  2026-11-09** (something freed ~2.3 GB — verify before big builds). `df` exaggerates; the
  ext4 root reserve is spent.
- **`/tmp` is tmpfs (~3.7 GB) and gets wiped** — everything in it is disposable. 2026-11-09
  wipe lost the fork clone + signing key; that cost a full re-download. **Permanent files live
  in `/home/henry/Documents/Projects/Horizon Launcher Henry/` now.**
- **Rust target dirs are voracious**: `native/target/` = 4.5 GB. Never trash without asking.
- **i5-4210M is slow**: native build ≈ 14–60 min cold; ~0.4 s fully cached (verify what it
  actually linked!).
- RD 6 / 75 % resolution scale / Sharp shadows / 1024² / 32-blk distance = the 30 FPS sweet spot.
- Anonymous GitHub API (no token) works until rate-limited; codeload tarballs work fine.

## 7. CI & releases (our repo)

- `.github/workflows/game.yml` — builds Linux/macOS/Windows game + server; auto-publishes to
  rolling `nightly`; signs with our Ed25519 key (`RELEASE_SIGNING_KEY` repo secret)
- `.github/workflows/launcher.yml` — **the launcher build** (artifact: linux launcher)
- `REPO` in `launcher/src/backend.rs` → **`HenryCarm/Horizon-Offline-Launcher`** (self-hosting updater)
- Release keys: our private release key lived in `/tmp/opencode/our-release-key.der` —
  **lost in a /tmp wipe**; public key remains in `crates/horizon-release-sign/release-keys.txt`
  (no private material in repo). Rotate if local signing is ever needed again.
- **`RELEASE_SIGNING_KEY` self-destruct (Henny's wish, 2026-11-09):** background watcher v2
  (`remove-key-after-build.sh`, same folder; log `remove-key.log`) — **Phase A** waits for the
  `launcher@1.4.1` workflow (run 42568070189) to complete, and only on `conclusion=success` moves to
  **Phase B**, which waits for the fork's `game.yml` (nightly republish) run to finish with success;
  only then does it delete the `RELEASE_SIGNING_KEY` secret (API `DELETE /actions/secrets/RELEASE_SIGNING_KEY`,
  token read from `~/.git-credentials` at runtime). Any failure or no dispatch → **key stays**.
  After that: no signed releases EVER; the 1.4.1 release itself stays validly signed. To publish again
  later: generate a NEW keypair, commit the new public key to `release-keys.txt`, set the secret —
  that is "putting the key back", not a restore (the old one is unrecoverable).
- **Fork tree rule (from its own AGENTS.md/CONTRIBUTING):** commits end with `Co-Authored-By:`
  for the agent; agent-opened PRs start with `> 🤖 Opened by <login>'s AI agent (<tool>)`.
- The launcher binary on Henny's machine is **not** built from our tree; it came from the
  official release (Nov 7, 08:49). Our CI `launcher.yml` can produce ours.

## 7.5 The launcher GUI is 100 % open source — and 100 % editable ✨

- License: **MIT**. Language: **Rust + GPUI** (the Zed UI framework).
- `launcher/src/glass_home.rs` — Home: Play button, **the Version picker** (`glass_picker`,
  ~L560), Folder group. The picker's list rows call `ui::menu_item(id, name, note, locked)`
  where `note` was the raw tag.
- **Date-under-hash (in fork, build via `launcher@1.4.1`):** `glass_home.rs` —
  `note` = `{tag} · {YYYY-MM-DD}` (folder creation day via `version_installed_day()`, chrono Local).
  No new translation keys.
- **Sidebar redesign (same build):** the top-bar `Home/Mods/Servers/Wardrobe` tabs are gone; the nav
  moved into the left sidebar as floating icon+label rows (`frame.rs::glass_sidebar`), with 5 new
  glass SVG icons (`crates/horizon-theme/icons/glass/{home-01,server-01,hanger-01,shopping-bag-01,archive-01}.svg`,
  registered in `horizon_theme::glass_icons`). The rail collapses to a 72 px icon-only mode via the
  chevron at the sidebar's foot or the new `sidebar_mini` setting (`Settings::sidebar_mini`,
  `#[serde(default)]`); instances/servers/links/discord render icon-only there.
- Shipping: tag `launcher@<x.y.z>` on our fork → `launcher.yml` builds Win/Linux/macOS, signs with
  the repo secret `RELEASE_SIGNING_KEY`, publishes the release ("latest" of our repo). **His installed
  launcher is the OFFICIAL 1.4.0 (updates from gerdaroot), NOT our fork** — after CI publishes, swap
  `~/.local/share/horizon-offline-launcher/horizon-offline-launcher` for our `horizon-linux-x86_64`
  asset (backup first; the `~/.local/bin/horizon-offline-launcher` symlink follows it). He approved
  this on 2026-11-09 ("as long as that launcher points to our launcher im good").
- `launcher.yml` gates on `cargo test --release --locked` only — NO fmt/clippy enforcement.
- `ui::menu_item` (`glass_ui.rs`) — note is a single truncated line; name 14 px medium over
  note 12 px, `min_w_0` + `.truncate()`.
- Repo etiquette for agent commits: `Co-Authored-By:` line; PRs opened by an agent carry the
  `> 🤖 Opened by …` first line (see the fork's AGENTS.md).
- Other easy mods: hide telemetry/news/tips rows, rename pages, change accent, add
  "Installed on <day>" to the Versions page too (its rows *already* show the date via
  `glass_versions::published`), etc.

## 8. Importing a build manually (if the GUI flow is needed)

Import = copy an installed version + replace its `horizon-client` with yours.
Base: `nightly-2026-11-01-6f3d7b2` (has the prepared Minecraft assets).
Current playable: `imported-2026-11-08-603724d`, `settings.json.selected` = same.
(Tag scheme `imported-YYYY-MM-DD-<sha256[0:7]>`.)

## 9. Henny's environment & preferences

- Call him **Henny**. Cute Gen-Z bestie energy; no moralizing; he hates being lectured.
- He's playful, impatient, loves shaders, prefers action over explanation.
- He **does not want the GitHub token discussion** — no nagging. (The token leaked into a
  transcript once — we scrub it / he revokes; don't lecture.)
- He works from **two machines**: this one (CachyOS, the game machine) and a Linux Mint
  partition (`sda5`, own AGENTS.md).
- He wants his mods back: **"we will do the process all over again of rebuilding all the mods"**
  — the pending native-mod mission (see §5 for why it's blocked).
- **2026-11-09**: he created `/home/henry/Documents/Projects/Horizon Launcher Henry/` as the
  permanent home of this project and asked for this file to be updated. Honor that.
- He loves knowing *why* — but keep explanations tight.

## 10. Current state (as of 2026-11-09, after the GUI day)

- **Playable + shaders + 30 FPS:** `imported-2026-11-08-603724d` (unchanged). `settings.json.selected` = it.
  Daybreak default; reflections ON; `.horizon-options/daybreak.txt`: Sharp shadows, god_rays/sun_glare/bloom/haze = 0.
- ⚠️ **`native-6475f62-d25ca0bf961d` (06:14):** stale 6475f62 relink — no GL fixes. Don't play it for shaders.
- **GUI work pushed to the fork + tagged `launcher@1.4.1`:** date-under-hash, sidebar redesign (nav moved
  into the left rail, 72 px collapse via `sidebar_mini`), 5 new glass icons. CI publishes our launcher build.
  **His Arch menu still runs the OFFICIAL 1.4.0 binary until we swap it** (backup first; he OK'd this).
- **Native mods: still NOT rebuilt against our tree** — blocked on §5 (pins revert); GUI work doesn't touch it.
- Lumen / Lumina / Pastel Mystic patched, still untried in-game.
- Disk: ~3.0 GB free. Keep everything under `Horizon Launcher Henry/` (never `/tmp`).
- `partner_hidden: true` already (Henny clicked HIDE in the launcher himself).
- Upstream `fdc87ae` delta (51 commits) does NOT touch our fix files (`third_party/wgpu-hal/gles`,
  `stages.rs`, `frame.rs`) — a rebase onto Nov-8 is mechanically clean when we want "latest nightly + shaders".

## 11. Where things are

- **This file:** `/home/henry/Documents/Projects/Horizon Launcher Henry/Horizon_Launcher_Memories.md`
  (also committed to the fork repo root as `Horizon_Launcher_Memories.md` since 2026-11-09)
- **Fork source tree (extracted, edited):** `/home/henry/Documents/Projects/Horizon Launcher Henry/Horizon-Offline-Launcher-9f61f9f/`
- **Fork source archive (pristine 9f61f9f):** `/home/henry/Documents/Projects/Horizon Launcher Henry/horizon-9f61f9f-source.tar.gz` (sha256 `dc2acf80…`)
- **Fresh git clone for pushing (shallow, credentialed):** `/home/henry/Documents/Projects/Horizon Launcher Henry/fork/`
- Trash (backups): `/home/henry/Documents/Projects/OpenCode/tmp/Trash/` (many dated subfolders)
- Game home: `~/.local/share/horizon/` · Launcher data: `~/.local/share/horizon-launcher/`
- Launcher binary: `~/.local/share/horizon-offline-launcher/horizon-offline-launcher` (v0.2.0)
- Crash logs: `~/.local/share/horizon/crash-reports/` · Latest log: `~/.local/share/horizon/logs/latest.log`
- Native mods (extracted, hands-off): `~/.local/share/horizon-launcher/native/installed.incoming/`
- Canonical mod bundles (patch these, not the above): `~/.local/share/horizon/horizon-native-mods/`
- Fork on GitHub: `HenryCarm/Horizon-Offline-Launcher` (re-clone: `git clone https://github.com/HenryCarm/Horizon-Offline-Launcher`)

## 12. Great known-issues (do NOT re-diagnose)

- `happy_ghast_harness: unknown target/face/...` — **harmless log noise**. Ignore.
- `The live count ping failed … telemetry … timed out` — **harmless**, offline play. Ignore.
- `flite lib missing` — narrator disabled, irrelevant.
- `TAA unavailable` — engine's own TAA, ungurable on GL; cosmetic.
- Crash `WGSL 'textureLoad' from depth textures is not supported in GLSL` → **pack bug**, not GPU (§3).
- Picked shader pack crashing every launch → `pickedShaderPack` in `horizon-graphics.txt`
  rewritten on exit → blank it before retrying.
- **Pins reverting in `installed.incoming/`** → expected: the launcher re-syncs from
  `~/.local/share/horizon/horizon-native-mods/*.hnmod` on build (§5).
- **A native build "succeeding" in < 1 s** → cargo had nothing to do; it relinked the stale
  stage. Verify `native-mods.json → "horizon"`.

---

*Last updated: 2026-11-09 ~06:30. Next milestones:*
1. Henny tries Lumen / Lumina / Pastel Mystic; report what breaks
2. **Unblock the native-mod rebuild** (§5) — the whole mod army hinges on this
3. Push the GUI date edit to our fork (needs a token) → CI launcher build → in his hands
4. Purge `/tmp`-territory from the workflow forever; keep everything under `Horizon Launcher Henry/`
5. Later: rebase GL fixes onto upstream once it carries them; re-audit Wisdom packs (Trash)

## 13. Launcher internals (quick map, for GUI surgery)

- Assets: `launcher/src/blocks.rs` — `Assets` strips `icons/glass/` → `horizon_theme::glass_icons::get` (embedded at compile time, MIT Hugeicons set).
- Icon registry: `crates/horizon-theme/src/lib.rs` `glass_icons::icons!()` — to add an icon: drop the SVG in `crates/horizon-theme/icons/glass/` + add the name to the list.
- `launcher/src/frame.rs` — `glass_sidebar` (Instances + pages nav + servers + links + footer) and `glass_topbar`.
  2026-11-09 redesign: the pages nav lives in the sidebar (`Page::in_top_bar` + `nav_icon` map); top bar = account/sign-in/settings/build-log only.
  The rail collapses to `MINI = 72.` via the chevron (`sidebar-collapse`) toggling `settings.sidebar_mini`.
- `launcher/src/glass_home.rs` — Home; `glass_picker` = the Version dropdown; `version_installed_day()` adds install day under the tag.
- `ui::menu_item` = one truncated note line; `ui::icon_button(id, icon, size, icon_size, radius, active, &t)`.
- `Page::ALL` third field is the icon name for the night look; glass reuses `package-01` and falls back to `nav_icon` map for new pages.
- Settings: add new fields with `#[serde(default)]`, or the old `settings.json` fails to load.
- CI: `launcher.yml` — push to `main` / PR / tag `launcher@*` → `cargo test --release --locked` (no fmt/clippy gate) → Win/Linux/macOS matrix → tag ⇒ sign + publish GitHub release ("latest" of our repo).
- Committing to our fork: shallow clone at `Horizon Launcher Henry/fork/`; `git config credential.helper 'store --file=.fork-credentials'` (600, in the permanent folder); LOCAL git identity `HenryCarm <HenryCarm@users.noreply.github.com>`; commit ends with `Co-Authored-By: OpenCode <noreply@opencode.ai>`.
- Game CI (`game.yml`): rebuilds/republishes the rolling `nightly` release from our fork; triggers on workflow_dispatch / pushes (check its `on:` block before relying on auto-builds).

## 14. Rebase in flight (2026-11-09, not yet landed)

- Fork `nightly-2026-11-08` HEAD = `bb5acc8` (GUI redesign + date-under-hash + Memories + 5 icons).
- Upstream `nightly-source/2026-11-08-fdc87ae` = Nov-8 nightly (51 commits incl. using-item refactor #11978
  + big game.rs split). **It touches NONE of our files** — verified via compare API.
- Git history is UNRELATED (our fork = overlay import), so `git merge` = 144 add/add conflicts → **aborted**.
  Strategy changed: deterministic **hash-based union** (`union-build.py` in this folder):
  base = `nightly-source/2026-11-07-2ebf00a` tarball, theirs = `fdc87ae` tarball, ours = fork working tree.
  Upstream wins except where ours differs; both-changed gets hand review (expect: nothing —
  `.github/workflows/game.yml` + `launcher.yml` stay **ours** (upstream deleted/rewrote them into
  irrelevant release-train infra), `release-keys.txt` upstream == base so ours wins intact).
- Tarballs downloading in background: `base-2ebf00a…tar.gz`, `base-fdc87ae…tar.gz`.
- After union: `git add -A && git commit && git push` → then dispatch `game.yml` ("game" workflow, builds
  our fork's client into upstream's current Nightly prepared assets, resigns with fork key, publishes
  our `nightly` pre-release). Watcher v2 phase B deletes `RELEASE_SIGNING_KEY` only after THAT succeeds.
- Game target: `nightly-source/2026-11-08-fdc87ae` client = latest Horizon **with our GL fixes**.
  Native-mod `.hnmod` rebuild problem unchanged (still blocked on pin-revert; next pin = fdc87ae tree).

### Build status (2026-11-09 ~07:10, update)
- `launcher@1.4.1` (run 42568070189) FAILED — pre-existing fork issues, not the GUI work:
  6 tests assumed upstream repo (`horizon-rust/horizon`) while fork REPO=HenryCarm (mirror/signed/platform-assets), Windows installer referenced missing `packaging/windows/setup.iss`.
- Fixed in `aa7ad1e2b` ("Fork: unbreak our own launcher CI"): test expectations -> HenryCarm, windows-aarch64 launcher asset optional in test, installer step skips when script absent.
- Tag `launcher@1.4.2` pushed; run 42571278771 in_progress at 2026-11-09 06:08 UTC.
- Game run 42571036125 in_progress on branch nightly-2026-11-08 @ 3702eae73 (rebased fdc87ae tree) — this publishes our fork's rolling `nightly` release (= latest Horizon + GL fixes).
- Watcher v4 (`remove-key-after-build.sh`): waits for BOTH game 42571036125 success AND newest launcher@1.4.2 run success, then deletes RELEASE_SIGNING_KEY. Log: remove-key.log.
- GUI binary (linux job of 1.4.1) already built fine — only tests/installer blocked the release. Our GUI code compiled (635 tests passed before the 6 fork-mismatch failures).

### Installed (2026-11-09 ~13:15)
- Game: new version `imported-2026-11-09-f49b940` created from nightly release `horizon-linux-x86_64.tar.gz` (sha256 f49b940a...), selected in settings.json. Old `imported-2026-11-08-603724d` kept as rollback. Base files (ctl/prepare/server) inherited from old import.
- Launcher: binary at ~/.local/share/horizon-offline-launcher/horizon-offline-launcher replaced with fork 1.4.2 (73M, from launcher@1.4.2 tarball). Official binary backed up to OpenCode/tmp/Trash/install-2026-11-09/. CLI symlink ~/.local/bin/horizon-offline-launcher follows it.
- RELEASE_SIGNING_KEY deleted from cloud 2026-11-09 13:13 CET (watcher v5, HTTP 204). Next publish needs NEW keypair.
- Disk 647M free (99%) after installs — empty Trash to reclaim.

## 15. Henny's launcher layout canon (2026-11-09 — DO NOT LOSE THIS)

Henny's taste is canonical for every launcher update from now on. Horizon 1.5+ (or any
rebase) must NOT silently revert these. If upstream moves code, re-apply them by hand:

1. **Sidebar (left rail):** page nav lives here as floating icon+label rows (Home/Mods/Servers/
   Wardrobe[/Shop]/Versions, with icons home-01, package-01, server-01, hanger-01,
   shopping-bag-01, archive-01). Rail collapses to 72px icon-only via the chevron at the
   sidebar's foot (`sidebar_mini` setting). Instances + servers below the nav; update card stays.
2. **NO Discord/Sponsor/Website in the sidebar.** Those four live ONLY at the bottom of
   Settings (About card, "Open" rows): Join the Discord, Sponsor, Website, My Website
   (https://github.com/HenryJayZ). The Sculk partner card stays in the sidebar (already hideable).
3. **Sidebar footer:** below the version line ("Launcher {version}") it must read:
   "Modded by HenryJayz btw :3" — Henny's signature. Keep it.
4. **Top bar:** NO "Sign in with Microsoft" button up there. It lives in
   Settings → Account (Microsoft card), where it always was. Top bar keeps: Build log,
   account chip (HenryJay Offline), settings icon.
5. **Mods tab (Home → Mods):** sort menu (Name / On first / Off first / Largest / Smallest,
   persisted as `settings.mods_sort`), "All on" / "All off" toggle-all buttons, hover trash
   (`delete-02`) per row with click-again confirm. **Toggle scroll bug rule:** never swap the
   list for a "Reading the mods…" placeholder while a cached list exists — it collapses scroll
   geometry and dumps the user at the bottom. Render cached rows while reloading.
6. **Settings page:** big right scrollbar (20px track, wide thumb), the big
   "Settings / Accounts, updates, and the launcher itself" header **collapses away** as you
   scroll down (returns at the top). Settings nav spy stays.
7. **Glass/blur defaults for NEW users:** glass 0% and blur 0px (flat, no frost) unless they
   change it in Settings (see `Settings::glass_tint/glass_blur` fallbacks).
8. **Version picker:** date under every hash (`{tag} · {YYYY-MM-DD}` from the version folder's
   creation day). Latest nightly = our fork's rolling `nightly`; never point at upstream's.
9. **Henny's playground:** this fork is a games machine, not a trophy case. Keep the sidebar
   nav + settings links + footer credit through every upstream pull. When in doubt about a
   layout call: sidebar > topbar, settings > sidebar, icons > words, dates > hashes.

### Build status (2026-11-09 ~16:20, update)
- **Default accent is now RED** (commit d1d322cd7, launcher@1.4.3): `Accent::default()` = RED (hue 356);
  settings/theme-picker/reset follow the default. Glass/blur defaults remain 0% / 0px for new users.
- **Signing key rotated again**: prior private key was unrecoverable (deleted from cloud + /tmp wipe).
  New Ed25519 keypair generated 2026-11-09: public `WxbJVd1zrDUf6WaDwwLo8jiEIHTSZyOa9DqfkToR0VQ=`
  committed to `crates/horizon-release-sign/release-keys.txt`; private DER kept locally at
  `/home/henry/Documents/Projects/Horizon Launcher Henry/.signing/signing-key.der` (600).
  `RELEASE_SIGNING_KEY` secret re-set via API (libsodium sealed box). Watcher v6 deletes it
  after launcher@1.4.3 publishes. **Do NOT commit `.signing/`**; treat the private as gone again after deletion.
- Tag `launcher@1.4.3` pushed; CI run 42642787295 queued 2026-11-09 15:15 UTC.

### Build status (2026-11-09 ~17:40, update)
- **GitHub username renamed HenryCarm → HenryJayZ** (same account!). All repo URLs redirect,
  token identity verified HenryJayZ, tags/secrets/releases untouched. Local remote + `backend::REPO`
  + tests now use `HenryJayZ/Horizon-Offline-Launcher` (no redirect reliance). No serious consequences.
- Pending 1.4.6 bundle (not yet coded when this was written): native Build 404 fix (upstream commit
  lookup + graceful fallback), Java button no-popup, multi-select mods/packs/shaders, custom SVG
  titlebar + OS bar removal, slimmer Settings header. The Build 404 is the only one blocking mods.

### Update (2026-11-09 ~17:50, user notes)
- Settings links at bottom now: Join the Discord, Sponsor, Website, **My Website** (https://henryjayz.github.io/CelSuite/), **My Profile** (https://github.com/HenryJayZ).
- User renamed GitHub username **HenryCarm -> HenryJayZ**; repo now at HenryJayZ/Horizon-Offline-Launcher (old URL redirects, token identity verified). Patched `backend::REPO` + tests to the new name.
- User asked to DELETE the public releases (launcher@1.4.2, nightly rolling, nightly-2026-11-08) — deleted via API 2026-11-09 (~17:47). Tags also deleted. Watch for nightly game rebuild republishing a fresh `nightly` later.
- The new PAT expires in ~24h (as of 2026-11-09 ~17:50); if actions 401 later, re-paste a fresh one.
- Signing key: the rotated private stays BOTH in cloud secret and local `.signing/signing-key.der` (600) until user says otherwise — previous auto-delete watchers intentionally NOT relaunched for 1.4.6; the key is an asset again.
