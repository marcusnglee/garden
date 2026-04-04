A hybrid fixed/live composition using federated learning across campus field recordings, driven through Max for Live into Ableton Live.

---

## Concept

Record the acoustic atmosphere of several campus locations. Train a federated autoencoder across those recordings — without centralizing any audio — to learn a shared acoustic grammar of campus. Extract per-location embeddings and use them to drive a fixed electronic composition in Ableton. During the installation/presentation, add a live inference layer that listens to the room in real time and modulates the pre-composed bed.

**Core tension:** The global model = campus as institution (shared, anonymous acoustic identity). Local embeddings = specific human experience of each place. The composition enacts the tension between the two.

---

## Conceptual Frames

- **Memory vs. presence** — the fixed composition is a crystallized memory of campus; the live layer is the room talking back to its own memory
- **Convergence as narrative** — FL training rounds move from divergence (chaos, individuality) toward convergence (shared structure); this trajectory can be the piece's temporal arc
- **Acoustic geography** — after training, acoustically similar locations cluster in latent space without being told to. That self-organized map of campus is compositional material
- **Privacy as aesthetic** — only gradients travel between nodes, never raw audio. The piece is built from what places _share_, not what they _sound like_

---

## Locations (draft)

|Location|Expected character|Reverb|Transient density|
|---|---|---|---|
|Library|Quiet, studious|Long|Low|
|Dining hall / student union|Busy, social|Mid|High|
|Outdoor courtyard|Open, wind, distant voices|None|Medium|
|Hallway / stairwell|Boxy, footsteps|Short|Medium-low|
|_(optional 5th)_|TBD|—|—|

**Recording target:** 20–30 min continuous per location, consistent mic placement.

---

## Technical Architecture

### Each node (Raspberry Pi 4, 4GB)

- USB audio interface (e.g. Behringer UMC22) — onboard Pi audio is too noisy
- `python-sounddevice` + `numpy` for audio capture
- `librosa` or custom FFT for mel spectrogram computation (128 mel bins × 64 time frames per window, ~1.5s)
- Saves spectrogram `.npy` files to disk during recording phase
- During installation: runs inference → extracts atmosphere parameter vector → sends via OSC to Ableton machine

### Federated training (offline, on laptop)

- Model: small convolutional autoencoder
    - Input: 128 × 64 mel spectrogram window
    - Encoder: 2 conv layers → flatten → dense → **8-dimensional latent vector**
    - Decoder: mirrors encoder
    - Loss: MSE reconstruction
    - ~50,000 parameters total
- Process: pull `.npy` files from each Pi → `local_train.py` per node → `aggregate.py` (FedAvg) → redistribute global weights → repeat
- **10–20 rounds** is sufficient; full run takes ~30 min on laptop
- Stack: PyTorch for training, export to ONNX, run `onnxruntime` on Pi for inference

### What training produces

- **Global model** — shared acoustic grammar across all locations; represents what campus spaces have in common
- **Per-location embeddings** — 8-dimensional vectors encoding how each space deviates from the global average
- **Atmosphere parameters** extracted from embeddings:
    - Transient density
    - Spectral centroid ("brightness")
    - Reverb tail estimate
    - Low-frequency energy (HVAC, traffic)
    - Model divergence from global (how acoustically unique the space is)

### Ableton / Max for Live

- Parameter vectors loaded as automation data or M4L LFO sources
- OSC bridge: Python (`python-osc`) → `udpreceive` in Max → mapped to Ableton parameters
- Key instruments:
    - **Granulator II** — grains from actual field recordings, density driven by transient density parameter
    - **MIDI / clip triggering** — transient density → rhythm and clip length
    - **Reverb wet/dry** — driven by reverb tail estimate
    - **Filter cutoff** — driven by spectral centroid
- High divergence locations → more harmonic dissonance, more solo space
- Low divergence → blend into shared textural beds

---

## Hybrid Fixed / Live Structure

### Fixed composition (weeks 1–6)

The trained model's understanding of campus, crystallized. Moves between locations using embeddings as waypoints — musically "traveling" from library to courtyard to dining hall. The piece has a defined arc.

### Live layer (weeks 7–8, added on top)

One Max for Live device runs inference on a mic in the presentation room in real time. Outputs gentle modulation over the pre-composed bed. The composition plays regardless of the live layer's stability — any instability is graceful degradation, not failure.

**The live layer makes the installation self-aware** — it hears the audience and responds.

---

## Timeline

|Week|Focus|
|---|---|
|1–2|Field recording at all locations. Mel spectrogram extraction.|
|3–4|FL training pipeline. Feature extraction. Embedding analysis.|
|5–6|Composition in Ableton. Parameter mapping. Fixed piece locked.|
|7–8|Live M4L inference device. Rehearsal. Installation setup.|

---

## Open Questions / Brainstorm Threads

- [ ] How many FL rounds before embeddings stabilize? Worth visualizing convergence per location
- [ ] Should the FL round trajectory itself be audible in the composition? (early rounds = dissonance, late rounds = resolution)
- [ ] What does "divergence" sound like? Map to: dissonance, grain scatter, filter resonance, rhythmic irregularity?
- [ ] Spatial audio — if presenting in a room, can location embeddings map to spatial position (ambisonics / stereo field)?
- [ ] Is there a visual component? Latent space clustering visualized in real time would be a strong installation element
- [ ] How to handle the OSC latency budget — target <50ms from mic to parameter change
- [ ] Recording ethics — ambient recording in public campus spaces, any consent considerations?
- [ ] Artist statement angle: privacy-preserving ML as a metaphor for how institutions flatten individual experience into aggregate identity

---

## Key Tools & Libraries

|Tool|Purpose|
|---|---|
|`librosa`|Mel spectrogram computation|
|`python-sounddevice`|Audio capture on Pi|
|`PyTorch`|Local model training|
|`onnxruntime`|Fast inference on Pi|
|`python-osc`|OSC bridge to Max|
|`numpy`|Spectrogram storage / manipulation|
|Max for Live|Parameter mapping, live inference device|
|Granulator II|Granular synthesis from field recordings|
|Ableton Live|DAW, automation, composition|

---

## References / Inspiration to Explore

- Alvin Lucier — _I Am Sitting in a Room_ (room as instrument, acoustic feedback as composition)
- Jana Winderen — field recording practice, texture-as-music
- Federated learning: McMahan et al. 2017 "Communication-Efficient Learning of Deep Networks from Decentralized Data" (original FedAvg paper)
- Granular synthesis: Curtis Roads — _Microsound_
- Acoustic ecology: R. Murray Schafer — _The Tuning of the World_