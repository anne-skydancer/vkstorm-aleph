> Scope note: this report cites `doc/vulkan/*` from the vulkanstorm release branch in a few places. Those documents belong to the separate C++ Vulkan port and are not adopted by Project Aleph; the citations are left in place only as pointers to where the PPLL policy was previously written down.

# Vulkanstorm PPLL order-independent transparency: carry-over study for vkstorm-aleph

Source tree: `C:\Dev\vkstorm-release-ref` at `origin/vkstorm-release` = `08c07b4cba`. All `path:line` citations below are relative to `indra/` in that tree unless the path starts with `doc/` or `scripts/`. The feature entered the release line in commit `82245f3e86` ("Retain PPLL OIT with correctness fixes and upstream alpha fallback"; body: "Exclude depth peeling and rigged/world interleaving, including avatar stamps and comparators."). After that commit only `settings.xml` (line shifts, no OIT key changes), `panel_preferences_graphics1.xml` (tab labels "Rendering 1/2", unrelated backend combo moved) and `scripts/perf/test_ppll_capture.py` (bootstrap file rename, sampler rename to `alpha_oit_opaque_depth`) changed; the C++ and GLSL are identical to the commit. There is no upstream (Firestorm `upstream/master`) counterpart: every OIT line is a Vulkanstorm addition, marked `<FS> Vulkanstorm`.

Related in-tree docs that already discuss this feature for the Vulkan port: `doc/vulkan/source-audit.md:112-118` (PPLL contract), `doc/vulkan/architecture.md:94,98,120-121,131`, `doc/vulkan/component-designs.md:31,69`, `doc/vulkan/shader-contracts.md:13,40,42`, `doc/vulkan/qualification-roadmap.md:17,55,69`. This report is consistent with them and adds the line-level detail.

---

## 1. The algorithm as implemented

### 1.1 Where it sits in the frame

`LLPipeline::renderGeomPostDeferred` (`newview/pipeline.cpp:4352-4490`) walks the draw pools in enum order (`newview/lldrawpool.h:57-78`: ... `POOL_ALPHA_PRE_WATER` (74), `POOL_WATER` (76), `POOL_ALPHA_POST_WATER` (77)). Interleaved full-screen passes: `doWaterExclusionMask()` before `POOL_WATEREXCLUSION`; `doWaterHaze()` before `POOL_ALPHA_PRE_WATER` (`pipeline.cpp:4382, 4420-4424`); `doAtmospherics()` before `POOL_ALPHA_POST_WATER`, or before `POOL_WATER` when `sUnderWaterRender` (`pipeline.cpp:4374-4379, 4414-4418`). Using the upstream study's numbering (`study/render-pipeline.md:146-165`): 10b water haze -> pre-water alpha -> water -> 10c atmospheric haze -> **post-water alpha** -> DoF depth pass -> `renderHighlights()`/`renderDebug()` (`pipeline.cpp:4479-4486`) -> glow/exposure/tonemap/post.

PPLL replaces only the **main-view `POOL_ALPHA_POST_WATER`** forward pass. Everything before (opaque G-buffer, lighting, haze, pre-water alpha, water) and after (highlights, glow extraction, DoF, AA) is unchanged. The haze has already been composited into `screen` before post-water alpha, and each alpha shader applies its own per-fragment sky/water fog (`applySkyAndWaterFog` at `class2/deferred/alphaF.glsl:339`, `class2/deferred/pbralphaF.glsl:248`, `class3/deferred/materialF.glsl:453`, `class1/deferred/fullbrightF.glsl:151`) *before* the OIT append, so captured node colours are already fogged exactly as the sorted path's fragments would be.

### 1.2 Eligibility gate (what goes through PPLL vs the sorted fallback)

`LLDrawPoolAlpha::renderPostDeferred` (`newview/lldrawpoolalpha.cpp:199-278`):

```
oit_view_eligible = POOL_ALPHA_POST_WATER && gPipeline.mRT == &mMainRT
                 && !sRenderingHUDs && !sImpostorRender && !sReflectionRender && !gCubeSnapshot   (:206-212)
use_ppll = RenderAlphaSortMethod == 1 && oit_view_eligible && sRenderAlphaOITSupported          (:215-224)
```

So the following **never** use PPLL and keep the stock two-pass sorted blend (`forwardRender(true)` rigged with depth write, then `forwardRender()`; `:259-277`):

| Excluded | Why / where |
|---|---|
| `POOL_ALPHA_PRE_WATER` (alpha on the far side of the water plane, and HUDs, which only run pre-water) | `:206`; pre-water must write depth for water fog (`:319-325`) |
| HUD attachments | `:209` |
| Avatar impostor renders | `:210` |
| Mirrors / hero reflection probes (`sReflectionRender`) | `:211` |
| Reflection-probe cube snapshots (`gCubeSnapshot`) | `:212` |
| Any auxiliary render target (`mRT != &mMainRT`) | `:207`; resources exist only for the main RT at render resolution (`pipeline.cpp:975-988`) |
| macOS, or GL < 4.4, or (Windows) missing `glBufferStorage`/`glCopyImageSubData` | `pipeline.cpp:7988-8004`, `llrender/llshadermgr.cpp:661-665`, `newview/llviewershadermgr.cpp:3048-3061` |
| Budget/allocation failure | `pipeline.cpp:8023-8033, 8079-8090` (see 1.6) |

2D snapshots are rendered through the main RT, so PPLL applies to them (at a different resolution the buffers are reallocated: `pipeline.cpp:975-988` via `allocateScreenBufferInternal`, and again lazily at `lldrawpoolalpha.cpp:215-220`).

Within an eligible post-water pass (`lldrawpoolalpha.cpp:225-257`) the sequence is:

1. `drawGLTFScene()` (`:227`, `:375-387`): the GLTF-scene alpha-blend objects (`LL::GLTFSceneManager::render(false, ...)` = `ALPHA_BLEND` variant, `newview/gltfscenemanager.cpp:578-596`) are drawn through their own stock path with depth write **on**, before capture. They are **not** captured (no OIT code in the gltf shaders; `doc/vulkan/shader-contracts.md:27` says the same). Their depth lands in the detached snapshot (step 2), so captured fragments behind them are rejected as occluded. Note the blend/colour-mask state here is inherited from the pool loop (`pipeline.cpp:4387-4388` sets BT_ALPHA / colormask(true,false)); in the stock path the same draws occur inside `forwardRender(true)` with colormask(true,true) (`:340-350`).
2. `gPipeline.beginAlphaOITCapture()` (`:228`; `pipeline.cpp:8112-8156`).
3. `setOITMode(1)` (`:230`; `:391-439`): pushes `oit_mode=1`, `oit_node_cap`, and binds the detached depth sampler on `simple_shader`, `fullbright_shader`, `pbr_shader`, their rigged variants, and the four alpha-blend material shader variants `gDeferredMaterialProgram[i]` with `(i & 3) == 1` (`:426-433`).
4. `forwardRender(true, ALPHA_OIT_CAPTURE)` then `forwardRender(false, ALPHA_OIT_CAPTURE)` (`:231-232`): rigged alpha then world alpha, both appended into the **same** per-pixel lists. In capture phase: depth test on, depth write **off** (`:317-329`), colour mask untouched (`:310-314`), no glow replay (`:1043-1046, 1082-1085`), particles and custom-blend batches skipped (`:816-825, 851-862`).
5. `setOITMode(0)`, `endAlphaOITCapture()` (barrier), `compositeAlphaOIT()` (resolve) (`:233-235`).
6. **Rigged depth replay** (`:237-251`): `renderAlpha(..., depth_only=true, rigged=true, ALPHA_OIT_NONE)` with depth test+write on and colour mask off, re-establishing the rigged-alpha depth the stock `forwardRender(true)` would have written.
7. `forwardRender(true, ALPHA_OIT_RESIDUAL)` and `forwardRender(false, ALPHA_OIT_RESIDUAL)` (`:253-256`): stock sorted blending of the residual set, plus the glow replay for captured faces.
8. DoF depth-only pass, unchanged (`:280-298`, minimum alpha 0.33, colour mask off).

**Captured set** (per draw, `lldrawpoolalpha.cpp:851-862`): every `LLDrawInfo` in the sorted post-water alpha groups whose blend func is exactly `(SRC_ALPHA, ONE_MINUS_SRC_ALPHA)` and whose spatial partition is not `PARTITION_PARTICLE`/`PARTITION_HUD_PARTICLE`. That covers legacy lit alpha (`gDeferredAlphaProgram` -> `class2/deferred/alphaF.glsl`), fullbright alpha (`gDeferredFullbrightAlphaMaskAlphaProgram` -> `class1/deferred/fullbrightF.glsl` with `IS_ALPHA`), legacy materials with blend alpha mode (`class3/deferred/materialF.glsl`, `DIFFUSE_ALPHA_MODE_BLEND`), and GLTF PBR alpha-blend faces on prims/avatars (`gDeferredPBRAlphaProgram` -> `class2/deferred/pbralphaF.glsl`), rigged or not.

**Residual set** (stock path, drawn *after* the resolve): particle partitions (`:816-825`, "must never consume PPLL nodes, even when their batches happen to be fullbright") and any batch with a custom `mBlendFuncSrc/Dst` (additive, etc.; `:854-857`). The residual pass also handles, for *captured* faces with a `TYPE_EMISSIVE` channel, the additive glow draw only (`:865-882`): they are pushed onto the emissive lists and `continue`d so their RGB is not blended twice.

**Rigged/world interleaving**: the commit message says avatar "stamps and comparators" (an archived mechanism to interleave rigged and world alpha draws by depth) were excluded; no such code exists in the tree (grep for `stamp|comparator|interleav` finds only the upstream glow comment at `:1042`). With PPLL this is moot for the captured set: rigged and world fragments land in the same per-pixel list and are depth-sorted together in the resolve. Depth peeling is likewise absent: `fullbrightF.glsl:63-89` still carries `oit_mode` 2/3/4 code under `ALPHA_DEPTH_PEEL`, but that macro is never defined anywhere in `indra/` and the CPU only sets modes 1/0 (`lldrawpoolalpha.cpp:230,233`); `doc/vulkan/shader-contracts.md:42` classifies it as dormant.

### 1.3 Per-fragment node layout

`oit_append(vec4 c, float z)` is textually identical in the four capture shaders (`alphaF.glsl:52-66`, `pbralphaF.glsl:44-58`, `materialF.glsl:48-62`, `fullbrightF.glsl:46-60`):

```glsl
if (!(z <= texelFetch(alpha_oit_opaque_depth, ivec2(gl_FragCoord.xy), 0).r)) discard;  // occluded by opaque/GLTF depth
uint idx  = atomicCounterIncrement(oit_counter);
if (idx >= uint(oit_node_cap)) return false;                                         // pool exhausted -> caller blends normally
uint prev = imageAtomicExchange(oit_head, ivec2(gl_FragCoord.xy), idx);
uint base = idx * 4u;
oit_nodes[base+0] = packHalf2x16(max(c.rg, vec2(0.0)));
oit_nodes[base+1] = packHalf2x16(vec2(max(c.b, 0.0), clamp(c.a, 0.0, 1.0)));
oit_nodes[base+2] = floatBitsToUint(z);
oit_nodes[base+3] = prev;
return true;
```

Node = **16 bytes = 4 x uint32** (`pipeline.cpp:8012` `NODE_BYTES = 4*sizeof(U32)`; `pipeline.h:778`):

| word | content |
|---|---|
| 0 | R,G as two IEEE half floats (`packHalf2x16`), negatives clamped to 0; HDR linear scene colour, **straight (non-premultiplied)** |
| 1 | B half + alpha half, alpha clamped to [0,1] |
| 2 | window-space depth `gl_FragCoord.z` as raw float bits (standard GL 0 = near, 1 = far; no reverse-Z) |
| 3 | previous head index (`0xFFFFFFFF` terminates) |

There are no glow/emissive/material bits in the node. The colour is the shader's final `frag_color` (after lighting, fog, `final_scale` classic-mode tweak and `max(..., 0)`), i.e. exactly what the sorted path would have blended (`alphaF.glsl:350-354`, `pbralphaF.glsl:254-257`, `materialF.glsl:461-464`, `fullbrightF.glsl:157-160`). The call site is `if (oit_mode == 1 && oit_append(oit_out, gl_FragCoord.z)) { discard; } frag_color = oit_out;` so a successful append produces **no framebuffer write at all**, and a failed append (pool full) falls through to ordinary source-over blending with the pass's current blend state.

Alpha/minimum-alpha test: the shaders' existing `discard` on `alpha < minimum_alpha` (0.004, `lldrawpoolalpha.cpp:64,130`) precedes the append, so near-invisible fragments do not consume nodes.

### 1.4 Head image, node pool, counter, depth snapshot (allocation)

`LLPipeline::allocateAlphaOITBuffers(w,h)` (`pipeline.cpp:7986-8101`), called from `allocateScreenBufferInternal` for the main RT when `RenderAlphaSortMethod == 1` (`:975-988`) and lazily from the pool (`lldrawpoolalpha.cpp:215-220`). Released from `releaseScreenBuffers` (`pipeline.cpp:1396-1398`) and `releaseAlphaOITBuffers` (`:8103-8110`).

| Resource | GL object | Details |
|---|---|---|
| Head image `mAlphaOITHead` | `GL_TEXTURE_2D`, `glTexStorage2D(GL_R32UI, w, h)`, NEAREST/CLAMP | `:8049-8059`; bound as image unit 0 `GL_READ_WRITE` for capture (`:8147`), `GL_READ_ONLY` for resolve (`:8215`); cleared each frame to `0xFFFFFFFF` with `glClearTexImage` (`:8140-8141`) |
| Node pool `mAlphaOITNodes` | `GL_SHADER_STORAGE_BUFFER`, immutable `glBufferStorage(size, nullptr, flags=0)` (device-only) | `:8062-8067`; SSBO binding 0; size `capacity * 16` |
| Allocator `mAlphaOITCounter` | `GL_ATOMIC_COUNTER_BUFFER`, one `uint`, `GL_DYNAMIC_DRAW` | `:8071-8075`; atomic binding 0; zeroed each frame with `glBufferSubData` (`:8135-8138`) |
| Depth snapshot `mAlphaOITOpaqueDepth` | `LLRenderTarget` depth-only (`allocate(w,h,0,true)` -> `GL_DEPTH_COMPONENT24`, `llrender/llrendertarget.cpp:105,307`) | `:8078`; filled each frame by `glCopyImageSubData` from `mRT->deferredScreen` depth (`:8124-8128`) |

Binding model: all three resources use **binding 0 in three different GL namespaces** (image unit 0, SSBO index 0, atomic-counter index 0; `alphaF.glsl:48-50`, `alphaOITResolveF.glsl:38-39`). `ALPHA_OIT` is injected into *every* fragment shader compiled on GL >= 4.4 (`llrender/llshadermgr.cpp:661-665`), but only the four alpha shaders and the resolve contain `#ifdef ALPHA_OIT` blocks. The detached-depth sampler is a reserved uniform `alpha_oit_opaque_depth` (`llshadermgr.cpp:1577-1580`, enum `ALPHA_OIT_OPAQUE_DEPTH` at `llshadermgr.h:363`, whose trailing comment still says `"alpha_peel_depth"`), and is re-bound on every shader switch inside `renderAlpha` because texture units are global state (`lldrawpoolalpha.cpp:908-915, 975-981`).

### 1.5 Memory budget maths

`pipeline.cpp:8007-8036`:

```
pixels     = w*h
requested  = pixels * clamp(RenderAlphaOITNodesPerPixel, 1, 32)          // nodes
budget     = clamp(RenderAlphaOITMemoryMB, 32, 2048) MiB
driver     = GL_MAX_SHADER_STORAGE_BLOCK_SIZE (GLint64)
capacity   = min(requested, min(budget, driver) / 16)
pixel_layers = clamp(RenderAlphaOITMaxPixelLayers, 4, 32)
if capacity < pixels or capacity > 0x7fffffff -> disable (see 1.6)
```

Defaults (`settings.xml:11785-11820`): 8 nodes/px, 512 MiB, 24 layers. Worked numbers (arithmetic, not measured; agrees with `doc/vulkan/source-audit.md:114`):

| Resolution | pixels | 8 nodes/px = bytes | vs 512 MiB budget | head image (R32UI) | depth copy (D24) |
|---|---|---|---|---|---|
| 1920x1080 | 2,073,600 | 16,588,800 nodes = 253.1 MiB | fits; capacity = 16.6 M nodes | 7.9 MiB | ~7.9-8.3 MiB |
| 2560x1440 | 3,686,400 | 29,491,200 nodes = 450.0 MiB | fits | 14.1 MiB | ~14 MiB |
| 3840x2160 | 8,294,400 | 66,355,200 nodes = 1012.5 MiB | capped at 33,554,432 nodes = 512 MiB = **4.05 nodes/px** | 31.6 MiB | ~32 MiB |

The driver limit matters: `GL_MAX_SHADER_STORAGE_BLOCK_SIZE` is 2^31-1 (or more) on the major desktop drivers, but some Mesa drivers have historically reported 128 MiB, which would cap capacity at 8,388,608 nodes (4.0/px at 1080p; 1.01/px at 4K, barely passing the >= 1 node/px floor). The `0x7fffffff` ceiling exists because `oit_node_cap` is pushed as a GLSL `int` (`lldrawpoolalpha.cpp:395`).

`RenderAlphaOITMaxPixelLayers` does not affect allocation; it is the resolve's exact-sort depth (`oit_max_pixel_layers` uniform, `pipeline.cpp:8211-8212`), bounded by the shader's `OIT_MAX_LAYERS 32` arrays (`alphaOITResolveF.glsl:46,61-63`).

### 1.6 Overflow and failure behaviour

* **Global pool exhaustion** (`idx >= oit_node_cap`): the fragment is *not* lost; `oit_append` returns false and the shader writes `frag_color`, which blends source-over in the capture pass's inherited state (`pipeline.cpp:8151-8154` comment; `lldrawpoolalpha.cpp:851` onward). Because the CPU still submits alpha groups in sorted order, overflow degrades to legacy ordering for the fragments past the cap, which are blended *before* the resolve composites the captured lists over them. The counter keeps incrementing past the cap (no clamp), which is harmless for a 32-bit counter at realistic fragment counts.
* **Per-pixel overflow** beyond `oit_max_pixel_layers`: handled in the resolve by a weighted tail (1.7).
* **Allocation refused** (capacity < 1 node/px, or > 2^31-1): warning, buffers released, `sRenderAlphaOITSupported=false`, `RenderAlphaPPLLAvailable=true`, and **`RenderAlphaSortMethod` is written back to 0** (`pipeline.cpp:8023-8033`). The same happens on a GL allocation error or a short `GL_BUFFER_SIZE` (`:8079-8090`). Note this contradicts the preference panel text "Unsupported methods safely use legacy sorting without changing your saved choice" (`panel_preferences_graphics1.xml:1803-1806`); on a *capability* failure (GL < 4.4, macOS) the setting is left alone (`:7999-8004`), on a *budget* failure it is overwritten.
* **Capture refused at frame time** (`beginAlphaOITCapture` returns false when any resource, the resolve program, or the size match is missing; `pipeline.cpp:8112-8120`): the pool falls back to the stock two-pass path for that frame (`lldrawpoolalpha.cpp:259-265`).

### 1.7 The resolve pass

`LLPipeline::compositeAlphaOIT()` (`pipeline.cpp:8171-8238`) draws one full-screen triangle (`mScreenTriangleVB`, `:8218-8219`) with `gAlphaOITResolveProgram` (`postDeferredNoTCV.glsl` + `class1/deferred/alphaOITResolveF.glsl`; `llviewershadermgr.cpp:3048-3061`) onto the still-bound `mRT->screen` (RGBA16F HDR, alpha = glow). State: depth test off, blend on, **colour mask (true,false)**, blend `(ONE, ONE_MINUS_SRC_ALPHA)` (`:8203-8207`). Inputs: head image read-only on image unit 0, node SSBO on binding 0, and the *same detached depth snapshot* as `depthMap` (`:8214-8216`).

Shader (`alphaOITResolveF.glsl:49-146`), per pixel:

1. `opaque_z = texelFetch(depthMap, coord)`; `idx = imageLoad(oit_head, coord)`; `discard` if empty (`:51-59`).
2. **Walk 1** (`:61-83`): follow `next` pointers collecting up to `min(32, oit_max_pixel_layers)` nodes with `nz <= opaque_z` (LEQUAL guard, same test as capture) into three local arrays (packed RG, packed BA, depth), bounded by `OIT_MAX_TRAVERSAL = 256` total hops to make corrupt lists terminate.
3. **Walk 2** (`:85-106`): the remaining nodes (still within 256 hops) are folded, order-independently, into `tail_wsum += rgb*a`, `tail_asum += a`, `tail_trans *= (1-a)`.
4. **Insertion sort** of the collected layers, *descending* window depth, i.e. back-to-front (`:108-126`). O(n^2) with n <= 32; three parallel arrays are moved.
5. **Composite** (`:128-143`): start from the tail as a single farthest layer with colour `tail_wsum/tail_asum` and coverage `1 - tail_trans`; then premultiplied over for each sorted layer: `accum = c.rgb*c.a + accum*(1-c.a)`, `cov = c.a + cov*(1-c.a)`.
6. Output `vec4(accum, cov)` (premultiplied). With the fixed-function blend above: `screen.rgb = accum + screen.rgb*(1-cov)`; `screen.a` untouched.

Because the linked list's head is the most recently appended node and the CPU still submits groups back-to-front, walk 1 collects the *last-drawn* (nearest-ish) layers and the tail is the *earliest-drawn* (farthest-ish) ones, so placing the tail behind the sorted layers is a reasonable approximation rather than arbitrary.

**Glow-in-alpha convention.** `screen.a` carries the emissive/glow signal consumed by `generateGlow`. The resolve masks alpha off so the existing glow survives (`pipeline.cpp:8201-8203`, `alphaOITResolveF.glsl:31-34`). The stock alpha pass uses alpha blend factors `(ZERO, ONE_MINUS_SRC_ALPHA)` to *attenuate* glow under alpha surfaces ("glow suppression", `lldrawpoolalpha.cpp:334-336`); captured fragments do **not** do this (see 2.2). Glow *emission* from captured faces is re-added in the residual phase via the emissive draws with blend `(ZERO, ONE, ONE, ONE)` (`:1082-1123`), which require `applyModelMatrix` on each emissive draw because the face's normal batch may not have been drawn in that phase (`:666-668, 696, 744`).

After the draw, `compositeAlphaOIT` restores colour mask `(true,false)` and blend `(SRC_ALPHA, ONE_MINUS_SRC_ALPHA)` because `POOL_ALPHA_POST_WATER` is the last pool and `renderHighlights()` inherits GL state without setting its own (`:8227-8237`).

**Water / haze / underwater / DoF interaction.** Water (`POOL_WATER`) has already been drawn and `doAtmospherics` already blended the haze into `screen` before this pool; node colours are per-fragment fogged (1.1). Underwater flips `water_sign` and `above_water` (`lldrawpoolalpha.cpp:152-161, 766-771`) and moves `doAtmospherics` before `POOL_WATER` (`pipeline.cpp:4376-4379`) but does not change the OIT flow: the post-water pool is still the one captured, now holding the geometry on the camera's (underwater) side. Depth of field: the stock depth-only pass at `lldrawpoolalpha.cpp:280-298` still runs after the OIT block and writes >= 33%-opaque alpha faces into the shared scene depth; the rigged replay (`:240-251`) additionally restores rigged alpha depth that the stock rigged pass would have written. The capture itself never writes scene depth (`:317-329`).

### 1.8 Synchronisation in the GL implementation

`beginAlphaOITCapture` (`pipeline.cpp:8112-8156`): copy depth -> `glMemoryBarrier(BUFFER_UPDATE | TEXTURE_UPDATE)` (orders the previous frame's shader writes before the API resets, even when reusing unchanged allocations) -> zero counter -> clear head -> `glMemoryBarrier(SHADER_IMAGE_ACCESS | ATOMIC_COUNTER)` -> bind image/SSBO/atomic -> colour mask (true,true). `endAlphaOITCapture` (`:8158-8168`): `glMemoryBarrier(SHADER_IMAGE_ACCESS | SHADER_STORAGE | ATOMIC_COUNTER)`. The optional stats readback issues `BUFFER_UPDATE` and a synchronous `glGetBufferSubData` (`:8179-8200`). No `layout(early_fragment_tests)` is declared anywhere; the shader-side depth test against the detached snapshot is what prevents occluded fragments from allocating nodes (GL must run side-effecting fragment shaders even if the late depth test would fail), and the detached copy avoids sampling the currently attached depth buffer (`:8077`, `doc/vulkan/architecture.md:131`).

### 1.9 Fallback path and requirements

Requirements as coded: `gGLManager.mGLVersion >= 4.4f` on Windows/Linux (`pipeline.cpp:979, 7990`; `llshadermgr.cpp:662`; `llviewershadermgr.cpp:3049`), i.e. core SSBO (4.3), image load/store (4.2), atomic counters (4.2), `glClearTexImage` and `glBufferStorage` (4.4), `glCopyImageSubData` (4.3); Windows additionally verifies the two 4.4/4.3 entry points were resolved (`:7995-7997`). Shaders declare `GL_ARB_shader_storage_buffer_object`, `GL_ARB_shader_image_load_store`, `GL_ARB_shader_atomic_counters` with `: enable` (warn-only) (`alphaF.glsl:28-30`, `alphaOITResolveF.glsl:26-27`). `mGLVersion` is `major + minor*0.1f` (`llrender/llgl.cpp:1132`) and the comparison against `4.4f` happens to be exact in float, but `llgl.cpp:1467` warns such comparisons should bias down by 0.01.

When unsupported or off (`RenderAlphaSortMethod == 0`, the default; `settings.xml:11773-11783`): `ALPHA_OIT` is undefined (or `oit_mode` stays 0), the resolve program is not built, no buffers are allocated, and `renderPostDeferred` runs the upstream path exactly: `forwardRender(true)` (rigged, depth write, GLTF scene to depth) then `forwardRender()` (`lldrawpoolalpha.cpp:267-277`).

---

## 2. Correctness fixes and known limitations

### 2.1 Fixes carried in commit 82245f3e86 (relative to the archived pre-reset implementation, per code comments)

1. **Detached opaque-depth snapshot** shared by capture and resolve, taken *after* the GLTF alpha depth draw and never the draw attachment (`pipeline.cpp:8077-8078, 8124-8128, 8214-8216`; test asserts it is unaffected by later writes to the live depth, `scripts/perf/test_ppll_capture.py:59-62`).
2. **LEQUAL parity incl. equality** between the capture reject and the resolve guard; reject happens *before* the allocator so hidden fragments consume no node, and `discard` keeps them off the overflow blend path (`alphaF.glsl:54-56`; test at `test_ppll_capture.py:70-80` uses a depth exactly equal to the stored one).
3. **Capture never writes scene depth**, and the rigged depth contribution is replayed after the resolve so avatar-linked glow is depth-tested properly and DoF/residual see avatars (`lldrawpoolalpha.cpp:237-251`, `:317-329`).
4. **GLTF scene drawn to depth with depth write explicitly enabled** in both paths (`:340-350`, `:375-387`).
5. **Particles and custom-blend batches excluded** from capture; **captured faces' glow re-added** in residual; `applyModelMatrix` added to the three emissive draw helpers (`:816-825, 851-882, 666-668, 696, 744`).
6. **Depth sampler re-bound on every shader transition** (`:908-915, 975-981`) and registered as a reserved uniform so `bindTexture` works (`llshadermgr.cpp:1577-1580`).
7. **Memory barriers before API resets and after capture**, including the allocation-reuse case (`pipeline.cpp:8130-8144, 8163-8165`).
8. **Immutable, device-only node pool** (`glBufferStorage`, flags 0) with allocation verified via `GL_BUFFER_SIZE` and `glGetError` (`:8062-8090`; test asserts immutability, `test_ppll_capture.py:37`).
9. **Budget against driver `GL_MAX_SHADER_STORAGE_BLOCK_SIZE`**, >= 1 node/px floor, 2^31-1 ceiling (`:8014-8033`).
10. **Resolve bounds**: fixed 32-slot arrays, user cap 4..32, 256-hop traversal cap, order-independent weighted tail instead of dropping layers (`alphaOITResolveF.glsl:43-47, 85-106`).
11. **GL state restoration after resolve** for the selection highlight (`pipeline.cpp:8227-8237`).
12. **Stats readback gated** behind `RenderAlphaOITStats` (`:8179-8200`).
13. **Exclusions**: depth peeling (dormant modes 2/3/4), rigged/world interleaving, avatar stamps/comparators (commit message; absent from tree).
14. Build/staging: shaders and XUI staged next to the dev executable so stale preference panels do not ship (`newview/CMakeLists.txt:2346-2347`, `viewer_manifest.py:96-110`).

### 2.2 Known limitations (visible behaviour the port must either reproduce or consciously change)

* **Residual alpha is always on top.** Particles and custom-blend faces are blended *after* the resolve, depth-tested only against opaque + GLTF-alpha + rigged-replay depth. A particle behind a captured world alpha surface still draws over it. (Stock path sorts them together.)
* **Glow suppression lost under captured alpha.** The resolve writes RGB only, so `screen.a` (glow) under captured surfaces is not attenuated as the stock `(ZERO, 1-SRC_ALPHA)` alpha blend would do; and captured faces' own glow is added afterwards without z-ordering relative to other captured layers.
* **Overflow is approximate.** Pool exhaustion reverts to draw-order blending beneath the resolved lists; per-pixel overflow beyond the layer cap is a WBOIT-style average placed behind the sorted layers. Both are deterministic given draw order but are not exact OIT (`doc/vulkan/source-audit.md:116`).
* **Precision.** Node colour is FP16 with negatives clamped; alpha clamped to [0,1]; depth stored as full float but compared with `<=` against a D24 snapshot sampled as float.
* **Pre-water alpha, GLTF-scene alpha, HUD, impostors, mirrors/hero probes, reflection probes** keep sorted blending (1.2). GLTF-scene alpha even occludes captured fragments behind it because its depth is in the snapshot.
* **Shadows** are unaffected (alpha shadow casting uses the shadow pools, not this pass).
* **Settings side effect**: budget failure persists `RenderAlphaSortMethod=0` (1.6).
* **Memory**: one set of buffers for the main RT only, reallocated on resolution change (e.g. high-res snapshots), ~253 MiB + 16 MiB at 1080p default.
* **Binding namespace**: all three OIT resources at "binding 0" (`doc/vulkan/shader-contracts.md:13`); fine in GL, must be separated in Vulkan.
* The `ALPHA_OIT` macro is added to *all* fragment shaders on GL 4.4+, so any shader that later gains an `#ifdef ALPHA_OIT` block silently joins the feature.

---

## 3. Settings, UI, defaults, perf script

Settings (`newview/app_settings/settings.xml`):

| Key | Type | Default | Persist | Role |
|---|---|---|---|---|
| `RenderAlphaSortMethod` | U32 | 0 | yes | 0 upstream sorted, 1 PPLL (`:11773-11783`) |
| `RenderAlphaOITNodesPerPixel` | U32 | 8 | yes | average node budget per pixel, clamped 1..32 (`:11785-11795`) |
| `RenderAlphaOITMemoryMB` | U32 | 512 | yes | node pool budget, clamped 32..2048 MiB, excludes head image (`:11797-11807`) |
| `RenderAlphaOITMaxPixelLayers` | U32 | 24 | yes | exact-sort layer cap, clamped 4..32 (`:11809-11819`) |
| `RenderAlphaPPLLAvailable` | Boolean | false | no | runtime capability flag for the preferences panel (`:11823-11833`) |
| `RenderAlphaOITStats` | Boolean | false | yes | per-frame counter readback + log every 64 frames; forces a GPU sync (`:11836-11846`; `pipeline.cpp:8179-8200`) |

UI (`newview/skins/default/xui/en/panel_preferences_graphics1.xml`): the Graphics preferences gained a "Rendering 2" tab (`:1776-1903`) with a radio group "Alpha sorting method" (Upstream sorted alpha / "PPLL (Windows/Linux, OpenGL 4.4+)", `:1809-1835`, commit callback `Pref.RenderOptionUpdate`), descriptive text listing the exclusions (`:1795-1807`), and spinners for memory budget (32..2048 step 16), nodes per pixel (1..32), max layers (4..32), plus the stats checkbox (`:1848-1901`). The original tab was renamed "Rendering 1" (`:1274`; Italian skin updated likewise). Defaults are therefore: feature **off**; when on, 8 nodes/px, 512 MiB, 24 layers.

Perf/regression script `scripts/perf/test_ppll_capture.py` (84 lines): an offscreen Windows GL harness (`gl_test_context.py` bootstrap, optional `--opengl PATH` to use Mesa) that extracts the real `bool oit_append(...)` text from each of the four alpha shaders, wraps it in a minimal fragment shader, and over a 3x1 framebuffer verifies: detached depth copy is independent of the live depth attachment (`:55-62`), LEQUAL equality (fragment at exactly the stored depth is kept, `:70-72`), occlusion (hidden fragment allocates no node, `:79`), overflow (cap 1 vs 8 leaves `min(cap, 2)` heads set, `:80`), counter reset between runs, immutable pool (`:37`), and no GL errors. Prints `N PPLL cases passed` (`:83`). It is a correctness test of the append helper, not a frame-time benchmark.

---

## 4. Vulkan translation for vkstorm-aleph

### 4.1 Construct-by-construct mapping

| GL construct (where) | Vulkan equivalent | Notes |
|---|---|---|
| `GL_R32UI` head texture, `glBindImageTexture(..., GL_READ_WRITE)` (`pipeline.cpp:8049-8059, 8147`) | `VkImage` `VK_FORMAT_R32_UINT`, usage `STORAGE | TRANSFER_DST`, layout `GENERAL` for capture and resolve; descriptor `STORAGE_IMAGE` | `R32_UINT` is required to support `STORAGE_IMAGE_ATOMIC` (needed for `imageAtomicExchange`); check `VkFormatProperties` anyway. Feature `fragmentStoresAndAtomics` is mandatory for any store/atomic from a fragment shader. |
| `glClearTexImage(0xFFFFFFFF)` (`:8140-8141`) | `vkCmdClearColorImage` with `uint32[4] = {~0u,...}` in `TRANSFER_DST_OPTIMAL`, then barrier to `GENERAL` | Alternative: clear in the same compute/graphics pass that resets the counter. |
| SSBO node pool, `glBufferStorage(flags=0)` (`:8062-8067`) | `VkBuffer` usage `STORAGE_BUFFER | TRANSFER_DST`, `DEVICE_LOCAL`; `std430 buffer { uint nodes[]; }` | Size <= `VkPhysicalDeviceLimits::maxStorageBufferRange` (replaces `GL_MAX_SHADER_STORAGE_BLOCK_SIZE`; spec minimum is 128 MiB, desktop drivers report 2^31-1 or 2^32-1). Consider `uvec4 nodes[]` for one 16-byte load per node. |
| `atomic_uint oit_counter` + `atomicCounterIncrement` (`alphaF.glsl:50,57`) | Vulkan GLSL has **no atomic counters**: a `uint` in a small storage buffer (or the first 16 bytes of the node pool) with `atomicAdd(counter, 1u)` | Reset per frame with `vkCmdFillBuffer` (transfer). `doc/vulkan/architecture.md:131` already prescribes this. |
| `glBufferSubData` zeroing (`:8135-8138`) | `vkCmdFillBuffer` | |
| Detached depth: `glCopyImageSubData` D24->D24 (`:8124-8128`) + `texelFetch` in capture/resolve | Either `vkCmdCopyImage` depth->depth (same format, `TRANSFER_SRC/DST`, then `SHADER_READ_ONLY_OPTIMAL`), **or** drop the copy: bind the scene depth as a read-only depth attachment (`DEPTH_READ_ONLY_OPTIMAL` / `DEPTH_STENCIL_READ_ONLY_OPTIMAL`) and sample the same image in the fragment shader, which Vulkan permits when depth writes are off | Capture does not write depth; GLTF alpha depth is written in a *preceding* rendering, rigged replay in a *following* one, so there is no in-pass hazard. |
| Shader-side depth reject (no `early_fragment_tests`) (`alphaF.glsl:56`) | `layout(early_fragment_tests) in;` in the capture fragment shader with pipeline depth compare `LESS_OR_EQUAL`, depth write off | Vulkan then guarantees occluded fragments never execute (no node allocated), reproducing the exact LEQUAL guard without any depth sampling, and the pool-overflow colour write still works for fragments that pass. Keep the sampled-depth variant behind a debug toggle for A/B parity, since `architecture.md:131` prefers the explicit sample. Minimum-alpha `discard` still precedes the append in shader order, so it keeps working. |
| `discard` after a successful append (`alphaF.glsl:353`) | identical in SPIR-V (`OpKill`/`OpTerminateInvocation`); with `early_fragment_tests` the discard no longer blocks depth writes, which is irrelevant because depth write is off in capture | |
| `packHalf2x16/unpackHalf2x16`, `floatBitsToUint` | same GLSL built-ins (GLSL.std.450) in Vulkan GLSL; no `shaderFloat16` feature needed | |
| `coherent uimage2D` + `imageAtomicExchange` | same; SPIR-V `OpAtomicExchange` on an `R32ui` image, `Coherent` decoration or Vulkan memory model | Within the capture pass nothing reads nodes, so only atomicity of the head exchange matters. |
| `glMemoryBarrier(...)` before capture (`:8130-8144`) | `vkCmdPipelineBarrier2`: (a) execution+memory barrier from last frame's resolve `FRAGMENT_SHADER / SHADER_STORAGE_READ` to `TRANSFER / TRANSFER_WRITE` for head, counter (WAR); (b) `TRANSFER / TRANSFER_WRITE` -> `FRAGMENT_SHADER / SHADER_STORAGE_READ|WRITE` for head (with layout `TRANSFER_DST_OPTIMAL -> GENERAL`), counter, and the depth copy (`TRANSFER_WRITE -> SHADER_SAMPLED_READ`, layout to `SHADER_READ_ONLY_OPTIMAL`) if the copy is kept | On a single in-order graphics queue this also makes **one** node pool safe across frames in flight; no per-slot duplication (`architecture.md:98`). |
| `glMemoryBarrier` after capture (`:8163-8165`) | `vkCmdPipelineBarrier2` `FRAGMENT_SHADER / SHADER_STORAGE_WRITE` -> `FRAGMENT_SHADER / SHADER_STORAGE_READ` for the node buffer and head image (image stays `GENERAL`) | Must be **outside** a render pass instance (or a non-by-region subpass self-dependency, which gives tilers nothing). |
| Capture draw state (`LLGLSPipelineAlpha`, `LLGLDepthTest(TRUE,FALSE)`, colour mask inherited, blend source-over) | Graphics pipeline: depth test on `LESS_OR_EQUAL`, depth write off, blend enabled `SRC_ALPHA/ONE_MINUS_SRC_ALPHA` on RGB (overflow path only), colour write mask RGB, per-material cull/double-sided | Capture, resolve, rigged depth replay and residual are four `vkCmdBeginRendering` scopes on `screen` (RGBA16F, `loadOp LOAD`) and the scene depth. |
| Resolve full-screen triangle, blend `(ONE, ONE_MINUS_SRC_ALPHA)`, mask (true,false) (`:8203-8219`) | Fragment pipeline, no depth attachment, `VkPipelineColorBlendAttachmentState { srcColor ONE, dstColor ONE_MINUS_SRC_ALPHA, colorWriteMask R|G|B }`, vertex-less `vkCmdDraw(3)` | Could instead be a compute pass doing the "over" with `imageLoad/imageStore` on `screen`, but the fragment+fixed-function blend keeps glow (`screen.a`) read-only for free and matches GL bit-for-bit in the blend. |
| Three "binding 0" resources (`alphaF.glsl:48-50`) | Distinct bindings in one OIT descriptor set, e.g. set 2: 0 = head storage image, 1 = node SSBO, 2 = counter SSBO, 3 = (optional) depth sampler; resolve reuses the set with `readonly` | `shader-contracts.md:13` flags this explicitly. |
| `oit_mode`, `oit_node_cap`, `oit_max_pixel_layers` uniforms | push constants or a small UBO; or specialise two pipeline variants (capture vs normal) and keep only `node_cap` dynamic | Specialising removes the `oit_mode` branch from every alpha shader. |
| Stats readback (`glGetBufferSubData`) | `vkCmdCopyBuffer` counter -> host-visible ring buffer, read `frames_in_flight` later | No sync stall. |
| Depth convention | GL window z 0..1 near..far and `<=`; `component-designs.md:31` keeps standard depth. If vkstorm-aleph adopts reverse-Z, flip to `>=` in the guard and sort *ascending* in the resolve | Resolve sort comment at `alphaOITResolveF.glsl:108-109`. |

Resource lifetime: head image + node pool + counter sized to the main render resolution; re-create on resize; `maxStorageBufferRange` and a user MiB budget bound the node count exactly as 1.5; keep the >= 1 node/px floor and the fallback to sorted blending.

### 4.2 Alternatives the Rust renderer could choose

| Technique | Visual relation to shipped PPLL | Memory (1080p) | Cost / requirements | Verdict |
|---|---|---|---|---|
| **PPLL as shipped** (R32 head, 16 B nodes, K<=32 insertion sort, weighted tail, draw-order overflow, residual particles) | Reference; bit-level reproducible given same policy and FP16 packing | 253 MiB nodes @8/px + 8 MiB head (+ 8 MiB depth copy if kept) | One geometry pass + full-screen resolve; atomics contention on counter/head; O(K^2) sort per covered pixel; `fragmentStoresAndAtomics` + R32_UINT atomic format | **Recommended parity target** |
| **Fixed-k A-buffer / k-buffer** (k slots per pixel in a 2D array or linear buffer, insertion with pixel-local ordering) | Exact up to k like PPLL's walk-1; no global overflow (bounded memory), but needs either `VK_EXT_fragment_shader_interlock` (not universal; AMD exposure has lagged) for race-free insertion, or atomics on depth-packed keys (lossy) | k=8: 8x16 B x 2.07 M = 265 MiB (same as PPLL @8/px) but allocated per pixel, so no overflow fallback needed | Cheaper resolve (slots already pixel-local), no counter contention | Good later optimisation where interlock exists; not a drop-in equivalent (per-pixel truncation differs from PPLL's global budget + tail) |
| **Weighted-blended OIT** (McGuire-Bavoil) | Approximate: order-independent weighted average, like PPLL's *tail* applied to *every* layer; wrong for high-alpha surfaces and saturated colour stacks (SL alpha clothing, windows) | 2 extra targets (RGBA16F accum + R16F revealage) ~ 12 MiB | Single pass, no atomics, no sort; cheapest | Not visually equivalent; viable for the *particle* residual set only |
| **Moment-based OIT** (Munstermann et al.) | Approximate but much closer than WBOIT; transmittance from 4 power moments; artefacts on sharp high-contrast stacks | 2 passes over alpha geometry (moments, then composite) + RGBA32F/RGBA16F moment targets ~ 24-50 MiB | Two geometry passes, float blending, no atomics | Attractive bounded-memory middle ground, but changes output; would need its own qualification |
| **Depth peeling** (front or dual) | Exact for n peels; n passes over all alpha geometry | 2-3 depth targets | Geometry cost x n; SL scenes with thousands of alpha batches make n>=4 prohibitive | Explicitly excluded by Vulkanstorm (commit message); do not pursue |
| **Sorted blending only** (upstream) | The existing fallback; per-object sort, intra-object and interpenetration errors | 0 | Cheapest geometry pass | Required anyway as the fallback and for pre-water/HUD/impostor/probe views |

### 4.3 Recommendation

1. Implement the shipped PPLL policy as the first-tier target, because (a) the sorted path must exist regardless (pre-water, HUD, impostor, probe, mirror views and the unsupported/overflow fallback), (b) the qualification docs define parity by this policy (`doc/vulkan/source-audit.md:118`, `qualification-roadmap.md:17,69`), and (c) every construct has a direct Vulkan 1.3 equivalent with no optional extension beyond `fragmentStoresAndAtomics`.
2. Keep: 16-byte node (two `packHalf2x16` words, float depth bits, next), R32_UINT head cleared to `~0u`, `K = clamp(4..32)` insertion sort descending z, weighted tail, 256-hop cap, global-pool overflow -> immediate source-over blend, particle/custom-blend residual after the resolve, rigged depth replay, glow replay, RGB-only resolve blend `(ONE, 1-SRC_ALPHA)`, `>= 1 node/px` floor and the MiB budget. Fix the one policy wart: do not overwrite the user's method setting on budget failure; expose an "available" flag instead.
3. Replace the atomic counter with an SSBO `atomicAdd`, separate the three bindings, use `vkCmdClearColorImage`/`vkCmdFillBuffer` + `vkCmdPipelineBarrier2`, and run capture / resolve / depth replay / residual as four dynamic-rendering scopes on one in-order queue with a single node pool shared across frames in flight.
4. Prefer `early_fragment_tests` with a read-only depth attachment over the depth copy; keep the sampled-snapshot variant as a validation mode and compare node counts/images against the GL build (the existing `test_ppll_capture.py` cases: equality, occlusion, overflow, reset) before dropping it.
5. Treat k-buffer-with-interlock and moment-based OIT as post-parity experiments with their own A/B qualification; use WBOIT, if at all, only for the particle residual set, which is already outside the exact path.
