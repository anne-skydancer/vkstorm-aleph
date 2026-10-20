# Vulkanstorm branding, artwork and login page: carry-over survey for vkstorm-aleph

Source tree surveyed read-only: `C:\Dev\vkstorm-release-ref`, detached at `origin/vkstorm-release` = `08c07b4cba`
("Merge pull request #159 ... codex/release-chat-ui-fixes"). All paths below are relative to that repo root.

Baselines used for "what Vulkanstorm changed":

- Merge-base with Firestorm `upstream/master` is `fbc9ee9624` (`git merge-base HEAD upstream/master`). `upstream/master` is 1093
  commits *past* that base, so `git diff upstream/master` mixes in newer upstream Firestorm work (e.g. the Aditi grid
  host change in `llviewernetwork.cpp`, Omnifilter, the `AgniGridLabelShort` strings, the `fstrusted` icon). Where it
  matters (settings, default colours, login XML, grid code) this report diffs against `fbc9ee9624` instead and says so.
- The AnsaStorm Modern skin (`indra/newview/skins/ansastorm_modern/`) does not exist at `fbc9ee9624`; it was ported from
  Firestorm (commit `b7a82cc6f0`, author Ansariel, "Add AnsaStorm Modern skin") and then modified by Vulkanstorm. Modern
  colour diffs are therefore against `upstream/master`'s copy.
- The authoritative in-repo narrative is `doc/branding_defaults.md` and `doc/archive_defaults_audit.md`; the reset base is
  described in `doc/clean_release_base.md` (Firestorm 7.2.4 release, `10bd3c9f93`).

Companion data files written next to this report (pure data for the Rust theme):

- `vulkanstorm_modern_colors.tsv` -- all 495 named colours of the Modern skin, columns: name, raw value or `->reference`,
  resolved RGBA (0..1), hex.
- `vulkanstorm_default_colors.tsv` -- the 480 entries of the default skin table (what Modern inherits from).

---

## 1. Asset inventory

Dimensions were read from PNG/BMP/ICO/ICNS headers (no `identify` available). "sha" is the first 10 hex of SHA-1,
used to show which files are byte-identical copies.

### 1.1 Project-owner artwork (blue "VK" lightning-and-ring icon) and its format derivatives

Provenance: `indra/newview/icons/vulkanstorm/README.md:3-6` states the blue VK artwork was supplied by the project owner
in `vulkanstorm-blue-icons.zip` on 2026-10-03; `vulkanstorm-blue.png` is the unmodified master, `firestorm_icon.ico` the
supplied recommended ICO, `vulkanstorm-blue-legacy.ico` the supplied uncompressed fallback; all other sizes/formats are
derivatives of the master (README.md:8-10). `doc/branding_defaults.md:49-54` says the same. This artwork is NOT covered
by the Linden CC-BY-SA notice; it is the owner's own work and can be relicensed/reused freely in the new repository.

| Path | Format / size | Purpose | sha |
|---|---|---|---|
| `indra/newview/icons/vulkanstorm/vulkanstorm-blue.png` | PNG RGBA 1254x1254 | **Master** (unmodified) | 4e01e7f36e |
| `indra/newview/icons/vulkanstorm/firestorm_1024.png` | PNG RGBA 1024x1024 | derivative; source for ICNS ic10 | 08f85830c0 |
| `.../firestorm_512.png` | PNG RGBA 512x512 | Linux desktop icon (`viewer_manifest.py:2342` copies it to `firestorm_icon.png`) | 5ffc332212 |
| `.../firestorm_256.png` | PNG RGBA 256x256 | README/doc logo, `vulkanstorm_logo.png` (identical) | b9144645f4 |
| `.../firestorm_128.png`, `_96`, `_64`, `_48`, `_40`, `_32`, `_24`, `_20`, `_16.png` | PNG RGBA at those sizes | ICO frame sources / small icons | -- |
| `.../firestorm_256.bmp` | BMP 256x256 24bpp | installer bitmap source | b6f54d3b83 |
| `.../firestorm_icon.ico` | ICO, 10 frames: 16,20,24,32,40,48,64,96,128 (32bpp BMP) + 256 (PNG-compressed) | Windows exe icon (`res/viewerRes.rc:57-59`), Inno `SetupIconFile` (`viewer_manifest.py:1075`, `installer_template.iss:61`) | ed73632ab5 |
| `.../vulkanstorm-blue-legacy.ico` | ICO, 10 frames, all BMP (256 uncompressed) | fallback if the RC compiler rejects PNG frames (README.md:13-15) | c94bcdbffd |
| `.../firestorm_icon.icns` | ICNS: ic07(128) ic08(256) ic09(512) ic10(1024) ic11(32@2x) ic12(64@2x) ic13(256@2x) ic14(512@2x) + TOC | macOS bundle icon (`CMakeLists.txt:2835`) | 13cb1e04d0 |
| `indra/newview/firestorm_icon.icns`, `indra/newview/firestormos_icon.icns` | ICNS (identical to above) | legacy mac icon paths kept for compatibility | 13cb1e04d0 |
| `indra/newview/installers/windows/firestorm_icon.ico`, `firestorm_icon_os.ico` | ICO (identical to `firestorm_icon.ico`) | NSIS installer icon (SL and OpenSim variants) | ed73632ab5 |
| `indra/win_crash_logger/firestorm_icon.ico` | ICO (identical) | crash-logger exe icon | ed73632ab5 |
| `indra/newview/res/install_icon.BMP`, `res/uninstall_icon.BMP`, `installers/windows/install_icon.BMP`, `installers/windows/uninstall_icon.BMP` | BMP 256x256 24bpp | NSIS header/wizard bitmaps | 3dc590ac9d |
| `indra/newview/res/firestorm_icon.png`, `res-sdl/firestorm_icon.png` | PNG RGBA 512x512 | Linux/SDL window icon | 8917224f3e |
| `indra/newview/res-sdl/firestorm_icon128.png`, `skins/default/textures/firestorm_icon.png` | PNG RGBA 128x128 | SDL small icon; in-viewer `firestorm_icon` texture | 1cbf6b56b5 |
| `indra/newview/skins/default/textures/windows/vulkanstorm_logo.png` | PNG RGBA 256x256 (= firestorm_256.png) | registered as `login_vulkanstorm_logo` (`textures.xml:777`); not referenced by any XUI (grep) | b9144645f4 |
| `indra/newview/skins/default/textures/windows/login_fs_logo.png` | PNG RGBA 444x444 (VK icon on rounded dark square) | registered as `login_fs_logo` (`textures.xml:778`); was the logo of the previous NUI login; the Kokua-style layout uses a text title, so it is now unused by `en` XUI | 7b042ab670 |
| `indra/newview/skins/default/textures/windows/startup_logo.png` plus the same file in `skins/ansastorm_modern/textures/windows/`, `skins/metaharper/...` (4 copies), `skins/starlight/themes/*/...` (6), `skins/starlightcui/...` (3) | PNG RGBA 1024x1024 (VK icon) | `startup_logo` texture (`textures.xml:775`, preload); no C++/XUI consumer found by grep, kept because every skin ships one | d8657676e1 |
| `doc/vulkanstorm_256.png` | PNG RGBA 256x256 | README logo (`README.md:1`) | b9144645f4 |
| `indra/newview/installers/darwin/{firestorm,firestormos}-{release,beta,nightly,private}-dmg/background.png` (8 identical copies) | PNG RGB 300x180 | DMG window background: VK icon top-left, "Vulkanstorm" caption, "VULKANSTORM VIEWER" bottom-right, white background | edfe5e47be |

Visual check (Read of `login_fs_logo.png`, `firestorm_256.png`, `startup_logo.png`): rounded dark-navy square tile,
chrome/silver "V" and "K" letters, cyan lightning bolt through the middle, cyan ring behind. The DMG background was also
viewed and matches the description above.

### 1.2 Non-image branding data added by Vulkanstorm (own work, no third-party licence)

| Path | What |
|---|---|
| `indra/newview/skins/default/html/en/vulkanstorm/welcome.html` | bundled login welcome page (33 lines, inline CSS only, no images); see section 3.4 |
| `indra/newview/installers/windows/installer_template.iss` | Inno Setup installer script (section 5) |
| `indra/newview/skins/ansastorm_modern/xui/en/panel_progress.xml`, `panel_progress_mini.xml` | Modern-skin progress panels (copies of the default ones with the blue bar colour) |
| `indra/newview/skins/default/xui/en/panel_fs_nui_login.xml` (+ `az de fr it ja pl ru zh` overlays) | the Kokua-style sidebar login layout |
| `indra/newview/skins/ansastorm_modern/colors.xml`, `skins/default/colors.xml` | colour tables with the Kokua palette import (section 2) |

### 1.3 Firestorm / Linden artwork that is still used by the login and progress screens (CC-BY-SA 3.0)

These were NOT replaced and are needed to reproduce the look; they fall under `doc/LICENSE-logos.txt`
("Second Life(TM) Viewer Artwork. Copyright (C) 2008 Linden Research, Inc." CC-BY-SA 3.0, with the trademark carve-out and
the "include this Notice and clearly identify any changes" obligation, `LICENSE-logos.txt:1-21`). Firestorm's own
additions to the skin are distributed under the same terms in the Firestorm tree.

| Texture name (textures.xml) | File | Size | Used by |
|---|---|---|---|
| `PushButton_Login`, `_Over`, `_Pressed` | `skins/default/textures/widgets/PushButton_Login*.png` | 299x64, 9-slice 4/19/28/4 (`textures.xml:684-686`) | Log In button (`panel_fs_nui_login.xml:375-377`) |
| `TrashItem_Off`, `TrashItem_Press` | `textures/icons/TrashItem_*.png` (`textures.xml:863-864`) | -- | "Remove this username" button (`panel_fs_nui_login.xml:132-134`) |
| `Profile_Group_Visibility_Off/On(_Pressed)` | `textures/icons/profile_group_visibility_eye_*.png` (`textures.xml:653-656`) | -- | password show/hide eye buttons (`panel_fs_nui_login.xml:166-167,179-180`) |
| `ProgressTrack`, `ProgressBarSolid` | `textures/widgets/ProgressTrack.png` 152x15 (9-slice 4/13/148/2), `ProgressBarSolid.png` 52x15 greyscale (9-slice 4/11/48/3) (`textures.xml:671-672`) | -- | progress bar track and fill (`ansastorm_modern/xui/en/widgets/progress_bar.xml:2-3`) |
| `Rounded_Square` | `rounded_square.j2c` 9-slice 16/16/112/16 (`textures.xml:708`) | -- | progress panel's translucent box (`panel_progress.xml:52-60`) |
| `login_sl_logo` (225x94), `login_sl_logo_small` (165x73), `first_login_image.jpg`, `Login_Pod.png` | `textures/windows/` | -- | only by Linden's `panel_login.xml` / `panel_login_first.xml`, which Vulkanstorm does not load (see 3.1) |
| `installers/darwin/*/_VolumeIcon.icns` | ICNS 490807 bytes, unchanged | -- | DMG volume icon: still Firestorm art |
| `res/loginbackground.bmp` (280x400), `res/bitmap2.bmp` | unchanged | -- | Windows resource bitmaps |
| Modern-skin widget art: `ansastorm_modern/textures/**` (flat tabs, accordions, bottomtray, build icons ...) | PNG | -- | Firestorm (Ansariel) artwork ported with the skin; registry entries for the flat tabs/TextField_Highlight were added by Vulkanstorm at `textures.xml:1207-1216` |

### 1.4 Kokua-derived material

Not images: the colour table ("imports all 394 unique entries from Kokua commit `20493b5e73`", `doc/branding_defaults.md:33-36`;
`ansastorm_modern/colors.xml:3` comment) and the login layout ("Layout adapted from Kokua 20493b5e73",
`panel_fs_nui_login.xml:2`; `doc/branding_defaults.md:43-47`). Kokua is an LGPL-2.1 viewer whose skins derive from the
same Linden CC-BY-SA artwork. The Vulkanstorm tree carries no separate Kokua notice; see section 6.

### 1.5 Images removed relative to merge-base

None of the branding images were deleted by Vulkanstorm versus `fbc9ee9624`. The deletions that show up in
`git diff upstream/master --stat` (Camera_*.png, fstrusted.png, login_sl_logo_horizontal.png, the Firestorm-skin copies of
the flat-tab PNGs, ansa_modern_blue.jpg ...) are files that upstream Firestorm added after the base, not Vulkanstorm removals.

---

## 2. Colour palette

### 2.1 Structure

- Default skin table: `indra/newview/skins/default/colors.xml` (480 entries; 445 at merge-base, 35 added, 118 resolved
  values changed by the Kokua import). Modern skin table: `indra/newview/skins/ansastorm_modern/colors.xml` (495 entries,
  header comment line 3: "Default palette imported from Kokua 20493b5e73; Vulkanstorm extensions retained."). Modern
  overrides default; the viewer loads default then the skin's file.
- The identity colour is the single **Kokua purple** `Kokua = 0.749 0.467 0.808 1` (`ansastorm_modern/colors.xml:476`)
  with alpha variants `Kokua_11/_13/_23/_42` (lines 477-480). `AccentColor`, `EmphasisColor`, `SchemeLightest`,
  `FilterTextColor`, `WornOutfitTextColor`, `AudioStreamTitle`, `FocusColor` are references to it (lines 6-10, 210, 216, 346, 470).
  Nearly every "selected / highlighted / friend / link-ish" colour resolves to it (see `vulkanstorm_modern_colors.tsv`).
- The **registered blue** for the login progress bar is `LoginProgressBarFgColor = 0 0.6211 0.8477 0.96`
  (`ansastorm_modern/colors.xml:257`; commit `2ad07ca03b`); it is the blue the Modern skin used as its accent base
  (0 0.6211 0.8477) before the purple import. In the default skin `LoginProgressBarFgColor` still references
  `Unused?` (#d4ff00) and the default progress panels use a literal `1 1 1 0.96` white (`default/xui/en/panel_progress.xml:112`).
- The **flash colour** `ButtonFlashBgColor = 0.498 0.498 1 1` = `#7f7fff` (both tables; `ansastorm_modern/colors.xml:155`,
  commit `2b1381c6ac` "set existing flash color to #7f7fff").
- The **teal cloud** (avatar loading cloud) colours are settings, not palette entries: `FSCloudColorStart = 0 0.5 0.5 0.1`,
  `FSCloudColorEnd = 0 0.5 0.5 0.9` (`app_settings/settings.xml:27579,27595`) mirrored in `app_settings/cloud.xml:55-69`.
  The palette names `FSCloudColorStart/End` do not exist in colors.xml.
- The login page itself uses literal colours in the XML, not palette names (section 3.2).

### 2.2 The ~40 colours that define the look (Modern skin, resolved)

Values are RGBA 0..1 as in the XML; hex is sRGB 8-bit. Line numbers refer to `indra/newview/skins/ansastorm_modern/colors.xml`.

| Name | Role | RGBA | Hex | Line |
|---|---|---|---|---|
| `Kokua` | brand purple, root of all accents | 0.749 0.467 0.808 1 | #BF77CE | 476 |
| `Kokua_42` / `EmphasisColor_35` | selection/highlight fill | 0.749 0.467 0.808 0.42 | #BF77CE a.42 | 480, 8 |
| `Kokua_13` / `EmphasisColor_13` | subtle highlight | 0.749 0.467 0.808 0.13 | #BF77CE a.13 | 478, 13 |
| `Kokua_11` / `ChicletFlashColor` | chiclet flash | 0.749 0.467 0.808 0.11 | #BF77CE a.11 | 477, 100 |
| `AccentColor`, `EmphasisColor`, `FocusColor`, `FilterTextColor`, `SchemeLightest`, `NameTagFriend`, `ObjectChatColor`, `ChatToastAgentNameColor`, `InventoryItemLibraryColor`, `ContextSilhouetteColor` | all -> Kokua | 0.749 0.467 0.808 1 | #BF77CE | 6, 7, 216, 210, 10, 296, 313 |
| `SelectionColor` | text selection | 0.75 0.45 0.81 1 | #BF73CF | 9 |
| `MenuItemHighlightBgColor`, `ScrollSelectedBGColor`, `PieMenuSelectedColor`, `HighlightPanelBackground`, `ListViewSelectedBgColor` | -> EmphasisColor_35 | 0.749 0.467 0.808 0.42 | #BF77CE a.42 | 279, 340, 115, 459 |
| `ScrollHoveredColor` | list hover | 0.375 0.225 0.405 1 | #603967 | 339 |
| `ConversationTabFlashBgColor` | IM tab flash | 0.749 0.467 0.808 0.20 | #BF77CE a.20 | 497 |
| `ButtonFlashBgColor` | button flash | 0.498 0.498 1 1 | #7F7FFF | 155 |
| `MenuItemFlashBgColor` -> `BeaconColor` | menu flash / beacons | 0.749 0.298 0 1 | #BF4C00 | 280, 14 |
| `ToolbarButtonFlashBgColor` | toolbar flash | 0.258 0.363 0.472 1 | #425D78 | 458 |
| `MentionFlashBgColor` | @mention flash | 1 1 0 0.5 | #FFFF00 a.5 | 457 |
| `LoginProgressBarFgColor` | login/teleport progress fill (blue) | 0 0.6211 0.8477 0.96 | #009ED8 a.96 | 257 |
| `LoginProgressBoxCenterColor` | progress box fill | 0 0 0 0.78 | #000000 a.78 | 259 |
| `LoginProgressBoxTextColor` -> White | progress text | 1 1 1 1 | #FFFFFF | 261 |
| progress bar track `color_bg` (widget default) | track | 0.3254 0.4000 0.5058 1 | #536681 | `widgets/progress_bar.xml:5-8` |
| `HTMLLinkColor`, `ChatMentionFont` | links | 0.3 0.82 1 1 | #4CD1FF | 225, 454 |
| `DkGray` -> `PanelDefaultBackgroundColor`, `MenuBarBgColor` | base dark panel | 0.125 0.125 0.125 1 | #202020 | 27, 316, 274 |
| `DkGray2` -> `PanelFocusBackgroundColor`, `FloaterFocusBackgroundColor`, `MenuDefaultBgColor`, `InventoryBackgroundColor` | focused/menu bg | 0.169 0.169 0.169 1 | #2B2B2B | 30, 318, 213, 276, 241 |
| `FloaterDefaultBackgroundColor` -> DkGray_66 | unfocused floater | 0.125 0.125 0.125 0.66 | #202020 a.66 | 212 |
| `DkGrayLogin` | Kokua login grey (unused by the current login XML) | 0.129 0.133 0.137 1 | #212223 | 465 |
| `PanelDark` | darkest panel | 0.078 0.078 0.078 1 | #141414 | 493 |
| `LtGray` -> `LabelTextColor`, `ButtonLabelColor`, `SystemChatColor`, `IMChatColor`, `ChatHistoryTextColor`, `MenuItemEnabledColor` | default UI text | 0.75 0.75 0.75 1 | #BFBFBF | 21, 255, 157, 356 |
| `TextFgColor` | text in editable fields (fields are light) | 0.102 0.102 0.102 1 | #1A1A1A | 365 |
| `TextBgWriteableColor` -> LtGray, `TextBgFocusColor` -> White | field backgrounds | 0.75 grey / white | #BFBFBF / #FFFFFF | -- |
| `AgentChatColor` -> White, `UserChatColor` -> Yellow | chat | 1 1 1 1 / 1 1 0 1 | #FFFFFF / #FFFF00 | 131, 373 |
| `ChatNameColor`, `ChatNameObjectColor` | chat names | 0 0.6 0.82 1 | #0099D1 | 311 |
| `Green5` -> `AvatarListItemChatRange`, `NameTagChatDistanceColor` | in-chat-range | 0.2 0.6 0.4 1 | #339966 | 468 |
| `Orange1` -> `KokuaNameTagTypingColor`, `NameTagUsername` | typing / username tag | 1 0.5 0 1 | #FF8000 | 466, 481 |
| `MdBlue` -> `MenuBarProjectBgColor` | non-release menubar | 0.07 0.38 0.51 1 | #126182 | 74 |
| `NotifyCautionBoxColor`, `AlertCautionBoxColor` | caution | 1 0.82 0.46 1 | #FFD175 | 307 |
| `AlertTextColor` | alert text | 0.58 0.66 0.84 1 | #94A8D6 | 137 |
| `ToolTipBgColor` / `ToolTipTextColor` -> DkGray2 / `ToolTipBorderColor` | tooltips | 0.937 0.89 0.655 1 / 0.169 grey / 0.812 0.753 0.451 1 | #EFE3A7 / #2B2B2B / #CFC073 | 125, 371 |
| `ScriptDialog` / `ScriptDialogFg` | script dialogs (Modern only) | 0.223 0.216 0.207 1 / white | #393735 / #FFFFFF | 4-5 |
| `Red`, `Yellow`, `White`, `Black` | primaries (Kokua uses pure red, not FS's 0.729 0 0.121) | -- | #FF0000 #FFFF00 #FFFFFF #000000 | 38, 40, 15, 32 |

Modern vs default differ in only 21 resolved names (AccentColor, ChatNameColor, LoginProgressBarFgColor, ScriptDialog*,
SchemeLightest, NameTagUsername, InventoryItemHighlightBgColor, BottomTrayOpaqueColor, ChatHeader*, HintText/Title,
MapPickRadiusColor, ParcelNormalColor, Slidermark, UploadDialogBackground, UriQueryPartColor, AvatarListItemAgeAlert);
otherwise Modern's visible palette *is* the default table. Compared with upstream Firestorm's Modern table, 145 resolved
values changed and 38 names were added (`Kokua*`, `ListView*`, `DkGrayLogin`, `PanelDark`, `ChatEntryBgColour*`,
`AudioStream*`, `NetMap*Physical`, `Resizebar*`, `OutfitSnapshotMacMask*`, ...). The Firestorm blue accent
(0.3 0.82 1 / 0 0.6211 0.8477) survives only in `HTMLLinkColor`, `ChatMentionFont`, `LoginProgressBarFgColor`, `ChatNameColor`.

---

## 3. The login page

### 3.1 Which XML is live

`FSPanelLogin::FSPanelLogin` loads `panel_fs_nui_login.xml` unless `FSUseLegacyLoginPanel` is set, in which case it loads
`panel_fs_login.xml` (`indra/newview/fspanellogin.cpp:208-215`; default 0 at `settings.xml:26071-26080`). Linden's
`LLPanelLogin` (`llpanellogin.cpp:214,218` -> `panel_login_first.xml` / `panel_login.xml`) is not instantiated by
Firestorm/Vulkanstorm startup (`llstartup.cpp` uses `FSPanelLogin` only, lines 1357-1673); both Linden files are unchanged
from the merge-base and can be ignored (they are listed here because the task named them: `panel_login_first.xml` is the
old "first run" layout with `login_sl_logo` + `first_login_image`, `panel_login.xml` is Linden's 1024x768 top-bar layout
with a `login_html` browser at top=154).

The legacy `panel_fs_login.xml` is also unchanged. Translations of the live panel exist for `az de fr it ja pl ru zh`
(`indra/newview/skins/default/xui/*/panel_fs_nui_login.xml`), restructured to the new hierarchy (`doc/branding_defaults.md:45-46`).

### 3.2 Layout spec of `indra/newview/skins/default/xui/en/panel_fs_nui_login.xml` (Kokua-style sidebar)

Coordinate system: XUI "topleft" layout, y grows downward from the parent's top; design size 1024x768; everything
`follows="all"` so it stretches with the window. Fonts: `SansSerifSmall` 7.6pt, `SansSerif` (medium) 8.6pt,
`SansSerifLarge` 10.6pt, `SansSerifHuge` 16pt (`skins/default/xui/en/fonts.xml:170-185`); the default face is DejaVu Sans
(`strings.xml` `font_fonts.xml` = "Deja Vu").

```
panel "panel_login" 1024x768, opaque bg 0.035 0.065 0.09 (#090F17)                  [lines 3-12]
+- layout_stack "main_stack" horizontal, no borders, no animation                     [21-31]
   |- layout_panel "left_spacer"  auto_resize, min 0                                   [32]
   |- layout_panel "center_stage" fixed 1008x768, bg #090F17, min_width 300            [33-43]
   |  +- layout_stack "ui_stack" horizontal                                            [44-54]
   |     |- layout_panel "ui_container" fixed 300x768, bg 0.055 0.09 0.12 (#0E171F)    [55-65]  <- the SIDEBAR
   |     |  +- scroll_container "login_scroll" (fills, right=-1 bottom=-1)             [66-74]
   |     |     +- panel "login_content" 280x677, focus_root                            [75-84]
   |     |        |- text "vulkanstorm_title" "Vulkanstorm" SansSerifHuge, centred,
   |     |        |     x20 y39 w260 h80, colour 0.15 0.85 1 (#26D9FF)                [85]
   |     |        |- panel "login" x0 y126 w280 h160                                   [86-93]
   |     |        |   |- text "username_text" "Username" SansSerifSmall 0.72 0.83 0.9  x40 y0  w220 h18 [95]
   |     |        |   |- combo_box "username_combo" x40 y23 w220 h28, editable, SansSerifLarge,
   |     |        |   |     max 128 chars, ascii prevalidator, text pad 8/8            [96-116]
   |     |        |   |- check_box "remember_username_check" (FSRememberUsername) x40 y54 w185 h22 [117-128]
   |     |        |   |- button "remove_user_btn" trash icon x235 y54 25x22, tooltip "Remove this username" [129-141]
   |     |        |   |- text "password_text" "Password" x40 y75                       [94]
   |     |        |   |- line_editor "password_edit" x40 y98 w190 h28, is_password, max 16, select_on_focus [142-158]
   |     |        |   |- button "password_show_btn" eye-off icon x235 y98 25x28 (visible) [159-171]
   |     |        |   |- button "password_hide_btn" eye-on icon, same rect (hidden)    [172-184]
   |     |        |   +- check_box "remember_check" (RememberPassword, enabled only if FSRememberUsername) x40 y129 w220 h22 [185-197]
   |     |        +- layout_stack "login_stack" vertical x0 y297 w280 h380             [199-209]
   |     |            |- layout_panel "start_location_panel" h60                       [210-252]
   |     |            |   |- text "Start location" x40 y0
   |     |            |   +- combo_box "start_location_combo" (NextLoginLocation) x40 y18 w220 h28,
   |     |            |         items: "Last location"=last, "Home"=home, "<Type region name>"=""
   |     |            |- layout_panel "grid_panel" h78                                  [253-309]
   |     |            |   |- text "grid_login_text" "Grid:" x40 y0
   |     |            |   |- combo_box "server_combo" x40 y24 w220 h28, editable (grid URI), max 256, ascii
   |     |            |   +- text "grid_builder_text" "+ Click to add more grids" 0.28 0.85 1, hidden by default [295-308]
   |     |            |- layout_panel "mode_selection" h60                              [310-365]
   |     |            |   |- text "Mode:" x40 y0
   |     |            |   +- combo_box "mode_combo" x40 y24 w220 h28:
   |     |            |         "Vulkanstorm"=settings_firestorm.xml, "Phoenix"=settings_phoenix.xml,
   |     |            |         "[VIEWER_GENERATION]"=settings_v3.xml, "Hybrid"=settings_hybrid.xml, "Text"=settings_text.xml
   |     |            +- layout_panel "login_actions" h182                              [366-413]
   |     |                |- button "connect_btn" "Log In" x40 y3 w220 h30, PushButton_Login art, white label, SansSerif
   |     |                |- text "forgot_password_text" "Forgot password?" centred x40 y110 w220 h20, 0.28 0.85 1
   |     |                +- text "create_new_account_text" "Create an account" centred x40 y150 w220 h20, 0.28 0.85 1
   |     |- layout_panel "middle_gutter" fixed 18px, bg #090F17                          [418]
   |     +- layout_panel "web_container" auto_resize, 690x768 at design size, bg #090F17 [419-429]
   |        +- web_browser "login_html" fills, trusted_content, no border, bg #090F17, start_url="" [430-441]
   +- layout_panel "right_spacer" auto_resize, min 0                                     [445]
```

Colour summary for the login screen (literal, not palette): page/web background `#090F17`, sidebar `#0E171F`, title
`#26D9FF`, field labels `#B8D4E6` (0.72 0.83 0.9), link-style texts `#47D9FF` (0.28 0.85 1), Log In label white on the
Firestorm login-button art.

Behaviours implemented in C++ that the layout depends on:

- Centring: `FSPanelLogin::reshape` sets `center_stage`'s target width to `min(window_width, 1008)` and re-runs both
  stacks, so the sidebar + 18px gutter + welcome page are centred with elastic spacers either side
  (`fspanellogin.cpp:473-489`; commits `f25590afc3`, `dd0b844163`).
- Focused controls scroll into view inside the sidebar (`FSPanelLogin::onUpdateScrollToChild`, `fspanellogin.cpp:491-503`;
  commit `21f164196a` also restored tab navigation).
- `password_show_btn`/`password_hide_btn` toggle each other and the editor's password masking (existing Firestorm code);
  `remove_user_btn` deletes the selected saved username; `remember_check` is disabled unless `FSRememberUsername`.
- `server_combo` lists known grids plus free-text URIs (tooltip string `ServerComboTooltip`, lines 17-20); on OpenSim
  builds the hidden `grid_builder_text` opens `FSGridBuilderURL` (`settings.xml:176-187`).
- `mode_combo` writes `SessionSettingsFile` (`fspanellogin.cpp:220-222`); the first entry is labelled "Vulkanstorm"
  while keeping Firestorm's internal name/value (`panel_fs_nui_login.xml:344-347`).
- `forgot_password_url` = `http://secondlife.com/account/request.php` (lines 13-16); the "Create an account" link uses
  the grid's register URL from the grid info (existing Firestorm behaviour).

What changed versus the Firestorm layout at merge-base (`git show fbc9ee9624:.../panel_fs_nui_login.xml`): Firestorm had
the browser on top (`top=18 height=598`) and a 152px-high horizontal bar at the bottom with the 125x125 `login_fs_logo`
icon, two rows of controls (username/password/remember, then grid/mode/Log In) and background `0.16 0.16 0.16`. Vulkanstorm
replaced this with the vertical sidebar above and dropped the logo icon in favour of the text title; the control set
(username removal, password visibility, grid selection, viewer modes, saved locations) was preserved (`doc/branding_defaults.md:43-45`).

### 3.3 Welcome page selection

`FSPanelLogin::loadLoginPage` (`fspanellogin.cpp:919-932`): after computing the grid's `login_page` and reading
`ForceLoginURL`, if no forced URL is set and the grid's login page is empty or equals the stock Firestorm splash
(`http://phoenixviewer.com/app/loginV3/` or `https://...`), the browser navigates to the local page
`navigateToLocalPage("vulkanstorm", "welcome.html")`, i.e. `findSkinnedFilename("html", "vulkanstorm/welcome.html")`
(`llmediactrl.cpp:692-708`), which resolves `skins/default/html/<lang>/vulkanstorm/welcome.html` with the `en` fallback.
Otherwise grid-provided splash pages (OpenSim `login_page`, SL grid pages) and the tester override are used unchanged,
with the usual query string appended (version, channel, grid, lang, etc.). The page was moved from `html/common/` to
`html/en/` in commit `3e8b11f251` because the skinned lookup is language-scoped.

### 3.4 `welcome.html` content (`indra/newview/skins/default/html/en/vulkanstorm/welcome.html`)

- `<title>Welcome to Vulkanstorm</title>`; body background `#091119` with a radial teal glow `#123C50` at 95%/10%;
  text `#E5F0F6`, Arial; 24px/20px padding; max-width 620px (lines 9-11).
- Eyebrow "VULKANSTORM VIEWER" in `#51C7E8`, 12px, letter-spacing .2em (lines 12, 23).
- H1 "Your world. / Your next adventure." clamp(32px, 6.7vw, 48px), weight 500 (lines 13, 24).
- Paragraph (`#A8BECB`, 17px): "Welcome to Vulkanstorm. Sign in to explore, create, and connect in Second Life and OpenSim worlds." (line 25).
- Nav links (`#72D6EF`, 14px, top border `#294250`): "Project & releases" -> `https://github.com/anne-skydancer/vulkanstorm`,
  "Report an issue" -> `.../issues` (lines 26-29).
- Footer (`#7F97A6`, 12px): "Vulkanstorm is an independent viewer derived from Firestorm and the Second Life viewer.
  Second Life is a trademark of Linden Research, Inc." (line 30).

### 3.5 Progress panels (login / teleport / logout)

Two panels are instantiated from `main_view.xml:162-174` as classes `progress_view` and `progress_view_mini`
(`llprogressview.cpp:67`, `llviewerwindow.cpp:2359-2360`). Skin file selection takes the most specialised English file,
so the Modern skin ships full copies (`scripts/tests/test_progress_panels.py:18-25` asserts the Modern files equal the
default ones except for the bar colour; commit `b510ea583f`).

`panel_progress.xml` (default: `skins/default/xui/en/panel_progress.xml`; Modern: `skins/ansastorm_modern/xui/en/panel_progress.xml`):

- Full-screen 1024x768 panel, placed off-screen (`top=768`) and slid/faded in by code (lines 2-9). Background is the
  start texture (`gStartTexture`, last-frame snapshot or black) drawn aspect-correct, fading (`llprogressview.cpp:125-160`).
- Horizontal stack: 150px elastic | 670px centre column (min 670) | 150px elastic (lines 10-29, 178-183).
- Vertical centring: 200px flex | fixed 255px box "panel4" | 200px flex (lines 39-51, 170-175).
- The box: icon `Rounded_Square` tinted `LoginProgressBoxCenterColor` (black at 0.78 alpha) 670x255 (52-60); inside a
  non-animated vertical stack: 30px spacer (71-78); "panel_login" 100px (79-120) containing `title_text` (SansSerifHuge,
  no shadow, x47 y32, right -47, `LoginProgressBoxTextColor`; text = `getSecondLifeTitle()` i.e. the viewer/grid title,
  `llprogressview.cpp:165`), `progress_text` (SansSerif, 5px below, word-wrap) and `login_progress_bar`
  (height 16, x45..-45, 5px below; Modern: `color_bar="LoginProgressBarFgColor"`, default: `1 1 1 0.96`);
  "panel_motd" 90px (121-143) with `message_text` (SansSerifLarge, valign centre, x45, right -90, line spacing 2);
  "panel_icons" 40px (144-167) with label `logos_lbl` "[APP_NAME] uses" (SansSerifLarge, x47 y6, w240) after which
  the third-party logos (FMOD/Havok/etc.) are drawn by code relative to that label (`llprogressview.cpp:326-345`).
- `cancel_btn` 90x22 anchored bottom-right at (-106, 700) (184-191); hidden `web_browser` `login_media_panel` full-size
  for intro video (192-199).
- Progress-bar widget defaults (`ansastorm_modern/xui/en/widgets/progress_bar.xml`): track image `ProgressTrack`, fill
  image `ProgressBarSolid`, `color_bar = LoginProgressBarFgColor` (blue #009ED8 a.96), track colour
  0.3254 0.4000 0.5058 (#536681). The default skin's widget (`skins/default/xui/en/widgets/progress_bar.xml`) keeps
  Firestorm's orange fill 0.898 0.533 0.173 with the `ProgressBar` image; the default panels override it to white 0.96.

`panel_progress_mini.xml` (default + Modern copies): a 200x20 strip for the "mini" teleport progress: `progress_bar_mini`
spanning left..-64 (height 20, same colour rule), `cancel_btn` "Cancel" 60x20 at the right, `progress_text` overlaid at
x10 y5 h16 ("Progress...") (lines 13-40). Vulkanstorm moved the text from `top=0 h20 valign=center` to `top=5 h16`
(diff vs upstream).

---

## 4. Defaults that constitute Vulkanstorm identity beyond art

Method: parsed `indra/newview/app_settings/settings.xml` (LLSD) at HEAD and at merge-base `fbc9ee9624` and compared every
shared `Value` (2397 vs 2379 entries). Values that differ only because upstream Firestorm moved on after the base are
listed separately. Line numbers are HEAD `settings.xml`.

| Setting | Vulkanstorm | Firestorm (merge-base) | Area | Line |
|---|---|---|---|---|
| `SkinCurrent` | `ansastorm_modern` | `firestorm` | skin | 15290 |
| `SkinCurrentTheme` | `` (empty) | `grey` | skin | 941 |
| `RestrainedLove` | 1 | 0 | RLVa on by default (restart required) | 1209 |
| `CacheSize` | 4096 | 2048 | cache (MB) | 2935 |
| `ChatPersistTime` | 45.0 | 20.0 | chat fade seconds | 3265 |
| `ConsoleMaxLines` | 12 | 40 | nearby chat console | 3511 |
| `AutohideChatBar` | 1 | 0 | chat bar | 134 |
| `AutoCloseOOC` | 1 | 0 | auto-close "((" OOC | 1097 |
| `EmotesUseItalic` | 1 | 0 | chat formatting | 5181 |
| `FSSecondsinChatTimestamps` | 1 | 0 | timestamps | 1031 |
| `FSFontChatLineSpacingPixels` | 0 | 2 | chat line spacing | 21167 |
| `PlayChatAnim` | 0 | 1 | no chat animation | 9776 |
| `PlayTypingAnim` | 0 | 1 | no typing animation | 9787 |
| `PlayModeUISndTyping` | 0 | 1 | no typing sound | 22989 |
| `CameraOffsetRearView` | `0.0 0.0 -0.2` | `-3.0 0.0 0.75` | camera | 3007 |
| `FocusOffsetRearView` | `1.0 1.0 0.2` | `1.0 0.0 1.0` | camera | 5758 |
| `CameraOffsetScale` | 0.75 | 1.0 | camera | 3022 |
| `TextureCameraBoost` | 8.0 | 7.0 | texture priority of camera-facing | 15665 |
| `RenderGLContextCoreProfile` | 1 | 0 | GL core context | 12430 |
| `RenderGLMultiThreadedTextures` | 1 | 0 | texture threading | 12441 |
| `GridListDownloadURL` | `https://phoenixviewer.com/app/fsdata/grids.xml` | `http://...` | HTTPS grid list | 156 |
| `OmnifilterEnabled` | 0 | 1 | feature disabled on release | 27523 |
| `FSCloudColorStart` (new key) | `0 0.5 0.5 0.1` | n/a | teal avatar cloud | 27579 |
| `FSCloudColorEnd` (new key) | `0 0.5 0.5 0.9` | n/a | teal avatar cloud | 27595 |
| `FSEnabledLanguages` | `az de en es fr it ja pl ru zh` | upstream/master adds `pt` | languages (release ships no pt) | 25661 |

Other catalogues:

| File | Setting | Vulkanstorm | Firestorm |
|---|---|---|---|
| `app_settings/settings_per_account.xml:449` | `LogNearbyChat` | 1 | 0 (owner's explicit choice, `doc/archive_defaults_audit.md:24-35`) |
| `app_settings/settings_firestorm.xml:75-96` | `SkinCurrent` / `SkinCurrentTheme` | `ansastorm_modern` / `` | `firestorm` / `grey` |
| `app_settings/settings_firestorm.xml:127-136`, `settings_phoenix.xml:114`, `settings_text.xml:114`, `settings_v3.xml:126` | `LetterKeysAffectsMovementNotFocusChatBar` | 1 (letter keys move the avatar) | 0 |
| `app_settings/cloud.xml:55-69` | StartColor / EndColor | `0 0.5 0.5 0.1` / `0 0.5 0.5 0.9` (teal) | `0.666 0.228 0.13 0.1` / `0.766 0.328 0.23 0.9` (orange) |
| `featuretable.txt`, `featuretable_linux.txt` | `RenderGLMultiThreadedTextures`, `RenderGLContextCoreProfile` | 1 in base rule / Intel rule | 0 |
| `skins/skins.xml:3-17` | skin registry | "AnsaStorm Modern" (theme "Blue", folder `ansastorm_modern`) listed first | listed after AnsaStorm |
| `app_settings/settings_modern.xml` | removed (upstream added it later; release does not ship a Modern mode catalogue) | -- | -- |

Settings that only differ against `upstream/master` because upstream changed after the base (not Vulkanstorm choices):
`FSDisableLabeledChatLinks` (upstream now 1), the `Omnifilter*`, `FSContactSets*`, `FSCmdLine*`, `InventoryShow*Tab`
keys absent from the release, and the Vulkanstorm-only renderer/editor keys (`RenderBackend=OpenGL`, `RenderAlphaOIT*`,
`ExternalWebsocket*`, `FSMusicSpatialSound`, `TextureBiasUnimportantFactor`) which are not branding.

Owner-stated intent table (`doc/archive_defaults_audit.md:27-35`): RLVa on, local chat logs on, nearby/IM/transcript
timestamps on, seconds in timestamps on, `LogTimestamp` on, auto-close OOC on.

---

## 5. Channel, version and branding strings

| Item | Value | Where |
|---|---|---|
| Product name | `Vulkanstorm` | `indra/llcommon/indra_constants.h:35` (`APP_NAME`), `strings.xml:10-11` (`APP_NAME`, `CAPITALIZED_APP_NAME`=VULKANSTORM), `APP_NAME_ABBR`=`VS` (`strings.xml:18`) |
| Channel vendor base | `Vulkanstorm` | `indra/lib/python/indra/util/llmanifest.py:97` (`CHANNEL_VENDOR_BASE`) |
| Channel names | `Vulkanstorm-Release` (CI release), `Vulkanstorm-RelWithDebInfo` (CI dev), `Vulkanstorm-<chan>` from `configure_firestorm.sh` (default `private-<hostname>`); the bare `Vulkanstorm` channel counts as the base release | `scripts/ci_release_config.py:35`, `scripts/configure_firestorm.sh:409-419`, `viewer_manifest.py:323-354`, `scripts/tests/test_inno_payload.py:12-15` |
| Channel types recognised | release, beta, alpha, nightly, manual, profiling, private | `viewer_manifest.py:335-354` |
| Version scheme | `1.0.0.<count>` where `<count>` = `git rev-list --count HEAD` (merges included); canary `1.0.0-canary.<count>`; shallow clones rejected; Windows RC file/product version `1.0.0.0` (16-bit limit) while string ProductVersion keeps the full value; macOS bundle versions `1.0.0` | `doc/version_numbering.md:3-13`, `indra/newview/VIEWER_VERSION_FS.txt` (`1.0.0`), `indra/cmake/BuildVersion.cmake:10-66`, `fsversionvalues.h.in:44` (`LL_VIEWER_VERSION_SUFFIX`), `llversioninfo.cpp:52-54`, `res/viewerRes.rc` (FILEVERSION ...,0), `CMakeLists.txt:2838-2839` |
| Strict version parser | `LLViewerVersion::parse/compare` accept `M.m.p[-canary].count` only | `indra/newview/llversionparser.h:16-49` |
| Underlying Linden version | `26.1.1` | `indra/newview/VIEWER_VERSION.txt` (reported as `LL_VIEWER_VERSION_LL`) |
| Build target / exe | CMake target `vulkanstorm-bin`; packaged exe = app name + `.exe`: `Vulkanstorm.exe` for the bare channel, `Vulkanstorm-Release.exe` for `Vulkanstorm-Release` (OpenSim builds insert `OS`) | `indra/newview/CMakeLists.txt:156`, `indra/CMakeLists.txt:206-209`, `viewer_manifest.py:399-432,608-609,674-675`, `installer_template.iss:12-15` |
| Installer file name | `Vulkanstorm[<variant>]_1_0_0_<count>_x86-64[_Setup].exe`; mac `..._universal.dmg` | `viewer_manifest.py:374-398` |
| Windows resources | CompanyName "The Vulkanstorm Project", FileDescription/InternalName/ProductName "Vulkanstorm", OriginalFilename `Vulkanstorm.exe`, LegalCopyright "Copyright (c) 2010-2026, The Vulkanstorm Project", splash text "Loading Vulkanstorm ..." | `indra/newview/res/viewerRes.rc` |
| Inno installer identity | AppId `{7A4C2E91-3F5B-4D6A-9C1E-5E8F2A6B9D04}` (stable upgrade GUID), AppPublisher "The Vulkanstorm Project", default dir `{autopf64}\<AppNameOneWord>`, registry `HKLM\SOFTWARE\The Vulkanstorm Project\<AppNameOneWord>` (Version/Shortcut/Exe), tasks "desktopicon" and "protocols" registering `secondlife`, `x-grid-location-info`, `x-grid-info` and (OpenSim only) `hop` URL schemes with `-url "%1"`, LGPL-2.1 licence page (`doc/LGPL-license.txt`), "Launch Vulkanstorm now" on finish, x64 only, MinVersion 6.1sp1 | `installers/windows/installer_template.iss:33-141`; generated via `viewer_manifest.py:1055-1120` as `vulkanstorm_setup.iss` |
| Velopack (updater) app name | prefix `Vulkanstorm` + channel remainder (+`OS`) | `indra/newview/llvelopack.cpp:300-317` |
| Linux | install dirs `/opt/vulkanstorm`, `~/vulkanstorm`; wrapper still named `firestorm`, binary `do-not-directly-run-vulkanstorm-bin`; desktop entry `vulkanstorm-viewer.desktop` "Vulkanstorm Viewer", icon `firestorm_icon.png` (512 VK icon), `StartupWMClass=do-not-directly-run-vulkanstorm-bin`; D-Bus service name unchanged `com.secondlife.ViewerAppAPIService`; libnotify app name "Vulkanstorm" | `linux_tools/install.sh:15-16`, `refresh_desktop_app_entry.sh:28-37`, `handle_secondlifeprotocol.sh:13-19`, `wrapper.sh:180`, `viewer_manifest.py:2342-2348`, `desktopnotifierlinux.cpp:237,267` |
| macOS | **not rebranded**: product/bundle name `Firestorm`, `MACOSX_BUNDLE_GUI_IDENTIFIER=org.firestormviewer.firestorm`, info string "Firestorm Viewer", copyright "The Phoenix Firestorm Project, Inc.", nib `Firestorm.nib`; only the icon (`firestorm_icon.icns`) and the DMG background/volume name (`"Vulkanstorm Installer"`) are Vulkanstorm | `indra/newview/CMakeLists.txt:2831-2842`, `viewer_manifest.py:1946` |
| Growl/notification app name | "Vulkanstorm" | `growlmanager.cpp:212` |
| Avatar client tag (visible to other viewers' tag readers) | texture-entry tag string `"Vulkanstorm"`, colour from per-account `FirestormTagColor` default (1,0,1,1) | `llvoavatarself.cpp:3363-3372` |
| Browser/HTTP User-Agent | `SecondLife/<version> (<channel>; <SkinCurrent> skin)` e.g. `SecondLife/1.0.0.NNN (Vulkanstorm-Release; ansastorm_modern skin)` | `llviewermedia.cpp:417-438` |
| Crash dialog text | "We are sorry, but Vulkanstorm has crashed and needs to be closed..." | `llappviewer.cpp:2682` |
| Support / download URLs | **inherited from Firestorm, unchanged**: `SUPPORT_SITE`="Firestorm Support Portal", `DOWNLOAD_URL`/`ViewerDownloadURL`=`https://www.firestormviewer.org/choose-your-platform/`, help wiki `https://wiki.firestormviewer.org/[TOPIC]`, installer AppURL from `dl_url_from_channel()` (firestormviewer.org pages); the only Vulkanstorm URLs are the welcome page's GitHub links and README | `strings.xml:12-13,22`, `settings.xml:6052,6065`, `viewer_manifest.py:1019-1032`, `welcome.html:27-28` |
| Grid list handling | `GridListDownload` stays 1; stock URL is HTTPS; a saved `http://phoenixviewer.com/.../grids.xml` is upgraded in memory; for the stock HTTPS feed the request enables SSL peer+host verification and disables redirects, custom feeds keep their existing options | `llstartup.cpp:1019-1052`, commits `8b73e2f8f7`, `37b35e5dfe`, `doc/archive_defaults_audit.md:62-63` |
| Grid labels / login grids | unchanged from base (the `util.aditi.lindenlab.com` change in `llviewernetwork.cpp` is upstream's) | `git diff fbc9ee9624 -- llviewernetwork.cpp fsgridhandler.cpp` is empty |
| About floater | credits lists trimmed (Chaser Zaks, Lassie, Willow Wilder removed from the FS team list etc.), SoLoud and miniaudio licence lines added; otherwise Firestorm's | `floater_about.xml` diff |
| Project description | "Vulkanstorm is a free client for 3D virtual worlds such as Second Life and OpenSim ... a viewer derived from Firestorm and the Second Life viewer." | `README.md:1-5` |
| Preferences tab labels | graphics tabs "Rendering 1"/"Rendering 2" (incl. Italian override) | `doc/branding_defaults.md:40-41`, `panel_preferences_graphics1.xml` |

---

## 6. What transfers as data vs what must be re-authored; licence obligations

### 6.1 Transfers unchanged (pure data)

- The owner's icon set: copy `indra/newview/icons/vulkanstorm/` wholesale (master PNG 1254px, 12 PNG sizes, 2 ICOs, ICNS,
  BMP). Regenerate nothing; the Rust viewer only needs the master plus whichever ICO/ICNS/PNG its packaging uses. The
  `firestorm_*` filenames are compatibility artefacts (`README.md:9-10`); rename freely in the new repo.
- The 8 DMG `background.png` copies are one 300x180 image; keep one.
- `welcome.html`: copy as-is; it has no external assets. If the new viewer embeds a browser (CEF/webview) it can be
  served from disk exactly as Firestorm does; if not, the text/colours above are the spec.
- Colour tables: `vulkanstorm_modern_colors.tsv` / `vulkanstorm_default_colors.tsv` (generated from the XML). The Rust
  theme should define the Kokua purple, the login-bar blue, the flash `#7f7fff`, the dark greys and the login literals
  (sections 2.2, 3.2) and keep the names so settings/UI code can refer to them.
- Strings: `APP_NAME`/`VULKANSTORM`/`VS`, channel base `Vulkanstorm`, channel suffix conventions, Inno AppId GUID and
  registry hive name, URL-scheme list, client tag "Vulkanstorm", user-agent format, version scheme `1.0.0.<count>` and
  the strict parser rules. These are what the login server, grid tools and other viewers "see".
- Settings defaults (section 4 table, cloud.xml teal, per-account LogNearbyChat) as a key/value list.
- Fonts: DejaVu Sans at 7.6/8.6/10.6/16 pt (`fonts.xml`) if the login should look identical.

### 6.2 Must be re-authored (XUI-specific)

- `panel_fs_nui_login.xml` and its 8 translation overlays: the layout_stack/layout_panel/scroll_container semantics
  (elastic spacers, `follows`, `auto_resize`, `min_width`) and the C++ centring/scroll helpers have no equivalent outside
  XUI; use section 3.2 as the spec. The translated label strings in `xui/<lang>/panel_fs_nui_login.xml` are reusable text.
- `panel_progress.xml` / `panel_progress_mini.xml` and the 9-slice widget art (`ProgressTrack`, `ProgressBarSolid`,
  `Rounded_Square`, `PushButton_Login*`): either redraw native equivalents (recommended, avoids CC-BY-SA obligations) or
  carry the PNGs with the notice.
- `textures.xml` registrations, `skins.xml`, `colors.xml` alias/reference mechanics (names referencing other names).
- `installer_template.iss` is reusable almost verbatim (replace the `%%TOKEN%%` substitutions); the Linux scripts are
  reusable but still name the wrapper `firestorm`.

### 6.3 Licence / notice obligations that must travel with the art

- Owner artwork (`icons/vulkanstorm/*`, `login_fs_logo.png`, `startup_logo.png`, `vulkanstorm_logo.png`,
  `doc/vulkanstorm_256.png`, DMG `background.png`): no third-party notice required; it is the project owner's own work
  (`icons/vulkanstorm/README.md:3-4`). Decide on and state a licence for it in the new repo (currently none is declared
  beyond the repo's LGPL `doc/LICENSE-source.txt`).
- Any Firestorm/Linden skin PNG/J2C reused (section 1.3 list, the Modern-skin widget art, `_VolumeIcon.icns`,
  `loginbackground.bmp`): ship `doc/LICENSE-logos.txt` verbatim, identify the changes made, and include the notice "where
  copyright notices are usually included" (text file, About window or credits), `LICENSE-logos.txt:16-21`. Do not use the
  Second Life name or eye-in-hand logo as branding (trademark carve-out, lines 11-14). The welcome page's footer sentence
  ("Second Life is a trademark of Linden Research, Inc.") is the existing attribution and should be kept.
- Kokua-derived data (colour palette values, login structure): Kokua is LGPL-2.1 and its skin files are CC-BY-SA like
  Firestorm's. Add a line to the new repo's credits/NOTICE: "Colour palette and login layout adapted from the Kokua
  viewer (commit 20493b5e73)". The Vulkanstorm tree currently records this only in `doc/branding_defaults.md:33-36,43-44`
  and the XML comments (`ansastorm_modern/colors.xml:3`, `panel_fs_nui_login.xml:2`).
- The `welcome.html`, `installer_template.iss`, progress/login XML and settings are LGPL-2.1 Vulkanstorm/Firestorm source
  (`doc/LICENSE-source.txt`); reusing them in a Rust codebase means that code stays LGPL-2.1 unless re-written.
- Remaining Firestorm identity leaks to resolve before calling the Rust viewer "the same product": macOS bundle
  identifier/name/copyright (`CMakeLists.txt:2831-2842`), support/download/help URLs pointing at firestormviewer.org,
  the `firestorm` wrapper name on Linux, `_VolumeIcon.icns`, and the `FirestormTagColor` setting name.
