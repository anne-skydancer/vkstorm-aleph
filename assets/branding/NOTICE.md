# Branding assets

All files in this directory are the project owner's own artwork and text,
carried over from the Vulkanstorm release branch
(`anne-skydancer/vulkanstorm`, `vkstorm-release` at 08c07b4cba).

- `icon/`: the blue "VK" icon. `vulkanstorm-blue-1254.png` is the unmodified
  master supplied by the owner on 2026-10-03; every other size and format is a
  derivative of it. `vulkanstorm.ico` is the supplied ICO (PNG-compressed 256px
  frame); `vulkanstorm-legacy.ico` is the uncompressed fallback.
- `login/`: the login logo tile, the startup logo, and the bundled welcome
  page shown in the login sidebar when no grid splash URL applies.
- `dmg-background-300x180.png`: macOS disk-image window background.

No Linden Lab or Firestorm artwork is included here, so the Second Life
viewer artwork notice (CC-BY-SA 3.0) does not apply to this directory. The
Vulkanstorm login and progress screens also used Firestorm widget art
(buttons, progress track, rounded squares) which was deliberately not copied;
the Rust viewer draws its own.

The welcome page keeps the sentence "Second Life is a trademark of Linden
Research, Inc." as attribution. The colour palette and login layout were
adapted from the Kokua viewer (commit 20493b5e73); see
`docs/upstream-study/carry-over/branding-login.md`.

This artwork is licensed under the GNU Lesser General Public License,
version 2.1, the same licence as the Vulkanstorm viewer and as this
repository. The licence text is in `LICENSE` beside this file and at the
repository root.
