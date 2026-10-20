# Vulkanstorm release-branch audio survey for vkstorm-aleph

Source: `C:\Dev\vkstorm-release-ref` at `origin/vkstorm-release` = `08c07b4cba`, diffed against
Firestorm `upstream/master`. All paths below are relative to that tree; `file:line` cites the
release tree unless marked `(upstream)`. The two commits that carried the audio work are
`31e9a81f1c Preserve SoLoud engine and current accelerated library packages` and
`b47c83f925 Preserve VLC fixes, stream metadata, and spatial music`.

Audience assumption: vkstorm-aleph has its own Rust mixer with HRTF, owns the output device,
and does not link SoLoud. Everything below is therefore judged as *behaviour to re-specify*,
not code to port.

---

## 1. SoLoud backend (`indra/llaudio/llaudioengine_soloud.{h,cpp}`, `lllistener_soloud.{h,cpp}`)

### 1.1 What it is

`LLAudioEngine_SoLoud` is a third `LLAudioEngine` implementation beside FMOD Studio and OpenAL,
selected by `USE_SOLOUD` (default ON, `indra/cmake/SOLOUD.cmake:5-9`) and instantiated in
`llstartup.cpp` under `LL_SOLOUD`. It handles sound effects, UI sounds and wind only. Parcel
music stays in the VLC media plugin: `createDefaultStreamingAudioImpl()` returns `nullptr`
(`indra/llaudio/llaudioengine_soloud.h:50`) and the adapter test asserts that
(`indra/llaudio/tests/llsoloud_adapter_test.cpp:109`).

The actual device/format negotiation and the bounded mixer live in the *package*, not in this
tree: 3p-soloud "with bounded 1024-frame miniaudio mixing and device-selection /
native-format-negotiation patches" (`doc/soloud_audio.md:18-20`). `autobuild.xml:1816-1843`
now pins `202002.e82fd32c1f.vulkanstorm3` with both `windows64` and `linux64` archives, and
`indra/cmake/SOLOUD.cmake:16-18` accepts Windows x64 and Linux x64; `doc/soloud_audio.md:22-27`
still describes revision 2 as Windows-only, so the doc is one package revision stale.

### 1.2 Behaviour worth keeping (with the numbers)

All line references in this table are `indra/llaudio/llaudioengine_soloud.cpp` unless noted.

| Behaviour | Where | Numbers / rule |
|---|---|---|
| Device identity by endpoint ID, not display name | `:244-272` | UUID = MD5(backend name + raw endpoint id bytes); WASAPI uses the wide endpoint-ID string, DSound the GUID bytes, WinMM the numeric index. Duplicate names and renames survive (`doc/soloud_audio.md:39-41`). |
| Device list refresh + change signal | `:450-457` | Re-enumerate every **5 s** in `idle()`; emit `OnOutputDeviceListChanged` only when the map differs. |
| Serialised device access with voice | `:203, :229, :316` | `gAudioDeviceMutex` is a `std::timed_mutex` taken with a **1 s** timeout (shared with the voice subsystem, FIRE-36022). |
| Deferred retry on mutex timeout | `:318-323, :446-449` | Mark pending, retry in `idle()` after **1 s**. |
| Unknown endpoint -> no change | `:298-306` | Log and keep the current device. |
| Reopen failure -> explicit restore | `:324-338` | Restore previous device id; if that also fails, "audio unavailable", no fallback engine (`doc/soloud_audio.md:46-47`). |
| Restart semantics | `:355-385` | Stop wind, invalidate every channel's voice handle, `deinit()`, detach every retained WAV's `mSoloud`/`mAudioSourceID`, destroy and recreate the mixer, restart wind if enabled. Test: 5 restarts at 2/6/8 ch, voice count 0, global volume restored (`llsoloud_adapter_test.cpp:183-193`). |
| Media stream keeps updating when effects device is down | `:458-462` | `idle()` calls `updateInternetStream()` and returns early when the backend is not ready. Test `:194-198`. |
| Init flags | `:169-173`, `h:75-76` | `CLIP_ROUNDOFF`, backend MINIAUDIO, samplerate AUTO, buffer AUTO, channels param defaults to 2; the package negotiates the native rate and 1/2/4/6/8 layouts (`doc/soloud_audio.md:54-55`). |
| Max active voices | `:181` | **64**. |
| 7.1 channel order remap | `:183-189` | When backend reports 8 channels: speaker 4 = (+2, 0, -1), 5 = (-2, 0, -1), 6 = (+2, 0, 0), 7 = (-2, 0, 0): SoLoud slots 4/5 become the Windows **back** pair (z = -1) and 6/7 the **side** pair (z = 0). Verified by the adapter test (`:117-124`). |
| Attenuation model | `:68, :841-842` | `INVERSE_DISTANCE`, min **1 m**, max **512 m** (SoLoud has no unclamped mode); rolloff factor is forwarded per voice (`:650-655, :811-814`). |
| Doppler + velocities | `:644-649, :805-810`, `lllistener_soloud.cpp:50-56` | Source velocity and doppler factor per voice; listener position/at/up/velocity committed via `set3dListenerParameters` + `update3dAudio()`. Doppler/rolloff factors are merely stored in the listener (`lllistener_soloud.cpp:59-77`) and read back per voice. |
| Voices start paused and fully configured | `:597-668` | Avoids the "unattenuated first granule" thump: create paused, set volume/loop/3D params, `update3dAudio()`, then unpause. |
| Baked head/tail PCM ramp | `:849-860` | **64 samples** (~1.5 ms at 44.1 kHz) linear fade baked into both ends of every decoded WAV with > 256 samples, replacing FMOD's implicit play/stop ramps. |
| Declicked stop | `:563-565` | `fadeVolume(0)` + `scheduleStop` over **20 ms** (`DECLICK_FADE_SEC = 0.02`), used on cleanup and on buffer switch (`:751-757`). |
| Synchronised (slave) start | `:712-728` | Seek to `fmod(master_pos, master_length)`, arm **20 ms** fade-in from 0 while paused, then unpause. |
| Loop detection | `:779-791` | Use the mixer's per-voice loop counter (stream position is monotonic), sets `mLoopedThisFrame`. |
| Inaudible voices keep ticking | `:662` | `setInaudibleBehavior(h, true, false)` so loop timing/completion are correct off-screen. |
| UI / forced-priority sounds are 2-D | `:617-621` | `play()` not `play3d()`. |
| Wind as a protected voice | `:71-137, :511-520` | Target freq/gain/pan cross to the audio thread as relaxed atomics; generator runs in **512-frame** chunks into a preallocated **1024-float** scratch (stereo interleaved -> planar). Protected voice so voice-limit eviction never kills it. |
| Wind parameter mapping | `:542-556` | Updated every `LL_WIND_UPDATE_INTERVAL` = **0.1 s** (`indra/llaudio/llaudioengine.h:51`); Linden (+X fwd, +Y left, +Z up) -> DS3D/OpenAL (+X right, +Y up, +Z back); `pitch = 1 + mapWindVecToPitch`, `center_freq = 80 * pitch^(2.5 * (gain + 1))`, gain = `mapWindVecToGain * mMaxWindGain`, pan = `mapWindVecToPan`. |
| Per-frame call throttling | `:582-605` | Only push volume/loop to the mixer on change (every SoLoud call takes the global mixer mutex). |
| Headless test contract | `llsoloud_adapter_test.cpp:101-203` | Null driver at 2/6/8 channels; 1024-frame mix with guard samples (`:134-137`); master/mute gains; listener velocity; wind finite output; category gain x rolloff (`0.4/19`, `0.02`); loop detect; synced slave; restart x5; stream update when unavailable. |

### 1.3 SoLoud-specific plumbing to drop

- `SoLoud::gSoloudRequestedDeviceId` / `gSoloudUseRequestedDeviceId` globals patched into the package (`:40-44, :324-335`); in Rust the mixer selects the cpal/WASAPI device directly.
- The `ma_context` / `sMaDeviceIds` static map and MD5 UUID derivation (`:47-65, :244-272`): keep the *semantics* (stable endpoint identity), not the MD5 trick; cpal exposes device names only, so a WASAPI `IMMDevice::GetId` path (or the `wasapi` crate) is needed for the same guarantee.
- The "repeated-init mixer-table leak" workaround that destroys and recreates the whole `Soloud` object (`:375-378`, `doc/soloud_audio.md:48-50`).
- `LLSoLoudWindSource/Instance` adapter around `LLWindGen` (`:81-137`); the generator itself (`indra/llaudio/llwindgen.h`, Firestorm base) is what to carry.
- Diagnostic heartbeat log every 5 s (`:468-482`).
- `OPENAL` macro undef hack (`llaudioengine_soloud.h:31-35`).
- All `SoLoud::handle` lifetime handling (`resetVoiceHandle`, `isValidVoiceHandle` polling).

### 1.4 Qualification status

Only the headless null-driver adapter test is automated (`scripts/validate_soloud.sh`,
`doc/soloud_audio.md:62-78`). "Physical acceptance remains required for output selection,
duplicate device names, unplug/replug and voice contention, stereo/surround directionality,
motion/Doppler, mute/category gains, wind, queued/synchronized looping"
(`doc/soloud_audio.md:80-83`). Treat the numbers above as design intent, not as listening-tested.

---

## 2. VLC streaming-audio work

### 2.0 What is live vs retired (read this first)

Two generations of plugin-side audio exist in the tree:

1. **Retired custom PCM engine** (`indra/media_plugins/libvlc/llvlcaudio.*`, `llvlcaudiobridge.*`,
   `llvlcwasapi.h`, `llvlcendpoint.h`, `prepare_miniaudio.cmake`, `doc/vlc_audio_engine.md`):
   VLC `amem` callbacks -> private miniaudio/WASAPI render loop with an endpoint-consumption
   fence. It is compiled out: `target_compile_definitions(media_plugin_libvlc PRIVATE
   LL_VLC_PCM_AUDIO=0)` (`indra/media_plugins/libvlc/CMakeLists.txt:32`); "the custom PCM
   device/bridge is not linked into it" (`doc/vlc_audio_plugin.md:7-8`). Only its bridge tests
   build behind `LL_VLC_AUDIO_BRIDGE_TESTS` (`CMakeLists.txt:72-91`). The `media_audio: 1.0`
   capability is advertised **only** in that configuration
   (`indra/media_plugins/libvlc/media_plugin_libvlc.cpp:1276-1280`).
2. **Live VLC-native path** (`llvlcspeakerfillconfig.*`, `vlc_speakerfill.cpp`,
   `prepare_speakerfill.cmake`, `media_plugin_libvlc.cpp` `receiveNativeMusic`/`idleNativeMusic`):
   VLC keeps its own audio output; the viewer drives a VLC audio *filter* (gain, mute, fade
   envelope, stereo upmix) and a patched copy of VLC's WASAPI output that reports playback
   progress. Advertised as `media_music: 1.0` on Windows (`media_plugin_libvlc.cpp:1278-1279`).

Consequence: the viewer's `object` / `nonpositional` roles and the `spatial` command only
work against `media_audio 1.0`; `pluginSupportsMediaAudio()` accepts `media_music` **only for
role `music`** (`indra/llplugin/llpluginclassmedia.cpp:1745-1750`). In the shipped release
build, audio-MIME object media therefore negotiates to "unsupported" and falls back to legacy
`set_volume` (`llpluginclassmedia.cpp:1721-1726`, `indra/newview/llviewermedia.cpp:2244-2268`
`else if (audible)` branch). The positional media branch is **dormant code**, not delivered
behaviour.

### 2.1 Protocol spec: viewer <-> plugin `media_audio` / `media_music`

**Capability negotiation** (`indra/llplugin/llpluginmessageclasses.h:53-54`,
`llpluginclassmedia.cpp:1745-1750`, `media_plugin_libvlc.cpp:1270-1281`)

- Plugin `init_response.versions` may contain `media_audio: "1.0"` (full protocol incl. object
  roles) or `media_music: "1.0"` (music-only subset; Windows native build).
- Viewer predicate `pluginSupportsMediaAudio` = exact `media_audio == "1.0"` OR
  (`role == "music"` AND `media_music == "1.0"`).
- `audioControlsAvailable` = role set AND (plugin still loading OR supported)
  (`llpluginclassmedia.cpp:1815-1819`). Commands issued while loading are queued; on
  `sendMessage` after negotiation, `media_audio` messages are dropped if unsupported, and legacy
  `media_time.set_volume` is dropped if supported (`:1490-1498`). So exactly one gain path
  reaches the plugin.

**Roles** (`llpluginclassmedia.cpp:1829-1836`): `""` (none), `music`, `object`,
`nonpositional`. Set by `LLStreamingAudio_MediaPlugins::initializeMedia`
(`indra/newview/llviewermedia_streamingaudio.cpp:264`) for parcel music, and by
`LLViewerMediaImpl` for `audio/*` MIME media: `object` when the first object is not a HUD
attachment, else `nonpositional` (`llviewermedia.cpp:1968-1971, 2027-2029`).

**Tokens**: `generation` and `serial` are canonical decimal U64 **strings** (never LLSD
integers). Generation starts at 0 and is incremented per `loadURI`; `U64::max` refuses further
loads with state `failed` (`llpluginclassmedia.cpp:801-806`). Serial restarts at 0 per
generation and increments per `transition`; `U64::max` refuses (`:1903`). Plugin side
additionally caps serial at `0x7fffffff` (`media_plugin_libvlc.cpp:1215`) because it packs it
into 32 bits.

**Viewer -> plugin messages** (`llpluginclassmedia.cpp:796-847, 1840-1911`)

| class.name | fields | rules |
|---|---|---|
| `media_audio.configure` | `role`, `generation` | First message of a new URI. Plugin retires the old decoder, clears the old URL, and waits for `load_uri`; never replays the old URL (`media_plugin_libvlc.cpp:1181-1193`). Generation must strictly increase. |
| `media_audio.set_gain` | `generation`, `target` real 0..1, `duration` real (viewer always sends 0.0), `hard_mute` bool | Persistent user/category gain, independent of the transition envelope. Deduplicated on unchanged (target, mute) (`:1862-1869`). Always accompanied by a legacy `media_time.set_volume` fallback (`:1848-1850`) that is filtered when supported. |
| `media_audio.spatial` | `generation`, `right`, `forward` | Object role only; direction normalised, zero-length rejected, unchanged deduplicated (`:1874-1897`). |
| `media_audio.transition` | `generation`, `target` 0..1, `duration` 0..60 s, `serial` | Fade envelope. Target/duration finite; serial strictly increasing. Rejected if controls unavailable or generation 0. |
| `media.load_uri` | `uri`, `audio_generation`, `audio_role` (music only), `speaker_fill` S32 (music only, one of 21/41/51/71) | Sent after the controls (`:838-846`). |

**Fade-in start sequence** (`llpluginclassmedia.cpp:826-833`): `configure` -> `set_gain` ->
[`spatial`] -> `transition(0, 0)` -> `transition(target, duration)` -> legacy `set_volume 0` ->
`load_uri`. Both transitions precede `load_uri` because decoding may start immediately.

**Plugin -> viewer message** `media_audio.state` (`media_plugin_libvlc.cpp:1167-1175`,
`llpluginclassmedia.cpp:1752-1806`)

- Fields: `generation`, `serial`, `state`, `detail` (fixed diagnostic token, `[a-z0-9_]{1,96}`
  else rewritten to `invalid_audio_failure_detail`), optional `diagnostic` map (whitelisted
  numeric/bool fields, logged only).
- States: `priming | running | buffering | paused | drained | failed | silent | cancelled`.
- Acceptance: discarded unless role set, not stopped, generation matches, and serial is `"0"`
  or equals the current serial. Once `failed`/`cancelled` is latched for the generation, later
  states are ignored (`:1769`).
- **Completion rule**: `mAudioTransitionComplete` becomes true only when `serial` equals the
  pending serial AND (`target == 0` => state `silent`) or (`target != 0` => state `running`)
  (`:1800-1805`). Serial-0 routine states update playback state without touching completion.
  `buffering`, `paused`, `drained`, `failed` never count as audible silence.
- `audioTransitionResult()` (`:1808-1813`): `Cancelled` if stopped or state `cancelled`;
  `Failed` if `failed`; else `Complete`/`Pending`.
- `isAudioPlaying` (`:1821-1826`): not stopped, plugin not exited, state not in
  {paused, drained, failed, cancelled}, so `priming`/`buffering` count as playing.

**Plugin side, native music build** (`media_plugin_libvlc.cpp:1177-1253`)

- `configure` with role != music or non-increasing generation is ignored. Accepted configure
  resets serial/ack/target/duration/mute/failure and stops the player.
- `set_gain`: stores `mCurVolume`, `mNativeMute`; applies via `setMusicGain` (cubic curve, see
  3.2). Invalid types -> `failed/invalid_native_music_control` or `.._mute`.
- `transition`: serial must increase and fit 31 bits; records target/duration/start time; if no
  player yet and duration == 0 the target becomes the filter's *initial* envelope value
  (`:1222`), which is how the pre-decode zero hold is installed.
- `idleNativeMusic` emits one-shot `silent`/`running` with the pending serial when
  `musicFadeComplete()` (playback clock passed the final-sample fence); otherwise after
  `duration + 30 s` without completion -> `failed/native_music_playback_clock_timeout`
  (`:1239-1242`). Routine state is derived: failure -> `failed`; no player -> `priming`; VLC
  paused -> `paused`; buffering < 100 % -> `buffering`; else `running`; sent on change with
  serial 0 (`:1244-1251`). Output error (`music-clock-error`) -> `failed/native_music_output_failed`.

**Streaming-audio adapter** (`indra/newview/llviewermedia_streamingaudio.cpp`)

- `hasAudioFade` = no plugin yet, or controls available (`:199-202`).
- `beginAudioFade(target, duration)` validates 0..1 / 0..60 s; if there is no plugin or URL it is
  stored as *pending* and attached to the next `start()` via `loadURI(url, target, duration)`
  (`:204-216, :92-100`).
- `getAudioFadeResult` mapping (`:224-236`): no plugin -> Pending if a fade is pending else
  Cancelled; plugin process exited -> Cancelled; else the plugin result, except
  plugin-Cancelled with a pending fade -> Pending.
- `stop()` sends `media_time.stop` but keeps the plugin object alive when controls are available
  (so the fade-out can be acknowledged); deletes it only if unsupported or exited (`:113-123`).
- `isPlaying()` returns 2 = paused, 1 = playing, 0 = stopped from the audio state (`:163-168`).
- `setAudioHardMute` re-sends gain with the mute flag (`:238-243`).
- On Windows `setMusicSpeakerFill(FSMusicSpatialSound)` is pushed before every `loadURI` (`:89-91`).

**Fade state machine** (`indra/newview/llvieweraudio.cpp`; base states
`FADE_IDLE / FADE_IN / FADE_OUT`, `llvieweraudio.h:51-53`)

- Preference policy: `FSFadeAudioStream` (default on), `FSAudioMusicFadeIn` **3.0 s**,
  `FSAudioMusicFadeOut` **2.0 s** (`indra/newview/app_settings/settings.xml:23111-23142`);
  minimum **0.01 s** (`llvieweraudio.cpp:327`); backend durations clamped to **60 s** (`:348`).
- `startFading()` (`:318-353`): choose duration by state; `mLoginFadePending` = FADE_IN while
  `LLStartUp < STATE_STARTED`; call `beginAudioFade(target, duration)` with target 0 for
  FADE_OUT or login-hold, else 1; duration 0 for the login hold. `mBackendFade` records whether
  the backend accepted.
- Login gate (`:109-120, :203-212`): stream is created and buffers immediately when parcel music
  is known (with the startup mute applied first), held at envelope 0, and on `STATE_STARTED`
  `startFading()` is called once more to submit the real sample-timed fade-in. Covered by
  `scripts/perf/test_login_music_gate.py`, which compiles the five production methods in a
  harness.
- `onIdleUpdate()` (`:183-286`): **before** the normal state machine, a backend result of
  `Failed`/`Cancelled` logs, clears `mNextStreamURI`, sets FADE_IDLE, hard-stops the stream and
  unregisters the idle listener; it does **not** mark success and does **not** start the queued
  replacement; a new explicit start is required (`doc/vlc_audio_viewer.md:26-33`). Then: when
  `mDone`, FADE_OUT -> stop stream, and if a next URI exists go FADE_IN and start it; FADE_IN ->
  if the URL changed meanwhile go FADE_OUT, else IDLE.
- `getFadeVolume()` (`:355-410`): returns 0 during the login hold; 1 when idle; with a backend
  fade it only polls `isAudioFadeComplete()` into `mDone` and returns 1 (the viewer never
  multiplies a frame-clock ramp into the engine gain); if the backend turned out unsupported it
  falls back to the legacy frame-timer ramp (`:376-379`).
- `audio_update_volume()` (`:620-634`): with backend fade -> `setAudioHardMute(mute_audio ||
  mute_music || startup)` and gain = `master * music` once; legacy -> `mute * master * music *
  fade`.
- Buffer policy: FMOD gets 7000/1000 ms stream/decode buffers (`:54-55`); the VLC plugin sets
  `:network-caching=5000` and `:no-video` for 1x1 (audio-only) instances
  (`media_plugin_libvlc.cpp:651-662`) because SLPlugin runs at background priority.

**Tests that are the spec**: `indra/newview/tests/llviewermedia_audio_test.cpp` (12 TUT cases:
queue order/field types, stale generation/serial, submitted-drain rejection, invalid tokens,
spatial normalisation, U64 exhaustion, unnegotiated state, reset, duplicate gain, stop/replay,
failure latching/recovery, speaker-fill opt-in, completed fade retention) built by
`LL_VLC_AUDIO_PROTOCOL_TESTS=ON` (`doc/vlc_audio_viewer.md:60-75`).

### 2.2 Stream metadata (title / artist) path

Two sources, both fork work on top of Firestorm's `ndMediadata_change` receiver
(`indra/llplugin/llpluginclassmedia.cpp:1040-1043 (upstream)`) and
`metadata_update_callback_t` signal (`indra/llaudio/llstreamingaudio.h:63-66`):

1. **In-band via the plugin** (`media_plugin_libvlc.cpp:700-778`; absent in upstream's plugin):
   attach `libvlc_MediaMetaChanged` on the *media* event manager (`:572-581`); react to the field
   that changed: `NowPlaying` (MP3/AAC ICY `StreamTitle`) is split on `" - "` into artist/title;
   `Title`/`Artist` (chained-Ogg Vorbis comments) are read together; a title equal to the URL
   filename placeholder is suppressed; identical pairs are deduplicated; emits
   `media.ndMediadata_change {title, artist}`. Events only set atomic flags; IPC happens in idle
   (`:442-447, :484-486`).
2. **Icecast `status-json.xsl` sidechannel** (`llviewermedia_streamingaudio.cpp:333-520`): first
   poll **5 s** after play, then every **20 s**, give up after **3** consecutive all-candidate
   failures; candidates walk the mount path upward (`/a/b/x.ogg` -> `/a/b/status-json.xsl`,
   `/a/status-json.xsl`, `/status-json.xsl`) to cope with reverse-proxied AzuraCast; picks the
   `icestats.source` whose `listenurl` ends with the mount (or the single source); splits a
   combined `"Artist - Title"`; the sidechannel **wins** over in-band once it has delivered data
   for the current stream (`:280-290`). Needed because libVLC does not surface chained-Ogg
   comments reliably (`:333-337`).
3. `resetMetadata()` on start/stop emits an empty map so the UI clears the old title
   (`:294-318`); `clearStreamMetadata()` also runs on `loadURI`/`stop` in the plugin wrapper
   (`llpluginclassmedia.cpp:798, 1670`).
4. Consumers: `indra/newview/fsfloaterstreamtitle.cpp`, `indra/newview/streamtitledisplay.cpp`
   (Firestorm base).

For Rust: if music is decoded in-process the ICY `icy-metaint` parsing and Ogg comment reading
replace source 1 directly; source 2 (status-json polling with the path walk, the 5/20 s cadence,
3-strike give-up and "sidechannel wins") is worth re-specifying verbatim.

### 2.3 "Spatial music": what was actually delivered

`FSMusicSpatialSound` is an S32 setting, Windows-only, "0=Stereo, 21=2.1, 41=4.1, 51=5.1,
71=7.1. Applies on the next stream start" (`settings.xml:906-916`), shown as **Spatial sound** in
Preferences -> Sound & Media -> Sounds under Output device
(`indra/newview/skins/default/xui/en/panel_preferences_sound.xml:782-807`), hidden on
non-Windows (`indra/newview/llfloaterpreference.cpp:850-853`). It is passed as `speaker_fill` on
`load_uri` and selects the VLC **speaker-fill upmix layout** (section 3).

It is **not** positional. The protocol's own policy says "Music is nonpositional
all-main-speaker audio. Only explicit audio MIME objects use positional routing"
(`doc/vlc_audio_viewer.md:49-51`), and as shown in 2.0 the object branch is unreachable in the
release build (no plugin advertises `media_audio`). The dormant positional code is: role
assignment by `audio/*` MIME (`llviewermedia.cpp:1968-1971`), the per-frame direction
projection `right = -(dir . camera.left)`, `forward = dir . camera.at`, ear position from
`MediaSoundsEarLocation` (0 = camera, 1 = avatar), default forward when coincident, HUD
exemption (`:2250-2263`), and a 2-D (right, forward) unit vector on the wire, i.e. a pan
direction, not a 3-D position; distance attenuation stays in `updateVolume()` on the viewer
side.

So "Preserve ... spatial music" delivered: (a) multichannel speaker fill for stereo parcel
music, (b) sample-accurate native fades with completion acknowledgement, (c) the metadata
paths, (d) the login gate, (e) the generation/serial protocol. Nothing in it positions music in
the world.

---

## 3. The speaker-fill plugin, precisely

### 3.1 Pieces

| File | Role |
|---|---|
| `indra/media_plugins/libvlc/vlc_speakerfill.cpp` | VLC `audio filter` module `speakerfill` (C++, MSVC-built, `libspeakerfill_plugin.dll`). Upmix + gain + mute + fade envelope + completion fence publication. |
| `indra/media_plugins/libvlc/prepare_speakerfill.cmake` | Downloads VLC 3.0.21 `modules/audio_output/wasapi.c`, `mmdevice.h`, `include/vlc_codecs.h` over HTTPS with SHA-256 pinning (`:9-17`), string-patches them into `speakerfill_output.c` (anchor-checked `speakerfill_replace`, `:30-188`). |
| `indra/media_plugins/libvlc/CMakeLists.txt:34-70` | Compiles `speakerfill_output.c` with a **GNU C** compiler (`find_program(VLC_MODULE_CC NAMES x86_64-w64-mingw32-gcc gcc REQUIRED)`, `-std=gnu11`) into `libspeakerfill_output_plugin.dll`; builds the filter; registers the `vlc_speakerfill` test. Whole block is `WINDOWS AND 64-bit` only. |
| `indra/media_plugins/libvlc/llvlcspeakerfillconfig.{h,cpp}` | Plugin-side helper that sets VLC *player* variables (`mmdevice-backend = "speakerfill_output,none"`, `speakerfill-layout`, `audio-filter = "speakerfill"`, gain/mute/fade variables) before `libvlc_audio_output_set(player, "mmdevice")` (`media_plugin_libvlc.cpp:545-559`). `static_assert` pins libVLC 3.0.21 (`llvlcspeakerfillconfig.cpp:17-18`) because it reinterprets the player as `vlc_object_t`. |
| `indra/media_plugins/libvlc/tests/vlc_speakerfill_test.cpp` | Loads both real DLLs, fakes `IMMDevice`/`IAudioClient`, checks negotiation masks, headphone bypass, failure cleanup, filter samples, LFE silence, passthrough, fade timing (`:185-321, :400-590`). |
| `indra/newview/viewer_manifest.py` | Stages the two DLLs under `plugins/audio_filter` and `plugins/audio_output`, plus the fetched sources, cmake script, filter source and `doc/vlc_audio_plugin.md` as `README.md` under `speakerfill-source/` (LGPL source availability). There is no `indra/llplugin/speakerfill-source` directory in the tree; it is a build/staging path. |

### 3.2 What it does

**Upmix matrix** (`vlc_speakerfill.cpp:58-73`), float PCM, VLC WG4 channel order:
- L -> FL, SL (MIDDLELEFT), RL at **0.70710678**; R -> FR, SR, RR at 0.70710678.
- CENTER = (L' + R') x 0.5 (i.e. (L + R) x 0.3536).
- LFE = **0** (no synthesized bass; "use sound-card bass management", `settings.xml:909`).
- Only stereo bitmap input is filled (`:172-173`); native multichannel (incl. authored LFE)
  passes through untouched; PTS/DTS/length/flags/frame count preserved (`:51-55`).
- Accepted target masks (`:166-171`): 2.0+LFE, 4.0+LFE, 2.0+sides+LFE, 4.0, 2.0+sides, 5.1,
  5.0(side)+LFE, 7.1.

**Gain / mute / fade** (same filter):
- User gain uses VLC's cubic curve: `speakerfill-gain = volume^3`
  (`llvlcspeakerfillconfig.cpp:41`), joined by a per-sample linear ramp of `rate / 20` =
  **50 ms** (`vlc_speakerfill.cpp:103-105`).
- Hard mute gates samples independently (`:108-109, :133`).
- Transition envelope: packed command `serial<<32 | ms<<16 | round(target*65535)`
  (`llvlcspeakerfillconfig.cpp:76-78`, so <= 65.5 s, 16-bit target); per-sample linear from the
  current value over `ms * rate / 1000` frames (`vlc_speakerfill.cpp:87-97, :120-122`). Initial
  value comes from `music-fade-initial` (`:189`), which is how fade-ins start at exactly 0 before
  the first decoded sample.
- Completion: on the final target sample, publish `music-fade-fence` = that sample's PTS end,
  bind to the current output `music-clock-epoch`, set `music-fade-completed = command`
  (`:123-130`). Flush clears it (`:148-156`).

**Output variant** (`prepare_speakerfill.cmake` patches to `wasapi.c`):
- Eligibility: endpoint form factor is not `Headphones`/`Headset` via
  `PKEY_AudioEndpoint_FormFactor` (`:104-118`), and the source is stereo bitmap (`:145-146`).
- Layout request (`:123-136`): 2.1 = 2.0+LFE; 4.1/5.1 = 2.0+LFE + **rear pair if the mix format
  has it, else side pair**; 5.1 adds CENTER; 7.1 = `AOUT_CHANS_7_1`; the request must be a subset
  of the endpoint mix-format mask.
- Negotiation (`:149-180`): take the endpoint `GetMixFormat` rate and sample type, substitute the
  requested mask, `IsFormatSupported` (shared mode); on failure retry with the untouched source
  format; publish `speakerfill-mask` only if the final negotiated mask equals the request
  (`:181-185`). The filter reads that mask at open (`vlc_speakerfill.cpp:164`).
- Playback-progress reporting for fades (`:47-101`): after each successful `Play` record the
  block's PTS end as `music_tail`; `TimeGet` publishes `music-clock-pts = tail - queued delay`;
  epoch increments on Start/Flush/Stop; failed submission sets `music-clock-error`.
- Module scored 0 so it is used only when explicitly named (`:187`).

### 3.3 Scope, option, acceptance, build cost

- Windows x64 only; applies to parcel music only ("not to effects or voice", object media and
  video "are not opted in", `doc/vlc_audio_plugin.md:13-16`).
- Operator acceptance on record: "Sound output nominal across speakers with soundcard surround
  off" (`doc/vlc_audio_plugin.md:45-47`) and "This crossfade fix works. Stream transition is
  smooth" (`:124-126`). Explicitly **not** qualified: every selectable layout, long playback,
  device recovery, overlapping/gapless A/B (`:48-49, :77-79, :127-128`).
- Build complexity: a second C toolchain (LLVM-MinGW 22.1.8 validated, `:71-72`), network
  fetch at configure time with pinned hashes, anchor-based source patching that breaks on any
  VLC bump, a `static_assert` ABI pin to libVLC 3.0.21, and a `vlc_object_t` reinterpret of the
  player. Linux/macOS get nothing.

### 3.4 Is it worth remaking?

**No, not as a plugin. Yes, as ~20 lines in the Rust mixer's output stage.** The entire reason
this became three modules, a MinGW toolchain and a 4-variable clock/epoch handshake is that the
*plugin process* owned the WASAPI endpoint, so the viewer could neither mix the music nor know
when a fade had been heard. When the Rust mixer owns an N-channel output stream, speaker fill
is a fixed matrix applied to the music bus after HRTF rendering of the spatial buses:
`FL = FR = 0.7071`, `SL/RL`, `SR/RR` = 0.7071, `C = 0.3536(L+R)`, `LFE = 0`, gated by (a) the
negotiated channel mask, (b) a headphone/headset form-factor check, (c) the user's layout
choice with the rear-vs-side preference rule. The fade envelope, cubic gain, 50 ms gain
smoothing and "completion = final sample passed through the device clock" all collapse into
the mixer's own sample clock plus the output stream's reported latency. The value to keep is
the **behavioural spec** (numbers above, test cases in `vlc_speakerfill_test.cpp` and the
protocol tests), not any code.

**Parcel-music decoding in-process instead of VLC.** What SL parcel streams look like in the
wild (background knowledge, not derivable from this repo): overwhelmingly Icecast/SHOUTcast
HTTP(S) with MP3, then AAC/AAC+ (HE-AAC v1/v2 is common on SHOUTcast "aacp" stations), some Ogg
Vorbis and a growing minority of Ogg Opus (Icecast/AzuraCast); FLAC rarely; HLS essentially
absent (the land media URL field is a bare stream URL, and the historical FMOD-based viewers
could not play HLS, so parcel owners do not set it). Many URLs are `.pls`/`.m3u` playlist
indirections or redirect, and SHOUTcast v1 answers with the non-HTTP `ICY 200 OK` status line.
The plugin doc also warns "music URLs can redirect to video and audio/* may be wrong"
(`doc/vlc_audio_viewer.md:238-240`).

What symphonia covers: MP3, AAC-**LC**, Vorbis, FLAC, chained Ogg demux, ADTS, ISO-MP4. What it
does **not** cover and you must add: HE-AAC/SBR/PS (symphonia's AAC is LC only; this is the
real gap for "aacp" stations; options are fdk-aac bindings, ffmpeg, or accepting LC-only), Opus
(`opus`/`audiopus` crate or a pure-Rust decoder), WMA (legacy Windows Media stations; probably
droppable), the ICY handshake (`Icy-MetaData: 1`, `icy-metaint` interleaved metadata blocks,
tolerating `ICY 200 OK`), playlist resolution, redirects/TLS, reconnect-with-backoff, and the
"was that actually a video stream" guard. VLC gives all of that plus a battle-tested network
cache; symphonia gives you the samples *in your mixer* with sample-accurate fades, no IPC, no
generation/serial protocol, no plugin-process priority buffering (`:network-caching=5000`),
and metadata without a side channel for Ogg. For a mixer-owned design the trade is clearly in
favour of in-process decoding, with HE-AAC as the one codec decision to make deliberately.

**Video (MOAP) still needs a decoder regardless.** Nothing in this branch qualifies VLC video
audio under the viewer's control: "Video/browser sources remain on native VLC/CEF and are not
claimed as PCM/A-V qualified" (`doc/vlc_audio_viewer.md:50-52`). vkstorm-aleph will need *some*
video pipeline (libvlc via `vlc-rs`, GStreamer, or ffmpeg) for prim video, and for that pipeline
the right pattern is the one this branch retired for reasons that no longer apply: pull PCM out
of the decoder (libvlc `amem` S16/FL32 callbacks, or a GStreamer appsink) into the Rust mixer so
video audio, distance attenuation and speaker fill all go through one device. The endpoint
fence problem that killed the PCM path (`doc/vlc_audio_engine.md:65-99`) does not exist when
the mixer, not the plugin, owns the device.

---

## 4. Keep / drop table

| Area | Item | Verdict | Re-specify as | Spec source |
|---|---|---|---|---|
| Effects | Endpoint-ID device identity, 5 s list refresh, change signal | **Keep** | Device registry in the audio crate keyed on WASAPI endpoint ID / PipeWire node id | `llaudioengine_soloud.cpp:244-272, :450-457`; `doc/soloud_audio.md:39-43` |
| Effects | 1 s device-mutex timeout, 1 s retry, unknown endpoint -> no change, failed reopen -> restore previous, no engine fallback | **Keep** | Device-switch state machine | `:298-338, :446-449`; `doc/soloud_audio.md:44-47` |
| Effects | Restart invalidates all voice handles and detaches buffers; stream/media update continues while device down | **Keep** | Mixer restart contract | `:355-385, :456-462`; test `:183-198` |
| Effects | Native rate, 1/2/4/6/8 channel negotiation; 7.1 order back pair (z=-1) then side pair (z=0) | **Keep** | Output-format negotiation + channel map table | `:183-189`; test `:117-124`; `doc/soloud_audio.md:54-55` |
| Effects | Inverse-distance, min 1 m / max 512 m, rolloff forwarded; Doppler factor + velocities forwarded | **Keep** (HRTF renderer replaces panning, keep the attenuation law) | Source/listener parameter set | `:68, :641-656, :805-814, :841-842`; `lllistener_soloud.cpp:50-56` |
| Effects | Create-paused-then-configure; 64-sample baked head/tail ramp; 20 ms declick stop; 20 ms synced-slave fade-in; `fmod` seek for sync | **Keep** | Voice start/stop ramps | `:563-565, :597-668, :712-728, :849-860` |
| Effects | Loop count detection, inaudible voices keep ticking, UI sounds 2-D | **Keep** | Voice semantics | `:617-621, :662, :779-791` |
| Effects | Max 64 active voices, per-frame call throttling | Keep as defaults | Mixer config | `:181, :582-605` |
| Wind | LLWindGen + atomic targets, 512-frame chunks, 0.1 s update, freq/gain/pan mapping, coordinate flip | **Keep** | Wind DSP node on the ambient bus | `:71-137, :542-556`; `llaudioengine.h:51` |
| Effects | SoLoud globals, `ma_context` plumbing, Soloud recreate workaround, OPENAL undef, heartbeat log | **Drop** | none | `:40-65, :375-378, :468-482`, `h:31-35` |
| Effects | Headless adapter test shape (null device, 2/6/8 ch, 1024-frame mix with guard samples, gains, restart x5) | **Keep** | Rust unit tests | `llsoloud_adapter_test.cpp:97-203` |
| Music | Fade policy: enable flag, 3.0 s in / 2.0 s out, 0.01 s min, 60 s cap, login hold at 0 then one full fade | **Keep** | Music-bus envelope spec | `llvieweraudio.cpp:109-120, :203-212, :318-353`; `settings.xml:23111-23142`; `scripts/perf/test_login_music_gate.py` |
| Music | Completion = final target sample has passed the device clock; buffering/paused/drained never count as silent; failure/cancel => hard stop, never implicit replay | **Keep** (as mixer-internal guarantees) | Envelope completion callback | `llpluginclassmedia.cpp:1800-1813`; `llvieweraudio.cpp:187-201`; `doc/vlc_audio_viewer.md:11-33` |
| Music | Gain = master x music applied once; hard-mute latch separate from envelope; gain changes do not restart the envelope | **Keep** | Bus gain model | `llvieweraudio.cpp:620-634`; `doc/vlc_audio_viewer.md:47-49` |
| Music | Cubic user-volume curve + 50 ms smoothing | Keep (number) | Gain smoothing | `llvlcspeakerfillconfig.cpp:41`; `vlc_speakerfill.cpp:103-105` |
| Music | `media_audio`/`media_music` IPC, generation/serial tokens, packed 64-bit fade command, VLC variable handshake, process-exit = Cancelled | **Drop** (no plugin process); keep only as the *test-case list* for the in-process envelope | none | `llviewermedia_audio_test.cpp`; `doc/vlc_audio_viewer.md:136-201` |
| Music | Deep buffering for background-priority plugin (`network-caching=5000`), FMOD 7000/1000 ms | Drop the numbers, keep the lesson: music ring buffer of several seconds, tuned to your own thread priority | none | `media_plugin_libvlc.cpp:651-662`; `llvieweraudio.cpp:54-55` |
| Music | Speaker-fill matrix (0.7071 mains, 0.3536 centre, LFE 0, passthrough for native multichannel) | **Keep** | ~20-line output-stage matrix | `vlc_speakerfill.cpp:58-73, :166-173` |
| Music | Layout selection: 0/21/41/51/71, rear-else-side rule, request subset of endpoint mask, headphone/headset bypass, "applies on next stream start" (can become immediate) | **Keep** | Layout negotiation rules + user option | `prepare_speakerfill.cmake:104-136, :149-185`; `settings.xml:906-916`; `panel_preferences_sound.xml:782-807` |
| Music | VLC output-module patching, MinGW build, hash fetch, libVLC 3.0.21 ABI pin, `vlc_object_t` reinterpret | **Drop** | none | `prepare_speakerfill.cmake`; `CMakeLists.txt:34-70`; `llvlcspeakerfillconfig.cpp:17-18` |
| Music | Speaker-fill test cases (mask negotiation, failure publishes no mask, exact samples, LFE silence, passthrough, fence timing) | **Keep** as test list | Rust tests | `tests/vlc_speakerfill_test.cpp:185-321, :400-590` |
| Metadata | ICY `NowPlaying` "Artist - Title" split; Ogg Title/Artist; URL-filename placeholder suppressed; dedup; clear on start/stop | **Keep** | In-process ICY/Ogg metadata extraction | `media_plugin_libvlc.cpp:700-778`; `llviewermedia_streamingaudio.cpp:294-330` |
| Metadata | Icecast `status-json.xsl` sidechannel: 5 s first poll, 20 s interval, 3-strike give-up, mount path walk, listenurl match, sidechannel wins | **Keep** | HTTP metadata poller | `llviewermedia_streamingaudio.cpp:333-520` |
| Object media | `audio/*` MIME -> object/nonpositional role; direction = (-dir.left, dir.at) with ear-location option and HUD exemption; distance gain stays viewer-side | **Keep the intent, upgrade** to a real 3-D source on the HRTF bus (dormant in the release build anyway) | Media-source position feed | `llviewermedia.cpp:1968-1971, :2027-2029, :2244-2268` |
| Video | VLC/CEF native audio output for video/browser | **Replace**: route decoder PCM into the mixer (`amem`/appsink) | Video-audio bus | `doc/vlc_audio_viewer.md:50-52`; retired design `doc/vlc_audio_engine.md` |
| Retired | `llvlcaudio*`, `llvlcaudiobridge*`, `llvlcwasapi.h`, `llvlcendpoint.h`, `prepare_miniaudio.cmake`, bridge/engine tests | **Drop** (compiled out; its endpoint-fence problem vanishes when the mixer owns the device) | none | `CMakeLists.txt:32`; `doc/vlc_audio_plugin.md:3-8` |
| UI | `indra/newview/llfloatersounddevices.cpp` (restored LL voice-device floater, no engine logic) | Not audio-engine work; ignore | none | diff vs upstream |

### Doc / test files that serve as the spec

- `doc/soloud_audio.md` (behaviour lines 33-58, validation limits 62-83)
- `doc/vlc_audio_viewer.md` (current contract 1-92; wire table 146-155; semantics 156-201)
- `doc/vlc_audio_plugin.md` (speaker fill 1-96; native fades 98-131; wire/state 200-224)
- `doc/vlc_audio_engine.md` (retired; useful only for the "submitted is not heard" reasoning 41-99)
- `indra/llaudio/tests/llsoloud_adapter_test.cpp`
- `indra/newview/tests/llviewermedia_audio_test.cpp`
- `indra/media_plugins/libvlc/tests/vlc_speakerfill_test.cpp`
- `scripts/perf/test_login_music_gate.py`
- `indra/newview/app_settings/settings.xml` keys `FSMusicSpatialSound`, `FSFadeAudioStream`,
  `FSAudioMusicFadeIn`, `FSAudioMusicFadeOut`, `MediaSoundsEarLocation`
