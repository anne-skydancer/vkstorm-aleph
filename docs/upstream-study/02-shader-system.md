# Firestorm shader system survey (for the Vulkan/Rust port)

Source: `C:\Dev\vkstorm-aleph-ref` (git worktree, branch `vkstorm-aleph` = Firestorm master as of 2026-10-18). All paths below are relative to `indra/`. Line numbers were taken from the actual files on 2026-10-20; nothing in the tree was modified.

Corpus size: 224 `.glsl` files (222 under `class1/2/3` + `errorF/errorV`), 25,320 lines total; `class1/deferred` alone is 99 files / 14,981 lines, of which three vendored post-process files (CASF 2,575, fxaaF 2,137, SMAA 1,475) are 24% of the entire corpus.

---

## 1. How a program is assembled

### 1.1 Class-level file lookup

`LLShaderMgr::loadShaderFile(filename, shader_level, type, defines, texture_index_channels)` (llrender/llshadermgr.cpp:462) resolves a logical name such as `deferred/softenLightF.glsl` by iterating `gpu_class` from the requested `shader_level` **down to 1** and opening the first file that exists at `<prefix><class>/<filename>` (llrender/llshadermgr.cpp:516-544). The prefix is `app_settings/shaders/class` (newview/llviewershadermgr.cpp:3588-3591), so the probe order is `class3/` > `class2/` > `class1/`. The level actually used is written back into `shader_level` (llrender/llshadermgr.cpp:941). If compilation fails and `shader_level > 1`, the function decrements the level and recurses (llrender/llshadermgr.cpp:945-949), so a broken class3 file silently falls back to class2/class1.

At the program level, `LLGLSLShader::createShader()` (llrender/llglslshader.cpp:407) compiles every entry in `mShaderFiles` at `mShaderLevel`, attaches the library objects via `attachShaderFeatures`, then `mapAttributes()` (which links) and `mapUniforms()`. If any step fails and `mShaderLevel > 0` it decrements `mShaderLevel` and calls itself again (llrender/llglslshader.cpp:486-502).

Class levels are **hard-coded** in `LLViewerShaderMgr::setShaders()` (newview/llviewershadermgr.cpp:609-616): lighting 3, interface 2, environment 2, object 2, effect 2, windlight 2, water 3, deferred 3; avatar is forced to 1 (newview/llviewershadermgr.cpp:705-708). There is no longer any user-facing "shader level" selection; the class directories are now simply a layered override mechanism. Only 28 logical names exist above class1 (see Section 4.4); 8 of those shadow a class1 file of the same name (e.g. `materialF` class1 52 lines vs class3 445 lines, `reflectionProbeF` class2 81 vs class3 914, `waterF` class1 34 vs class3 349, `pbralphaF` class1 33 vs class2 277). The class1 copies of these 8 are stubs that are never compiled at the shipped levels.

`errorF.glsl`/`errorV.glsl` are referenced only inside an `#if 0` block (llrender/llshadermgr.cpp:496-513) and are dead; they are magenta-output fallbacks (`frag_color = vec4(1,0,1,1)`).

### 1.2 Header the loader prepends

`loadShaderFile` builds the source as an array of lines in this order (llrender/llshadermgr.cpp:569-752):

1. `#version` chosen from `gGLManager.mGLSLVersionMajor/Minor` (llrender/llshadermgr.cpp:569-623): GL >= 4.2 -> `#version 420`; GL 4.0/4.1 -> `#version 400`; GL 3.3+ -> `#version 330`; GL 3.0-3.2 -> `#version 150`; GL 2.x -> `#version 140`/`150` plus `precision mediump int; precision highp float;`. No `core`/`compatibility` profile token is ever emitted. Nothing in the corpus contains its own `#version` (only commented references in fxaaF.glsl:689,709,722).
2. `#define FRAGMENT_SHADER 1` or `#define VERTEX_SHADER 1` (llrender/llshadermgr.cpp:625-632). Note: geometry shaders get `VERTEX_SHADER` because the test is only `type == GL_FRAGMENT_SHADER`.
3. G-buffer flag constants (llrender/llshadermgr.cpp:636-640):
   `GBUFFER_FLAG_SKIP_ATMOS 0.0`, `GBUFFER_FLAG_HAS_ATMOS 0.34`, `GBUFFER_FLAG_HAS_PBR 0.67`, `GBUFFER_FLAG_HAS_HDRI 1.0`, `GET_GBUFFER_FLAG(data, flag) (abs(data-flag)< 0.1)`.
4. One `#define K V` per entry of the `defines` map passed in (llrender/llshadermgr.cpp:642-649). For a program this is `LLGLSLShader::mDefines` (llrender/llglslshader.cpp:458); for the library objects it is the `attribs` map built in `loadBasicShaders` (newview/llviewershadermgr.cpp:805-862), which is also copied to `LLGLSLShader::sGlobalDefines` (newview/llviewershadermgr.cpp:862) but **only for hashing**; `sGlobalDefines` is never injected into program sources (llrender/llglslshader.cpp:2072 is its only consumer).
5. `#define IS_AMD_CARD 1` when the vendor is AMD (llrender/llshadermgr.cpp:651-654). Consumed by avatarSkinV:46, objectSkinV:65, blurLightF:113, multiPointLightF:179 as "touch the array by static index" workarounds.
6. On macOS non-Apple-GPU: `OLD_SELECT=1` (llrender/llshadermgr.cpp:466-471, llrender/llglslshader.cpp:424-430), consumed by srgbF.glsl:32,51,71 (`mix(vec3,vec3,bvec3)` workaround).
7. **Indexed-texture synthesis** (fragment stage only, when `texture_index_channels > 0`, llrender/llshadermgr.cpp:656-749): emits `#define HAS_DIFFUSE_LOOKUP`, `uniform sampler2D tex0..texN-1;`, `flat in int vary_texture_index;` (if N>1) and a generated `vec4 diffuseLookup(vec2 texcoord)` that is a `switch` on `vary_texture_index` (or an `if/else if` chain on NVIDIA, llrender/llshadermgr.cpp:716-725). N is `LLGLSLShader::sIndexedTextureChannels`, **hard-coded to 4** (newview/llviewershadermgr.cpp:572; the clamp against `RenderMaxTextureIndex` is commented out). Consumers: diffuseIndexedF, diffuseAlphaMaskIndexedF, emissiveF, fullbrightF, shadowAlphaMaskF, lightF, lightAlphaMaskF, class2 alphaF, class3 fullbrightShinyF.
8. `struct GBufferInfo { vec4 albedo; vec4 specular; vec3 normal; vec4 emissive; float gbufferFlag; float envIntensity; };` is injected into **every** shader (llrender/llshadermgr.cpp:751-752) so that gbufferUtil.glsl can return it.

### 1.3 Text patching: `[EXTRA_CODE_HERE]`

The only in-body rewrite is the extra-code marker. If a file contains the literal `[EXTRA_CODE_HERE]` (conventionally `/*[EXTRA_CODE_HERE]*/` right after the licence block; 66 files do), the header block built above is spliced in at that line instead of being prepended after `#version` (llrender/llshadermgr.cpp:771-845). Files without the marker get the block immediately after `#version` (llrender/llshadermgr.cpp:832-845). This exists so that files that start with `#extension` lines (fxaaF.glsl:26-27, SMAA.glsl:26-28, exoVignetteF.glsl:6) keep them before the defines.

There is **no** `textureLod`/`texture2D` rewriting, no `#include` expansion, no version-dependent patching of function names. Lines are capped at 1024 chars and files at 4096 lines (llrender/llshadermgr.cpp:560-566; CASF.glsl at 2,575 lines is the largest).

### 1.4 "Library" objects and `attachShaderFeatures`

Firestorm does not `#include`. It compiles each library file once as a standalone GL shader object (stored in `mVertexShaderObjects`/`mFragmentShaderObjects` keyed by logical name, llrender/llshadermgr.cpp:934-940) and `glAttachShader`s it to each program that needs it. Cross-object calls are resolved at **GL link time** through forward declarations (311 prototype lines across the corpus, e.g. deferredUtil.glsl:77-81 declares `srgb_to_linear`, `atmosFragLightingLinear`, `decodeNormal`). Vulkan/glslang has no multi-object linking of this kind, so the port must concatenate (or `#include`) the library text into each entry point; see Section 5.

`loadBasicShaders()` (newview/llviewershadermgr.cpp:762-920) precompiles the library set with these levels:

| Stage | File | Level requested | Indexed channels |
|---|---|---|---|
| V | windlight/atmosphericsVarsV, atmosphericsHelpersV, atmosphericsFuncs, atmosphericsV | windlight (2) | - |
| V | lighting/lightFuncV, lightV, lightFuncSpecularV, lightSpecularV | lighting (3) | - |
| V | lighting/sumLightsV, sumLightsSpecularV | `sum_lights_class` = clamp(3, 1, RenderShaderLightingMaxLevel) (:767-779) | - |
| V | environment/srgbF (yes, as a vertex object), avatar/avatarSkinV, avatar/objectSkinV, deferred/textureUtilV, objects/indexedTextureV, objects/nonindexedTextureV | 1 | - |
| F | windlight/atmosphericsVarsF, atmosphericsHelpersF, gammaF, atmosphericsFuncs, atmosphericsF | windlight (2) | -1 |
| F | environment/waterFogF (water 3), environment/srgbF (env 2) | | -1 |
| F | deferred/deferredUtil, gbufferUtil, globalF, shadowUtil, aoUtil, pbrterrainUtilF, tonemapUtilF | 1 | -1 |
| F | deferred/reflectionProbeF | 3 if probes enabled else 2 (:902) | -1 |
| F | deferred/screenSpaceReflUtil | 3 if SSR else 1 (:903) | -1 |
| F | lighting/lightNonIndexedF, lightAlphaMaskNonIndexedF | lighting | -1 |
| F | lighting/lightF, lightAlphaMaskF | lighting | `ch` = 4 (:879-884) |

All of them are compiled with the global `attribs` define map (Section 1.5), so e.g. shadowUtil.glsl sees `SUN_SHADOW`/`SPOT_SHADOW` and reflectionProbeF.glsl sees `REFMAP_LEVEL`, `REF_SAMPLE_COUNT`, `SSR`, `HERO_PROBES`.

`LLShaderMgr::attachShaderFeatures(shader)` (llrender/llshadermgr.cpp:68-366) then attaches objects based on `shader->mFeatures` (an `LLShaderFeatures`, llrender/llglslshader.h:36-63). Order is significant for GL link ("NOTE order of shader object attaching is VERY IMPORTANT", :81). Exact rules:

Vertex side (:82-180):
- `calculatesAtmospherics || hasGamma || isDeferred` -> `windlight/atmosphericsVarsV`
- `calculatesLighting || calculatesAtmospherics` -> `windlight/atmosphericsHelpersV`
- `calculatesLighting`: if `isSpecular` -> `lightFuncSpecularV`, (+`sumLightsSpecularV` unless `isAlphaLighting`), `lightSpecularV`; else -> `lightFuncV`, (+`sumLightsV` unless `isAlphaLighting`), `lightV`
- `calculatesAtmospherics` -> `environment/srgbF`, `windlight/atmosphericsFuncs`, `windlight/atmosphericsV`
- `hasSkinning` -> `avatar/avatarSkinV`; `hasObjectSkinning` -> `avatar/objectSkinV` (and sets `mRiggedVariant = shader`, :170)
- **always** `deferred/textureUtilV` (:177)

Fragment side (:188-363):
- **always** `deferred/globalF` (:188) — provides `mirrorClip`, `encodeNormal`, `decodeNormal` and uniforms `mirror_flag`, `clipPlane`, `clipSign`
- `hasSrgb || hasAtmospherics || calculatesAtmospherics || isDeferred` -> `environment/srgbF`
- `calculatesAtmospherics || hasGamma || isDeferred` -> `windlight/atmosphericsVarsF`
- `calculatesLighting || calculatesAtmospherics` -> `windlight/atmosphericsHelpersF`
- `isDeferred || hasReflectionProbes` -> `deferred/deferredUtil`
- `hasFullGBuffer` -> `deferred/gbufferUtil`
- `hasScreenSpaceReflections || hasReflectionProbes` -> `deferred/screenSpaceReflUtil`
- `hasShadows` -> `deferred/shadowUtil`
- `hasReflectionProbes` -> `deferred/reflectionProbeF`
- `hasAmbientOcclusion` -> `deferred/aoUtil`
- `hasGamma || isDeferred` -> `windlight/gammaF`
- `hasAtmospherics || isDeferred` -> `windlight/atmosphericsFuncs`, `windlight/atmosphericsF`
- `isPBRTerrain` -> `deferred/pbrterrainUtilF`
- `hasTonemap` -> `deferred/tonemapUtilF`
- `hasAtmospherics` -> `environment/waterFogF`
- `hasLighting`: `mIndexedTextureChannels <= 1` ? (`hasAlphaMask` ? `lightAlphaMaskNonIndexedF` : `lightNonIndexedF`) : (`hasAlphaMask` ? `lightAlphaMaskF` : `lightF`), and in the indexed case forces `mIndexedTextureChannels = max(sIndexedTextureChannels,1)` (:346)
- finally `mIndexedTextureChannels <= 1` ? `objects/nonindexedTextureV` : `objects/indexedTextureV` (vertex, :350-363)
- `attachNothing` short-circuits everything (:73-76). `hasTransport`, `hasHeroProbes`, `mGLTF` are **not** consulted by `attachShaderFeatures`; `mGLTF` is only set (newview/llviewershadermgr.cpp:354) and `hasTransport` only appears on the PBR terrain program (:1564).

Side effect: `mFeatures` is also hashed into the program-binary cache key (llrender/llglslshader.cpp:2078) and `hasLighting/calculatesLighting/calculatesAtmospherics` drive `LLRender::syncMatrices` -> `syncLightState()` (llrender/llrender.cpp:1146-1149). Several alpha shaders therefore set `calculatesLighting=false; hasLighting=false; isAlphaLighting=true` **before** `createShader()` (so no forward-lighting library gets linked) and flip them back to `true` afterwards so the light uniforms still get synced ("Alpha Shader Hack", newview/llviewershadermgr.cpp:339-344, 1848-1852, 1516-1523, 2434-2437).

### 1.5 Define catalogue

Global define map built in `loadBasicShaders` (newview/llviewershadermgr.cpp:805-860), applied to the library objects (and used in the cache hash):

| Define | Source | Value |
|---|---|---|
| `MAX_JOINTS_PER_MESH_OBJECT` | `LLSkinningUtil::getMaxJointCount()` = `LL_MAX_JOINTS_PER_MESH_OBJECT` (newview/llskinningutil.cpp:92, llcharacter/lljoint.h:48) | 110 |
| `HAS_EMISSIVE` | `RenderEnableEmissiveBuffer` (default 1) | 1 |
| `SUN_SHADOW` | `RenderShadowDetail >= 1` (default 2) | 1 |
| `SPOT_SHADOW` | `RenderShadowDetail >= 2` | 1 |
| `SSR` | `RenderScreenSpaceReflections` (default 0) | 1 |
| `REFMAP_LEVEL` | `RenderReflectionProbeLevel` clamped 0..3 (default 3), only if `RenderReflectionsEnabled && GL > 3.99` | 0-3 |
| `REF_SAMPLE_COUNT` | with probes | 32 |
| `HERO_PROBES` | `RenderMirrors` (default 0) | 1 |
| `TERRAIN_PLANAR_TEXTURE_SAMPLE_COUNT` | `RenderTerrainPBRPlanarSampleCount` (default 3) | 1 or 3 |
| `TERRAIN_TRIPLANAR_BLEND_FACTOR` | `RenderTerrainPBRTriplanarBlendFactor` | float |
| `TERRAIN_PBR_DETAIL` | `RenderTerrainPBRDetail` clamped -4..0 | int |

Per-program permutations (`addPermutation`, all in newview/llviewershadermgr.cpp): `HAS_SKIN` (make_rigged_variant :258, GLTF :303), `HAS_EMISSIVE` (add_common_permutations :267-275), `HAS_SUN_SHADOW` (many, when `use_sun_shadow` = deferred level > 1 && RenderShadowDetail > 0, :1096), `HAS_SSAO` (soften :2159 when RenderDeferredSSAO), `HAS_NOISE` (glow extract :1056, post no-DoF :2870), `TRANSPARENT_WATER` (:955,983), `HAS_NORMAL_MAP`/`HAS_SPECULAR_MAP`/`DIFFUSE_ALPHA_MODE`/`HAS_ALPHA_MASK` (material loop :1330-1345; PBR alpha :1495-1498), `HAS_EMISSIVE_MAP` (:1498), `USE_VERTEX_COLOR`, `USE_INDEXED_TEX`, `IS_HUD`, `FOR_IMPOSTOR`, `USE_DIFFUSE_TEX`, `IS_AVATAR_SKIN`, `IS_ALPHA`, `IS_FULLBRIGHT`, `DEPTH_CLAMP` (:2271 — defined but **no GLSL consumes it**), `LIGHT_COUNT` (:1670), `MULTI_SPOTLIGHT`, `AVATAR_CLOTH`, `USE_LAST_EXPOSURE`, `LEGACY_GAMMA`, `GAMMA_CORRECT`, `NO_POST`, `HAS_HDRI`, `COPY_DEPTH`, `HAS_ATTRIBUTE_TANGENT`, `TERRAIN_PAINT_TYPE`, `TERRAIN_PAINT_PRECISION`, `PROBE_FILTER_SAMPLES`, `FXAA_QUALITY__PRESET`/`FXAA_GLSL_400`/`FXAA_GLSL_130`, `SMAA_GLSL_4|3|2`, `SMAA_PRESET_*`, `SMAA_PREDICATION`, `SMAA_REPROJECTION`; GLTF: `MAX_NODES_PER_GLTF_OBJECT`, `MAX_MATERIALS_PER_GLTF_OBJECT`, `MAX_UBO_VEC4S` (derived from `gGLManager.mMaxUniformBlockSize`, :288-299), `UNLIT`, `MULTI_UV`, `ALPHA_BLEND`.

Shader constants: `addConstant(SHADER_CONST_CLOUD_MOON_DEPTH|STAR_DEPTH)` adds `LL_SHADER_CONST_CLOUD_MOON_DEPTH=0.99998` / `LL_SHADER_CONST_STAR_DEPTH=0.99999` (llrender/llglslshader.cpp:74-85, newview/llviewershadermgr.cpp:2931,2973,2989). **No GLSL file references these macros**; the sky shaders hard-code their own depth (`moonV.glsl:43 pos.z = pos.w*0.999991`, `starsV.glsl:46 pos.z = pos.w`, `sunDiscV.glsl:48 pos.z = pos.w*0.999999`). Dead plumbing.

Defines actually tested in GLSL, by frequency (from a census of `#if/#ifdef` lines, FXAA/SMAA/CAS internals excluded): `TERRAIN_PBR_DETAIL*` (~210), `HAS_SKIN` 42, `TERRAIN_PLANAR_TEXTURE_SAMPLE_COUNT` 30, `HAS_EMISSIVE` 25, `HAS_SUN_SHADOW` 14, `UNLIT` 13, `TERRAIN_PAINT_TYPE` 12, `DIFFUSE_ALPHA_MODE` 9, `ALPHA_BLEND` 8, `IS_HUD` 7, `HAS_NORMAL_MAP` 7, `HAS_SSAO` 6, `HAS_ALPHA_MASK` 6, `USE_VERTEX_COLOR` 5, `IS_AVATAR_SKIN` 5, `HAS_ATTRIBUTE_TANGENT` 5, `TRANSPARENT_WATER` 4, `MULTI_UV` 4, `LEGACY_GAMMA` 4, `IS_AMD_CARD` 4, `HAS_SPECULAR_MAP` 4, `HAS_HDRI` 4, `HAS_DIFFUSE_LOOKUP` 4, `USE_INDEXED_TEX` 3, `SUN_SHADOW` 3, `SSR` 3, `SPOT_SHADOW` 3, `OLD_SELECT` 3, `NO_POST` 3, `IS_ALPHA` 3, `HAS_NOISE` 3, `GAMMA_CORRECT` 3, `AVATAR_CLOTH` 3, `USE_LAST_EXPOSURE` 2, `USE_DIFFUSE_TEX` 2, `MULTI_SPOTLIGHT` 2, `COPY_DEPTH` 2, `REFMAP_LEVEL` 1, `IS_FULLBRIGHT` 1, `HERO_PROBES` 1, `FOR_IMPOSTOR` 1, `WATER_MINIMAL`/`WATER_MINIMAL_PLUS`/`SHORELINE_FADE`/`TRANSPARENT_SURFACE` 1 each (never defined by C++ — latent). `HAS_SHADOW`, `HAS_SKY`, `WATER_FOG`, `HAS_REFLECTION_PROBES`, `IS_PBR`, `HAS_SPECULAR`, `MIP_LEVELS` from the brief **do not exist** in this tree.

### 1.6 Variants and the binary cache

- `make_rigged_variant(shader, rigged)` (newview/llviewershadermgr.cpp:251-265) clones name/features/defines/files/level/group, sets `hasObjectSkinning=true`, adds `HAS_SKIN=1`, links `shader.mRiggedVariant`, and calls `createShader()` on the clone **before** the base program is created.
- `make_gltf_variants` (:352-371) builds 16 variants of `gGLTFPBRMetallicRoughnessProgram` from bits ALPHA_BLEND(1)/RIGGED(2)/UNLIT(4)/MULTI_UV(8) (llrender/llglslshader.h:347-355); the base object itself is never linked.
- Program binaries are cached under `shader_cache/<hash>.shaderbin` keyed by `LLGLSLShader::hash()` = XXH128 of name, group, level, file list, defines, global defines, `mFeatures` bytes, vendor/renderer/version strings (llrender/llglslshader.cpp:2055-2083; llrender/llshadermgr.cpp:1140-1218). When a cached binary loads, `attachFragmentObject`/`attachObject` are skipped (llrender/llglslshader.cpp:599-600, 621-622) but `attachVertexObject` is **not** guarded (:578-595) — harmless because `mVertexShaderObjects` is still populated.

---

## 2. The uniform contract

### 2.1 Reserved vertex attributes

`mReservedAttribs` (llrender/llshadermgr.cpp:1224-1237) must match `LLVertexBuffer::AttributeType` (llrender/llvertexbuffer.h:132-147). Before linking, every reserved name is bound to its enum index with `glBindAttribLocation` (llrender/llglslshader.cpp:659-663); after linking the real locations are read back into `mAttribute[]` and `mAttributeMask` (:682-696). `LLVertexBuffer::setupVertexBuffer` (llrender/llvertexbuffer.cpp:1727-1826) sets the pointers from `sCurBoundShaderPtr->mAttributeMask` with these formats:

| Loc | Name | Stride (`sTypeSize`, llvertexbuffer.cpp:688-704) | glVertexAttribPointer format (llvertexbuffer.cpp:1734-1824) | GLSL decl seen |
|---|---|---|---|---|
| 0 | `position` | 16 (vec4) | 3 x FLOAT | `in vec3 position` (87 files) |
| 1 | `normal` | 16 | 3 x FLOAT | `in vec3 normal` |
| 2 | `texcoord0` | 8 | 2 x FLOAT | `in vec2 texcoord0` |
| 3 | `texcoord1` | 8 | 2 x FLOAT | `in vec2 texcoord1` |
| 4 | `texcoord2` | 8 | 2 x FLOAT | `in vec2 texcoord2` |
| 5 | `texcoord3` | 8 | 2 x FLOAT | (reserved; unused in GLSL) |
| 6 | `diffuse_color` | 4 (RGBA8) | 4 x UNSIGNED_BYTE normalized; **points at the emissive array if `MAP_EMISSIVE` is also set** (:1774) | `in vec4 diffuse_color` |
| 7 | `emissive` | 4 | 4 x UNSIGNED_BYTE normalized; also aliased onto loc 6 when color absent (:1783-1787) | `in vec4 emissive` |
| 8 | `tangent` | 16 | 4 x FLOAT | `in vec4 tangent` |
| 9 | `weight` | 4 | 1 x FLOAT | `in vec4 weight` (avatarSkinV: index in integer part, lerp in fraction) |
| 10 | `weight4` | 16 | 4 x FLOAT | `in vec4 weight4` (objectSkinV: index=floor, weight=fract; GLTF: pure weights) |
| 11 | `clothing` | 16 | 4 x FLOAT normalized | unused in GLSL |
| 12 | `joint` | 8 (U64) | `glVertexAttribIPointer` 4 x UNSIGNED_SHORT | `in uvec4 joint` (pbrmetallicroughnessV.glsl:204) |
| 13 | `texture_index` | 16 (shares `position` stride; data is `position.w` at byte offset 12) | `glVertexAttribIPointer` 1 x UNSIGNED_INT (:1816-1817) | `in int texture_index` (indexedTextureV.glsl:27) |

Note the **two aliasing tricks** (emissive->color, texture_index->position.w) and that `position` is stored as vec4 but declared vec3.

### 2.2 Reserved uniforms

`mReservedUniforms` (llrender/llshadermgr.cpp:1240-1577; enum llrender/llshadermgr.h:42-337) has **292 entries** (`END_RESERVED_UNIFORMS`). A program's `mUniform[i]` is the GL location (or -1) of reserved uniform `i`, filled in `mapUniform` by string comparison against every active uniform name (llrender/llglslshader.cpp:779-794). Array uniforms are registered by base name after stripping `[0]` (:768-772).

Per-frequency setters, verified in C++:

**Per draw / per matrix change — `LLRender::syncMatrices()` (llrender/llrender.cpp:1004-1150)**, called before every draw via `gGL.flush()`; compares a per-matrix-mode hash with `shader->mMatHash[]` and only re-uploads changed ones:
`modelview_matrix` mat4, `projection_matrix` mat4, `texture_matrix0..3` mat4, `normal_matrix` mat3 (inverse-transpose of MV, :1048-1066), `inv_modelview` mat4 (:1069-1071), `modelview_projection_matrix` mat4 (:1076-1089, :1119-1131), `inv_proj` mat4 (:1101-1104), `identity_matrix` (:1108-1110). Of these, `identity_matrix` and `inv_modelview` are declared in **no** shader (dead).
`LLRender::syncLightState()` (:953-1001) when `hasLighting||calculatesLighting||calculatesAtmospherics` (:1146): `light_position[8]` vec4, `light_direction[8]` vec3, `light_attenuation[8]` vec4, `light_deferred_attenuation[8]` vec2, `light_diffuse[8]` vec3, `light_ambient` vec3 (declared nowhere), `sun_up_factor` int, plus for non-deferred `ambient_color`, `sunlight_color`, `moonlight_color` vec3.
`color` vec4 via `gGL.diffuseColor4f/4fv/4ub` (:2000-2055) — per draw; this is the "light_color"/"diffuse color" uniform shared by lighting and UI shaders.

**Per material / per batch**:
- Blinn-Phong materials (newview/lldrawpoolalpha.cpp:435-446, 464, 788-790; newview/pipeline.cpp render passes): `diffuseMap`, `bumpMap`, `specularMap` samplers via `bindTexture`; `specular_color` vec4, `env_intensity` float, `emissive_brightness` float, `minimum_alpha` float (`setMinimumAlpha`, llrender/llglslshader.cpp:2024-2029).
- PBR materials (`LLFetchedGLTFMaterial::bind`, newview/llfetchedgltfmaterial.cpp:82-143): `minimum_alpha`, `diffuseMap`, `bumpMap`(normal), `specularMap`(ORM), `emissiveMap`, `texture_base_color_transform` vec4[2], `texture_normal_transform`, `texture_metallic_roughness_transform`, `texture_emissive_transform` (packed KHR_texture_transform, llrender/llglslshader.cpp count 2 each), `roughnessFactor`, `metallicFactor`, `emissiveColor` vec3.
- Rigged meshes: `matrixPalette` as `uniformMatrix3x4fv(AVATAR_MATRIX, ...)` (newview/lldrawpool.cpp:675,711,749) — declared `mat3x4 matrixPalette[MAX_JOINTS_PER_MESH_OBJECT]` in objectSkinV.glsl:27 but `vec4 matrixPalette[45]` in avatarSkinV.glsl:29 (the avatar path uploads with `uniform4fv`). `translationPalette` is reserved but declared nowhere.
- Shiny: `origin` vec4 (`SHINY_ORIGIN`, newview/lldrawpoolbump.cpp:228,336).
- Terrain (newview/lldrawpoolterrain.cpp, 12 setters): `detail_0..3`, `alpha_ramp`, `paint_map`, `detail_N_base_color/normal/metallic_roughness/emissive`, `baseColorFactors` vec4[4], `metallicFactors` vec4, `roughnessFactors` vec4, `emissiveColors` vec3[4], `minimum_alphas` vec4, `terrain_texture_transforms` vec4[5], `region_scale`.
- Avatar wind/cloth: `gWindDir`, `gSinWaveParams`, `gGravity` vec4 (newview/llvoavatar.cpp; declared in deferred/avatarV.glsl).
- GLTF scene: `gltf_node_id`, `gltf_material_id` int (newview/gltfscenemanager.cpp).

**Per pass — `LLPipeline::bindDeferredShader(shader, light_target, depth_target)` (newview/pipeline.cpp:9169-9400)**, plus `bindLightFunc` (:9109-9122), `bindShadowMaps` (:9124-9151), `bindReflectionProbes` (:10276-10340):
samplers `diffuseRect`, `specularRect`, `normalMap`, `emissiveRect`, `depthMap` (g-buffer attachments 0-3 + depth, :9177-9215), `exposureMap`, `noiseMap`, `lightMap`, `environmentMap` (TT_CUBE_MAP), `lightFunc`, `brdfLut`, `shadowMap0..5` (depth-compare targets), `reflectionProbes`/`irradianceProbes`/`heroProbes` (TT_CUBE_MAP_ARRAY), `sceneMap`, `sceneDepth`; uniforms `viewport` vec4, `shadow_matrix[6]` mat4 (:9280), `env_mat` mat3 (:9302), `shadow_clip` vec4, `sun_wash`, `shadow_noise`, `blur_size`, `ssao_radius`, `ssao_max_radius`, `ssao_factor`, `ssao_factor_inv`, `ssao_effect_mat` mat3, `screen_res` vec2, `near_clip`, `shadow_offset`, `shadow_bias`, `spot_shadow_offset`, `spot_shadow_bias`, `sun_dir`/`moon_dir` vec3 (view space), `shadow_res`/`proj_shadow_res` vec2, `depth_cutoff`, `norm_cutoff`, `modelview_delta`/`inv_modelview_delta` mat4, `cube_snapshot` int, `norm_mat` (declared nowhere), `sunlight_color`/`moonlight_color`, `max_probe_lod`; SSR: `iterationCount`, `rayStep`, `distanceBias`, `depthRejectBias`, `glossySampleCount`, `noiseSine`, `adaptiveStepMultiplier` (all float).
Per-light (renderDeferredLighting, :9405+): `center` vec3, `size` float, `falloff` float, `light[N]`/`light_col[N]` vec4 arrays, `far_z`, projector set `proj_mat`, `proj_near`, `proj_p`, `proj_n`, `proj_origin`, `proj_range`, `proj_ambiance`, `proj_shadow_idx`, `shadow_fade`, `proj_focus`, `proj_lod`, `proj_ambient_lod`, `projectionMap` sampler.
Post: DoF `focal_distance`, `blur_constant`, `tan_pixel_angle`, `magnification`, `max_cof`, `res_scale`, `dof_width`, `dof_height`; FXAA `tc_scale`, `rcp_screen_res`, `rcp_frame_opt`, `rcp_frame_opt2`; SMAA `edgesTex`, `areaTex`, `searchTex`, `blendTex`; glow `minLuminance`, `maxExtractAlpha`, `lumWeights`, `warmthWeights`, `warmthAmount`, `glowStrength`, `glowDelta`, `glowNoiseMap`; `display_gamma` (set at lldrawpoolalpha.cpp:110 but declared nowhere); `vignette` vec3; snapshot `border_color`, `border_thickness`, `frame_rect`; RLVa `rlvEffectMode` int, `rlvEffectParam1..5`.

**Per frame — environment (WindLight/EEP)**: `LLEnvironment::updateSettingsUniforms` (newview/llenvironment.cpp:1854-1872) rebuilds four `LLShaderUniforms` groups (`SG_DEFAULT/SG_SKY/SG_WATER/SG_ANY`, llrender/llglslshader.h:142-149) from the current sky and water settings; it marks every shader in `mShaderList` (and its rigged/GLTF variants) `mUniformsDirty` (newview/llenvironment.cpp:1786-1797); `LLGLSLShader::bind()` applies them lazily (llrender/llglslshader.cpp:1071-1075 -> `LLEnvironment::updateShaderUniforms`, newview/llenvironment.cpp:1840-1852: SG_ANY groups plus the shader's own group). The 48-program list that receives these is `finalizeShaderList` (newview/llviewershadermgr.cpp:415-483).
Sky SG_ANY (newview/llsettingsvo.cpp:725-747, 790-827): `ambient_color`, `blue_density`, `blue_horizon` vec3; `haze_density`, `haze_horizon`, `density_multiplier`, `distance_multiplier` float; `cloud_pos_density2` vec3, `cloud_scale`, `cloud_shadow`, `cloud_variance`; `glow` vec3; `max_y`, `moon_brightness`, `moisture_level`, `droplet_radius`, `ice_level`, `reflection_probe_ambiance`; `scene_light_strength`, `sky_sunlight_scale`, `sky_ambient_scale`, `sky_hdr_scale`, `classic_mode` int. SG_DEFAULT: `lightnorm`, `camPosLocal` (:756-759). SG_SKY: `lightnorm`, `cloud_pos_density1`, `sunlight_color`, `moonlight_color`, `cloud_color` (:762-788).
Water SG_ANY (newview/llsettingsvo.cpp:1063-1142): `waterPlane` vec4, `clipPlane` vec4, `mirror_flag`, `waterFogKS`, `waterFogDensity`, `waterFogColor` vec4, `waterFogColorLinear` vec3, `blend_factor`, `lightnorm`.
Water draw pool (newview/lldrawpoolwater.cpp, 16 setters): `screenTex`, `screenDepth`(declared nowhere), `refTex`(nowhere), `exclusionTex`, `bumpMap`, `bumpMap2`, `eyeVec`, `time`, `waveDir1/2`, `lightDir`, `specular`, `lightExp`, `refScale`, `waterHeight`, `normScale`, `fresnelScale`, `fresnelOffset`, `blurMultiplier`, `sunAngle`/`scaledAngle`/`sunAngle2`(nowhere), `water_edge`(nowhere).
Sky draw pool (newview/lldrawpoolwlsky.cpp, 20 setters): `cloud_noise_texture`, `cloud_noise_texture_next`, `blend_factor`, `rainbow_map`, `halo_map`, `sun_size`(nowhere), `moon_brightness`, `sun_moon_glow_factor`, etc.

**Reserved names that no GLSL file declares (dead, 40)**: `baseColorFactors`* `base_color_texcoord`, `bloomMap`, `display_gamma`, `emissiveColors`*, `emissive_texcoord`, `fog_color`, `identity_matrix`, `inscatter`, `inv_modelview`, `irradiance_texture`, `light_ambient`, `metallic_roughness_texcoord`, `norm_mat`, `normal_texcoord`, `occlusion_texcoord`, `positionMap`, `refTex`, `scaledAngle`, `scattering_texture`, `screenDepth`, `sh_input_r/g/b`, `shadow_noise`, `single_mie_scattering_texture`, `sunAngle`, `sunAngle2`, `sun_size`, `terrain_texture_transforms`*, `texture_*_transform`*, `texture_matrix2`, `texture_matrix3`, `translationPalette`, `transmittance_texture`, `water_edge`. (* = false positives: these are declared with array syntax `uniform vec4[2] name` which my scan skipped; the genuinely dead set is the remaining ~32.) Most are leftovers of the pre-EEP Bruneton atmosphere (`transmittance_texture`, `scattering_texture`, `inscatter`, `sh_input_*`) and of the old forward water shader.

**Uniforms declared in GLSL but NOT reserved (61)**: these are set by name through `LLStaticHashedString` and `getUniformLocation(hashed)` (llrender/llglslshader.cpp:1709-1733, 1767-2006): `tex0`, `tex1`, `texture0`, `texture1`, `dither_tex`, `glowMap`, `screenMap` (newview/llviewershadermgr.cpp:52-58, bound to fixed units right after link at :3075-3076, 3265-3266, 3300-3302, 3317, 3333), `kern`, `kern_scale`, `delta`, `dist_factor`, `stepX`, `stepY`, `bump_code`, `norm_scale`, `mipLevel`, `sourceIdx`, `u_width`, `screenRes`, `out_screen_res`, `resScale`, `zNear/zFar/znear/zfar`, `far_clip`, `exposure`, `dynamic_exposure_params/2`, `tonemap_type`, `tonemap_mix`, `dither_scale[_s/_t]`, `above_water`, `waterSign`, `is_mirror`, `clip_plane`, `alpha_scale`, `custom_alpha`, `tint`, `tolerance`, `diffuse_luminance_scale`, `blur_fidelity`, `ssao_irradiance_scale/max`, `hdri_split_screen`, `cas_param_0/1` (uvec4), `SMAA_RT_METRICS`, `predicationTex`, `velocityTex`, `fogCol`, `ambiance`, `direction`, `dt`, `noiseVec`, `fbScale`, `morphFactor`, `kd`, `srcMap`, `transparent_surface` (bool, declared at class3/deferred/reflectionProbeF.glsl:39 but **no C++ setter exists** — it always reads as `false`), `POISSON3D_SAMPLES[128]` (a `uniform` with an initializer, class3/deferred/screenSpaceReflUtil.glsl:187). The complete list of 73 hashed-string names in C++ is in newview/newview grep output (e.g. `NoiseTexture`, `RenderTexture`, `bloomStrength`, `texelSize`, `texture_index_in`...). This is the second, non-table-driven binding path that the Rust side must also reproduce or replace.

Type conventions worth noting: `sun_up_factor` and `classic_mode` are `int` but set with `uniform1i`; `mirror_flag`, `is_mirror`, `waterSign`, `emissive_brightness` are floats used as flags; `rlvEffectParam3` is `bvec2`; `cas_param_0/1` are `uvec4` (set with `uniform4uiv`); `env_mat` is `mat3` uploaded with `transpose=true` (pipeline.cpp:9302). `LLGLSLShader` caches last-set values in `mValue` (map location -> LLVector4) and skips redundant `glUniform*` calls (llrender/llglslshader.cpp:1304-1311 and siblings) — only for scalar/vec setters, not for arrays or matrices.

### 2.3 Uniform buffer objects

Four named blocks, bound by **name -> fixed binding index** after link (llrender/llglslshader.cpp:1002-1021; enum llrender/llglslshader.h:151-158):

| Binding | Block | Declared | Layout | Producer |
|---|---|---|---|---|
| 0 | `ReflectionProbes` | class3/deferred/reflectionProbeF.glsl:45-82 | `layout(std140)`: `mat4 refBox[256]; mat4 heroBox; vec4 refSphere[256]; vec4 refParams[256]; vec4 heroSphere; ivec4 refIndex[256]; ivec4 refNeighbor[1024]; ivec4 refBucket[256]; int refmapCount; int heroShape; int heroMipCount; int heroProbeCount;` (MAX_REFMAP_COUNT 256 = `LL_MAX_REFLECTION_PROBE_COUNT`, newview/llreflectionmapmanager.h:38) | `LLReflectionMapManager` uploads `ReflectionProbeData` with `glBufferData(GL_STREAM_DRAW)` (newview/llreflectionmapmanager.cpp:1322-1324) and binds with `glBindBufferBase(..., UB_REFLECTION_PROBES, mUBO)` (:1354). Size ~ 256*64 + 64 + 256*16*3 + 16 + 1024*16 + 256*16 + 16 = ~49 KB |
| 1 | `GLTFJoints` | class1/gltf/pbrmetallicroughnessV.glsl:198-201 | `vec4 gltf_joints[MAX_NODES_PER_GLTF_OBJECT]` (3 vec4 per joint, mat3x4 packed) | newview/gltf/animation.cpp:455-457; bound newview/gltfscenemanager.cpp:735 |
| 2 | `GLTFNodes` | pbrmetallicroughnessV.glsl:235-238 | `vec4 gltf_nodes[MAX_NODES_PER_GLTF_OBJECT]` | newview/gltf/asset.cpp:186-188; bound gltfscenemanager.cpp:692 |
| 3 | `GLTFMaterials` | pbrmetallicroughnessV.glsl:44-60, F:38-42 | `vec4 gltf_material_data[MAX_UBO_VEC4S]`, 12 vec4 per material (5 packed texture transforms, emissive, factors) | newview/gltf/asset.cpp:235-237; bound gltfscenemanager.cpp:695 |

Everything else — including the 8-light forward arrays, the 6 shadow matrices, the 110-joint `mat3x4` palette (110*48 = 5,280 bytes of default-block uniforms per rigged draw), the 128-entry Poisson table — lives in the **default uniform block** and is pushed with `glUniform*`.

### 2.4 Texture binding model

Texture units are assigned **at link time, in `glGetActiveUniform` enumeration order**, by `mapUniformTextureChannel` (llrender/llglslshader.cpp:818-848): every active uniform whose type is a sampler (`GL_SAMPLER_1D..GL_SAMPLER_2D_RECT_SHADOW`, `GL_SAMPLER_2D_MULTISAMPLE`, `GL_SAMPLER_CUBE_MAP_ARRAY`) gets `mActiveTextureChannels++` and the unit number is written into the sampler uniform with `glUniform1i` immediately. Sampler arrays get consecutive units, capped at 16 (:835-837). `mTexture[reserved_index]` stores the unit.

Two superstitions layered on top:
- `mapUniforms` first force-maps `diffuseMap`, `specularMap`, `bumpMap`, `environmentMap`, `altDiffuseMap`, `reflectionMap` (`reflectionMap` is declared nowhere) **in that order** if `diffuseMap` would otherwise come after one of them, so that `diffuseMap` is unit 0 (llrender/llglslshader.cpp:875-999). The comment at :892 says the author believes this is "superstitious".
- For indexed programs (`mIndexedTextureChannels > 0`), after link `createShader` binds `tex0..texN-1` to units 0..N-1 and shifts every other sampler by N (llrender/llglslshader.cpp:503-538), asserting `mActiveTextureChannels <= 16` ("canary in the coal mine", :533-536).

Only samplers that survive GL's dead-code elimination count. This matters because the library files declare many samplers unconditionally: deferredUtil.glsl declares `normalMap, depthMap, projectionMap, brdfLut` (:51-54), shadowUtil.glsl re-declares `normalMap` plus `shadowMap0..5` (:26-38), gbufferUtil `diffuseRect, specularRect, emissiveRect`, aoUtil `noiseMap, depthMap`, class3 reflectionProbeF `reflectionProbes, irradianceProbes, sceneMap, heroProbes` (:32-35, 696), tonemapUtilF `exposureMap`, class3 screenSpaceReflUtil `sceneMap, sceneDepth`. A program such as the deferred alpha shader links deferredUtil + shadowUtil + reflectionProbeF + screenSpaceReflUtil + its own `diffuseMap`/`tex0..3`, and only the referenced subset is assigned.

Runtime binding goes through `bindTexture(reserved_index, LLTexture*|LLRenderTarget*, type/filter)` -> `gGL.getTexUnit(mTexture[idx])->bindFast/bindManual` (llrender/llglslshader.cpp:1127-1175), or `enableTexture(idx, TT_*)` returning the unit for manual binding (:1223-1242). Texture target types are `TT_TEXTURE, TT_CUBE_MAP, TT_CUBE_MAP_ARRAY, TT_MULTISAMPLE_TEXTURE, TT_TEXTURE_3D` (llrender/llrender.h:71-77); 32 tracked units (`LL_NUM_TEXTURE_LAYERS`, llrender/llrender.h:60). Sampler state (filter/wrap/compare) is **per texture object**, set through `LLTexUnit::setTextureFilteringOption` etc., never per shader; shadow maps are bound with `bind(target, depth=true)` which enables `GL_TEXTURE_COMPARE_MODE` on the depth texture so `sampler2DShadow` works.

Sampler types used in the corpus: `sampler2D` (overwhelmingly), `sampler2DShadow` x6 (shadowUtil only), `samplerCube` x1 (`environmentMap`, class2 reflectionProbeF/class3 materialF), `samplerCubeArray` x3 (`reflectionProbes`, `irradianceProbes`, `heroProbes`). No `sampler2DRect` (despite the `GL_ARB_texture_rectangle` extension lines), no integer samplers, no `sampler2DArray`, no `sampler3D`, no images.

---

## 3. Program inventory

Source: newview/llviewershadermgr.cpp. "Rigged" = a separate program produced by `make_rigged_variant` (HAS_SKIN + objectSkinV). "Pass" is the draw-pool/pipeline stage that binds it (from pool names and `finalizeShaderList`; not every pool was read in depth for this survey). Levels: W=water 3, E=effect 2, I=interface 2, O=object 2, A=avatar 1, D=deferred 3.

### 3.1 Water (loadShadersWater, :922-1013)
| Program | V / F | Lvl | Features | Defines | Variants | Pass |
|---|---|---|---|---|---|---|
| Water Shader `gWaterProgram` | environment/waterV, environment/waterF (class3) | W | calcAtmos, atmos, gamma, srgb, reflectionProbes, tonemap, shadows=use_sun_shadow; group SG_WATER | TRANSPARENT_WATER, HAS_SUN_SHADOW | - | LLDrawPoolWater (above water) |
| Underwater `gUnderWaterProgram` | waterV, environment/underWaterF (class3) | W | calcAtmos, atmos; SG_WATER | TRANSPARENT_WATER | - | LLDrawPoolWater (camera under water) |

### 3.2 Effects (:1015-1091)
| Program | V / F | Defines | Pass |
|---|---|---|---|
| Glow `gGlowProgram` | effects/glowV, glowF | - | bloom blur |
| Glow Extract `gGlowExtractProgram` | effects/glowExtractV, glowExtractF | HAS_NOISE (RenderGlowNoise) | bloom extract |
| Vignette Post `gPostVignetteProgram` (FS) | post/exoPostBaseV, exoVignetteF | - | post |
| Snapshot Frame Post `gPostSnapshotFrameProgram` (FS) | post/exoPostBaseV, snapshotFrameF | - | post |

### 3.3 Interface (:3188-3585)
| Program | V / F | Features/Defines | Rigged | Notes |
|---|---|---|---|---|
| Highlight `gHighlightProgram` | interface/highlightV, highlightF | - | yes | selection outline |
| Highlight Normals / Spec | highlightNormV / highlightSpecV, highlightF | - | - | normal/spec-map highlight |
| UI `gUIProgram` | uiV, uiF | - | - | all 2D UI |
| Pathfinding, PathfindingNoNormals | pathfindingV / pathfindingNoNormalV, pathfindingF | - | - | navmesh overlay |
| Glow Combine, Glow CombineFXAA | glowcombine[FXAA]V, glowcombine[FXAA]F | `glowMap`=0, `screenMap`=1 forced after link | - | final composite |
| Two Texture Compare, One Texture Filter (`LL_WINDOWS` only, :3288-3320) | twotexturecompareV/F, onetexturefilterV/F | tex0/tex1/dither_tex forced | - | texture-compare tools |
| Solid Color `gSolidColorProgram` | solidcolorV, solidcolorF | tex0 forced | - | |
| Occlusion, Skinned Occlusion, Occlusion Cube | occlusionV / occlusionSkinnedV / occlusionCubeV, occlusionF | skinned: hasObjectSkinning | manual | occlusion queries; occlusionF writes frag_data[4] |
| Debug `gDebugProgram` | debugV, debugF | | yes | |
| Normal Debug [2] + Skinned [2] | normaldebugV, **normaldebugG (geometry)**, normaldebugF | HAS_ATTRIBUTE_TANGENT for variant 1 | yes | the only geometry shader (:3391-3397) |
| Clip, Benchmark | clipV/F, benchmarkV/F | | | |
| Reflection Probe Display | reflectionprobeV/F (class2) | reflectionProbes, srgb, calcAtmos, atmos, gamma, isDeferred | | debug probe spheres |
| Copy, Copy Depth | copyV, copyF | COPY_DEPTH (writes gl_FragDepth, copyF.glsl:40) | | |
| Draw Color | objects/simpleNoAtmosV, objects/simpleColorF | | | |
| Terrain Bake RGB<n> (`LocalTerrainPaintEnabled`, default off) | pbrTerrainBakeV/F | isPBRTerrain, TERRAIN_PAINT_PRECISION | | |
| Alpha Mask `gAlphaMaskProgram` | alphamaskV/F | | | used by llappearance texture baking (llappearance/lltexlayer.cpp:145) |
| Reflection Mip, (mis-named) "Reflection Mip Shader" = `gGaussianProgram` | splattexturerectV, reflectionmipF / gaussianF | isDeferred, gamma, atmos, calcAtmos | | probe mip chain |
| Radiance Gen, Hero Radiance Gen, Irradiance Gen (need cube-map arrays) | radianceGenV/F, irradianceGenV/F (class2) | PROBE_FILTER_SAMPLES 32 / 4+HERO_PROBES | | probe prefilter |

### 3.4 Object (:3055-3145)
| Program | V / F | Features | Rigged |
|---|---|---|---|
| Bump `gObjectBumpProgram` | objects/bumpV, bumpF | `texture0`=0,`texture1`=1 forced (:3070-3078) | yes |
| No color alpha mask `gObjectAlphaMaskNoColorProgram` | objects/simpleNoColorV, simpleF | calcLighting, calcAtmos, gamma, atmos, hasLighting, alphaMask (forward lighting path) | - |
| Impostor `gImpostorProgram` | objects/impostorV, impostorF | srgb | - |
| Object Preview | objects/previewV, previewF | hasLighting set *after* create (:3118) | yes |
| Preview Physics | objects/previewPhysicsV/F | none | - |

`gObjectFullbrightAlphaMaskProgram` is declared (:109) and placed in `mShaderList` (:422) but **never created** — dead.

### 3.5 Avatar (:3147-3186, "DEPRECATED -- forward rendering")
| Avatar Shader `gAvatarProgram` | avatar/avatarV, avatarF | hasSkinning, calcAtmos, calcLighting, gamma, atmos, hasLighting, alphaMask | used only for avatar previews (LLVisualParamHint) |

### 3.6 Deferred (:1093-3053) — `use_sun_shadow` = RenderShadowDetail > 0
| Program | V / F | Features | Defines | Variants | Pass |
|---|---|---|---|---|---|
| Deferred Highlight | interface/highlightV, deferred/highlightF | | | | selection into gbuffer |
| Deferred Diffuse | deferred/diffuseV, diffuseIndexedF | srgb, indexed=4 | | rigged | opaque simple |
| Deferred Diffuse Alpha Mask | diffuseV, diffuseAlphaMaskIndexedF | indexed=4 | | rigged | opaque alpha-mask |
| Deferred Diffuse Non-Indexed Alpha Mask | diffuseV, diffuseAlphaMaskF | | | | |
| ... No Color | diffuseNoColorV, diffuseAlphaMaskNoColorF | | | | |
| Deferred Bump | bumpV, bumpF | | | rigged | legacy bump/shiny |
| Material Shader 0..15 / Skinned 16..31 | materialV, materialF (class3) | srgb, calcAtmos, atmos, gamma, shadows, reflectionProbes; hasAlphaMask when alpha_mode!=0; hasLighting on 1,5,9,13(+16) (blend modes) | i&3 -> `DIFFUSE_ALPHA_MODE` (0 none,1 blend,2 mask,3 emissive), i&4 `HAS_SPECULAR_MAP`, i&8 `HAS_NORMAL_MAP`, i&16 `HAS_SKIN`, `HAS_SUN_SHADOW`, `HAS_ALPHA_MASK` | 32 programs, rigged = index+16 | opaque/mask/emissive materials; blend variants used by alpha pool |
| Deferred PBR Opaque | pbropaqueV, pbropaqueF | srgb | HAS_EMISSIVE | rigged | PBR opaque + mask |
| GLTF PBR Metallic Roughness (GLTFEnabled, default off) | gltf/pbrmetallicroughnessV/F | srgb (+alpha-blend variants get atmos/shadows/probes/isDeferred) | ALPHA_BLEND, HAS_SKIN, UNLIT, MULTI_UV, MAX_* | 16 variants | GLTF scene manager |
| PBR Glow | pbrglowV, pbrglowF | srgb | | rigged | glow pass |
| HUD PBR Opaque | pbropaqueV/F | srgb | IS_HUD | | HUD |
| Deferred PBR Alpha | pbralphaV, pbralphaF (class2) | isAlphaLighting, srgb, calcAtmos, atmos, gamma, shadows, isDeferred, (reflectionProbes) | DIFFUSE_ALPHA_MODE=1, HAS_NORMAL_MAP, HAS_SPECULAR_MAP, HAS_EMISSIVE_MAP, USE_VERTEX_COLOR, HAS_SUN_SHADOW | rigged | alpha pool |
| HUD PBR Alpha | pbralphaV/F | srgb | IS_HUD + same | | HUD |
| Deferred PBR Terrain [2] | pbrterrainV, pbrterrainF | srgb, isAlphaLighting, calcAtmos, atmos, gamma, hasTransport, isPBRTerrain | TERRAIN_PBR_DETAIL, TERRAIN_PAINT_TYPE (0 heightmap+noise / 1 paintmap), TERRAIN_PLANAR_TEXTURE_SAMPLE_COUNT | 2 | terrain |
| Deferred Tree / Tree Shadow / Skinned Tree Shadow | treeV/treeF, treeShadowV/treeShadowF, treeShadowSkinnedV | | | | trees |
| Deferred Impostor | impostorV, impostorF | srgb | | | avatar impostors |
| Deferred Light | pointLightV, pointLightF (class3) | isDeferred, fullGBuffer, shadows, srgb | | | local point lights (sphere volume) |
| Deferred MultiLight 0..15 | multiPointLightV/F (class3) | same | LIGHT_COUNT = i+1 | 16 | fullscreen batched lights |
| Deferred SpotLight / MultiSpotLight | pointLightV / multiPointLightV, spotLightF (class3) | same | MULTI_SPOTLIGHT | | projectors |
| Deferred Sun | sunLightV, sunLightF or sunLightSSAOF (class2) | isDeferred, shadows, ambientOcclusion=RenderDeferredSSAO | HAS_EMISSIVE | | sun shadow + SSAO into lightMap |
| Deferred Sun Probe | sunLightV, sunLightF | isDeferred, shadows | | | probe capture |
| Deferred Blur Light | blurLightV/F | isDeferred | | | SSAO/shadow blur |
| Deferred Alpha / Skinned / HUD Alpha | alphaV, alphaF (class2) | isAlphaLighting, srgb, calcAtmos, atmos, gamma, shadows, reflectionProbes, indexed=4 | USE_VERTEX_COLOR, HAS_ALPHA_MASK, USE_INDEXED_TEX, HAS_SUN_SHADOW, HAS_SKIN, IS_HUD | 3 | alpha pool (forward-lit) |
| Deferred Alpha Impostor / Skinned | alphaV, alphaF | srgb, isAlphaLighting, shadows, reflectionProbes, indexed=4 | USE_INDEXED_TEX, FOR_IMPOSTOR, HAS_ALPHA_MASK, USE_VERTEX_COLOR, HAS_SKIN | 2 | impostor capture |
| Deferred Avatar Eyes | avatarEyesV, diffuseF | calcAtmos, gamma, atmos, srgb, shadows | | | |
| Deferred Fullbright (+Alpha Mask, +Alpha Mask Alpha, +Shiny) and HUD twins | fullbrightV, fullbrightF / fullbrightShinyV, fullbrightShinyF (class3) | calcAtmos, gamma, atmos, srgb, indexed=4; shiny: reflectionProbes; "Alpha" variant: isDeferred | HAS_ALPHA_MASK, IS_ALPHA, IS_HUD | 4 rigged + 4 HUD + 4 base = 12 | fullbright pools |
| Deferred Emissive | emissiveV, emissiveF | calcAtmos, gamma, atmos, indexed=4 | | rigged | glow/emissive |
| Deferred Soften | softenLightV (class2), softenLightF (class3) | srgb, calcAtmos, atmos, gamma, isDeferred, fullGBuffer, shadows | HAS_SUN_SHADOW, HAS_SSAO | | fullscreen lighting resolve |
| Haze / Water Haze | softenLightV, hazeF (class3) / waterHazeV, waterHazeF (class3) | same minus fullGBuffer; water: SG_WATER | | | atmospheric haze |
| Deferred Shadow / Skinned Shadow / Shadow Cube | shadowV / shadowSkinnedV / shadowCubeV, shadowF | skinned: isDeferred, shadows, objectSkinning | | | shadow depth |
| Shadow Fullbright Alpha Mask, Shadow Alpha Mask | shadowAlphaMaskV/F | indexed=4 | DEPTH_CLAMP, IS_FULLBRIGHT | rigged each | shadow depth (alpha-tested) |
| GLTF Shadow Alpha Mask / Alpha Blend | pbrShadowAlphaMaskV, pbrShadowAlphaMaskF / pbrShadowAlphaBlendF | | | rigged each | |
| Avatar Shadow / Avatar Alpha Shadow / Avatar Alpha Mask Shadow | avatarShadowV/F, avatarAlphaShadowV, avatarAlphaShadowF / avatarAlphaMaskShadowF | hasSkinning | | | |
| Deferred Terrain (legacy) | terrainV, terrainF | srgb, isAlphaLighting, calcAtmos, atmos, gamma | | | |
| Deferred Avatar | avatarV, avatarF | hasSkinning | AVATAR_CLOTH | | avatar body |
| Deferred Avatar Alpha | alphaV, alphaF | hasSkinning, isAlphaLighting, srgb, calcAtmos, atmos, gamma, isDeferred, shadows, reflectionProbes | USE_DIFFUSE_TEX, IS_AVATAR_SKIN, HAS_SUN_SHADOW | | |
| Exposure, Exposure (no fade), Luminance | postDeferredNoTCV, exposureF / luminanceF | srgb, isDeferred | USE_LAST_EXPOSURE | | auto-exposure |
| Gamma Correction (Deferred / Legacy) | postDeferredNoTCV, postDeferredGammaCorrect | srgb, isDeferred | LEGACY_GAMMA | | |
| Tonemap x6 (Deferred/NoPost x {plain, GammaCorrect, LegacyGammaCorrect}) | postDeferredNoTCV, postDeferredTonemap | srgb, isDeferred, tonemap | NO_POST, GAMMA_CORRECT, LEGACY_GAMMA | 6 | |
| FXAA [4] (GL > 3.9) | postDeferredV, fxaaF | isDeferred | FXAA_QUALITY__PRESET 12/23/28/39, FXAA_GLSL_400 | 4 | AA |
| SMAA Edge/BlendWeights/Neighborhood [4 each] (GL > 3.15) | SMAA*V, SMAA*F **plus SMAA.glsl attached as both V and F** (:2692-2695) | isDeferred | SMAA_GLSL_4, SMAA_PRESET_*, SMAA_PREDICATION 0, SMAA_REPROJECTION 0 | 12 | AA |
| CAS, CAS Legacy Gamma (GL > 4.05) | postDeferredNoTCV, CASF | srgb | GAMMA_CORRECT, LEGACY_GAMMA | 2 | sharpen |
| Deferred Post / CoF / DoFCombine / Post NoDoF / Post NoDoF Noise | postDeferredNoTCV, postDeferredF / cofF / dofCombineF / postDeferredNoDoFF | isDeferred | HAS_NOISE | 5 | DoF & final |
| Environment Map / WL Sky / WL Cloud / WL Sun / WL Moon / Star | skyV, skyF / cloudsV, cloudsF / sunDiscV, sunDiscF / moonV, moonF / starsV, starsF | calcAtmos, atmos, gamma, srgb; SG_SKY | HAS_HDRI (env map) | 6 | sky pool |
| Normal Map Gen | normgenV/F | SG_SKY (!) | | | bump-to-normal conversion |
| Brdf Gen | genbrdflutV/F | | | | one-shot BRDF LUT |
| Screen Space Reflection Post | screenSpaceReflPostV/F (class3) | SSR, isDeferred | | | |
| Deferred Buffer Visualization | postDeferredNoTCV, postDeferredVisualizeBuffers | | | | debug |
| RLVa Sphere Post (FS) | rlvV, rlvF | isDeferred | | | RLV @setsphere |

### 3.7 Count

Counting every `createShader()` reached on a Windows build with defaults except where noted ("max settings" = shadows 2, probes 3, SSAO on, emissive buffer on, SSR on, mirrors on, GLTF on, terrain paint on, cube-map arrays available):

| Group | Linked programs |
|---|---|
| Water | 2 |
| Effects | 4 |
| Interface | 34 (31 without the 2 Windows-only + 1 terrain-bake programs) |
| Object | 7 |
| Avatar | 1 |
| Deferred: materials 32, GLTF 16, multilight 16, SMAA 12, tonemap 6, sky 6, fullbright family 12, FXAA 4, alpha family 5, shadow family 17, post/DoF 5, misc | 178 (162 with GLTFEnabled=0, the default) |
| **Total** | **226 at max; 210 at shipped defaults** |

(The declared `LLGLSLShader` globals number 149 lines but include arrays: 32+16+4+4x3+2+2+2.) Of the 226, 44 are `make_rigged_variant` clones and 16 are GLTF variants, so the number of *distinct source combinations* is ~166. SSR/mirrors/SSAO do not add programs; they change defines in existing ones.

---

## 4. Corpus catalogue

Entry point (E) = has `main()` and appears in some `mShaderFiles`; Library (L) = no `main()`, attached by `attachShaderFeatures` or `loadBasicShaders`. Line counts in parentheses.

### 4.1 class1/avatar (6 files)
- `avatarSkinV` (51) L — `getSkinnedTransform()` from `weight` and `vec4 matrixPalette[45]` (15 joints x 3 rows).
- `objectSkinV` (72) L — `getObjectSkinnedTransform()` from `weight4` and `mat3x4 matrixPalette[110]`.
- `avatarV/F` (70/34) E — deprecated forward avatar (preview only).
- `eyeballV/F` (59/34) — **unreferenced** by any C++ (dead).

### 4.2 class1/windlight (8) + class1/lighting (9) + class1/environment (4) — the "atmospherics library"
- `atmosphericsVarsV/F` (73/48) L — the varyings bridge (`vary_AdditiveColor`, `vary_AtmosAttenuation`, `vary_position`...) with getters/setters.
- `atmosphericsHelpersV/F` (56/44) L — `atmosFragLighting`, `scaleSoftClip` wrappers; uniform `scene_light_strength`.
- `atmosphericsFuncs` (165) L — `calcAtmosphericVars()` — the EEP haze model; declares **all** the sky uniforms (`lightnorm, sunlight_color, moonlight_color, sun_up_factor, ambient_color, blue_horizon, blue_density, haze_*, cloud_shadow, density_multiplier, distance_multiplier, max_y, glow, scene_light_strength, sun_moon_glow_factor, sky_sunlight_scale, sky_ambient_scale, classic_mode`). Attached to both stages.
- `atmosphericsV` (55) L — `calcAtmospherics(pos)` vertex side; `atmosphericsF` (52) L — `atmosFragLighting*` fragment side.
- `gammaF` (55) L — `scaleSoftClipFrag`, legacy gamma.
- `srgbF` (133) L — `srgb_to_linear`, `linear_to_srgb`, ACES/other helpers; attached to vertex stage too.
- `waterFogF` (144) L — `applyWaterFogView*`; uniforms `waterPlane, waterFogColor, waterFogDensity, waterFogKS`.
- `lightFuncV`, `lightFuncSpecularV`, `sumLightsV`, `sumLightsSpecularV` (class1 57 / class3 66), `lightV` (class3 35), `lightSpecularV` L — forward 8-light vertex lighting (only used by the deprecated forward object/avatar programs).
- `lightF`, `lightAlphaMaskF`, `lightNonIndexedF`, `lightAlphaMaskNonIndexedF` L — forward fragment `default_lighting()` with/without `diffuseLookup`.
- `waterV` (109) E vertex for both water programs; `waterF` class3 (349) E above-water; `underWaterF` class3 (87) E; class1 `waterF` (34) stub.

### 4.3 class1/deferred utility libraries
- `globalF` (61) L, in every fragment program: `mirrorClip()` (discard against `clipPlane` when `mirror_flag>0`), `encodeNormal`/`decodeNormal` (spheremap-style xy encoding; `.z` = env intensity, `.w` = gbuffer flag).
- `deferredUtil` (648) L — depth/position reconstruction (`getPosition`, `getPositionWithDepth`, `linearDepth`, :307-330, :176-185), projector light helpers, PBR BRDF (`pbrIbl`, `pbrPunctual`, Sascha Willems code), `clampHDRRange`.
- `gbufferUtil` (59) L — `getGBuffer()` -> `GBufferInfo`.
- `shadowUtil` (242) L — 4-cascade PCF `sampleDirectionalShadow` + `sampleSpotShadow` with `sampler2DShadow` and `shadow_matrix[6]`.
- `aoUtil` (118) L — SSAO `calcAmbientOcclusion` with `noiseMap`.
- `reflectionProbeF` class3 (914) L — probe selection (`preProbeSample`, bucket/neighbor walk over the UBO), parallax-corrected box/sphere sampling, `sampleReflectionProbes`, `applyGlossEnvironment`, hero probe; class2 (81) is the no-probe fallback using `environmentMap` samplerCube + `reflection_probe_ambiance`.
- `screenSpaceReflUtil` class3 (394) L — ray-march SSR with a 128-entry Poisson `uniform` initializer; class1 (37) stub `random()`.
- `tonemapUtilF` (191) L — ACES/Khronos PBR neutral/etc. `toneMap`, exposure sampling.
- `pbrterrainUtilF` (522) L — triplanar/planar terrain mixing.
- `textureUtilV` (184) L, in every vertex program: KHR_texture_transform + SL texture-animation composition (`texture_transform`, `terrain_texture_transform`), with the Y-flip convention switch (:70-73).
- `SMAA` (1,475) vendored, attached to both stages.

### 4.4 class1/deferred entry points (grouped)
G-buffer writers (`out vec4 frag_data[4]`; 22 files): `avatarF, bumpF, cloudsF, diffuseF, diffuseIndexedF, diffuseAlphaMaskF, diffuseAlphaMaskIndexedF, diffuseAlphaMaskNoColorF, highlightF, impostorF, materialF(class1 stub + class3), moonF, pbropaqueF, pbrterrainF, skyF, starsF, sunDiscF, terrainF, treeF, gltf/pbrmetallicroughnessF, interface/occlusionF`. 13 of them `discard` (alpha masks, terrain, clouds, moon, sky objects).
Vertex entries: `alphaV, avatarEyesV, avatarV, avatarShadowV, avatarAlphaShadowV, bumpV, cloudsV, diffuseV, diffuseNoColorV, emissiveV, fullbrightV, fullbrightShinyV, impostorV, materialV, moonV, pbralphaV, pbrglowV, pbropaqueV, pbrterrainV, pbrShadowAlphaMaskV, shadowV, shadowSkinnedV, shadowCubeV, shadowAlphaMaskV, skyV, starsV, sunDiscV, terrainV, treeV, treeShadowV, treeShadowSkinnedV, normgenV, genbrdflutV, rlvV`; screen-quad vertices `postDeferredV` (with texcoord), `postDeferredNoTCV` (derives `vary_fragcoord = pos.xy*0.5+0.5`, :36), `blurLightV`, class2 `sunLightV`, `softenLightV`, class3 `pointLightV` (sphere volume: `position*size+center`), `multiPointLightV`, `screenSpaceReflPostV`, `waterHazeV`.
Forward-lit transparent fragments (single `frag_color`): class2 `alphaF` (319) — the general alpha shader (8 point lights + sun/moon + probes + shadows, in one file, with `USE_INDEXED_TEX`/`FOR_IMPOSTOR`/`IS_AVATAR_SKIN`/`IS_HUD` branches); class2 `pbralphaF` (277); `emissiveF`, `fullbrightF`, class3 `fullbrightShinyF`.
Lighting resolve: class3 `softenLightF` (285), `hazeF`, `waterHazeF`; class2 `sunLightF`/`sunLightSSAOF`; `blurLightF`; class3 `pointLightF`, `multiPointLightF`, `spotLightF`.
Shadow depth: `shadowF` (31, writes `frag_color = vec4(1,1,1,1)`), `shadowAlphaMaskF`, `avatarShadowF`, `avatarAlphaShadowF`, `avatarAlphaMaskShadowF`, `treeShadowF`, `pbrShadowAlphaMaskF`, `pbrShadowAlphaBlendF`.
Post: `exposureF, luminanceF, postDeferredGammaCorrect, postDeferredTonemap, postDeferredF (DoF), cofF, dofCombineF, postDeferredNoDoFF, fxaaF (2,137), CASF (2,575), SMAA*F/V, postDeferredVisualizeBuffers, rlvF, genbrdflutF, normgenF`.

### 4.5 class1/objects (14), effects (4), interface (44), post (3), gltf (2)
- objects: forward-era `bumpV/F` (dual-texture bump via `texture0/texture1`), `impostorV/F`, `previewV/F`, `previewPhysicsV/F`, `simpleColorF`, `simpleF`, `simpleNoAtmosV`, `simpleNoColorV`; libraries `indexedTextureV`/`nonindexedTextureV` (`passTextureIndex()`).
- effects: `glowExtractV/F`, `glowV/F` (bloom).
- interface: tiny utility pairs listed in 3.3 plus `splattexturerectV` (fullscreen quad), `reflectionmipF`, `gaussianF`, `radianceGenV/F` (GGX prefilter of `samplerCubeArray` into probe mips), class2 `irradianceGenF` (SH-ish irradiance convolution), `irradianceGenV`, `pbrTerrainBakeV/F`, `normaldebugG` (geometry: emits normal/tangent line strips).
- post (Firestorm additions): `exoPostBaseV`, `exoVignetteF`, `snapshotFrameF`.
- gltf: `pbrmetallicroughnessV/F` (329/330) — the new GLTF-scene renderer path using the three GLTF UBOs; 16 define-variants.

---

## 5. GLSL -> Vulkan GLSL migration analysis

Overall the corpus is **already GLSL 1.40+/core-style** (no `attribute`/`varying`, no `gl_FragColor`/`gl_FragData`, no `gl_ModelViewMatrix`, no `texture2D` outside vendored SMAA macros, `in`/`out` everywhere). The hard parts are structural (multi-object linking, implicit bindings, library-declared samplers) rather than syntactic.

### 5.1 Items that must change in every file (mechanical)
1. **`#version`**: the loader injects 140-420 (Section 1.2). Replace with `#version 450` (or 460) injected by the Rust loader; glslang targets Vulkan with `--target-env vulkan1.x`. No file has its own version line. Remove the `precision` fallback.
2. **Multi-object link -> single translation unit**: every program is a GL link of 3-15 objects with 311 forward declarations resolved at link time. For SPIR-V each stage must be one module, so the Rust assembler must textually concatenate the library files in `attachShaderFeatures` order after the entry file (or emit `#include` and use glslang's includer). Consequences: duplicate `uniform` declarations across objects (`modelview_projection_matrix` in 63 files, `diffuseMap` 40, `screen_res` 32, `inv_proj` 15, `normalMap` in both deferredUtil and shadowUtil, `depthMap` in deferredUtil and aoUtil, `sceneMap` in reflectionProbeF and screenSpaceReflUtil...) are legal in GL across separate objects but a **redefinition error in one unit** — they must be de-duplicated (a preprocessing step that hoists all `uniform` declarations into a generated header is the natural fix and also solves item 3). Same for duplicate `in`/`out` varyings between atmosphericsVarsF and entry files, and for `const` definitions (`M_PI` etc.).
3. **Implicit uniform locations -> explicit bindings**: 292 reserved + 61 named default-block uniforms, all set with `glUniform*` by location. Vulkan has no default uniform block. The port must move them into descriptor-set UBOs/push constants with `layout(set=S, binding=B)` and `std140`. Suggested partition (mirrors the setter frequency in 2.2): set 0 = per-frame (sky/water/env, light arrays, shadow matrices, deferred params, ~2.5 KB); set 1 = per-pass textures (g-buffer, shadow maps, probes, LUTs); set 2 = per-material (textures + factors + transforms); push constants = per-draw matrices (`modelview_matrix`, `modelview_projection_matrix`, `normal_matrix` = 48+64+... > 128 bytes, so MVP+MV only, or a per-draw dynamic UBO) plus `color`, `gltf_node_id`, `gltf_material_id`, `minimum_alpha`. The 110-joint `mat3x4 matrixPalette` (5.3 KB) must become an SSBO or dynamic-offset UBO. Shader source rewrite: `uniform T name;` -> member of a block; references in code are unchanged because block members are accessed unqualified, so this is a header-only change if the declarations are hoisted (item 2).
4. **Samplers -> `layout(set, binding) uniform sampler2D`**: combined image samplers map 1:1 onto GLSL `sampler2D`, so keeping combined samplers is the least-change path. But Firestorm declares samplers unconditionally in library files and relies on GL dead-sampler elimination for unit assignment (Section 2.4). In Vulkan every declared sampler needs a binding; two options: (a) give every known sampler a fixed binding in a global table (the 76 unique sampler names fit comfortably in one set; unused bindings can be left unbound only with `VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT` or must be filled with dummies), or (b) use SPIR-V reflection on the compiled module to find which samplers survived and build a per-program layout. (b) exactly reproduces today's behaviour; (a) is simpler to reason about. `sampler2DShadow` x6 requires `VkSampler` with `compareEnable`; `samplerCubeArray` x3 requires `imageCubeArray` device feature.
5. **Uniform initializers**: `uniform int gltf_node_id = 0;` (pbrmetallicroughnessV.glsl:240) and `uniform vec3 POISSON3D_SAMPLES[128] = vec3[128](...)` (class3/screenSpaceReflUtil.glsl:187) are **illegal in Vulkan GLSL**; move the Poisson table to a `const` array and drop the default.
6. **`uniform bool transparent_surface`** (class3/reflectionProbeF.glsl:39): never set by C++; make it a `const bool` or specialization constant.
7. **Vertex attribute locations**: declare `layout(location = N) in` using the reserved table (Section 2.1); today they are bound by `glBindAttribLocation` and the GLSL has no layout qualifiers. The two aliasing tricks (emissive->`diffuse_color`, `texture_index` = `position.w` as uint) become explicit vertex-input bindings/formats: `texture_index` must be `VK_FORMAT_R32_UINT` at offset 12 of the position stream, and `in int texture_index` must become `in uint`/`int` consistently (GL currently feeds UNSIGNED_INT into `in int`).
8. **Fragment outputs**: all files already use `out vec4 frag_color` (83) or `out vec4 frag_data[4]` (22); add `layout(location=0)` / `layout(location=0) out vec4 frag_data[4]` (array outputs consume locations 0-3). `occlusionF` writes 4 MRTs for an occlusion query — can be reduced.
9. **Y-flip / NDC**: no shader reads `gl_FragCoord`, and `vary_fragcoord` is derived from clip-space xy (`pos.xy*0.5+0.5`, postDeferredNoTCV.glsl:36, softenLightV.glsl:46, pointLightV via `vary_fragcoord = pos` then `/w`). Render targets sampled with these coords are consistent as long as the Rust side either uses a negative-viewport-height flip or flips the projection; in both cases the `0.5+0.5` math is unchanged. `textureUtilV.glsl:70-73` flips `texcoord.y` for KHR_texture_transform — this is a UV-convention flip, not a window flip; keep as-is. The sky-dome/cloud depth hacks (`pos.z = pos.w*0.999991`, moonV:43 etc.) are clip-space and survive the [0,1] depth range (z/w = 0.999991 in both conventions, ordering preserved).
10. **Depth range [-1,1] -> [0,1]** — this is the real correctness trap:
   - `deferredUtil.glsl:311` `ndc = vec4(sc.x, sc.y, 2.0*depth-1.0, 1.0)` and `:328`, `linearDepth` `:178 d = d*2.0-1.0` assume GL depth. With a Vulkan projection (GLM_FORCE_DEPTH_ZERO_TO_ONE or hand-built), these become `ndc.z = depth` and `linearDepth` must use the [0,1] formula. `getPosition`/`getPositionWithDepth`/`getPositionWithNDC` are called by every deferred light/soften/haze/SSR/SSAO/water shader, so this is one library fix but it is load-bearing for everything.
   - Shadow lookup: `mSunShadowMatrix = trans*proj*view*inv_view` with a `[-1,1]->[0,1]` bias matrix on **xyz** (newview/pipeline.cpp:11418-11431, 10104-10112 for spot). In Vulkan the z bias must be dropped (z already in [0,1]) and y must account for the flip; shadowUtil.glsl itself (`stc.xyz /= stc.w; stc.z += offset`) is unchanged.
   - The C++ projection matrices are plain `glm::perspective`/`glm::ortho` with GL defaults (newview/llviewercamera.cpp:381, pipeline.cpp:10123, 11278, 11555; `GLM_FORCE_DEPTH_ZERO_TO_ONE` is **not** defined anywhere — only `GLM_FORCE_SSE2/NEON`, llcommon/llpreprocessor.h:232-234). The Rust side will build its own; the shaders that multiply by `inv_proj`/`projection_matrix` (deferredUtil, shadowUtil, aoUtil, screenSpaceReflUtil, spotLightF, water) are the ones affected.
11. **`gl_FragDepth`** in `fxaaF.glsl:2136`, `postDeferredNoDoFF.glsl:91`, `copyF.glsl:40` (depth copy pass) — legal in Vulkan GLSL; keep, but note it disables early-Z for those passes and depth copies could become `vkCmdCopyImage`/blit instead.
12. **`gl_FrontFacing`** in pbropaqueF:106, pbrterrainF:178,415, pbrmetallicroughnessF:221, class2 pbralphaF:156 — legal, but front-face sign flips if the viewport y-flip changes winding; set `VkPipelineRasterizationStateCreateInfo.frontFace` accordingly.
13. **`discard`** in 40 files, including 13 g-buffer writers and 9 shadow-depth fragments — legal; only perf (no early-Z) implications. `layout(early_fragment_tests)` is used nowhere.
14. **`#extension` lines** (4 files): `GL_ARB_texture_rectangle`, `GL_ARB_shader_texture_lod`, `GL_EXT_gpu_shader4` (SMAA.glsl:26-28, fxaaF.glsl:26-27, exoVignetteF.glsl:6) — all no-ops under `#version 450`; delete. CASF.glsl's `GL_ARB_gpu_shader_int64`/`GL_KHR_shader_subgroup_*` are inside `#ifdef A_LONG`/`A_WAVE` which Firestorm does not define (:538-552) — delete the guarded block or leave (it is never reached).
15. **Legacy builtins**: `texture2D(` only in SMAA.glsl:619-621 macros (rename to `texture`); `texture2DLod` hits are the deferredUtil function `texture2DLodSpecular` and FXAA/SMAA macro fallbacks guarded by `FXAA_GLSL_120`/`SMAA_GLSL_2`, which are never selected (FXAA_GLSL_400 / SMAA_GLSL_4 are, newview/llviewershadermgr.cpp:2629-2636, 2673-2678). `textureLod` (9 files), `texelFetch` (CASF, radianceGenF), `textureGather` (CASF, fxaaF, SMAA), `textureSize` (1) are all fine in Vulkan GLSL 450.
16. **Not present, no work**: `gl_VertexID`, `gl_InstanceID`, `gl_ClipDistance`, `gl_PointSize`, `gl_Layer`, `dFdx/dFdy/fwidth`, integer samplers, `imageStore`/compute, tessellation, transform feedback, `subroutine`, `invariant`, `centroid`, `noperspective` (only in FXAA comments), `inverse()`.
17. **Geometry shader**: exactly one, `interface/normaldebugG.glsl` (debug normals). Vulkan supports geometry shaders where the device exposes `geometryShader`; it can also be dropped or replaced by a line-list vertex shader.
18. **`flat` varyings**: `flat out int vary_texture_index` (indexedTextureV) and `flat float vary_sign` (PBR) — supported; the `switch` on `vary_texture_index` in the synthesized `diffuseLookup` is dynamically-uniform-free sampler array indexing in disguise. In Vulkan, replace the generated `tex0..tex3` + switch with `sampler2D tex[4]` indexed by `vary_texture_index` (needs `shaderSampledImageArrayDynamicIndexing`, universally available) or a descriptor-indexed texture array; this removes the NVIDIA if-chain special case (llrender/llshadermgr.cpp:716-725) entirely.
19. **UBOs**: the 4 existing `layout(std140) uniform` blocks just need `set/binding` qualifiers; their layouts (vec4 arrays) are already std140-safe. `ReflectionProbes` at ~49 KB exceeds the 16 KB `maxUniformBufferRange` guaranteed minimum on some mobile/older parts but is fine on desktop (64 KB typical); an SSBO is the safer target.
20. **`#define FRAGMENT_SHADER/VERTEX_SHADER`** and the GBuffer flag macros + `GBufferInfo` struct injection must be reproduced by the Rust preprocessor (or moved into a common header). `[EXTRA_CODE_HERE]` can be honoured or ignored once `#extension` lines are gone.

### 5.2 Effort by family

| Family | Files / lines | Verdict |
|---|---|---|
| interface/*, effects/*, post/*, objects/*, deferred post-process (exposure, tonemap, gamma, DoF, cof, blur, luminance, visualize, rlv, normgen, genbrdflut), shadow depth passes, sky objects (stars/sun/moon) | ~110 files, ~5k lines | **Mechanical**: add layouts, hoist uniforms into blocks, done. |
| windlight/* + lighting/* + srgbF/gammaF/waterFogF (atmospherics library) | 21 files, ~1.1k | **Mechanical but central**: becomes the shared include; must be de-duplicated once and the per-frame UBO defined from its uniform list. |
| G-buffer writers (diffuse*, bump, material, pbropaque, pbrglow, tree, terrain, avatar, impostor, highlight) | ~25 files | Mechanical + MRT layout; `materialF` class3 (445) has 4 alpha modes x normal x specular compile-time branches — unchanged. |
| deferredUtil + shadowUtil + aoUtil + screenSpaceReflUtil | 4 files, ~1.4k | **Rework the math** for depth range and bias matrix (item 10); the rest mechanical. Highest correctness risk. |
| softenLightF, hazeF, pointLightF, multiPointLightF, spotLightF, sunLight*F | 8 files, ~1.3k | Mechanical once deferredUtil is fixed; spotLightF reads `proj_mat` (a GL-convention projector matrix) and `shadow_matrix[4..5]`. |
| reflectionProbeF (class3) | 914 | UBO -> set/binding (or SSBO); `samplerCubeArray` feature; `transparent_surface` fix; otherwise mechanical. Heavy but contained. |
| alphaF (class2), pbralphaF (class2), fullbrightShinyF, water*F | ~1.1k | Mechanical, but these are the programs that link the *most* libraries (atmos + shadows + probes + SSR + indexed textures), so they are the best test of the uniform-hoisting and sampler-binding strategy; alphaF hits ~15 samplers. |
| gltf/pbrmetallicroughnessV/F | 659 | Mechanical; already UBO-based; drop the uniform initializer; `uvec4 joint` attribute. |
| pbrterrainF/V + pbrterrainUtilF | ~1.2k | Mechanical but macro-heavy (TERRAIN_PBR_DETAIL etc.); up to 21 samplers in one program (4 materials x 4 maps + paint/ramp + libs) — check `maxPerStageDescriptorSamplers` is comfortably above 16 (it is on desktop). |
| fxaaF, SMAA, CASF | 3 files, 6.2k | Vendored; already have Vulkan/HLSL-portable code paths; strip the GL extension lines and `texture2D` macros; SMAA.glsl is attached to both stages — becomes an include. |
| normaldebugG | 76 | Only geometry shader; optional. |
| avatarSkinV / objectSkinV | 123 | `matrixPalette` arrays -> SSBO/dynamic UBO; `IS_AMD_CARD` dummy reads can go. |

Shaders where GL-specific assumptions run deepest: **deferredUtil.glsl** (depth reconstruction, lines 176-185 and 307-330), **shadowUtil.glsl** (expects the C++ `[-1,1]->[0,1]` xyz bias in `shadow_matrix`), **aoUtil.glsl** and **class3 screenSpaceReflUtil.glsl** (both re-project with `inv_proj`/`projection_matrix` and compare against sampled depth), **class3 spotLightF.glsl** (projector `proj_mat` plus spot shadows), and the loader-synthesized **`diffuseLookup`** (not a file: generated text in llrender/llshadermgr.cpp:656-749). Everything else is bindings and headers.

---

## 6. What the C++ side does that the Rust side must reproduce

1. **Program table and define injection** — `LLViewerShaderMgr::load*` (3,600 lines of `mName/mShaderFiles/mFeatures/addPermutation/createShader`) is the single source of truth for which (entry V, entry F, library set, define set) tuples exist. It is pure data; a Rust table (or a TOML/RON manifest generated from it) with the same ~166 source combinations x variant bits is the direct replacement. `attachShaderFeatures`' feature->library mapping (Section 1.4) is a 20-rule function — keep it as code.
2. **Preprocessing** — the `#version` header, `FRAGMENT_SHADER/VERTEX_SHADER`, the GBuffer flag macros + struct, `IS_AMD_CARD`, `[EXTRA_CODE_HERE]` splice, and the `diffuseLookup` synthesis (Section 1.2-1.3). ~150 lines of Rust; the `diffuseLookup` part should be replaced by a sampler array (5.1 item 18).
3. **Uniform name tables** — `mReservedUniforms` (292 names, llrender/llshadermgr.cpp:1240-1577) + the enum (llrender/llshadermgr.h:42-337), the 14 reserved attributes, the 4 UBO names, and the 73 ad-hoc `LLStaticHashedString` names. Today these exist so that C++ can address uniforms by integer without string lookups. In Vulkan the equivalent is **the layout of the UBO structs**: the Rust side defines `#[repr(C)]` structs for per-frame/per-pass/per-material/per-draw blocks and the GLSL header is generated from (or checked against) them. The 292-entry table collapses into struct field names; about 32 entries are dead (Section 2.2) and can be dropped.
4. **Uniform-location caching and redundant-set elimination** — `mUniform[]`, `mUniformMap`, `mValue` last-value map, `mMatHash` per-matrix-mode hashing in `syncMatrices` (llrender/llrender.cpp:1034-1141). With UBOs this becomes "write the struct into a ring buffer once per frame/pass/draw"; the hashing logic is replaced by dirty flags on the Rust structs. No per-uniform caching is needed.
5. **Texture channel assignment** — `mapUniformTextureChannel` + the diffuseMap-first and indexed-shift fixups (Section 2.4). Replaceable by SPIR-V reflection (spirv-reflect / naga / rspirv) at pipeline-build time: enumerate `OpVariable` with sampled-image type, read `DescriptorSet/Binding` decorations. If bindings are assigned by a fixed global table (option (a) in 5.1 item 4), reflection is only needed as a validation step.
6. **Material -> uniform binding** — the `bindTexture(LLShaderMgr::X, tex)` / `uniform1f(X, v)` call sites: `LLFetchedGLTFMaterial::bind` (newview/llfetchedgltfmaterial.cpp:82-143), alpha pool (newview/lldrawpoolalpha.cpp:435-446, 464, 788-790), bump pool, terrain pool, water pool, sky pool, `bindDeferredShader` (pipeline.cpp:9169-9400) and `bindReflectionProbes` (:10276-10340). These are ~230 reserved-uniform setter sites across 20 files (pipeline.cpp alone 134). In Rust each becomes a struct write + descriptor-set bind. They are not reflection-replaceable: *which value goes where* is renderer semantics, not shader metadata.
7. **Environment uniform broadcast** — `LLShaderUniforms` groups (SG_DEFAULT/SKY/WATER/ANY) and the lazy `mUniformsDirty` apply on bind (Section 2.2 "Per frame"). Replace with one per-frame UBO that every pipeline binds; the group distinction (sky programs get `lightnorm`/`cloud_pos_density1` variants) becomes two small UBOs or two fields.
8. **Shader cache** — `glGetProgramBinary` cache keyed by XXH128 of inputs (llrender/llglslshader.cpp:2055-2083). Replace with a SPIR-V (+ `VkPipelineCache`) cache keyed by the same hash; SPIR-V is driver-independent so the vendor/renderer strings leave the key.
9. **Fallback-on-failure** — level decrement in `loadShaderFile`/`createShader`. For glslang offline or at startup, make failures fatal in development and keep a single magenta fallback pipeline for release.

Rough split: of the C++ shader-system code (~8.8k lines in the 7 files surveyed), the table/enum data (~1.2k lines) ports as data; the loader/preprocessor (~500 lines) ports as ~200 lines of Rust; the uniform setter API and caching (~1.2k lines of `LLGLSLShader::uniform*`) disappears into struct writes; the texture-unit mapping (~250 lines) is replaced by reflection or a fixed table; the program manifest (~3.3k lines of `LLViewerShaderMgr`) becomes a ~600-line declarative table. What **cannot** be reflected and must be hand-written is the semantic side: which per-pass textures/values each pipeline needs (items 6-7), the feature->library rules, and the variant matrices (material 32, GLTF 16, multilight 16, fullbright 12, tonemap 6).

---

## 7. Things I could not determine / caveats

- Exact per-program active texture-unit counts (the `GLSLTextureChannels` debug log) require a running GL context; the "<=16" assertion and the sampler declarations give the upper bounds quoted above.
- Draw-pass attribution in Section 3 is from program names, `finalizeShaderList`, and the draw-pool files I grepped (alpha, bump, terrain, water, wlsky, pipeline); I did not read every draw pool, so a few "Pass" cells are inferred.
- `LLStaticHashedString` setter sites were listed by name (73) but not individually traced to lines.
- The report counts `createShader()` reach statically; runtime failures (e.g. SMAA/FXAA/CAS disabled on GL < 4.0) reduce the count.
- I did not read the full bodies of CASF/fxaaF/SMAA (vendored); the extension/macro findings come from grep.
