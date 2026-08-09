# H3 Motion Context

Chain MiniMax H3 clips together so that motion **and sound** continue across
the joins, instead of every clip re-deciding what's happening from a single
still frame.

Generate clip A. Feed its last frames and audio into this node. Generate
clip B. B picks up exactly where A left off: same motion, same direction and
speed, and the same audio - not similar audio, the *same waveform*,
continued. Repeat down a chain as long as you like.

This is done with inline, marker-gated runtime patches. No ComfyUI files are
edited on disk, and importing this node pack does not patch ComfyUI. The first
`H3 Motion Context` execution validates and activates the hooks; H3 workflows
that do not use Motion Context remain on stock behavior, including workflows
queued later in the same ComfyUI process. If a future ComfyUI update changes
something underneath, the opted-in node refuses to run rather than quietly
rendering something wrong.

## How is this different from LTX motion context?

LTX ships clip-chaining as a built-in feature: you pin frames from the
previous clip into the latent and the model continues them. H3 has no such
feature - but it turns out the machinery was already there. H3's keyframe
system tags frames with a time coordinate and re-injects them at every
sampling step. The only thing preventing a *run* of consecutive frames was
a single check in ComfyUI that rejected any keyframe that wasn't the first
or last frame. Mathematically, the position formula already worked for every
frame in between. This project lifts that restriction (and verifies its own
math against ComfyUI's when Motion Context first opts in).

The bigger difference is audio. H3 generates picture and sound together,
and this carries **both** streams across the join. Getting audio to
genuinely continue - rather than the model playing a sound-alike - turned
out to be the hard part, and the fix is the most interesting thing in the
repo (see "The audio story" below).

## Install

Drop the folder into `ComfyUI/custom_nodes/` and restart. Merely loading the
pack only registers its nodes. The first time a workflow executes
`H3 Motion Context`, watch the console for:

```
h3_motion_context: interior keyframe anchors enabled
h3_motion_context: keyframe/ref coexistence enabled
```

If a self-test fails instead, the reason is logged and that opted-in node
refuses to run. That's deliberate: a loud failure beats a subtly wrong render.

## Wiring

```
MiniMaxH3ImageToVideo / MiniMaxH3ReferenceToVideo (or the t2v path)
  -> H3 Motion Context      <- previous clip's frames + audio
  -> guider / sampler
  ...
  decoded IMAGE + AUDIO
  -> H3 Motion Context Trim         <- wire trim_frames across
  -> Create Video / save
```

Feed `context_frames` the decoded frames of the previous clip. For audio,
the best source is the previous clip's latent - but note you **cannot**
wire the sampler's output directly into `context_latent`; ComfyUI will
flag a circular connection, correctly, because the latent you need is from
the previous *run*, not the current one. Two helper nodes carry it across
runs the same way you already carry frames and audio through saved files:

```
this run:   SamplerCustomAdvanced -> H3 Motion Context Save Latent
next run:   H3 Motion Context Load Latent -> context_latent
```

Both nodes have a `clip_index`, and the numbers mean exactly what they
say: on the Load node, the clip to CONTINUE FROM; on the Save node, the
clip THIS is. Generating clip 2 from clip 1: Load 1, Save 2. Don't like
the result? Queue again and change nothing - the retry reloads clip 1 and
overwrites clip 2's reject. Accept it, bump both numbers, move on. Files
get the natural names (`clip_00002.safetensors` is clip 2). At the
default of 0 the loader instead takes the newest file in the folder,
which is NOT retry-safe - a re-roll loads its own rejected audio - and
auto-saved files are numbered by RUN, not clip, marked by a trailing
underscore (`clip_00002_.safetensors`) so indexed loading never confuses
them for real slots. Leave context unwired for clip 1. The loader can
also point straight at a specific file, which ignores the index. (Stock Save/Load
Latent won't work here; it can't handle H3's paired video/audio latent.)
The loader's output is only for `context_latent`; don't wire it into a
decode node. The older path - decoded audio into `context_audio` with the
H3 audio VAE in `audio_vae` - still works and is used when no latent is
wired; it costs one extra lossy VAE round trip per link (see Limitations).
Wire the `trim_frames` output into the Trim node so the duplicated head -
picture and sound together, in sync - comes off before you concatenate.

For **Ref2VA/R2V**, connect the conditioning and latent from the stock
`MiniMaxH3ReferenceToVideo` node exactly the same way. Motion Context preserves
its existing image, video, and audio references, then appends the continuation
audio as the final reference block. This ordering matters: older versions of
this repo replaced `minimax_refs`, which silently dropped the R2V references,
and the layout patch rejected the resulting multi-reference audio setup. Both
paths are now handled inside the Motion Context node; no separate patch script
or ComfyUI-core edit is required.

The Ref2VA multi-reference/audio compatibility
[fix](https://discord.com/channels/1076117621407223829/1535700117452226560/1535771676158206032)
and [six-clip global-ref demo workflow](https://discord.com/channels/1076117621407223829/1535700117452226560/1535771814452793474)
were contributed by **seitanism** in the Banodoco MiniMax H3
seamless-extension thread.
They are included here with attribution; this repo activates the shared
compatibility behavior inline when Motion Context executes, so users do not
have to run its external patching script and unrelated H3 workflows stay stock.

Motion Context also preserves a stock `first_frame`/`last_frame` anchor from
`MiniMaxH3ImageToVideo` (FL2VA) the same way: instead of replacing
`minimax_keyframes`, it appends its own pinned run after whatever anchors are
already there. This lets you pin the real, decoded final frame of a segment
with `last_frame` while Motion Context pins continuity from the previous
clip's tail - useful when you want every segment's end to land on a real
frame instead of a purely generated one. If an audio-continuation ref
(`context_latent`/`context_audio`) is wired at the same time, the stock
anchor is also given the layout patch's positioning tag so its coordinate
picks up the same shift the ref gives the rest of the timeline; without that,
the anchor would keep its un-shifted stock position and drift out of sync
with the rest of the clip.

## Automated disk-backed chains

The `H3 Chain` nodes turn a repeated Ref2VA graph into one recursive sampling
body. They are specialized for MiniMax rather than generic carry-value loop
nodes: shot lengths are placed on H3's `17k+5` frame grid, source-song windows
are computed from the delivered frame timeline, clip 1 bypasses Motion Context,
and every later clip receives the preceding frame tail and optional AV latent.
The recursion engine is a self-contained MiniMax adaptation of Ethanfel's SxCP
loop implementation; ComfyUI-Prompt-Builder is not a runtime dependency.

The graph shape is:

```
H3 Chain Plan -> H3 Chain Loop Start -> H3 Chain Current Shot
                                         | prompt / seed / length / audio slice
                                         v
                              MiniMaxH3ReferenceToVideo
                                         v
                                  H3 Chain Context
                                         v
                              guider / sampler / decode
                                         v
                               H3 Motion Context Trim
                                  |                |
                                  v                v
                     H3 Chain Segment +       H3 Chain Loop End
                         Checkpoint  ---------->    |
                                                   | recurse
                                                   v
                                          next planned clip

H3 Chain Loop End.manifest -> H3 Chain Assemble
```

Wire the original song into Loop Start, Current Shot, and Assemble when using a
source-track mode. All three nodes verify the same full-waveform hash, so a
miswire cannot render or mux a different song. Source audio must cover the full
planned video duration; a short track fails before sampling instead of silently
truncating the final video.

Wire `Loop Start.state` into `Current Shot.state`, then use the pass-through
`Current Shot.state` for Chain Context, Segment + Checkpoint, and Loop End.
Wire the Start's `flow` directly to Loop End. The stock Ref2VA node receives
the Current Shot outputs for `prompt`, `width`, `height`, `length`, and its
first standalone audio-reference socket. `noise_seed` goes to Random Noise and
`steps` goes to Basic Scheduler. The sampler's raw output goes to both decode
nodes, Segment + Checkpoint, and Loop End. Trimmed images go to Segment +
Checkpoint and Loop End; trimmed audio goes to Segment + Checkpoint.

The plan is compact JSON. Global prompt text can live in `prompt_prefix`; each
shot supplies only what changes:

```json
{
  "prompt_prefix": "Global subject and continuity instructions.",
  "defaults": {"duration_seconds": 15, "steps": 20},
  "shots": [
    {"id": "intro", "prompt": "Opening shot.", "seed": 123},
    {"id": "street", "prompt": "Continue into the street.", "seed": 456},
    {"id": "outro", "prompt": "Finish the take.", "duration_seconds": 5}
  ]
}
```

You may specify an exact `length` instead of seconds, but it must satisfy
`length % 17 == 5`. Missing seeds are deterministically derived from
`base_seed`, clip index, and shot id. Resolution and context settings are
global and checkpointed; changing them invalidates resume. Set
`generation_fingerprint` to a stable version tag for the external model, VAEs,
global references, CFG, and scheduler, and change that tag whenever any of those
inputs changes. It becomes part of the resume compatibility hash.

### Segments and resume

Every iteration writes these artifacts under
`output/h3_chains/<run_name>/` before the next clip begins:

- `segments/clip_0001.<transaction>.mp4`: video-only H.264 segment;
- `checkpoints/clip_0001.<transaction>.safetensors`: context frames, AV latent,
  and delivered generated audio;
- `checkpoints/clip_0001.json`: prompt/seed/timing hashes and artifact manifest.

The JSON file is the atomic commit point. A retry first writes a new immutable
video/checkpoint pair and their SHA-256 hashes, then switches the fixed JSON slot
to that pair in one rename. An interruption cannot combine a new video with an
old latent checkpoint; successful retries clean the superseded pair.

The loop carries only the last context frames and compact AV latent in memory;
it never builds the multi-gigabyte cumulative IMAGE tensor used by manually
duplicated long-chain workflows. `H3 Chain Assemble` stream-copies the video
segments and muxes audio only at the end.

To resume at clip N, keep the same `run_name` and set `start_clip=N`. The Start
node loads clip N-1, verifies every predecessor against the current plan, and
continues. Prompts for clip N and later may be changed. Any change to an earlier
prompt, seed, duration, resolution, context setting, or audio mode is rejected
until those earlier clips are regenerated. Changing the source song is also
detected from its waveform hash. Re-running a clip overwrites its fixed
segment/checkpoint slot, so rejected attempts do not accumulate.

If all clips were saved but the browser, Loop End, or final assembly stopped,
connect the same Plan (and source song when applicable) to `H3 Chain Load
Completed Manifest`. It validates every artifact and rebuilds the manifest for
Assemble without sampling the last clip again.

### Chain audio modes

- `source_track`: Current Shot slices the uploaded song for every Ref2VA clip;
  Motion Context carries video only; Assemble muxes the original song. This is
  the recommended music-video mode.
- `generated_audio`: no source reference is needed; Chain Context carries the
  previous raw audio latent on the timeline and Assemble concatenates the
  checkpointed generated audio. Segment + Checkpoint requires trimmed decoded
  audio in this mode and rejects any sample count that does not exactly match
  the delivered video frames.
- `source_plus_timeline`: supplies both the source-song Ref2VA window and the
  preceding generated audio latent. This is experimental; Assemble uses the
  source track by default.

Segment saving requires the PyAV/libx264 support shipped with normal ComfyUI
installations. Final assembly requires `ffmpeg` on `PATH`.

## Settings and what to pick

**context_length** - how many frames of the previous clip to carry over.
The video VAE only distinguishes certain run lengths, so useful values are
**5, 22, or 39**; anything else snaps down to the nearest. 5 is just barely
fluid, 22 is nearly seamless, 39 is untested. **Use 22.**

**encode_mode** - `video` (default) encodes the whole run in one VAE call
so the motion lives inside the latent. `frames` encodes each frame as a
separate still; it costs twice the rows and left a visible seam in testing.
**Use video.** `frames` remains only for comparison.

**anchor_mode** - `head` (default) pins the frames at the start of the
clip; they come back in the output and the Trim node removes them. `before`
places them at negative time instead so nothing needs trimming - but its
coordinates collide with the text conditioning, which weakens the anchors
and consistently darkens output, failing subtly rather than loudly.
**Use head.** `before` remains only so the failure can be reproduced.

**audio_mode** - `timeline` (default) places the pinned audio on the new
clip's own timeline so the model continues it. `ref` is the stock
placement, which the model *imitates* instead - similar music, not the
same recording, and an audible tick at every join. **Use timeline.**
`ref` remains only for comparison.

**audio_context_length** - how much tail audio to pin, in frames,
independent of the video window. It is end-aligned with the pinned video,
so both always finish at the same instant (the join) and this only controls
how far back the sound reaches. **Use 22** to overlay the video window
exactly; that's the tested config. Longer windows (44, 96) are legal and
land in safe coordinate space, but nobody has rendered one yet.

The Trim node also has `match_tail` (default on). Leave it on: H3 rounds
its audio grid up, so every clip carries ~8 ms more sound than picture, and
without the trim that error grows at every join in a chain.

## The audio story

The first version put pinned audio through H3's reference mechanism, which
is where audio conditioning normally lives. Joins had a small tick - the
audio seemed to briefly speed up and go offbeat. Waveform inspection showed
no splice error; both sides of every join were individually smooth.

Cross-correlating each clip's opening against the previous clip's ending
(the `tests/seam_probe.py` script in this repo) revealed the real problem: the
new clip's audio *resembled* the old clip's - same instruments, same
groove - but never matched it. A cover band, not the same recording. The
model was treating the reference as "a separate clip that sounds like
this," which is exactly what references are for, and exactly wrong for
continuation.

The fix mirrors what already worked for video: the rows the model sees are
identical between the two mechanisms; only their **time coordinates**
differ, and the coordinates are what tell the model "separate clip" versus
"this clip, earlier." So the pinned audio keeps riding the reference
machinery for construction, and its coordinates are rewritten onto the new
clip's own timeline, ending exactly where the pinned video ends. After the
change, measured correlation at the joins went from ~0.45 with incoherent
timing to 0.95+ with a flat, stable offset, and the tick disappeared. The
same measurement across a multi-clip chain shows the offset does **not**
grow from join to join - each clip re-anchors from absolute positions, so
timing errors don't compound.

`tests/seam_probe.py` is included. Point it at the previous clip's audio and the
new clip's **untrimmed** audio and it scores the join:

```
python tests/seam_probe.py clipA.flac clipB_untrimmed.flac --frames 22 --win-ms 100 --search-ms 60
```

## Limitations

**Sound quality degrades down a chain.** This is the big one. Each clip's
audio is generated from the previous clip's *output*, which was generated
from the clip before it, and so on. Like photocopying a photocopy, losses
compound, and (like most lossy audio compression) the top end goes first.
In practice: timing and tempo stay locked, but after several clips the
audio gets noticeably duller and more muffled. Video degrades far less
visibly. Two loss sources stack per link: the model's own regeneration
smoothing, and an extra pass through the audio VAE's encode/decode cycle.
The `context_latent` input eliminates the second one by slicing the pinned
audio straight from the previous clip's latent - wire it and the VAE
round trip is gone. How much of the muffling that removes is newly
measurable, not yet established; the model's own smoothing remains either
way. Whatever remains: treat long chains as territory to listen to
critically, and consider placing chain restarts at natural musical
transitions where a fresh start won't be noticed.

**A small constant audio offset.** Measurement shows each context-generated
clip's audio sits a fixed ~10 ms late. It is constant - it does not grow
down the chain and does not affect tempo - and it is below the threshold
where lip-sync errors become perceptible, but it is real and unfixed.

**Testing breadth.** Joins have been verified clean on two very different
kinds of material: dense beat-driven electronic music (where timing errors
are most audible) and spoken word via the latent path (where nothing masks
a seam and the ear is least forgiving about artifacts).

**One machine, one configuration.** Everything here was verified on a
single Windows machine at one resolution with one sampler. The math is
self-tested on first inline activation; the perceptual results are one person's
renders.

**ComfyUI's H3 support is young and moving.** The patches depend on the
current shape of ComfyUI's H3 code. They verify those assumptions when an
H3 Motion Context path first opts in and shut down loudly if anything changed,
so the failure mode is
"the node refuses to run after an update," not corrupted output.

**Turn Spectrum off.** Step-skipping optimizers like
ComfyUI-Spectrum-MiniMax-H3 forecast how the model's state evolves across
steps. Pinned rows never evolve, which is a degenerate case for the
forecaster. Keep it disabled for these graphs.

**License.** The H3 community license reportedly does not currently cover
the EU, UK, Korea, or the US. Verify independently before building
anything shipping on this.

## Recommended starting point

`context_length 22, encode_mode video, anchor_mode head, audio_mode
timeline, audio_context_length 22`, Trim node wired for both picture and
sound with `match_tail` on, Spectrum off. That is the configuration every
"it works" claim in this README refers to.

## Status and testing

Built and verified against ComfyUI master as of early August 2026, while
H3 support was days old. Importing the pack is side-effect free; the math
patches self-test against the live ComfyUI code on first Motion Context
execution, and remain behaviorally gated to its private conditioning markers.
An upstream change surfaces as a clear refusal, not a bad render. The repo also
ships two standalone patch/node tests
that run without ComfyUI or a GPU (only numpy needed), plus a CPU chain
integration test against an adjacent ComfyUI checkout:

```
python tests/_mock_harness.py       # patch logic against a faithful stock model
python tests/_node_smoke_test.py    # the node end to end, R2V refs + save/load
python tests/_chain_smoke_test.py   # timing, recursion, segments, resume, assemble
```

All three should print their checks and finish with a pass line.

## Files

| File | Role |
|---|---|
| `patch_layout.py` | Marker-gated wrapper that lifts the first/last-only keyframe restriction, moves pinned audio onto the clip timeline, and keeps R2V refs aligned. Self-tests on first inline activation. |
| `patch_payload.py` | Marker-gated wrapper that lets Motion Context video and refs coexist while leaving unmarked H3 payloads stock. |
| `nodes.py` | The four nodes: Motion Context, Trim, and the latent Save/Load pair. |
| `chain_nodes.py` | The eight MiniMax-specific plan, recursive loop, segment/checkpoint, resume/manifest recovery, and assembly nodes. |
| `tests/seam_probe.py` | Measures whether a join's audio is a true continuation, a sound-alike, or drifting. |
| `tests/` | Standalone patch/node tests plus the CPU chain integration test. |

The `example_workflows/` folder contains both the original compact FL2VA demo
and seitanism's six-clip Ref2VA/global-reference chain. See its README for the
additional workflow dependencies.
