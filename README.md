# vkstorm-aleph

A Second Life viewer written from the ground up in Rust on Vulkan.

Goals:

- Vulkan-native renderer with a focus on visual quality
- Spatial audio
- RLV support

Reference: the Firestorm viewer source is tracked on the `vkstorm-aleph` branch of
[anne-skydancer/vulkanstorm](https://github.com/anne-skydancer/vulkanstorm), which follows
Firestorm `master`. No code is shared with that repository; it is read for reference only.

## Documents

- [docs/upstream-study/](docs/upstream-study/README.md): reverse-engineering study
  of the Firestorm viewer that settles how much native code is unavoidable, how
  the Rust viewer is structured, how the GLSL corpus ports to Vulkan GLSL, and
  how to reach a visually equivalent render. Its
  [carry-over annex](docs/upstream-study/06-carry-over.md) covers the
  Vulkanstorm features brought into this project.
- [assets/branding/](assets/branding/NOTICE.md): the project's icon, login
  logos and welcome page, with provenance.

## Licence

This repository, including its source code, documentation and artwork, is
licensed under the GNU Lesser General Public License, version 2.1. See
[LICENSE](LICENSE).
