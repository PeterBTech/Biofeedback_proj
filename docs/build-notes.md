# Biofeedback Engine — Build Notes

Working reference for the real-time EEG pipeline project.
Last updated: 31 July 2026

---

## Contents

1. [Project overview](#1-project-overview)
2. [The four phases](#2-the-four-phases)
3. [Environment setup](#3-environment-setup)
4. [Terminal reference](#4-terminal-reference)
5. [Git reference](#5-git-reference)
6. [Day 1 findings](#6-day-1-findings)
7. [The three rules](#7-the-three-rules)
8. [Corrections to the original plan](#8-corrections-to-the-original-plan)
9. [Design decisions](#9-design-decisions)
10. [Real EEG data (Phase 1.5)](#10-real-eeg-data-phase-15)
11. [README template](#11-readme-template)
12. [Schedule](#12-schedule)

---

## 1. Project overview

A real-time pipeline that takes EEG voltage measurements, works out which
brain rhythms are present, decides what state the person is in, and displays
that back to them as ambient colour and motion.

The chain runs in one direction:

```
data source → ring buffer → filters + spectral analysis → classifier → visual feedback
```

Each stage is built and verified in isolation before the next one is allowed
to depend on it. The reason is that failures propagate silently: a buffer
handing out duplicated samples produces a plausible-looking but wrong
frequency spectrum, and you will spend days suspecting the filters.

**Repo:** `~/VSCode/Biofeedback_proj`

---

## 2. The four phases

### Phase 1 — Getting data in and holding it

Get measurements arriving continuously into the program, and always keep the
most recent two seconds in memory. Nothing older kept, nothing newer missed.

The problem: an EEG device is a firehose, not a file. It produces numbers 250
times per second per electrode and never stops. You cannot keep everything —
sixteen electrodes at 250 Hz is millions of numbers per hour, forever — and
you do not want to, because a measurement from forty minutes ago tells you
nothing about the person's state right now.

Two things get built. A **source**, which owns the hardware connection and
exposes a handful of simple operations, so nothing else in the program ever
touches the hardware library directly. And a **ring buffer**, which is the
intellectual core of the phase.

The ring buffer is a fixed number of slots arranged conceptually in a circle,
plus a marker showing where the next value goes. When the marker reaches the
last slot it wraps to the first, overwriting the oldest value. The array is
allocated exactly once and never grown or replaced; new samples are written
directly into existing memory. The physical order becomes scrambled as the
marker wraps, but the marker's position tells you which slot is oldest, so
correct chronological order can always be reconstructed on demand. Result: a
constantly updating window over the recent past, with no memory growth and no
garbage collection pauses in the real-time path.

Two silent failure modes to avoid. Asking the library for "the last 500
samples" rather than "whatever is new" returns mostly data you already
processed — your window stops representing two seconds of time and starts
representing half a second smeared five times over, with nothing crashing.
And stacking new samples onto old then trimming, which is the obvious thing to
write, quietly allocates a fresh array ten times a second forever.

**Done when:** shape locks at `(16, 500)` and never changes, iteration time
stays under a millisecond, and memory is flat in Activity Monitor for five
minutes.

### Phase 2 — Turning voltage over time into rhythms

Take a two-second window of jagged voltage and produce a small set of numbers
describing how much energy sits in each frequency range.

Raw EEG is nearly meaningless to look at — a wobbling line whose wobble is a
mixture of dozens of overlapping oscillations plus muscle tension, eye
movement, and powerline hum. The information is not in the shape of the line
but in which rhythms contribute to it and how strongly.

**Filtering** removes what you don't want. A notch filter cuts a narrow slice
around 60 Hz, the largest contaminant in any recording made indoors. A
bandpass keeps roughly 1–50 Hz, discarding slow drift from electrode movement
and sweat below, and mostly-muscle content above. Coefficients are designed
once at startup, never rebuilt inside the loop — that is one of the most
common ways real-time systems lose their timing budget. Filtering runs
forwards then backwards so it introduces no time delay, which matters because
the later calculation compares bands against each other.

**Spectral estimation** measures energy per frequency. Welch's method splits
the window into overlapping segments, computes a rough profile for each, and
averages them — trading frequency precision for stability. Integrate across
each named band to get one number per band.

| Band | Range | Associated with |
|---|---|---|
| theta | 4–8 Hz | drowsiness, memory encoding |
| alpha | 8–12 Hz | relaxed wakefulness, blocked by eye opening |
| beta | 13–30 Hz | active processing, cognitive load |

Report each band as a **fraction of total power**, not raw energy. Absolute
energy depends heavily on electrode contact quality and amplifier gain, both
of which vary enormously between and within sessions. Ratios don't. This is
what allows a fixed threshold to work across people and equipment.

**Done when:** eyes-closed alpha is clearly higher than eyes-open alpha,
averaged across twenty subjects. If filters, sampling rate, band boundaries,
or units are wrong, this fails.

### Phase 3 — Turning rhythms into a state

Reduce three band-power numbers into one continuous measure and one discrete
label. Two approaches, built side by side because they demonstrate different
things.

**The formula.** Engagement index = beta / (alpha + theta). Beta associates
with effortful processing; alpha and theta with relaxed and drowsy states. The
ratio rises with engagement. From mid-nineties human-factors research, used in
operator-alertness monitoring. Crude and variable between individuals, but
transparent, needs no training data, and computes in microseconds.

Two refinements make it usable. **Smoothing**, because the raw value jumps
around and a jittery number produces a jittery display. And **hysteresis** —
two thresholds instead of one. With a single threshold, whenever the value
hovers nearby the label flips several times a second and the interface
strobes. Requiring a clear rise to enter a state and a clear fall to leave it
is exactly how a thermostat avoids constant switching.

**The classifier.** Take thousands of labeled two-second windows, extract band
powers, let a model learn which combinations correspond to which condition.
Produces an accuracy number the formula alone cannot support.

**The trap, and it is the most consequential mistake available anywhere in the
project.** Consecutive windows overlap in time, and each person's EEG carries
a strong individual fingerprint from skull thickness, electrode contact, and
their own alpha frequency. Splitting data randomly lets the model see nearly
identical data on both sides, or simply identify the subject and recall their
label distribution. It will report 99% accuracy that means nothing.

Split by **person**, using `GroupKFold` grouped by subject, so every subject
appears in training or testing but never both. Expect 75–90% subject-
independent accuracy. A 99% result is a bug, and experienced reviewers look
for it specifically.

Also add timing instrumentation here, reporting the 95th percentile rather
than the average. The average hides stalls.

### Phase 4 — Closing the loop and presenting the work

Two halves that seem unrelated but aren't: make the system visible, and make
the repository legible.

**The interface.** A shape at screen centre expands and contracts with the
engagement index; background colour shifts along a gradient — teal and green
when relaxed, blue and violet when engaged. Nothing labeled, nothing charted.
The intent is not to display data but to give the person something to respond
to without conscious effort. That is what makes it biofeedback rather than
monitoring.

Two constraints shape the build. On macOS the window and its event loop must
own the main thread, so acquisition and analysis run on a background thread
publishing into a shared slot the display reads when it draws. Neither waits
for the other. And all visual changes must be eased rather than applied
instantly — a circle that jumps to its new size looks mechanical; one that
moves a fraction of the way each frame looks alive. Small amount of code,
disproportionate share of the impression.

**The presentation.** Code, tests, docs, and assets each in an obvious place.
Automated testing on every push so a green badge sits at the top. A written
description that leads with a short animation, states measured numbers rather
than adjectives, gives commands a stranger can paste, explains one or two
design decisions, and clearly states what the system does not demonstrate.

That last item is counterintuitive but important. Writing down that
eyes-closed alpha is a proxy for relaxed wakefulness rather than a measure of
focus, that no live hardware was tested, that no artifact removal was
performed — all of it makes the project look stronger. A project claiming to
read minds invites scepticism about everything in it. A project claiming to
classify two well-defined conditions at 83% on unseen subjects, and saying
plainly what that does and doesn't imply, is believable throughout.

---

## 3. Environment setup

Done once. Already complete.

```bash
cd ~/VSCode/Biofeedback_proj
git init
# create .gitignore FIRST, before venv exists
python3 -m venv venv
source venv/bin/activate
pip install brainflow numpy
mkdir -p src tests docs assets
touch src/__init__.py
git add . && git commit -m "chore: project scaffolding"
git branch -M main
```

Published to GitHub via GitHub Desktop (File → Add Local Repository →
Publish repository).

### `.gitignore`

```
__pycache__/
*.py[cod]
.pytest_cache/
*.egg-info/
venv/
.venv/
.DS_Store
._*
.vscode/
.idea/
*.code-workspace
*.edf
*.edf.gz
mne_data/
data/
*.joblib
*.log
scratch/
```

### VS Code

- Open the **project folder itself**, not its parent — the integrated terminal
  opens at whatever folder you opened, so opening the parent puts you one
  level up.
- Avoid `.code-workspace` files. Paths inside them are relative to the file's
  own location, and a multi-root workspace makes VS Code treat unrelated
  projects as one.
- `Cmd+Shift+P` → `Python: Select Interpreter` → pick the one with `venv` in
  the path.

---

## 4. Terminal reference

### Every session

```bash
cd ~/VSCode/Biofeedback_proj
source venv/bin/activate
```

Look at the prompt before running anything. It should show `(venv)` and the
right folder name. The overwhelming majority of confusing errors are one of
those two being wrong.

Activation is per-shell. A new tab means running it again — normal, not a
mistake.

### Keys

| Key | Does |
|---|---|
| Tab | Autocomplete paths |
| ↑ / ↓ | Command history |
| **Ctrl + C** | **Kill the running program** |
| Ctrl + A / E | Jump to line start / end |
| Cmd + K | Clear screen |
| Cmd + T | New tab |

Ctrl+C is Control, not Command. Several scripts here are infinite loops by
design.

### Commands

```bash
pwd                  # where am I
ls -la               # what's here, including dotfiles
cd ..                # up one level
python -m src.buffer # run a module from the project root
pytest -v            # run tests
open .               # open current folder in Finder
```

Always run from the project root using `python -m src.thing`, not
`cd src && python thing.py`. It keeps relative paths consistent and makes
imports work.

### Common errors

| Error | Almost always means |
|---|---|
| `command not found: python` | venv not activated |
| `ModuleNotFoundError` | venv not activated, or activated in a different tab |
| `No such file or directory` | wrong folder — run `pwd` |

---

## 5. Git reference

GitHub Desktop and the command line operate on the same repository
simultaneously. Use whichever suits the moment; the commits are identical.

```bash
git status                        # what changed
git diff                          # the actual line changes
git add .                         # stage everything
git commit -m "feat(x): thing"    # save a checkpoint
git push                          # send to GitHub
git log --oneline -10             # recent history
git switch -c branch-name         # new branch
git restore file.py               # discard uncommitted changes (DESTRUCTIVE)
git reflog                        # the undo button — recovers almost anything
```

**Commit rule:** commit whenever something goes from broken to working. Each
commit is a save point, and the history becomes a record of what you proved
and when.

**Message format:**

```
feat(buffer): handle wraparound across the array end
test(buffer): cover shape invariance over 1000 pushes
perf(dsp): precompute SOS coefficients, 8.1ms -> 2.3ms
docs: add benchmarks and architecture diagram
```

A `perf:` commit with before/after numbers is disproportionately persuasive.

**Anything committed is recoverable. Anything uncommitted is not.**

### Desktop ↔ command line

| GitHub Desktop | Command |
|---|---|
| Checkbox next to a file | `git add <file>` |
| Summary field | the `-m "..."` part |
| Commit to main | `git commit` |
| Push origin | `git push` |
| History tab | `git log` |
| Right-click → Discard | `git restore <file>` |

The checkboxes are the **staging area**: `working directory → git add →
staging → git commit → history`.

---

## 6. Day 1 findings

Measured directly from BrainFlow's synthetic board, not taken on authority.

```
board id:       -1
sampling rate:  250
total rows:     32
EEG rows:       [1, 2, ..., 16]
timestamp row:  30
raw shape:      (32, 501)   ← varies run to run
```

### The data matrix

Rows are sensor channels. Columns are time, oldest on the left.

| Row(s) | Contains | Observed magnitude |
|---|---|---|
| 0 | Package counter | 0, 1, 2, 3 — increments by one |
| **1–16** | **EEG, microvolts** | 12 to 600 |
| 17–29 | Accelerometer, temperature, battery | 0.8 to 235,000 |
| 26 | Temperature °C | 36.6, constant |
| 30 | Unix timestamp | 1.79e9 |
| 31 | Marker channel | zeros |

Three consequences:

**Different units per row.** Row 26 is degrees Celsius. Row 30 is seconds
since 1970. Rows 1–16 are microvolts. There is no single interpretation of
"a number in this matrix."

**Wildly different magnitudes.** Row 25 read about 235,000 while EEG topped
out near 600 — roughly a thousand times larger. Sweep it in by accident and
the frequency analysis describes that row and nothing else.

**No slice can be guessed.** `data[1:9]` grabs half the EEG. `data[0:16]`
includes the package counter. Only `get_eeg_channels()` is correct, and it
returns something different for every board model.

### Row indices are not channel numbers

`get_eeg_channels()` returns **row positions in the matrix**. For this board
they happen to be contiguous starting at 1; on other hardware they may be
scattered. Hence naming the variable `_eeg_rows`, not `_channels`.

```python
data[eeg, :]     # → (16, 501)
```

Comma separates dimensions. Before it selects rows (a list means "these
specific rows, in this order" — fancy indexing). After it, `:` means all
columns. That single line is the whole body of `poll()`.

### Sample count varies

501 on one run, 502 on the next. Two seconds at 250 Hz "should" be 500.
`time.sleep(2)` sleeps for *at least* two seconds, and BrainFlow collects
another sample or two during the overshoot.

`poll()` therefore returns a different number of columns every call. Code
assuming a fixed count breaks intermittently — the worst failure mode,
because it passes testing and fails during the demo.

### The EEG rows are pure sine waves

Row 1 moves gently: `11.8 → 13.5 → 15.3 → 17.0`.
Row 16 lurches: `452 → -255 → 353 → 381` across 16 milliseconds.

Amplitude and frequency both scale with channel index. Every channel is a
clean sinusoid.

Two consequences. Real scalp EEG sits around 5–100 µV; these reach 600, so the
board isn't even amplitude-realistic. And more importantly, a sine in gives a
sine out whether the filter works or not — this data proves plumbing, never
math. That is the gap the PhysioNet recordings close.

### The package counter

`0, 1, 2, 3...` If it ever jumps, samples were dropped in transit. A free
integrity check.

### BrainFlow lifecycle

```
prepare_session()   open the connection
start_stream()      begin sending; a C++ thread fills an internal buffer
   ...              data accumulates without your Python running
get_board_data()    hand me everything, and empty the buffer
stop_stream()       stop sending
release_session()   close the connection, free the port
```

Skip `release_session()` and the port stays claimed — the next run fails with
"unable to prepare session." The context manager exists to make that
impossible even on a crash.

`get_sampling_rate()` and friends are **static methods** on the class, not the
instance. They work before connecting because they're a lookup table keyed by
board type. This lets you size the buffer correctly before any hardware
exists.

---

## 7. The three rules

**1. Ask the library, never hardcode.** Sampling rate, channel count, row
positions — all queried at runtime.

**2. Sample counts vary.** Never assume a fixed number of columns.

**3. Rows are not channels.** Always select with `get_eeg_channels()`.

Every design decision in `source.py` and `buffer.py` traces to one of these.

---

## 8. Corrections to the original plan

| Plan said | Reality | Why it matters |
|---|---|---|
| `Fs = 256 Hz`, 512 samples | **250 Hz**, 500 samples | A 2.4% error shifts every computed frequency. Plots still look plausible |
| 8 EEG channels | **16** | Hardcoding `data[1:9]` silently drops half, with no error |
| 4 channels in the verification output | 16 | — |
| `get_current_board_data()` | `get_board_data()` | The former peeks and returns duplicates; the latter drains |
| 60 Hz notch + 1–45 Hz bandpass | Bandpass already kills 60 Hz | Either raise the bandpass to 50 Hz so the notch does visible work, or state in the README that it exists for real-hardware paths |
| Pygame + streaming thread | macOS requires the UI on the main thread | Acquisition goes on a daemon thread |

---

## 9. Design decisions

Worth writing up in the README.

**Drain, don't peek.** `get_current_board_data(n)` returns the last *n*
samples without consuming them. Polling every 100 ms at 250 Hz gives 25 new
samples but reads back 500 — 95% duplicates, silently corrupting the spectrum
while looking completely normal.

**Ring buffer over `np.hstack`.** The obvious
`np.hstack([win, new])[:, -500:]` allocates a fresh array ten times a second
forever. The ring buffer writes in place into a fixed allocation, trading a
little index arithmetic for a flat memory profile.

**Relative over absolute band power.** Absolute µV² varies with impedance and
gain, so fixed thresholds wouldn't transfer between recordings. Ratios do.

**Source protocol over an if-statement.** Declaring what a data source must
provide, once and in writing, means adding file replay touches exactly one
module. Buffer, DSP, decoder, and UI stay untouched.

**Return copies, not views.** NumPy slices share memory with the original. If
`get()` returned a view and a downstream filter modified it in place, it would
corrupt the buffer's internals. One small allocation per frame is a deliberate
trade for safety.

**Hysteresis over a single threshold.** Two thresholds — enter high, leave low
— stop the label flipping several times a second when the value hovers near
the boundary.

---

## 10. Real EEG data (Phase 1.5)

An addition to the original plan that changes the character of the project.

Synthetic data can't validate anything, because a working filter and a broken
one both pass a sine through unchanged. Real recorded EEG containing a known
physiological effect either gets reproduced by your pipeline or doesn't —
that's a test, not a vibe.

### The dataset

**PhysioNet EEG Motor Movement/Imagery Database**, recorded in Albany around
2004. 109 subjects, 64 channels, 160 Hz, EDF format. Downloads via
`mne.datasets.eegbci` with one command — no registration, no licence forms, so
a reviewer can reproduce your results.

Each subject has 14 runs:

| Run(s) | Duration | Task | Labels |
|---|---|---|---|
| **1** | 1 min | Rest, **eyes open** | whole-file |
| **2** | 1 min | Rest, **eyes closed** | whole-file |
| 3, 7, 11 | 2 min | Actually move left/right fist | per-event |
| 4, 8, 12 | 2 min | Imagine moving left/right fist | per-event |
| 5, 9, 13 | 2 min | Actually move both fists / both feet | per-event |
| 6, 10, 14 | 2 min | Imagine moving both fists / both feet | per-event |

Event annotations are `T0` (rest), `T1` (left fist or both fists), `T2` (right
fist or both feet). Note T1/T2 mean different things depending on the run.

### Is there a focused/relaxed label?

**No.** Nobody in 2004 recorded "how focused is this person." But two usable
contrasts exist.

**Contrast A — eyes open vs eyes closed (runs 1, 2).** Alpha blocking, the
Berger effect, described in 1929. Closing the eyes produces a large increase
in 8–12 Hz power over occipital electrodes. Best channels O1, Oz, O2, P3, Pz,
P4. Labels are per-file, so no annotation parsing. The effect is large,
reliable, and visible by eye in the raw trace. Honest class names:
`eyes_open` / `eyes_closed` — a reasonable proxy for cortical arousal, not a
measure of focus.

**Contrast B — rest vs motor imagery (T0 vs T1/T2 in runs 4, 8, 12).**
Event-related desynchronization: imagining movement decreases the mu rhythm
(8–13 Hz, overlapping alpha) over sensorimotor cortex. Best channels C3, Cz,
C4. Since alpha drops, the engagement index beta/(alpha+theta) should *rise*
during task — so the existing formula makes a directional prediction you can
test. Honest class names: `rest` / `motor_imagery`. This is task engagement —
a real cousin of focus, but motor rather than cognitive.

**Start with A, add B as a stretch.** A's per-file labels mean your first
encounter with real EEG isn't also your first encounter with event alignment.

If you later want genuine attention labels: **STEW** (rest vs multitasking,
with workload ratings) or **Sleep-EDF** (expert-scored sleep stages). Both
after EEGMMIDB works.

### The five gotchas

**Units.** MNE returns **volts**; BrainFlow returns **microvolts**. A factor
of one million. Multiply by `1e6` in the loader. Relative band power survives
the error, which is what makes it so hard to notice. Sanity check: scalp EEG
standard deviation should land between about 5 and 100 µV.

**Sampling rate.** The dataset is 160 Hz, your board is 250 Hz. Welch's
`nperseg` is in *samples*, so a fixed value gives different frequency
resolution at different rates — train at 160 and serve at 250 and the model
sees inputs that don't mean what they meant during training. Resample
everything to one canonical rate in the loader.

**Channel names.** Raw names in this dataset have trailing dots — `Oz..`,
`Fc5.`. Call `eegbci.standardize(raw)` immediately after loading, *before*
picking channels, or your pick matches nothing.

**Channel selection.** Don't average all 64. Alpha blocking is occipital;
frontal channels are dominated by blink and muscle artifact, and eyes-open
recordings have far more blinks — so averaging frontals in correlates with
your label for entirely the wrong reason.

**Leakage.** Covered in Phase 3 above. Use `GroupKFold` grouped by subject.

### Order of operations in the loader

```
load EDF → standardize() → pick channels → resample → multiply by 1e6
```

Not arbitrary. Standardize before pick, or the pick fails silently. Pick
before resample, because resampling 6 channels is ten times faster than 64.

### The architecture change

One change: an `EEGSource` protocol — a written contract stating what any data
source must provide.

```
SyntheticSource ─┐
ReplaySource ────┼─▶ EEGSource ─▶ RingBuffer ─▶ FilterChain ─▶ Decoder ─▶ UI
LiveSource ──────┘   (protocol)
```

Required members: `fs`, `n_channels`, `channel_names`, `start()`, `stop()`,
`poll()`.

A `Protocol` is a description, not a parent class — anything with the right
methods qualifies, no inheritance needed. Python doesn't enforce it; the value
is having one written place where the contract lives.

`ReplaySource` holds a recording in memory, watches a clock, and hands out
samples at the speed they were originally recorded. The pacing must anchor to
**absolute elapsed time**, not per-iteration increments:

```
elapsed = now - start_time
target  = int(elapsed * fs)      # samples that should exist by now
n_new   = target - cursor        # what we owe the caller
```

The tempting alternative — emit a fixed chunk then `sleep(0.1)` — drifts,
because sleep overshoots by a few milliseconds every iteration. After ten
thousand iterations you're thirty seconds behind. The version above is
self-correcting: a stall just produces a larger `n_new` next call.

Also note this must return an **empty array** when no samples are due, which
is exactly the case the ring buffer's `n_new == 0` guard handles.

---

## 11. README template

### The five principles

1. **Visual first.** A GIF above the fold. Four seconds decides whether anyone
   keeps reading.
2. **Numbers, not adjectives.** "83.4% ± 4.1%" beats "high accuracy."
3. **Reproducible in one paste.** Clone, install, run.
4. **State your limitations.** The highest-signal paragraph in the document.
5. **Explain one decision.** A real tradeoff you made, and why.

### Phrasing

| Don't write | Write |
|---|---|
| "Detects focus and relaxation" | "Classifies eyes-open vs eyes-closed, a proxy for cortical arousal" |
| "99% accuracy" | "83.4% ± 4.1%, subject-independent (GroupKFold)" |
| "12 ms end-to-end latency" | "12 ms p95 processing latency; acquisition latency untested" |
| "Real-time BCI" | "Real-time processing pipeline validated on recorded EEG" |
| "Advanced ML" | "RandomForest over 3 relative band-power features" |

Name the actual thing measured, attach the actual number, scope the claim.
Every downgrade makes you more credible, not less.

### Numbers to collect as you go

Keep these in `docs/benchmarks.md` rather than scrambling on day 25.

| Blank | How to get it |
|---|---|
| Accuracy ± std | `cross_val_score(...).mean()` and `.std()` |
| p95 / median latency | `np.percentile(lat, 95)`, `np.median(lat)` |
| Per-stage latency | Wrap each stage in `perf_counter_ns()` separately |
| Throughput headroom | 2000 ms ÷ p95 latency |
| PSD resolution | `fs / nperseg` |
| Alpha open vs closed | Mean relative alpha per condition, across subjects |
| Test count | What `pytest` prints |

### Mistakes that cost you

- No GIF, or it's below the fold
- Accuracy with no CV method stated
- Installation troubleshooting before anything interesting
- A `pip freeze` dump instead of your ~8 direct dependencies
- No limitations section
- "Feel free to contribute!" on a solo portfolio project
- Broken badge URLs

### Write it in three passes

**Now:** title, one-line description, WIP notice, phase checklist. Ten lines.
**End of each phase:** add numbers to `docs/benchmarks.md`, tick a checkbox.
**Phase 4:** full rewrite, using numbers you already collected.

### References

1. Berger, H. (1929). Über das Elektrenkephalogramm des Menschen.
   *Archiv für Psychiatrie und Nervenkrankheiten*, 87, 527–570.
2. Schalk, G., et al. (2004). BCI2000: A General-Purpose Brain-Computer
   Interface (BCI) System. *IEEE TBME*, 51(6), 1034–1043.
3. Goldberger, A.L., et al. (2000). PhysioBank, PhysioToolkit, and PhysioNet.
   *Circulation*, 101(23), e215–e220.
4. Welch, P.D. (1967). The use of fast Fourier transform for the estimation of
   power spectra. *IEEE Trans. Audio Electroacoust.*, 15(2), 70–73.
5. Pope, A.T., Bogart, E.H., & Bartolome, D.S. (1995). Biocybernetic system
   evaluates indices of operator engagement in automated task.
   *Biological Psychology*, 40(1–2), 187–195.
6. Pfurtscheller, G., & Lopes da Silva, F.H. (1999). Event-related EEG/MEG
   synchronization and desynchronization. *Clinical Neurophysiology*,
   110(11), 1842–1857.

---

## 12. Schedule

### Phase 1

| Day | Build | Commit when |
|---|---|---|
| 1 ✅ | Nothing — print raw `get_board_data()` and look at the matrix | — |
| 2 | `src/source.py` — constructor, `start()`, `stop()` | three clean start/stop cycles in a row |
| 3 | `poll()` + context manager | shapes print, `n_new` visibly varies |
| 4 | `src/buffer.py` — `__init__` and `push()`, non-wrapping only | it accepts samples without error |
| 5 | `push()` wraparound, `get()`, `is_ready`, verification loop | shape locks at `(16, 500)`, memory flat |

Day 4 is the one to slow down on. Work a capacity-5 example on paper until you
can predict the write index before running the code.

### Phase 1.5

| Day | Build |
|---|---|
| A | `pip install mne`. Load subject 1, runs 1–2. Print `ch_names` before and after `standardize()`. Plot 5 s of each and *see* alpha |
| B | `dataset.py` — `load_run` + `sanity_check` |
| C | `source.py` — the protocol and `ReplaySource` |
| D | Rename `EEGStreamer` → `SyntheticSource`, confirm interchangeable, add `--source` flag |
| E | `load_baseline_cohort` for 20 subjects. Find out whether the DSP is correct |

### Overall

Original plan was 5/7/6/7 days. Suggested: **5/9/5/6**, with the last day
reserved purely for the README and demo GIF. Phase 2's DSP is the
intellectual core and the part reviewers probe; Phase 4's UI is mostly
`pygame.draw.circle`.

### Verification ladder

Never skip a rung — each isolates a different failure.

1. Data loads — `load_run(1, 1)` returns the expected shape
2. Units sane — std between 5 and 100 µV
3. Alpha visible — plot eyes-open vs eyes-closed, see the 10 Hz rhythm
4. Replay paced — sum of `n_new` over 10 s ≈ 10 × fs, within 1%
5. Buffer intact — shape locks, memory flat, sub-millisecond
6. DSP correct — eyes-closed alpha clearly exceeds eyes-open
7. Index correct — E is *lower* eyes-closed
8. Classifier honest — `GroupKFold`, mean ± std, confusion matrix
