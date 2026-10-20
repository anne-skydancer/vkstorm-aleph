# Upstream study: Firestorm as the reference for vkstorm-aleph

This study reverse-engineers the Firestorm viewer to settle four questions for a
from-scratch Second Life viewer in Rust and Vulkan:

1. How much C or C++ is unavoidable, for shaders and everything else.
2. How the viewer should be structured in Rust.
3. How the GLSL shader corpus transposes to Vulkan GLSL compiled with glslang.
4. How to reach a visually equivalent render in Vulkan, raw or through an RHI,
   including post-processing.

The answers are in this file. The five numbered reports beside it are the source
material: each is a survey of one area of the Firestorm tree with every claim
cited as `path:line`.

## Source and provenance

| | |
|---|---|
| Reference tree | `anne-skydancer/vulkanstorm`, branch `vkstorm-aleph` |
| Commit | `0de5043af00b1b7dd3201d3b878bb7033e4bf067` (Firestorm `master`, 2026-10-18) |
| Local checkout used | `C:\Dev\vkstorm-aleph-ref` (git worktree) |
| Paths in reports | relative to `indra/` unless stated |
| Date of study | 2026-10-20 |

The reports were produced by five parallel code surveys, then synthesised here.
Line numbers are valid for the commit above only. Each report ends with a list of
what its author could not determine from code alone. Treat those lists as the
empirical-verification backlog.

Reports:

- [01-render-pipeline.md](01-render-pipeline.md): the ordered frame, G-buffer
  layout, lighting, probes, post chain, water and sky, culling, visual quirks.
- [02-shader-system.md](02-shader-system.md): how programs are assembled, the
  uniform contract, the program inventory, the corpus catalogue, the Vulkan
  GLSL migration analysis.
- [03-content-model.md](03-content-model.md): prim mesher, mesh assets,
  skinning, terrain, texture entries, legacy and PBR materials, EEP, camera.
- [04-viewer-architecture.md](04-viewer-architecture.md): login, messages,
  object model, textures, avatars, audio, RLVa hooks, inventory, UI, threading,
  proposed crate layout, anti-patterns.
- [05-native-dependencies.md](05-native-dependencies.md): every third-party
  package, pure-C++ subsystems that must be rewritten, the shader toolchain
  question, plugins, platform code, FFI verdict.
- [06-carry-over.md](06-carry-over.md): the Vulkanstorm release-branch work
  brought into Aleph (branding and artwork, login page, markdown chat, PPLL
  order-independent transparency, SLua editor and IDE bridge) and the audio
  decisions on SoLoud and speaker fill. Detail reports in
  [carry-over/](carry-over/).

## Scale of the thing

| Area | Size in Firestorm |
|---|---|
| `newview/` (the application) | 834 `.cpp`, ~740k lines, a quarter to a third of it UI |
| Lower libraries (`llcommon`, `llmath`, `llmessage`, `llprimitive`, `llcharacter`, `llappearance`, `llimage`, `llinventory`, `llcorehttp`, `llrender`) | ~290k lines |
| `llui` + XUI | 102k lines C++, 793 XML files / 179k lines |
| Shader corpus | 224 GLSL files, 25.3k lines (3 vendored post files are 24% of that) |
| Linked shader programs at max settings | 226 (210 at defaults) |
| Message template | 483 messages, ~150 UDP handlers, 117 capabilities requested per region |
| RLVa | ~17k lines, hooks in 219 non-RLV files |
| Globals | ~370 `g*` externs, 149 singletons, 265 `#if OPENSIM` sites |

## 1. How much C or C++ is necessary

### Shaders: none in our own code

Firestorm compiles GLSL at runtime through the OpenGL driver and nothing else.
There is no offline shader tooling anywhere in the tree. The only cache is of
driver program binaries.

For a Vulkan viewer the question becomes which SPIR-V compiler to use:

| Option | C++ involved | Verdict |
|---|---|---|
| glslang through the `glslang` crate (0.9.0, updated 2026-10-11) or `shaderc` (0.10.1, 2025-09) | glslang and SPIRV-Tools built by the crate's build script, shipped inside the binary | **Recommended.** The reference compiler. Keeps the existing define-driven variant model working on day one. |
| naga (pure Rust, 30.0.1) | none | Not viable as-is. Its GLSL front end is officially "secondary support", GLSL 440+ with Vulkan semantics, and it is not a full preprocessor. The corpus has 30+ arithmetic `#if` lines and relies on the loader splicing library files. Would need a separate preprocessor and shader clean-up first. |
| Offline precompile of all permutations | build-time only | Not viable without a redesign. The settings-driven matrix is over 10^4 variants and some defines are derived from hardware limits at runtime. |

So the answer is: **zero C++ written by us for shaders**, but glslang (C++) ships
inside the binary via a Rust crate as long as we keep runtime GLSL compilation.
A later migration to specialization constants would shrink the variant space
enough to precompile, and glslang could then become build-time only. Details
in [05 §3](05-native-dependencies.md) and [02 §5](02-shader-system.md).

### Everything else: three mandatory native libraries

The shipped Firestorm binary carries four proprietary components, all of which
disappear in the Rust viewer: FMOD Studio, Kakadu, Havok, BugSplat. Vivox is
OpenSim-only; Second Life voice is WebRTC.

What remains as C or C++ behind FFI, in order of necessity:

| Library | Why it stays native | Rust path |
|---|---|---|
| **OpenJPEG** | Texture decode is the most performance-critical codec in SL. It needs resolution-reduced decode of truncated byte ranges (discard levels) and encoding for uploads. `hayro-jpeg2000` (0.4.1, pure Rust, 4M downloads) is a strong full-image decoder but progressive partial decode is unconfirmed. | `openjp2-sys` FFI, ~45 symbols. Benchmark `hayro-jpeg2000` for the full-decode path. |
| **libwebrtc** | SL voice needs its audio processing (echo cancel, gain, noise), device I/O, NetEQ and Opus. ~60 symbols used today. | Wrap Firestorm's `llwebrtc.dll` interface (~25 virtuals) behind a C ABI, or LiveKit's `webrtc-sys`. |
| **CEF** | Chromium. Out of process either way. | Keep the C++ `SLPlugin` sidecar and reimplement only the host side of its LLSD-over-socket protocol (~3.5k lines). |
| meshoptimizer | 9 functions, mature `meshopt` crate. | FFI, no reason to port. |
| GStreamer or libvlc | Media playback, out of process. | `gstreamer-rs` or `vlc-rs`. |
| Steam Audio (optional) | HRTF, occlusion, reverb. Free binary, not OSI. | `audionimbus` 0.16.0 (2026-08). Alternative is the pure-Rust `hrtf` crate used by Fyrox. |

Estimated hand-written unsafe binding code: 2 to 4k lines, concentrated in J2C
and voice. Optional extras: Hunspell, libvorbis encode, VHACD for upload
physics, libndofdev.

Everything else goes pure Rust: networking (`reqwest`, `rustls`, `tokio` UDP),
compression (`flate2`), images (`png`, `zune-jpeg`), XML (`quick-xml`), math
(`glam`), tangents (`mikktspace`), text (`cosmic-text`), windowing (`winit`),
audio output (`cpal`) and decode (`symphonia`), crash dumps (`minidumper`).

### Pure-C++ subsystems that must be rewritten regardless

These have no external library behind them. They are the real porting work.

| Module | Lines | Notes |
|---|---|---|
| `llmath` | 30.6k | **`LLVolume` (7.4k) is the bit-exact procedural prim mesher.** Every prim in SL is generated client-side from 17 parameters; the vertex math must match or the world looks wrong. |
| `llcommon` | 81.8k | LLSD plus its three codecs (XML, binary, notation) is the wire format for all capabilities. The rest is threading and utilities that `std`/`tokio` replace. |
| `llmessage` | 56k | Template-driven UDP: `message_template.msg` parsing, zero-coding, reliability, circuits, throttles. Should be code-generated from the template at build time. |
| `llprimitive` | 18k | Texture entries, legacy materials, GLTF materials, mesh asset parsing. |
| `llcharacter` | 16k | Joints, motions, `.anim` format, blending. |
| `llappearance` | 13.9k | Wearables, visual params, `avatar_lad.xml` (17.5k lines of data). |
| `llimage` | 10k | Codec wrappers; mostly replaced by crates. |
| `llinventory` | 15k | Types and model. |
| `llcorehttp` | 12.7k | Policy-class HTTP; replaced by `reqwest` plus ~1k lines of policy. |
| `llrender` | 28.5k | ~16k is GL plumbing (discard), ~8k is portable font and 2D logic. |
| `llui` + XUI | 102k + 179k XML | **Replace, do not port.** Keep `notifications.xml` and `strings.xml` as data. |

## 2. How the viewer should be made in Rust

### Crate decomposition

The lower Firestorm libraries already have the property the rewrite needs:
`llcommon`, `llmath`, `llmessage`, `llinventory`, `llprimitive` and `llcorehttp`
never include `llui`. `newview` lacks it entirely. The proposed layout keeps that
discipline and splits `newview` along the seams Firestorm never cut. Full table
with module mapping in [04 §12](04-viewer-architecture.md).

```
viewer (bin)         startup/login state machine, settings, scheduler
  sl-ui              winit, UI toolkit, floaters as views, media plugin host
  vk-render          Vulkan renderer, spatial index, batching, residency
  sl-audio           source/listener model, HRTF mixer, wind, voice
  sl-rlv             RLVa parser, restriction state, Policy trait
  sl-comms           chat, IM, groups, notifications as typed events
  sl-avatar          skeleton, params, wearables, motions, attachments
  sl-world           regions, object store, VOCache, terrain, movement
  sl-inventory       model, AIS v3, COF
  sl-assets          ViewerAsset, mesh, J2C, texture fetch and cache
  sl-http            policy-class client, caps, event queue, XML-RPC login
  sl-messages        template codegen, packets, circuits, UDP driver
  sl-core            UUID, LLSD, coordinate newtypes, enums, packers
```

Dependency direction is strictly top to bottom. Every `lib` crate compiles
without the binary.

### Data-flow contracts

- `sl-messages` produces decoded, owned message structs on a channel. A dispatch
  table generated from the template routes them. Event-queue LLSD bodies decode
  through the same generated code, preserving Firestorm's two-transport bus.
- `sl-world` exposes per-frame immutable snapshots or a change log. The renderer
  and audio read snapshots, never the live store. Dirty flags become explicit
  `ObjectChange` events.
- Domain crates emit typed events (`OfferReceived`, `ScriptDialogRequested`,
  `RestrictionChanged`) and accept commands. `sl-ui` is the only consumer that
  turns them into widgets. This inverts Firestorm's loop of protocol handler to
  `LLNotificationsUtil::add` to XML-template-named callback to protocol reply.
- One `Session` struct owned by the binary replaces the ~370 globals and 149
  singletons. Two sessions in one process becomes possible, and so do tests.

### What to keep from the C++ design

The reports are explicit that several things are right and should be
reimplemented exactly: reliable-packet semantics (piggybacked acks, resend at
5x averaged ping, 3 retries, 5 s ping, 100 s timeout); visibility-driven object
instantiation from the VOCache (compressed updates are not instantiated on
receipt, only when their cached bounds enter the frustum); CRC cache probes;
time-sliced work budgets; dead reckoning with phase-out; progressive J2C by
discard level with byte-range fetches; AIS v3 with atomic COF updates and
server-side baking; the RLVa command grammar and behaviour table, because
scripts in the wild depend on its quirks.

### What not to copy

God objects (`LLViewerObject` 8.2k lines, `LLVOAvatar` 13.7k, `LLPipeline`
12.6k), global singletons, ad-hoc mutexes with lock-order comments, the
poll-everything frame loop, notification templates as control flow, localisation
and UI calls inside protocol handlers, compile-time grid switches, and the
immediate-mode UI with no batching. The table in [04 §13](04-viewer-architecture.md)
pairs each with its replacement.

### Spatial audio: what the server gives you

This matters for the audio goal, so it is stated plainly. The server sends world
audio as events only: asset UUID, owner and object IDs, gain 0 to 1, flags
(loop, sync master/slave, queue, stop), a cutoff radius, and a region-local
position for one-shot triggers. **All spatialisation is client-side today**, and
it is delegated to FMOD or OpenAL defaults: min and max distance are never set,
no HRTF, listener orientation from the camera but velocity from the agent.

Consequences:

- World sounds are always mono 44.1 kHz. Ideal HRTF point sources.
- Attached sounds sample the client-interpolated object position every frame.
  There is no audio-specific position stream.
- Wind is synthesised client-side from the region wind vector minus agent
  velocity (pink noise through a resonant low-pass). The algorithm is small and
  documented in [04 §6.1](04-viewer-architecture.md).
- **Voice cannot be HRTF-spatialised per speaker.** WebRTC voice arrives as one
  stereo Opus stream already mixed and panned by the server. The viewer only
  sends its own position over a data channel. Treat voice as a stereo bed unless
  the server changes.
- Media-on-a-prim audio bypasses the engine entirely (CEF plays to the OS, the
  viewer sets a scalar volume). HRTF for MOAP needs plugin audio capture.

So the spatial-audio feature is fully within our control for SFX, ambient and
wind, and out of reach for voice and media without server or plugin work.

### RLV: a policy service with 25 interception points

RLVa intercepts `@` commands on owner-say chat before anything else consumes
them, and then intrudes into 219 files. The architecture report reduces that to
25 interception points grouped as: command ingress and egress; chat and IM;
movement, teleport and sit; world interaction (one `canTouch`/`canSit`/`canEdit`
predicate); appearance and inventory; perception and UI (name and location
presentation services, a floater-open gate); camera and render (position solver,
far-clip clamp, vision-sphere post pass, overlay pass); environment and
settings. Each is a query to one `Policy` trait with refcounted behaviours,
per-UUID exceptions and min/max-composited modifiers, plus change signals. List
in [04 §7.9](04-viewer-architecture.md).

## 3. Transposing the GLSL corpus to Vulkan GLSL

The corpus is already core-style GLSL 1.40+: `in`/`out` everywhere, no
`gl_FragColor`, no `attribute`/`varying`, no `texture2D` outside vendored SMAA
macros, no compute, tessellation, transform feedback, `gl_VertexID`, integer
samplers or derivatives. Exactly one geometry shader exists (debug normals) and
can be dropped. **The hard parts are structural, not syntactic.**

### How programs are built today

Each program is an OpenGL link of 3 to 15 separately compiled objects: the entry
vertex and fragment files plus library files (atmospherics, lighting, shadows,
SSAO, probes, tonemap, skinning) chosen by 20 rules over 21 boolean feature
flags, resolved through 311 forward declarations at link time. Files are looked
up class3 > class2 > class1 with silent fallback. The loader injects a
`#version` between 140 and 420, stage defines, G-buffer flag macros, a struct,
the permutation define map, and synthesises a `diffuseLookup()` function with
`tex0..tex3` for indexed-texture programs.

### The migration, in order of effort

1. **Single translation unit.** SPIR-V stages are one module. Concatenate the
   library files after the entry file in the same order, or emit `#include` and
   use glslang's includer. This immediately exposes duplicate `uniform`
   declarations across files (`modelview_projection_matrix` in 63 files,
   `diffuseMap` in 40), duplicate varyings and duplicate constants. Fix: hoist
   every uniform declaration into a generated header.
2. **Default-block uniforms into descriptor sets.** 292 reserved plus 61 named
   uniforms are set by `glUniform*`. Vulkan has no default block. Suggested
   partition: set 0 per-frame (sky, water, lights, shadow matrices, ~2.5 KB);
   set 1 per-pass textures; set 2 per-material; push constants or a dynamic UBO
   for per-draw matrices. The 110-joint `mat3x4` palette (5.3 KB) becomes an
   SSBO. Because block members are accessed unqualified, shader bodies do not
   change once declarations are hoisted.
3. **Sampler bindings.** Library files declare samplers unconditionally and rely
   on GL dead-sampler elimination. Either give all 76 unique sampler names fixed
   bindings in one set (needs `PARTIALLY_BOUND` or dummy images) or reflect the
   compiled SPIR-V per program. Reflection reproduces today's behaviour exactly.
   Replace the synthesised `diffuseLookup` switch with `sampler2D tex[4]`
   indexed by the flat varying.
4. **Depth range, the one real correctness trap.** `deferredUtil.glsl` lines
   176 to 185 and 307 to 330 reconstruct position with `2*depth-1`, and the C++
   shadow matrix applies a `[-1,1]` to `[0,1]` bias on xyz. Every deferred
   light, soften, haze, SSR, SSAO and water shader calls these. GLM is built
   without `GLM_FORCE_DEPTH_ZERO_TO_ONE`. Fix once in the library, drop the z
   bias in the Rust shadow matrix, and account for the y flip in front-face
   winding. The sky-dome depth hacks (`pos.z = pos.w*0.999991`) survive the
   range change.
5. **Mechanical cleanups.** `#version 450`, `layout(location)` on the 14
   reserved attributes (including the two aliasing tricks: emissive reuses the
   `diffuse_color` slot, `texture_index` is `position.w` as uint) and on the
   fragment outputs, delete four `#extension` lines, rename three `texture2D`
   macros, remove two illegal uniform initialisers (a Poisson table and a node
   id default), turn one never-set `uniform bool` into a constant.
6. **The 49 KB `ReflectionProbes` UBO** exceeds the 16 KB guaranteed minimum.
   Fine on desktop, but an SSBO is the safe target.

Effort by family ([02 §5.2](02-shader-system.md)): about 110 files of interface,
effects, post, objects and shadow passes are mechanical; the atmospherics
library (21 files) is mechanical but central; four utility files (deferredUtil,
shadowUtil, aoUtil, screenSpaceReflUtil) need the depth math reworked; the
vendored FXAA, SMAA and CAS already have Vulkan-portable paths.

### What the C++ shader system does that Rust must reproduce

Of the ~8.8k lines of C++ shader machinery: the program manifest (3.3k lines of
`LLViewerShaderMgr`) becomes a ~600-line declarative table; the loader and
preprocessor (~500 lines) becomes ~200 lines; the uniform setter API and
location caching (~1.2k lines) disappears into struct writes to ring buffers;
texture-unit mapping (~250 lines) is replaced by reflection. What cannot be
reflected is the semantic side: which per-pass textures and values each pipeline
needs (~230 setter sites across 20 files), the feature-to-library rules, and
the variant matrices (material 32, GLTF 16, multi-light 16, fullbright 12,
tonemap 6).

## 4. A visually equivalent render in Vulkan

### The frame, compressed

Full pass list with targets, formats, state and toggles in
[01 §1](01-render-pipeline.md). No MSAA anywhere; no compute; no instancing.

1. Hero (mirror) probe faces, 1024², optional.
2. Sun cascaded shadow maps, 4 splits, depth24, roughly screen resolution,
   perspective best-fit with ortho fallback.
3. Two spot shadow maps.
4. Avatar impostors: mini G-buffers up to 512².
5. **G-buffer fill** in draw-pool order, occlusion queries issued mid-pass.
6. Sun, shadow and SSAO light map (full screen, RGBA16F).
7. Two-pass SSAO blur.
8. **Soften**: sun, IBL, emissive, full screen into the RGBA16F `screen` target.
9. Additive local lights: cube volumes when outside, batched full-screen when
   inside, two shadowed spots.
10. Forward post-deferred pools with injected full-screen water exclusion,
    water haze and atmospheric haze passes, water refraction copy, alpha
    pre-water and post-water (particles live here).
11. Finalize: SSR copy, 256² luminance, 1x1 temporal exposure, ACES or Neutral
    tonemap plus sRGB, CAS, glow extracted from `screen` alpha at 512², DoF at
    0.7 resolution, FXAA or SMAA, dithered present.
12. HUD attachments in ortho, UI, swap.

Reflection probes update one cube face per frame after the main display, up
to 256 probes at 128² R11G11B10F with GGX-prefiltered mips plus 16² irradiance.

G-buffer: RGBA8 diffuse (legacy sRGB, PBR linear, alpha is the fullbright flag),
RGBA8 spec (legacy sRGB spec plus gloss, PBR occlusion-roughness-metallic),
RGBA16 normal (spheremap xy, env intensity in z, a four-value type flag in w),
optional RGB16F emissive (Firestorm default on), shared depth24 with the HDR
screen target whose alpha channel is the glow accumulator.

### Where equivalence is cheap and where it is expensive

**Cheap.** Pass structure, targets, formats, blend states, culling and batching
can be redesigned freely. Bindless, instancing, descriptor indexing and a proper
frame graph are all fine, as long as the per-pixel maths of `softenLightF`, the
point, spot and multi-light shaders, `atmosphericsFuncs`, `reflectionProbeF`,
`shadowUtil`, `tonemapUtilF`, glow, water, sky and clouds, and the forward
alpha, material and PBR shaders is ported literally.

**Expensive.** The look is defined as much by CPU-side policy as by shaders:
sun-colour normalisation (max component clamped to 1), hardware-light
attenuation constants, nearby-light selection and fade, the alpha-mask
promotion rules (`canRenderAsMask`: blended faces become masks at cutoff 0.33
when alpha is 1, no glow, not rigged, not HUD, texture detected binary), face
bucketing into draw passes, LOD selection, probe origin and radius heuristics,
exposure parameters, and the legacy versus EEP versus auto-adjust branching in
sky settings. These must be reimplemented from the reports, not rediscovered.

**Fifteen quirks** that must survive are listed in [01 §8](01-render-pipeline.md).
The ones most likely to be missed: mixed colour spaces in the G-buffer; glow
living in the alpha channel with blend tricks that suppress it under blended
geometry; fullbright encoded as diffuse alpha; the legacy shiny "magic halves";
the gain constants (PBR sun ×3, lights ×3.25, classic-mode 1.35/0.9/1.1,
ambient `pow(x,0.9)*0.57`); tonemap before glow; the sky dome at depth 0.99999
followed by a depth clear; the legacy-sky path running with `tonemap_mix = 0`,
which most content was authored against.

### Raw Vulkan or an RHI

Recommendation: **raw Vulkan through `ash`, with a thin in-house RHI layer
(frame graph, descriptor and residency management) written for this viewer.**

Reasoning:

- The pipeline is about 22 passes, no MSAA, no compute, four UBOs, one SSBO
  candidate, cube arrays and compare samplers. That is well within what a small
  bespoke layer handles, and the layer is where the project's own value lives
  (bindless textures for the 4-textures-per-batch model, descriptor indexing for
  probes, async pipeline creation to hide runtime glslang compiles).
- **Diligent Engine is not recommended.** It is a C++ engine abstracting
  D3D11, D3D12, Vulkan, OpenGL and Metal. The only Rust binding is a hobby
  wrapper at version 0.2.0 (January 2026, under 2k downloads) that needs the C++
  source and install directories at build time and warns that everything is
  prone to change. Adopting it adds a large C++ dependency and a binding
  maintenance burden, which contradicts the goal in section 1, in exchange for
  backends the project does not target.
- **`wgpu` is the credible alternative** if the team prefers a safe, maintained
  Rust RHI over control. It covers the pass structure. Costs: naga is its shader
  front end (so the GLSL caveats above apply unless shaders are fed as SPIR-V
  via the `spirv` feature), bindless is behind optional features, and the
  abstraction hides Vulkan-specific residency and timeline tooling the texture
  budget loop will want.
- `vulkano` (0.35.2, 2025-08) is a safe wrapper with less adoption than `ash`
  and more opinion than the project needs.

### Post-processing

Everything in the finalize chain is plain full-screen GLSL and ports
mechanically: luminance mip chain, 1x1 exposure history, ACES and Neutral
tonemappers, gamma, CAS, the 512² three-target glow blur, DoF with circle of
confusion in alpha at 0.7 resolution, FXAA and SMAA (vendored, with
Vulkan-portable code paths already present). Order matters for equivalence and
is fixed: exposure, tonemap, glow, DoF, anti-aliasing, dither. Depth copies done
today with a fragment shader writing `gl_FragDepth` become image copies.

## Decisions this study settles, and what it leaves open

Settled:

- glslang via the `glslang` crate, runtime compile with a SPIR-V and pipeline
  cache, keeping the existing variant model. Precompile later if specialization
  constants are adopted.
- Three mandatory native libraries (OpenJPEG, libwebrtc, CEF), everything else
  pure Rust. No proprietary components.
- `ash` plus an in-house RHI. Not Diligent.
- The twelve-crate layout with strict downward dependencies and a single
  `Session` owner.
- UI reimplemented, not ported; XUI data files kept.
- Spatial audio is an own mixer with explicit rolloff and HRTF for SFX, ambient
  and wind; voice stays a stereo bed.
- RLVa behaviour table reimplemented exactly behind a `Policy` trait.

Open, needing empirical work against a live grid or a running Firestorm:

- Hollow-prim face index order (code places the inner face before the bottom
  cap; verify against live SL face numbers before hard-coding the mapping).
- Whether `hayro-jpeg2000` can serve the discard-level progressive path or only
  full decodes.
- Per-program active sampler counts at runtime (upper bounds known: 21 samplers
  in PBR terrain, ~15 in forward alpha).
- The exact linked program count (`NORMAL_DEBUG_SHADER_COUNT` and
  `TERRAIN_PAINT_TYPE_COUNT` were not resolved).
- The three `RenderDynamicExposure*` defaults are code-only (true, 0.1, 2.0),
  not in `settings.xml`.
- FMOD's effective rolloff defaults, which define current SL loudness curves,
  come from FMOD documentation rather than viewer code. Measure before choosing
  the new attenuation model.
- Terrain shading internals, the avatar body skinning shader, SMAA lookup
  texture data and local glTF scene rendering were skimmed, not read.
