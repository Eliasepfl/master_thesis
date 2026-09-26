# Reading Vision from EEG: presentation script

**Master's thesis presentation · Elias Naha · EPFL / MIT Media Lab**

This is the spoken text for the 29-slide deck, slide by slide. The same text is stored in each slide's speaker notes, so it also appears in presenter view.

- **Length:** 3,094 words, about 21 minutes at 145 words per minute (22 min at 140).
- **Deck:** https://claude.ai/artifact/HmB9MhZvwCare464b4YJWd

| Part | Slides | Approx. time |
|---|---|---|
| Introduction: vision in the brain and in AI models | 1–9 | 6 min |
| FREUD | 10–13 | 3 min 30 s |
| HINT | 14–19 | 4 min 25 s |
| Imagery transfer | 20–26 | 5 min 30 s |
| Synthesis and conclusion | 27–29 | 2 min |

**If you run long:** slide 11 (SwinEEG activations), slide 18 (cost) and slide 19 (benchmark controls) can each shrink to one sentence, saving about a minute and a half.

The introduction follows the thesis introduction (Sections 1.1 and 1.2): images from EEG, biological vision as a hierarchy and a timetable, vision models and the brain as a yardstick, and the converse question the thesis asks. The story then runs FREUD → HINT → imagery transfer. The figures and tables come from the HINT paper, the imagery paper and the thesis itself.

---

## Introduction: vision in the brain and in AI models

### Slide 1 · Title slide

*Starts at 0:00 · about 22 s*

Good morning, and thank you for being here. My name is Elias Naha, and I'm presenting my master's thesis, Reading Vision from EEG: where the information is, and what crosses into imagery. The work was carried out at the MIT Media Lab with Dr. Nataliya Kos'myna, and supervised at EPFL by Professor Mathieu Salzmann.

### Slide 2 · Images are reconstructed from scalp recordings

*Starts at 0:22 · about 46 s*

Images are reconstructed from non-invasive recordings of a person looking at them. On the left, a photograph a participant saw; next to it, a reconstruction from the EEG recorded while they looked at it. The usual recipe predicts a low-level latent and a semantic embedding from the brain signal and merges them in a generative model, and it works even for electroencephalography, a sensor with little spatial resolution.

Most of these pipelines pass the brain signal through a single global vector. So where does the brain's contribution enter, and which part of the image does it actually determine? That is not established, and it is the thread of this thesis.

### Slide 3 · “A photograph is an impoverished description of the world”

*Starts at 1:08 · about 29 s*

The reason is structural. A photograph is an impoverished description of the world: one object yields a vast family of images. An EEG recording is an impoverished description of the photograph, and a few recordings from one participant cannot fix a general visual representation without prior assumptions. So a pipeline must be told which part of the image the brain readout may determine, or it decides on its own.

### Slide 4 · Visual cortex is organised as a hierarchy

*Starts at 1:36 · about 33 s*

Where should the brain be allowed to write into an image? Biology offers a first answer. Simple cells in the primary visual cortex respond to oriented edges in a small patch of the visual field, a code that matches the statistics of natural images. Along the ventral stream, receptive fields grow, and later stages build more task-specific features, up to objects and categories. So visual cortex is a hierarchy that runs from local and fine to global and abstract.

### Slide 5 · The hierarchy is also a timetable

*Starts at 2:09 · about 37 s*

And the hierarchy is also a timetable. The gist of a scene reaches the higher areas first and is then refined by feedback: vision at a glance comes before vision with scrutiny. Electrical recordings show the same order, with coarse information about an image arriving before its fine detail.

This matters for EEG. A scalp recording resolves little in space, but it resolves time very well, and here time indexes coarseness. So EEG should be well placed to tell us about the coarse part of what a person sees.

### Slide 6 · Transformers drop the locality that convolutions build in

*Starts at 2:46 · about 38 s*

Artificial vision followed a related path. Convolutional networks make locality explicit: small filters applied everywhere and stacked into a hierarchy, much like the local receptive fields of V1. Vision transformers drop that constraint. They mix information across the whole image from the first layer, they organise it differently from convolutional networks, and they are now the state of the art.

But that freedom has a cost: the structure a convolution gets for free has to be learned from data, so transformers need more training data, and what they learn is harder to interpret.

### Slide 7 · Brain activity has become a yardstick for vision models

*Starts at 3:24 · about 45 s*

This is where brain activity offers a yardstick. Vision models are scored by how well their activations predict neural responses, as in Brain-Score, keeping in mind that a layer is not an area of cortex. Behind these comparisons is one question: does the structure biology uses appear inside a model that is free to ignore it? For orientation selectivity, for instance, the training paradigm decides rather than the architecture.

This thesis asks the converse question. Instead of reading a model with the brain, can we write the brain into a model: can information be routed from a brain signal into a generative model, along the coarse-to-fine order both claim?

### Slide 8 · Image generators also draw from coarse to fine

*Starts at 4:10 · about 49 s*

Image generators have their own coarse-to-fine order: diffusion resolves coarse structure before fine detail, and next-scale autoregression makes the hierarchy explicit. The strip at the top is VAR, a visual autoregressive generator, at work. The first token fixes the global colour, the next maps fix the layout, and the last ones only sharpen texture. It draws an image as ten token maps, from one token to a sixteen-by-sixteen grid, so every token has an address.

Yet in EEG-to-image pipelines the brain enters through one global vector: the conditioning stays flat. Visual cortex and a scale-wise generator both order their representations from coarse to fine, which invites joining them level by level. This thesis treats that invitation as a measurement.

### Slide 9 · Two questions, three projects

*Starts at 4:59 · about 59 s*

Two questions organise the thesis. The first is about perception: can a frozen image generator use the part of the EEG that carries appearance rather than category? The second is about imagination: does the code we read during perception persist in another person once the stimulus is gone?

They map onto three projects, which I'll present in order. FREUD routes the stages of a hierarchical EEG encoder into the scales of the generator during training. What FREUD taught us led to HINT, which writes brain-predicted tokens into the coarsest scales at inference. And the imagery project asks whether other people's perception can teach a decoder to read a new person's imagery. Throughout, every result sits between a floor, the same pipeline with its brain input shuffled, and a ceiling, the true target written in the same place, and the generator's weights never change.


---

## FREUD: routing a hierarchical EEG encoder into the generator

### Slide 10 · FREUD routes each encoder stage to a generator scale

*Starts at 5:58 · about 58 s*

The first project, FREUD, takes that invitation literally. EEG-to-image encoders at the time mapped each trial to a single vector and discarded everything computed along the way. So we built SwinEEG, a four-stage Swin Transformer V2 over a grid of electrodes by time. Each stage merges neighbouring time slices but never merges electrodes, so every stage is a readout of the whole scalp at a different depth.

FREUD routes those stages into the generator by depth: the deepest, most semantic stage drives the three coarsest scales, the shallowest drives the two finest, through small adaptors that start at zero, so a routed model starts out identical to an unrouted one. To test the direction of the mapping, we compared it with a global-vector baseline, a reversed routing and a learned router, all on THINGS-EEG2: ten participants viewing thousands of natural images.

### Slide 11 · SwinEEG keeps the scalp layout at every depth

*Starts at 6:56 · about 37 s*

What sets SwinEEG apart from other EEG encoders is that it never collapses the electrode axis. This figure shows, for two participants, how strongly each electrode is represented from the input embedding through the four stages. Every stage is still defined at every electrode, so each stage is a genuine readout of the scalp, and the pattern changes with depth: frontal at the second-to-last stage and posterior at the last, in both participants. The encoder was pretrained by masked reconstruction of EEG patches, then aligned to CLIP image embeddings.

### Slide 12 · Teacher forcing hides the brain signal during training

*Starts at 7:33 · about 58 s*

Training FREUD taught us something important about these generators. The routed models ended up behaving like their baseline, and understanding why was the key step. A next-scale generator is trained with teacher forcing: to predict scale k, it receives the ground truth of every coarser scale, upsampled, which is a blurred version of the answer. Given that context, any extra condition carries almost no information about the next token, so it receives almost no gradient and its path stays near zero. Even the true CLIP embedding of the image, given in place of the EEG, leaves the training loss unchanged, 6.047 against 6.046 nats. Only the very first scale has no coarser context, and it is a single token.

The implication is direct: the brain signal should be written in where the generator cannot see the answer, at inference.

### Slide 13 · Appearance and layout live in five coarse tokens

*Starts at 8:31 · about 54 s*

The second lesson is where the information lives. We measured it directly, by writing the true tokens of the viewed image into chosen scales of the frozen generator. The single coarsest token carries the global appearance: forcing it raises an appearance score from 0.69 to 0.85. The second scale, four tokens, carries the layout. Together, five tokens out of 680 reach 0.95, while the four finest scales add pixel detail but no appearance.

Twelve numbers, the image pooled to two by two, predict these coarse tokens better than the CLIP embedding the encoder was trained on. And a third observation simplified everything: the deepest encoder stage turned out to be the best source for every scale. So the recipe for HINT was set: one good readout, written into the coarsest scales.


---

## HINT: hierarchical injection of neural tokens

### Slide 14 · HINT moves FREUD's idea to inference

*Starts at 9:25 · about 55 s*

That is HINT, Hierarchical Injection of Neural Tokens: FREUD's idea, moved to inference. The paper is currently under review at the NeurIPS Brain and Body workshop. A per-subject EEG encoder feeds two closed-form ridge readouts. One predicts the global image embedding, which conditions the generator as before. The other predicts the coarse feature maps, which are quantised into tokens and written into the first K scales of a frozen VAR decoder; the decoder then completes the fine scales on its own. No gradient from EEG ever reaches the decoder, and adapting to a new person takes under a minute.

Writing a prefix involves two choices that prior work collapses into one: how many scales, K, and what fraction of the positions within them, q. We evaluate on THINGS-EEG2, ten participants and two hundred test images, against the published ENIGMA system on the field's seven metrics.

### Slide 15 · Forcing more scales trades semantics for structure

*Starts at 10:19 · about 32 s*

This plane shows the trade-off: structure on the horizontal axis, AlexNet identification; semantics on the vertical, CLIP identification. Forcing more scales with every position written slides the system along a front: each extra scale buys structure and costs meaning. At a fixed number of scales, freeing half the positions, the green arrows, moves it almost straight up, off that front. At ten scales, CLIP identification rises by 0.078 and Inception by 0.064, while AlexNet stays where it was.

### Slide 16 · Freeing half the positions helps in every participant

*Starts at 10:52 · about 53 s*

Paired image by image over the ten participants, freeing half the positions improves four metrics in every participant: CLIP, Inception, SwAV and the deeper AlexNet layer, each at p equal to 0.0137, the smallest a Holm-corrected exact test can reach with ten participants. It replicates across both samplers, two generation seeds, and bridge sizes spanning a factor of fifty-two.

The mechanism is simple: an unwritten position is filled by the generator's autoregressive prior, while a written one replaces that prior with a low signal-to-noise EEG estimate. The control confirms it: filling the freed positions with the prefix's own mean degrades every participant. So the pair K, q is a runtime trust parameter, how much of the brain estimate to impose, with no retraining and no added latency.

### Slide 17 · HINT beats the reference on five of seven metrics

*Starts at 11:44 · about 41 s*

Against the published reference, ENIGMA, the deployed configuration, six scales at half the positions, is higher on five of seven metrics with the 420-million-parameter decoder, and on six of seven with the deeper VAR-d30, where only CLIP remains below.

The lower rows frame these numbers. A prefix from another image brings every identification score down to chance, so the prefix carries information specific to the image that was seen. And the true prefix reaches seven of seven: the frozen generator has room to spare, and the readout from the brain is where the next gains will come from.

### Slide 18 · An order of magnitude lighter than the reference

*Starts at 12:25 · about 42 s*

HINT is also light where the reference stakes its claim. Measured back to back on the same GPU, the generator is ten times smaller, needs nine times fewer FLOPs, and takes 90 milliseconds per image, 19 at batch 25. Adapting to a new subject takes under a minute of encoder training plus two seconds of ridge solves on that subject's training set, and the bridge compresses fifty-two-fold at rank 32 while keeping both the count and the effect. That makes the system fast enough for closed-loop use on the compute side, where calibration time and latency matter as much as accuracy.

### Slide 19 · Controls make the benchmark harder to overstate

*Starts at 13:07 · about 43 s*

Alongside the method, we contribute controls for the benchmark itself. A single flat colour, the target's average colour, already beats every published system on pixel correlation and SSIM, so those two columns should be read against that bar, for our numbers as for anyone's. The shared scoring code counted a self-comparison, so every two-way identification score was understated by exactly one in 199; we report corrected reference values. And every claim sits between a floor, a prefix taken from another image, and a ceiling, the true prefix. These controls make the benchmark harder to overstate, which we see as a contribution in its own right.


---

## Imagery transfer: reading a new person's imagery

### Slide 20 · To imagine is to see without a stimulus

*Starts at 13:50 · about 57 s*

The third project leaves the screen. To imagine is to see without a stimulus. That makes imagery attractive as a control signal and expensive as a measurement: imagery decoders are calibrated on imagery trials, which are slow to collect, because imagining imposes a high cognitive load and cannot be serialised, whereas perception affords rapid designs that gather thousands of trials per hour. And nothing outside the participant marks the moment of imagining.

Within a person, perception and imagery are two readouts of one representation; in EEG, the shared part lives in posterior alpha oscillations. So the setting a calibration procedure needs is this: show images to a group once, then decode a new person who was never in it, with no labelled data from that person. To our knowledge, no study had occupied that setting for visual content. This work is currently under review at the NeurIPS Brain and Body workshop.

### Slide 21 · Train on 21 people's perception, test on the 22nd's imagery

*Starts at 14:47 · about 60 s*

This project was carried out with Ninon Lizé Masclef. The dataset is Gao and colleagues, 2026: twenty-two participants and three tasks, four everyday objects, three geometric figures, and three animals. The animal task was designated in advance, on published grounds, as the negative control. Each trial shows the image for four seconds, a mask for about two, then a cue to imagine it for four.

The pipeline was fixed in advance, following Xie and colleagues: log alpha power on ten posterior electrodes, each participant standardised to their own mean and variance without labels, trials averaged into pseudo-trials, and a linear SVM for each pair of classes. It trains on the pooled perception of twenty-one participants and is tested on the twenty-second, who contributes no labelled trial to any fit. Every number is in percentage points above its own permutation null, where chance is fifty percent.

### Slide 22 · Others' perception reads a held-out person's imagery

*Starts at 15:47 · about 43 s*

And it reads. On objects, a decoder fitted on other people's perception separates the held-out participant's imagery 5.3 points above its null, positive in eighteen of twenty-two participants, with p equal to 0.0002, and it survives Bonferroni correction over all thirty-seven primary comparisons. Figures are positive and smaller, 3.5 points, which we treat as convergent support.

The animal control sits at its null, even though animal perception crosses people at 13.5 points, so the null is not a failure to cross people. And pooling matters: with a single source participant the effect halves; twenty-one restore it to parity with the participant's own perception.

### Slide 23 · The effect passes every pre-declared control

*Starts at 16:30 · about 41 s*

Each of these checks was declared before it ran. The transfer grows with the number of source participants on objects and figures, not on animals. Across six bands and regions, posterior alpha, the boxed cell chosen in advance, is the only one positive on both signal tasks with the control at its null. Shifting the class labels cyclically abolishes the effect. Removing ocular components wipes out a frontal perception signal on the control task, from thirty points to nothing, and leaves the posterior readout unchanged. And source participants who share no presentation order with the target reproduce it.

### Slide 24 · The readout is posterior and precedes the cue

*Starts at 17:10 · about 42 s*

Where on the scalp? The decoder's forward patterns sit over posterior sites, in perception and in imagery alike. When? Reading the identical decoder across the trial, the pattern appears at image onset, falls to chance while the image is still on screen, rebuilds through the delay, and peaks in the imagery window.

Stating this honestly required guarding the windows: the wavelet reaches almost 0.4 seconds to either side, so an unguarded delay window leaks activity from after the cue. With a 450-millisecond guard band the leak falls a hundred-and-nineteen-fold, and the delay still reads 4.5 points, before any instruction to imagine.

### Slide 25 · A simple standardised signal is enough

*Starts at 17:52 · about 41 s*

What crosses is simple, which is good news for calibration. It is one dimension: the common level of posterior alpha across the ten electrodes, measured as a departure from each person's own distribution. Standardising within each participant is the only subject-specific step, and it is what makes the transfer work.

Larger models did not do better: across 205 scored architectures, nine alternative classifiers and four pretrained EEG foundation models, none beat the ten-feature linear readout. And crossing people replicates out of sample: on a scene dataset with forty-nine participants, pooled perception reads a new person's perception in forty-six of them.

### Slide 26 · Two experiments would separate imagery from memory

*Starts at 18:34 · about 47 s*

Two questions remain open, and each has a clean experiment. In this dataset each class is one fixed image, the same for everyone, so we cannot yet say whether what crosses is the category or the picture; a design with several images per class, testing on images never seen in training, would decide. And the pattern is already present before the cue, so it could reflect short-term memory, sustained attention, or imagery that starts as soon as the item is known; revealing the item to imagine only at the cue would separate them.

If the transfer holds, calibration inverts: images are shown to a group once, and a new person contributes only unlabelled data.


---

## Synthesis and conclusion

### Slide 27 · The three projects point the same way

*Starts at 19:21 · about 43 s*

The three projects point the same way. First, what EEG delivers is coarse: in FREUD and HINT, the brain's information lands in the coarsest scales of the generator; in the imagery transfer, a single standardised dimension crosses people.

Second, simple readouts go far: closed-form ridge regressions into a frozen generator, and a ten-feature linear readout for imagery, match or beat much heavier systems, and the next gains lie in the map from brain to target rather than in the machine on either side. Third, controls are what make these results credible: floors, ceilings and negative controls declared in advance turn an observation into a measurement.

### Slide 28 · What the thesis answers

*Starts at 20:04 · about 54 s*

So, the answers. Can a frozen image generator use the part of the EEG that carries appearance? Yes, at its coarsest scales: writing brain-predicted tokens into them at inference, with half the positions left to the generator, beats the published reference on five of seven metrics, at about a tenth of its compute. Does the perceptual code persist in another person once the stimulus is gone? On one corpus, yes: a decoder trained on other people's perception reads a held-out participant's imagery 5.3 points above its null, in eighteen of twenty-two participants.

The next steps are changes of design rather than larger models: spatial conditioning to open the finer scales, a per-participant alignment on a shared network, many images per class, and an imagery target revealed only at the cue.

### Slide 29 · Thank you · Questions

*Starts at 20:58 · about 22 s*

Thank you to my supervisors, Dr. Nataliya Kos'myna and Professor Mathieu Salzmann; to Ninon Lizé Masclef, who worked with me on the imagery project from its first day to its last; to my friends at the Media Lab; and to the MIT ORCD cluster, where every experiment ran. I'm happy to take your questions.
