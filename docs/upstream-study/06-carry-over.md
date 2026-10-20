# Carry-over from the Vulkanstorm release branch

Project Aleph starts from the Firestorm reference, but five pieces of work done
on the C++ Vulkanstorm viewer are to be brought across: branding and artwork,
the login page, markdown chat formatting, order-independent transparency, and
the Lua script editor. This annex records what each piece actually is, what of
it transfers, and how it lands in the Rust crates proposed in the main study.
It also settles the two audio questions: SoLoud is not needed, and the VLC
speaker-fill plugin is not worth remaking as a plugin.

| | |
|---|---|
| Source tree | `anne-skydancer/vulkanstorm`, `vkstorm-release` at `08c07b4cba` (2026-10-16) |
| Firestorm merge-base | `fbc9ee9624` |
| Local checkout used | `C:\Dev\vkstorm-release-ref` (detached worktree) |
| Detail reports | [carry-over/](carry-over/) directory, every claim cited `path:line` |

The other Vulkan port effort in that repository (`doc/vulkan/`, the
DiligentCore design) is a separate project and is deliberately not consulted
here. One detail report cites it incidentally as the place a policy was
previously written down; those citations carry no decision weight for Aleph.

## 1. Branding and artwork

Report: [carry-over/branding-login.md](carry-over/branding-login.md).

**What it is.** A single piece of owner artwork, the blue "VK" icon (rounded
dark-navy tile, chrome V and K, cyan lightning bolt and ring), supplied as a
1254 px master and rendered to twelve PNG sizes, two ICOs, one ICNS, one BMP,
a 444 px login tile, a 1024 px startup logo and a 300x180 DMG background. Plus
a colour identity: Kokua's purple `#BF77CE` as the single accent with four
alpha variants, login progress blue `#009ED8`, flash `#7F7FFF`, teal avatar
cloud `(0, 0.5, 0.5)`, dark greys `#202020` and `#2B2B2B`. The full palettes
are 495 resolved colours for the Modern skin and 480 for the default skin.

**Licensing.** The VK artwork, the welcome page, the colour tables as data and
the installer script are the project's own work. No Linden or Firestorm
artwork is needed for the icon set. The Vulkanstorm login and progress
screens did still draw on Firestorm widget art (buttons, progress track,
rounded squares, the macOS volume icon), which is CC-BY-SA 3.0 with a
trademark carve-out. Aleph avoids that obligation by drawing its own widgets.
Kokua is LGPL-2.1 with CC-BY-SA skins; the palette values and login structure
adapted from it need a one-line credit.

**Done in this pass.** The owner's artwork, the welcome page and a provenance
notice are now in [assets/branding/](../../assets/branding/NOTICE.md). The two
colour tables are in `carry-over/colors-modern.tsv` and
`carry-over/colors-default.tsv`. No CC-BY-SA material was copied.

**Licence.** The artwork is LGPL-2.1, the same as the Vulkanstorm viewer;
the licence text ships beside it in `assets/branding/LICENSE`.

**Still to do.** Define the Rust UI theme from the roughly forty named colours in report section 2.2,
keeping the names so settings and UI code can refer to them. Resolve the
identity leaks Vulkanstorm never fixed: macOS bundle identifier, support and
download URLs still pointing at firestormviewer.org, the `firestorm` Linux
wrapper name.

**Identity strings to reproduce** (report section 5): app name `Vulkanstorm`,
short `VS`; channel base `Vulkanstorm` with `-Release` and similar suffixes;
version scheme `1.0.0.<git count>`; executable `Vulkanstorm[-Release].exe`;
Inno Setup AppId GUID and registry hive; client tag `Vulkanstorm`; user agent
`SecondLife/<ver> (<channel>; <skin> skin)`. These are what the login server,
grid tools and other viewers see.

**Settings defaults that are part of the identity** (report section 4):
twenty-five values differ from Firestorm, among them RLVa on, cache 4096 MB,
chat persistence 45 s, custom camera offsets, typing and chat animations off,
GL core profile, HTTPS-only grid list download, Modern skin default, nearby
chat logging on. They carry over as a key-value list into the Rust settings
struct.

## 2. Login page

Report: [carry-over/branding-login.md](carry-over/branding-login.md), section 3.

**What it is.** The live file is `panel_fs_nui_login.xml`, a Kokua-style
layout: a 300 px dark sidebar (`#0E171F`) with an 18 px gutter on a `#090F17`
page, title "Vulkanstorm" in `#26D9FF`, containing in order the grid selector,
username field (the username removal control), password field with a
visibility toggle, viewer-mode selector, saved-locations dropdown, and the
login button. The rest of the window is an embedded browser showing either the
grid's splash page or, when the grid would show the stock Firestorm splash,
the bundled `welcome.html` (33 lines, inline CSS, no images). Progress panels
for login, teleport and logout exist in full and mini variants with the blue
fill bar. Fonts are DejaVu Sans at 7.6, 8.6, 10.6 and 16 pt.

**What transfers.** The layout spec in report section 3.2 (regions, sizes,
fonts, behaviours), the welcome page verbatim, the splash selection rule, the
eight translation overlays as reusable label text, and the progress-panel
behaviour. The XUI layout-stack mechanics themselves do not transfer; the Rust
UI reproduces the spec in its own toolkit. The welcome page needs an embedded
web view (CEF, which the main study already keeps out of process) or a native
rendering of its text and colours.

**Crate placement.** `sl-ui` for the panels; `viewer` for the splash-selection
policy and grid list; `sl-core` for the identity strings.

## 3. Markdown chat formatting

Report: [carry-over/markdown-chat.md](carry-over/markdown-chat.md).

**What it is.** A deliberately tiny dialect, 446 lines of C++ with 47 unit
tests that are the de facto specification. It recognises exactly: `**strong**`
in both modes; `_emphasis_` in normal chat with CommonMark-like flanking where
every non-ASCII byte counts as a word byte; `_` as an italic on/off toggle in
emotes; and `__` as an escaped literal underscore. Nothing else: no links, no
code, no strikethrough, no single `*`, no italic inside bold. Unclosed openers
format to the end of the message. Caller-supplied literal byte ranges (URL
matches) and a built-in emoticon detector (an ASCII eye set plus 41 Unicode
eyes, with a word-embedding guard) protect underscores in faces like `^_^` and
`o_o`. Labelled links `[url label]` are Firestorm URL-registry code, separate
from the parser, with confusable folding and a real-URL tooltip for HTTP
schemes. Vulkanstorm's only release change there was enabling labelled links
by default.

**Output model.** A flat span list with nine kinds and no colour, font or link
data. The text widget and the console overlay turn kinds into bold and italic
flags on top of the message base style, then linkify per span. One quirk: bold
nested in italic is classified as emphasis and renders italic only.

**Where it applies.** Local, IM and group chat history, nearby-chat toasts,
the console overlay and IM echo. Not input editors, logs, notifications, names
or timestamps. Emote italics are unconditional. There is no master on/off
setting.

**Decision.** Port the parser one to one in Rust, byte-oriented, about 250 to
350 lines plus the 47 tests verbatim. Do not use pulldown-cmark: its
delimiter-run algorithm, inline code, autolinks and Unicode flanking would
break the SL conventions the tests encode. Fix the bold-in-italic quirk
deliberately in the resolver that turns spans into styled runs, and keep the
span kinds verbatim so the ported tests still pass. Carry label and target as
separate fields on link runs and show the real target on hover for every
scheme, which closes a gap Vulkanstorm left for `secondlife://` and `hop://`.

**Crate placement.** Parser and resolver in `sl-comms`; styled-run rendering
and linkification in `sl-ui`.

## 4. Order-independent transparency

Report: [carry-over/oit.md](carry-over/oit.md).

**What it is.** A per-pixel linked list (PPLL) that replaces only the
main-view post-water forward alpha pass. Pre-water alpha, HUDs, impostors,
mirrors and hero probes, cube snapshots, non-main render targets, macOS and GL
below 4.4 all keep the upstream two-pass sorted blend. Sequence: glTF-scene
alpha drawn to depth but not captured; scene depth copied to a detached
snapshot; counter zeroed and the R32UI head image cleared to all ones; rigged
then world alpha captured with depth test on and write off, each fragment
appending a 16-byte node (two half-float pairs for colour, float depth bits,
next pointer) via an atomic counter and an atomic head exchange, then
discarding; a barrier; a full-screen resolve that walks up to
`min(32, RenderAlphaOITMaxPixelLayers)` nodes, insertion-sorts them by depth,
folds any excess into a weighted-blended tail placed behind, and composites
premultiplied-over with the alpha channel masked so the glow accumulator
survives; a rigged depth-only replay; a residual pass for particles and
custom-blend batches; then the stock depth-of-field depth pass.

**Budget.** Nodes per pixel 8 by default, 512 MiB cap, 24 layers, which is
253 MiB at 1080p and about 4 nodes per pixel at 4K. Below one node per pixel
the feature disables itself. Pool overflow falls through to ordinary blending.

**Known limitations to carry knowingly.** Residual alpha always composites
over captured alpha. Glow suppression under captured alpha is lost. Overflow
ordering is draw-order approximate. Colour is FP16. GLTF alpha occludes
captured fragments. Depth peeling, avatar stamps and comparators were
excluded on purpose.

**Vulkan mapping.** Every construct has a direct Vulkan 1.3 equivalent with no
optional extension beyond `fragmentStoresAndAtomics`: an `R32_UINT` storage
image cleared with `vkCmdClearColorImage`; the node pool as a storage buffer
bounded by `maxStorageBufferRange`; the GL atomic counter, which Vulkan GLSL
lacks, becomes `atomicAdd` on a small storage buffer reset with
`vkCmdFillBuffer`; `vkCmdPipelineBarrier2` between clear, capture and resolve;
capture, resolve, depth replay and residual as four dynamic-rendering scopes
on one in-order queue with a single node pool shared across frames in flight;
three distinct descriptor bindings instead of the three "binding 0" resources
the GL code gets away with. The detached depth copy can be dropped by using
`early_fragment_tests` with a read-only depth attachment, which guarantees
occluded fragments never allocate a node. Keep the sampled-depth variant as a
validation mode until node counts and images match the GL build on the
existing perf-script cases (equality, occlusion, overflow, reset).

**Decision.** Reproduce the shipped PPLL policy as the parity target, since
the sorted path must exist anyway for the excluded views and the fallback.
Fix one wart: do not overwrite the user's sort-method setting on budget
failure; expose an "available" flag instead. Treat k-buffer with fragment
shader interlock and moment-based OIT as post-parity experiments with their
own A/B qualification. Weighted-blended OIT, if used at all, only for the
particle residual. No depth peeling.

**Crate placement.** `vk-render`, as a pass pair in the frame graph between
the water pass and the atmospheric haze pass documented in the main study.

## 5. Lua script editor and IDE bridge

Report: [carry-over/slua-editor.md](carry-over/slua-editor.md).

**What it is.** Larger than an editor. It is the selected port of Linden
Lab's Viewer 26.4 SLua work:

- Four compile targets as literal strings: `lsl2`, `mono`, `lsl-luau`, `luau`.
- Simulator advertisement via the `LuaScriptsEnabled` feature and the
  `ScriptDefinitions` and `CreateTaskInventoryItem` capabilities; the
  `ScriptRunningReply` carries Mono, Luau and language fields.
- Script language as an inventory subtype in the low flags byte (LSL 0, Lua
  1) plus a runtime string serialised as `script.runtime`, with content
  sniffing as fallback.
- Lua lexing additions for single-quote and backtick strings, `--` comments,
  long brackets, dotted namespaces, driven by the `lsl_definitions` 0.6.12
  package (LLSD maps of controls, types, constants, events, functions).
- Preprocessor bypass for Lua, `.luabackup` recovery, external edit to a temp
  file or a `vscode://` URI.
- A local IDE bridge: websocketpp, no TLS, `127.0.0.1:9020`, any `Origin`
  header rejected, JSON-RPC 2.0 with batches rejected and unauthenticated
  requests answered with `-32002`. Handshake by a challenge file (user-only,
  create-new, random UUID) answered within 30 s. Methods for syntax
  definitions, script subscription, object and linkset exploration, content
  read, save, create, delete and modify, running state and reset, and viewer
  commands (teleport, camera focus, save back to contents, reset or recompile
  all). Notifications for object publish, update and unpublish, compile
  results, runtime debug and errors, syntax changes, disconnect. Diagnostics
  are `{row, column, level, message}`. The counterpart is the published
  Second Life VS Code extension.

**What is UI-independent.** The transport, the RPC layer, the bridge
semantics, the published-object model, inventory metadata, the compile and
diagnostic flow and the syntax cache. What is UI-bound: floaters, text-editor
segments, the build tab, queues and notifications.

**Decision.** New `sl-scripting` crate with modules for compile, syntax,
published objects, bridge and external edit; additions to `sl-inventory` for
the subtype and runtime fields and to `sl-comms` for the caps and messages;
the editor widget in `sl-ui`. Use tokio-tungstenite and serde_json with a
hand-rolled dispatcher of about 300 lines, because batch rejection, the
pre-auth gate and the main-thread hop do not fit the generic JSON-RPC crates.
Consume the same `lsl_definitions` package at runtime, since regions push
definitions by syntax id anyway. A hand-written lexer of about 400 lines gives
exact parity; tree-sitter can be layered later. Estimated 4.5 to 6k lines for
the service crate with tests, roughly three to five engineer-weeks excluding
the editor widget.

**Security rules to keep.** Loopback bind only, reject non-empty `Origin`,
challenge file deleted after use, no method dispatch before auth, frame-size
cap (the C++ has none), server-side permission and RLVa checks on every write,
bridge default off (the ported settings defaulted it on, the test plan says
off), and `vscode://` launched with separate arguments, never a shell string.

## 6. Audio: SoLoud and the speaker-fill plugin

Report: [carry-over/audio.md](carry-over/audio.md).

**SoLoud is not needed, and that is the right call.** The main study already
decided on an own Rust mixer with HRTF. SoLoud on the release branch handled
effects, UI sounds and wind only; music stayed in VLC. What transfers is its
behaviour, not its code, as a specification for the Rust mixer:

- Device identity by endpoint ID, a 5 s device-list refresh, a 1 s device
  mutex shared with voice with 1 s retry, restore-previous-on-failure with no
  engine fallback, a restart that invalidates all voice handles.
- Native rate and 1, 2, 4, 6, 8 channel negotiation; the 7.1 remap (slots 4
  and 5 back, 6 and 7 side).
- Inverse-distance attenuation, min 1 m, max 512 m, rolloff and Doppler
  forwarded with velocities.
- Create-paused voices, 64-sample baked head and tail ramps, 20 ms declick
  stop and synced-start fade, 64 active voices.
- The wind generator with atomic target crossing in 512-frame chunks at 0.1 s
  updates.
- The headless adapter test shape (null device, 2, 6 and 8 channels,
  1024-frame mix with guard samples, restart five times).

**The speaker-fill plugin is not worth remaking as a plugin.** It exists only
because the VLC plugin process owned the WASAPI endpoint, so the viewer could
neither mix the music nor know when a fade had been heard. That forced three
VLC modules, a MinGW toolchain to build patched VLC 3.0.21 output code, an ABI
pin, and a generation-and-serial handshake with a playback-clock fence. Once
the Rust mixer owns an N-channel output stream, speaker fill is a fixed matrix
on the music bus: left and right to all mains at 0.7071, centre at 0.3536
times their sum, LFE silent, native multichannel passed through, gated by the
negotiated channel mask, a headphone check, and the user's layout choice with
the rear-else-side rule. About twenty lines. Keep the test cases from
`vlc_speakerfill_test.cpp` as the Rust tests.

**"Spatial music" means speaker fill, not positioning.** The setting
`FSMusicSpatialSound` selects a layout (stereo, 2.1, 4.1, 5.1, 7.1). The
positional branch for object audio is dormant on the release branch because
no shipped plugin advertises the capability. Aleph upgrades that intent: an
object's `audio/*` media becomes a real 3D source on the HRTF bus.

**Parcel music in-process.** Decode with symphonia rather than VLC: MP3,
AAC-LC, Vorbis, FLAC, chained Ogg are covered, and the samples land in the
mixer with sample-accurate fades and no IPC. The gaps to decide on: HE-AAC
(common on SHOUTcast "aacp" stations; symphonia is LC only, so fdk-aac
bindings or accepting LC only), Opus (an `opus` crate), the ICY handshake
with interleaved metadata and the non-HTTP `ICY 200 OK` status line,
playlist resolution, reconnect with backoff, and the "this is actually video"
guard. Keep the music fade policy as a mixer-internal envelope: 3.0 s in,
2.0 s out, 0.01 s minimum, 60 s cap, held at zero during login; completion
means the final target sample has passed the device clock; failure or cancel
is a hard stop, never an implicit replay. Keep the ICY "Artist - Title" split,
Ogg comments, and the Icecast status-json poller (5 s first poll, 20 s
interval, three strikes, mount-path walk).

**Video still needs a decoder.** Prim video keeps libvlc, GStreamer or
ffmpeg, but its PCM is pulled into the Rust mixer so video audio, distance
attenuation and speaker fill all go through one device. The endpoint-fence
problem that killed Vulkanstorm's PCM path does not exist when the mixer owns
the device.

**Drop entirely.** SoLoud globals and handle plumbing, the media_audio and
media_music IPC protocol and its tokens, the VLC output-module patching and
build, the retired PCM bridge and its WASAPI endpoint code.

## 7. Summary table

| Item | Transfers as | Lands in | Effort note |
|---|---|---|---|
| VK artwork, welcome page, DMG background | Data, copied now | `assets/branding/` | LGPL-2.1, notice included |
| Colour palette | Data (two TSVs) plus a theme of ~40 named colours | `sl-ui` theme | Keep names |
| Identity strings and settings defaults | Key-value lists | `sl-core`, `viewer` | Fix the Firestorm leaks |
| Login page | Layout spec, labels in 8 languages | `sl-ui`, `viewer` | Needs a web view or native welcome |
| Markdown chat | 1:1 parser port, 47 tests | `sl-comms`, `sl-ui` | ~300 lines plus tests |
| PPLL OIT | Policy and shader logic, Vulkan-native resources | `vk-render` | Parity target, then experiments |
| SLua editor and bridge | Protocol and model spec, new crate | `sl-scripting`, `sl-inventory`, `sl-comms`, `sl-ui` | 4.5 to 6k lines |
| SoLoud | Behaviour spec only | `sl-audio` | No code |
| Speaker fill | 20-line output matrix plus tests | `sl-audio` | No plugin |
| Parcel music | In-process decode | `sl-audio` | Decide HE-AAC |
| Video audio | PCM routed into the mixer | `sl-audio` | Decoder choice open |
