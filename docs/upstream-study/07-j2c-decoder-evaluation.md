# JPEG 2000 decoder evaluation

Every texture in Second Life is a JPEG 2000 codestream, fetched progressively
by byte range and decoded at a discard level matching how much has arrived.
The main study left open whether a pure-Rust decoder can serve that path or
whether OpenJPEG must stay behind FFI. This evaluation measures the
`anne-skydancer/3p-openjpeg` fork, the first candidate, against the two
pure-Rust decoders on the viewer's actual decode pattern.

| | |
|---|---|
| Fork evaluated | `anne-skydancer/3p-openjpeg` main at `4d62eaf`, package `v2.5.4-cpu-avx2.1` (the one `vkstorm-release` pins for Windows and Linux) |
| Fork source | submodule `anne-skydancer/openjpeg` at `3b1a6a96` |
| Rust candidates | `hayro-jpeg2000` 0.4.1, `j2k` 0.12.0 (with `j2k-native`) |
| Machine | AMD Ryzen 9 9950X3D, Windows 11, clang (llvm-mingw) for C, rustc 1.99 with `target-cpu=x86-64-v3` |
| Harness and raw data | `tools/j2c-eval/` in [vkstorm-core-rust](https://github.com/anne-skydancer/vkstorm-core-rust/tree/main/tools/j2c-eval), results in `results/2026-10-20-ryzen9950x3d.csv` |
| Date | 2026-10-20 |

## 1. What the fork actually is

The `3p-openjpeg` repository is Autobuild packaging. The code lives in its
submodule, which is upstream OpenJPEG master as of 2026-09-05 (45 commits past
the 2.5.4 tag) plus three additions by the project owner:

- an OpenCL decoding and encoding backend (Tier-1, inverse DWT, MCT on GPU,
  with CPU fallback and exact-output tests),
- a CUDA decoding backend,
- an `OPJ_PRECISE_FP` option that keeps floating-point behaviour strict.

The `cpu-avx2` package used by the release viewer is that source built with
`/arch:AVX2` on MSVC or `-mavx2 -ffp-contract=off` on clang and gcc, with both
GPU backends compiled out. It adds no CPU SIMD code of its own: the AVX2 paths
are upstream OpenJPEG's existing SSE and AVX2 code in the wavelet and Tier-1
stages plus compiler auto-vectorisation. The fork's own README says the AVX2
gain over SSE2 was small and workload-dependent. This evaluation confirms it.

The GPU backends are not relevant to Aleph: the viewer decodes textures on
worker threads, and if GPU decode is ever wanted it belongs in Vulkan compute
inside this project, not in an OpenCL dependency.

## 2. Method

Three decoders, four builds, decoded single-threaded unless noted, best of
five runs:

- `opj_avx2`: the fork, AVX2 build, through a C harness that mirrors
  `LLImageJ2COJ::decode`: J2K codec, strict mode off, decoded resolution
  factor set to the discard level, memory stream cut at `max_bytes`.
- `opj_sse2`: the same source without `-mavx2`.
- `hayro`: `hayro-jpeg2000` with `strict = false` and `target_resolution`
  set to the reduced size.
- `j2k`: `j2k` with `DecodeSettings::lenient()` and
  `CpuDecodeParallelism::Serial`, through `decode_native_components_at_reduction`.
  `j2k-auto` is the same crate with its default parallelism, shown for context.

Corpus: synthetic photo-like images at 256², 512², 1024², 2048², a 1024²
mosaic of a real Second Life land picture, two RGBA logos, all encoded with
the viewer's own upload rules (J2K, RLCP, irreversible 9/7, layers by surface
area with rates 8·4^i, so a 1024² texture is 393 KB at 8:1), plus three real
skin textures shipped by the viewer. Scenarios: full decode at discard 0 to
3, and truncated input at the byte fractions the viewer would have fetched
for each discard level. Output equality is measured against `opj_avx2`.

## 3. Results

### Full decode, single-threaded, milliseconds

| Image | Size, bytes | opj_avx2 | opj_sse2 | hayro | j2k | j2k-auto |
|---|---|---|---|---|---|---|
| synth 2048², RGB | 1,572,873 | 219.6 | 214.8 | 176.7 | 173.6 | 85.6 |
| synth 1024², RGB | 393,062 | 53.5 | 53.2 | 45.1 | 44.8 | 22.2 |
| land mosaic 1024², RGB | 393,198 | 54.5 | 54.8 | 46.8 | 47.8 | 27.6 |
| logo 1024², RGBA | 524,259 | 83.0 | 83.2 | 74.5 | 84.3 | 44.1 |
| synth 512², RGB | 98,264 | 13.0 | 12.5 | 11.2 | 11.2 | 7.4 |
| skin startup logo 1024², RGB | 69,118 | 12.1 | 14.2 | 13.6 | 14.2 | 12.4 |
| skin land picture 256², RGB | 24,492 | 3.6 | 3.5 | 3.5 | 3.3 | 3.6 |

### Reduced decode of the 2048² image, milliseconds

| Discard level | Output | opj_avx2 | hayro | j2k | j2k-auto |
|---|---|---|---|---|---|
| 1 | 1024² | 67.5 | 60.9 | 67.5 | 35.4 |
| 2 | 512² | 26.8 | 25.5 | 29.0 | 17.6 |
| 3 | 256² | 12.0 | 11.9 | 15.1 | 11.6 |

### Accuracy against the fork

Across all 56 full and reduced cases, both Rust decoders are within one code
value of OpenJPEG on every sample (`max_abs` 1). hayro sits at 78 to 83 dB
PSNR against OpenJPEG, j2k at 92 to 98 dB, and several reduced cases are
bit-identical. The skin startup logo decodes bit-identically in hayro at every
level. These are rounding differences in the inverse wavelet and colour
transform, invisible in a texture.

AVX2 versus SSE2 in the fork: outputs bit-identical, timings within noise.

### Truncated input: the decisive test

With the input cut to the byte fraction the viewer fetches for a discard
level, OpenJPEG with strict mode off decodes what is present. Both Rust
decoders, as published, refuse every truncated stream: hayro with
`Tile(Invalid)`, j2k with "image contains no tiles". Appending an
end-of-codestream marker does not help. The cause is the same in both, since
j2k-native is a fork of hayro: the tile-part parser takes the declared
tile-part length from the SOT marker and fails when fewer bytes exist.

A seven-line change in hayro's `tile.rs`, clamping the tile-part slice to the
bytes available when strict mode is off, fixes it. The patch is in
`tools/j2c-eval/patches/` in vkstorm-core-rust. With it applied:

| synth 1024², bytes present | opj_avx2 ms | hayro patched ms | PSNR vs complete decode, opj | same, hayro |
|---|---|---|---|---|
| 50% | 31.4 | 36.0 | 36.6 | 36.2 |
| 25% | 23.5 | 18.9 | 34.9 | 34.7 |
| 12% | 19.0 | 12.1 | 33.1 | 33.4 |
| 6% | 16.3 | 10.0 | 30.0 | 31.4 |
| 3% | 15.7 | 8.1 | 25.0 | 25.0 |

The patched hayro reconstructs truncated streams as well as or better than
OpenJPEG at every truncation point tested, on all three images checked, and
does so faster at the small fractions where most progressive decodes happen.
The two decoders differ in how they treat the incomplete last packet, which
shows up as 35 to 44 dB between them at some points, but both are equally
close to the complete image. Untruncated decoding is unchanged by the patch.

### Other findings

- `j2k::decode_srgb8` rejects four-component input ("requires grayscale, RGB,
  or a restricted ICC input profile"). Its native-components path handles
  RGBA fine; the harness uses that path.
- hayro has no threading; j2k's `Auto` mode uses threads and reaches about
  2.5x on the 2048² image. For a viewer that decodes many textures
  concurrently on a worker pool, per-image threading is not needed.
- hayro outputs 8-bit only, which is all Second Life uses.
- hayro has no encoder. j2k has lossless and lossy encoders and HTJ2K.
- Header-only inspection (dimensions, components) is cheap in both:
  `Image::new` in hayro and `J2kDecoder::inspect` in j2k parse the main
  header without decoding, which is what the viewer's metadata read needs.

### Where the time goes, and what SIMD can and cannot buy

hayro's `simd` feature (on by default, `fearless_simd` with runtime CPU
detection) vectorises the inverse wavelet transform, the colour transform and
the final sample conversion. Measured on the same images:

| Build | synth 2048² | synth 1024² | logo 1024² RGBA |
|---|---|---|---|
| SIMD on, AVX2 compile target | 177.9 | 44.7 | 74.7 |
| SIMD on, baseline x86-64 target (runtime dispatch) | 189.0 | 47.9 | 80.3 |
| SIMD off | 171.8 | 43.0 | 74.3 |

SIMD on versus off is within noise. Timing each stage inside the decoder
explains why:

| Image | total ms | Tier-1 (bit-plane, MQ) | IDWT | MCT | parse |
|---|---|---|---|---|---|
| synth 2048² | 168.0 | 148.1 (88%) | 13.0 (8%) | 1.0 | 0.8 |
| synth 1024² | 41.3 | 36.7 (89%) | 3.0 (7%) | 0.1 | 0.3 |
| logo 1024² RGBA | 71.9 | 65.8 (91%) | 3.9 (5%) | 0.2 | 0.4 |
| synth 2048², discard 2 | 25.0 | 22.9 (92%) | 1.0 | 0.0 | 0.7 |

About nine tenths of decode time is Tier-1: the context-adaptive binary
arithmetic decoder walking bit-planes of each 64x64 code-block. That loop is
sequential by construction, one decision depends on the previous one, so SIMD
does not apply inside a code-block. OpenJPEG has the same profile, which is
why its AVX2 build is no faster than SSE2. Vectorising the remaining IDWT
further would gain at most the 5 to 8% it already costs.

The fork was measured on the same terms, with its SIMD confirmed rather than
assumed. Upstream OpenJPEG, which the fork's CPU path is, carries AVX2 and
SSE2 code in the inverse wavelet (`dwt.c`, 29 AVX2 intrinsic calls and 100
SSE calls), a small amount in Tier-1 (`t1.c`, 28 AVX2 calls for the pass-flag
handling) and SSE in the colour transform (`mct.c`). The AVX2 build used here
compiles with `-mavx2 -ffp-contract=off` and the library contains 2,782
AVX2 (ymm) instructions, 1,380 of them in `dwt.c` and 186 in `t1.c`; the SSE2
build contains none. The fork's own changes to the CPU decoder are the GPU
dispatch hooks in `tcd.c` and a debug capture hook in `t1.c`; it does not
add CPU SIMD. The AVX2 and SSE2 builds produce bit-identical output and
decode within noise of each other because, as in hayro, the wavelet stage
they accelerate is under a tenth of the work. This is consistent with the
fork's own README, which reports the AVX2 gain as small and
workload-dependent.

What does buy speed in Tier-1:

- **Parallelism across code-blocks and across textures.** A 2048² texture
  has over three thousand code-blocks, all independent. j2k's threaded mode
  reached 2.5x on this machine. A viewer decodes many textures at once on a
  worker pool, so aggregate throughput scales with cores whether or not a
  single image is threaded; single-image threading only shortens latency for
  the first textures after a teleport.
- **GPU Tier-1**, which is what the fork's OpenCL backend does. For Aleph that
  would be a Vulkan compute path, a later project, not a dependency.
- **HTJ2K**, the high-throughput block coder, is roughly an order of magnitude
  faster and SIMD-friendly, but Second Life assets are classic J2K and the
  grid would have to transcode. j2k supports it if that ever happens.
- Micro-optimisation of the MQ decoder (table-driven, branchless) is where
  hayro, OpenJPEG and Kakadu already compete; hayro is on par with OpenJPEG
  here and there is no large win left on the CPU.

So "SIMD-optimised" is already true of hayro for the stages where SIMD is
possible, and it is not the lever. The patch
`patches/hayro-jpeg2000-0.4.1-stage-timing-and-truncation.diff` adds the
stage timers (printed when `HAYRO_STAGE_TIMING` is set) for anyone repeating
this.

## 4. Verdict

> Outcome (2026-10-20, after this evaluation): the project chose a different
> route than the verdict below recommends. Both tracks use one Rust codec for
> decoding and encoding, `vkstorm-openjp2` in vkstorm-core-rust, a maintained
> port of the OpenJPEG fork, because uploads must be byte-identical to the C
> encoder, the cache and upload paths treat the bytes as identity, and two
> crates for the two directions were not wanted. That port now decodes
> bit-identically to the fork at every discard level and truncation, encodes
> byte-identically, and runs at or above the fork's speed on Windows and
> Linux (measurements in the core repository, `docs/j2c-port-plan.md`).
> `hayro-jpeg2000` and `j2k` are not in the product. The measurements below
> stand as the record of why a pure-Rust decoder was viable at all.


**A pure-Rust decoder can replace OpenJPEG for texture decode, and
`hayro-jpeg2000` is the primary candidate**, on three grounds measured here:
single-threaded speed equal to or better than the fork on every image, output
within one code value, and, with a small patch, truncated-stream decoding of
equal quality. It is MIT or Apache-2.0, forbids unsafe code outside its SIMD
feature, and has four million downloads as the PDF ecosystem's decoder.

Conditions and the remaining work:

1. **Upstream the truncation change.** Offer the non-strict tile-part clamp
   to hayro. Until it lands, vendor the crate with the patch applied. The
   change is small enough to carry.
2. **Validate on real textures.** This corpus is synthetic images encoded by
   OpenJPEG plus 21 skin textures. Live textures include server-baked
   avatar textures encoded by Kakadu and user uploads from many encoders.
   Run the harness over a sample of a real texture cache before committing.
   The fork's own OpenCL work did that with cache textures; the same
   procedure applies.
3. **Throughput, not SIMD.** Decode textures concurrently on a worker pool,
   and consider j2k-style intra-image code-block parallelism for latency on
   large textures. Do not spend effort on further SIMD; Tier-1 dominates.
4. **Corrupt streams.** Second Life has damaged assets in the wild. OpenJPEG
   with strict mode off tolerates them; hayro's non-strict mode also skips
   failed packets and code-blocks, but this was not exercised here.
5. **Encoding for uploads.** hayro has none. Use j2k's encoder for the upload
   path, or OpenJPEG through `openjp2-sys` behind a feature flag. Uploads are
   rare and not latency-critical, so either is fine.
6. **Keep the fork as the oracle.** The C harness and the fork stay in the
   test suite so every decoder change is compared against OpenJPEG output
   within one code value, and so a runtime fallback to OpenJPEG via FFI
   remains one feature flag away if a texture class turns up that hayro
   mishandles.

`j2k` remains a credible second candidate: slightly faster with threads, an
encoder included, HTJ2K support. Against it: six crates, a 0.12 API still
moving, the sRGB convenience path's RGBA gap, and the same truncation defect
inherited from hayro. Reassess if hayro declines the patch or stalls.

The fork's AVX2 packaging is sound and remains the right choice for the C++
viewer, but it brings nothing a Rust viewer needs beyond what upstream
OpenJPEG provides: the AVX2 build is upstream code with a compiler flag, and
the GPU backends do not fit this project.
