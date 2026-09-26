# Reading Vision from EEG: presentation script

**Master's thesis presentation · Elias Naha · EPFL / MIT Media Lab**

This is the spoken text for the 28-slide deck, slide by slide. The same text is stored in each slide's speaker notes, so it also appears in presenter view.

- **Length:** 3,225 words, about 22 minutes at 145 words per minute (21½ min at 150, 23 min at 140).
- **Deck:** https://claude.ai/artifact/HmB9MhZvwCare464b4YJWd

| Part | Slides | Approx. time |
|---|---|---|
| Introduction | 1–5 | 3 min 35 s |
| Part I · Perception | 6–17 | 9 min 55 s |
| Part II · Imagination | 18–25 | 6 min 40 s |
| Part III · Synthesis | 26–28 | 2 min 10 s |

**If you run long:** slide 16 (cost) and slide 19 (the two within-participant datasets) can each shrink to one sentence, saving over a minute.
**If you have time to spare:** slide 11 (the decoder bench) and slide 24 (what crosses) are where the thesis has the most extra detail to add.

The introduction follows the wording of the thesis introduction (Sections 1.1 and 1.2). The figures and tables come from the HINT paper (reconstruction), the imagery paper and the thesis itself.

---

## Introduction

### Slide 1 · Title slide

*Starts at 0:00 · about 22 s*

Good morning, and thank you for being here. My name is Elias Naha, and I'm presenting my master's thesis, Reading Vision from EEG: where the information is, and what crosses into imagery. The work was carried out at the MIT Media Lab with Dr. Nataliya Kos'myna, and supervised at EPFL by Professor Mathieu Salzmann.

### Slide 2 · Images are reconstructed from scalp recordings

*Starts at 0:22 · about 55 s*

Images are reconstructed from non-invasive recordings of a person looking at them. On the left is a photograph a participant saw; next to it, a reconstruction from the EEG recorded while they looked at it. The usual recipe predicts a low-level latent and a semantic embedding separately from the brain signal and merges them in a generative model. It works even for electroencephalography, a sensor with little spatial resolution, and a published system, ENIGMA, sets the reference.

But where the brain's contribution enters, and which part of the picture it determines, is not established. One measurement makes the point: an image that is a single flat colour, the target's average colour, beats the published system on pixel correlation and on structural similarity. Two of the seven metrics this field reports do not measure reconstruction.

### Slide 3 · “A photograph is an impoverished description of the world”

*Starts at 1:18 · about 29 s*

The reason is structural. A photograph is an impoverished description of the world: one object yields a vast family of images. An EEG recording is an impoverished description of the photograph, and a few recordings from one participant cannot fix a general visual representation without prior assumptions. So a pipeline must be told which part of the image the brain readout may determine, or it decides on its own.

### Slide 4 · The visual hierarchy is also a timetable

*Starts at 1:46 · about 64 s*

Where should the brain be allowed to write? Biology offers an answer. Simple cells in primary visual cortex respond to oriented edges in a small patch of the visual field; later stages build task-specific features. And the hierarchy is also a timetable: the gist of a scene reaches the higher areas first and is refined by feedback, and electrical recordings show coarse information arriving before fine detail. A scalp recording resolves little in space, but it resolves time, and here time indexes coarseness.

Image generators have their own coarse-to-fine order. The strip at the top is a next-scale generator at work: the first token fixes the global colour, the next maps fix the layout, and the last ones only sharpen texture. Both order their representations from coarse to fine, which invites joining them level by level. This thesis treats that invitation as a measurement, and the answer is not the one the analogy predicts.

### Slide 5 · Two questions organise the thesis

*Starts at 2:50 · about 44 s*

Two questions organise the thesis. Part one, perception: can a frozen image generator use the part of an EEG signal that carries appearance rather than category? Part two, imagination: does the code we read during perception persist in another person once the stimulus is gone?

The same rules run through both parts. Every result is reported between a floor, the same pipeline with its brain input shuffled, and a ceiling, the true target written in the same place. Success criteria are fixed before the run. And abandoned directions are reported with the measurement that closed them: at least twelve margins in this project did not replicate.


---

## Part I · Perception: where the information is

### Slide 6 · A frozen generator gives every token an address

*Starts at 3:34 · about 49 s*

Part one. The data is THINGS-EEG2: ten participants, 63 electrodes, about sixteen and a half thousand training images seen four times each, and two hundred test images seen eighty times each. Reconstructions are scored on the field's seven metrics: pixel correlation, SSIM, four two-way identification scores on AlexNet, Inception and CLIP features, where chance is one half, and a SwAV distance.

The probe is a visual autoregressive generator, VAR. It draws an image as ten token maps, from a single token to a sixteen-by-sixteen grid, 680 tokens in all, so every token has an address. Brain-predicted tokens can be written into the coarse maps, the fine ones, a random set, or nowhere, while the generator's weights never change.

### Slide 7 · SwinEEG and FREUD route encoder depth to generator scale

*Starts at 4:23 · about 58 s*

The first idea was the natural one. EEG-to-image encoders at the time mapped each trial to a single vector and discarded everything computed along the way. So we built SwinEEG, a four-stage Swin Transformer V2 over a grid of electrodes by time. Each stage merges neighbouring time slices but never electrodes, so every stage reads the whole scalp at a different depth.

FREUD then routed those stages into the generator by depth: the deepest stage drives the three coarsest scales, the shallowest drives the two finest, through small adaptors that start at zero. We trained a global-vector baseline, this aligned routing, a reversed routing as the negative control, and a learned router. The paper went to an ICML 2026 workshop and was rejected; the area chair asked for more subjects, a seed analysis, and evidence on the learned routing.

### Slide 8 · The audit found the routing path was never trained

*Starts at 5:20 · about 65 s*

So we audited it, reading the code against the paper. The published claim was a gap of 0.013 on one metric, for one participant and one seed, with no error bar. The code differed from the paper in at least seven places: the per-scale signal was added to the residual stream instead of entering through adaptive normalisation, the frozen decoder had every weight in the optimiser, and one line disabled the default initialisation, for speed, before the routing module was attached.

That line decided the outcome. In all nine surviving checkpoints the routing path was either dead, its weights and optimiser state still exactly zero after a hundred thousand steps, or ran on uninitialised memory; all nine score zero of seven against the reference. The premise fails on its own terms too: the deepest encoder stage predicts all 42 image targets better than every shallower stage, fine targets included. Scale k does not belong to stage k.

### Slide 9 · Scale-wise conditioning is inert under teacher forcing

*Starts at 6:25 · about 67 s*

Was it only a bug? No. Rebuilt correctly on another encoder, the per-scale signal matched its no-conditioning twin to three decimals, and so did seven other ways of injecting the EEG scale by scale. Even giving the generator the true CLIP embedding of the image, instead of the EEG, left the validation loss at 6.047 nats against 6.046.

One property of next-scale training explains them all. The generator is trained with teacher forcing: the input at scale k is the upsampled ground truth of every coarser scale, a blurred version of the answer. Given that context, the condition carries almost no information about the next token, so it gets no gradient and its path decays to zero; this is posterior collapse. Only the first scale has no coarser context, and it is one token of 680: pinning it takes 7.74 bits, and the best participant's EEG supplies 1.56. From then on, we judged mechanisms only on generated images, with correct against shuffled EEG.

### Slide 10 · Appearance and layout sit in five coarse tokens

*Starts at 7:33 · about 55 s*

So where is the information? We measured the ceiling directly, writing the true tokens of the viewed image into chosen scales of the frozen generator. Forcing the single coarsest token raises an appearance score from 0.69 to 0.85 while barely moving pixel correlation. The second scale, four tokens, carries layout and does the reverse. Together, five tokens reach 0.95. The four finest scales, 589 tokens, raise pixel correlation and leave appearance where it was.

And these coarse tokens are not what the encoder was trained to predict: twelve numbers, the image pooled to two by two, predict them better than the 1024-dimensional CLIP embedding, almost five times better on layout. The finer signal does exist, about two R-squared points of texture in all ten participants, but no generator we trained made use of it.

### Slide 11 · A rescaled ridge readout clears three of seven metrics

*Starts at 8:28 · about 42 s*

Next we fixed a bench: a published EEG encoder, frozen, whose retrieval we reproduced; twelve decoder families; and one reference, ENIGMA, on all ten participants. The strongest arm trains nothing on EEG. A ridge regression predicts the image feature, a VAR fine-tuned only on real image features generates from it, and a single scalar, computed on the training images, restores the amplitude the regression shrinks.

Without that scalar the arm wins one metric of seven; with it, five on one participant, and three of seven on the ten-participant mean. That three is what survived our own audit of the headline counts.

### Slide 12 · HINT writes the brain's tokens into the coarsest scales

*Starts at 9:10 · about 49 s*

The routing never ran in training, but the idea works at inference. That is HINT, Hierarchical Injection of Neural Tokens. A per-subject EEG encoder feeds two closed-form ridge readouts. One predicts the global image embedding, which conditions the decoder as before. The other predicts the coarse feature maps, which are quantised into tokens and forced into the first K scales of a frozen VAR decoder; the decoder completes the fine scales on its own.

No gradient from EEG ever reaches the decoder, so any difference between two configurations is what the readout delivered. Writing a prefix involves two choices that prior work collapses into one: how many scales, K, and what fraction of the positions within them, q.

### Slide 13 · Forcing more scales trades semantics for structure

*Starts at 9:59 · about 32 s*

This plane shows the trade-off: structure on the horizontal axis, AlexNet identification; semantics on the vertical, CLIP identification. Forcing more scales with every position written slides the system along a front: each extra scale buys structure and costs meaning. At a fixed number of scales, freeing half the positions, the green arrows, moves it almost straight up, off that front. At ten scales, CLIP identification rises by 0.078 and Inception by 0.064, while AlexNet stays where it was.

### Slide 14 · Freeing half the positions helps in every participant

*Starts at 10:31 · about 53 s*

Paired image by image over the ten participants, freeing half the positions improves four metrics in every participant: CLIP, Inception, SwAV and the deeper AlexNet layer, each at p equal to 0.0137, the smallest a Holm-corrected exact test can reach with ten participants. It replicates across both samplers, two generation seeds, and bridge sizes spanning a factor of fifty-two.

The mechanism is simple: an unwritten position is filled by the generator's autoregressive prior, while a written one replaces that prior with a low signal-to-noise EEG estimate. The control confirms it: filling the freed positions with the prefix's own mean degrades every participant. So the pair K, q is a runtime trust parameter, how much of the brain estimate to impose, with no retraining and no added latency.

### Slide 15 · HINT beats the reference on five of seven metrics

*Starts at 11:24 · about 47 s*

Against the published reference, the deployed configuration, six scales at half the positions, is higher on five of seven metrics with the 420-million-parameter decoder, and on six of seven with the deeper VAR-d30, where only CLIP stays below.

The lower rows bound these numbers. A prefix from another image drops every identification score to chance. The true prefix reaches seven of seven, so the frozen generator is not the limit; the bridge is. And the flat-colour row is why I do not claim a new state of the art: it beats everyone on pixel correlation and SSIM. Without those two columns, HINT wins three of five, and four of five with the deeper decoder.

### Slide 16 · An order of magnitude lighter than the reference

*Starts at 12:11 · about 37 s*

The system is also light where the reference stakes its claim. Measured back to back on the same GPU, the generator is ten times smaller, needs nine times fewer FLOPs, and takes 90 milliseconds per image, 19 at batch 25. Adapting to a new subject takes under a minute of encoder training plus two seconds of ridge solves, and the bridge compresses fifty-two-fold at rank 32 while keeping both the count and the effect. One caveat: adaptation uses the subject's full training set, and single trials are not tested.

### Slide 17 · Two of the seven metrics cannot support a claim

*Starts at 12:48 · about 39 s*

Before leaving perception, three facts about the benchmark bound every number I have shown. First, the flat colour: on pixel correlation and SSIM it sits above the published state of the art and above our own configuration, so those two columns cannot carry a reconstruction claim, ours included. Second, the shared scoring code counts a self-comparison, so every two-way identification score is understated by exactly one in 199; we report corrected reference values. Third, every participant sees the same 200 test images, which caps the smallest resolvable paired difference near 0.02, whatever the budget.


---

## Part II · Imagination: what crosses

### Slide 18 · To imagine is to see without a stimulus

*Starts at 13:26 · about 63 s*

Part two. To imagine is to see without a stimulus. That makes imagery attractive as a control signal and expensive as a measurement. Decoders are calibrated on imagery trials, which are slow to collect, because imagining imposes a high cognitive load and cannot be serialised, whereas perception affords rapid designs that gather thousands of trials per hour. And nothing outside the participant marks the moment of imagining.

Within a person, perception and imagery are two readouts of one representation; in EEG, the shared part lives in posterior alpha oscillations. So could a decoder trained on perception read imagery? There are two axes: from perception to imagery, and from a pool of people to a new person. Each has been crossed on its own. Crossing both at once, for what is seen, had not been done to our knowledge; the nearest precedent, in motor imagery, crossed people but failed on the step into imagery.

### Slide 19 · Within one person, perception decodes and imagery does not

*Starts at 14:30 · about 57 s*

We began inside one participant, asking whether a person's own perception can stand in for their imagery. On the Wilson dataset, with whole trials held out, perception beats a twin trained on shuffled labels by about five points on all four seeds, imagery sits on its null, and every route from perception into imagery lands within one to three points of zero. The published imagery accuracy there depends on the split: cutting windows instead of trials puts near-copies of a trial on both sides and adds thirty points.

On the Shimizu dataset, a rotation between class means put imagery at twelve times chance, until a rotation fitted on a random pairing did just as well: it was routing the recording clock, not revealing a shared geometry. We withdrew it the same day and took the question across participants.

### Slide 20 · Train on 21 people's perception, test on the 22nd's imagery

*Starts at 15:27 · about 56 s*

The dataset is Gao and colleagues, 2026: twenty-two participants and three tasks, four everyday objects, three geometric figures, and three animals. The animal task was designated in advance, on published grounds, as the negative control. Each trial shows the image for four seconds, a mask for about two, then a cue to imagine it for four.

The pipeline was fixed in advance, following Xie and colleagues: log alpha power on ten posterior electrodes, each participant standardised to their own mean and variance without labels, trials averaged into pseudo-trials, and a linear SVM for each pair of classes. It trains on the pooled perception of twenty-one participants and is tested on the twenty-second, who contributes no labelled trial to any fit. Every number is in percentage points above its own permutation null, where chance is fifty percent.

### Slide 21 · Others' perception reads a held-out person's imagery

*Starts at 16:23 · about 43 s*

And it reads. On objects, a decoder fitted on other people's perception separates the held-out participant's imagery 5.3 points above its null, positive in eighteen of twenty-two participants, with p equal to 0.0002, and it survives Bonferroni correction over all thirty-seven primary comparisons. Figures are positive and smaller, 3.5 points, which we treat as convergent support.

The animal control sits at its null, even though animal perception crosses people at 13.5 points, so the null is not a failure to cross people. And pooling matters: with a single source participant the effect halves; twenty-one restore it to parity with the participant's own perception.

### Slide 22 · The effect passes every pre-declared control

*Starts at 17:06 · about 41 s*

Each of these checks was declared before it ran. The transfer grows with the number of source participants on objects and figures, not on animals. Across six bands and regions, posterior alpha, the boxed cell chosen in advance, is the only one positive on both signal tasks with the control at its null. Shifting the class labels cyclically abolishes the effect. Removing ocular components wipes out a frontal perception signal on the control task, from thirty points to nothing, and leaves the posterior readout unchanged. And source participants who share no presentation order with the target reproduce it.

### Slide 23 · The readout is posterior and precedes the cue

*Starts at 17:46 · about 42 s*

Where on the scalp? The decoder's forward patterns sit over posterior sites, in perception and in imagery alike. When? Reading the identical decoder across the trial, the pattern appears at image onset, falls to chance while the image is still on screen, rebuilds through the delay, and peaks in the imagery window.

Stating this honestly required guarding the windows: the wavelet reaches almost 0.4 seconds to either side, so an unguarded delay window leaks activity from after the cue. With a 450-millisecond guard band the leak falls a hundred-and-nineteen-fold, and the delay still reads 4.5 points, before any instruction to imagine.

### Slide 24 · What crosses is one standardised dimension

*Starts at 18:28 · about 46 s*

What crosses is small. It is one dimension: the common level of posterior alpha across the ten electrodes, measured as a departure from the participant's own distribution. On the raw feature the transfer sits at its null; standardised within each participant, the same feature reads almost six points.

Standardisation is necessary, and capacity is not: 205 scored architectures, nine alternative classifiers including tabular foundation models, and four pretrained EEG foundation models, and none beats the ten-feature linear readout. Out of sample, crossing people replicates, on a scene dataset with forty-nine participants. The step into imagery does not, including on the dataset of Xie and colleagues, where the alpha band itself came from.

### Slide 25 · Two limits belong to the dataset

*Starts at 19:14 · about 50 s*

Two limits belong to the dataset, and each has a design that would decide it. Every class is one fixed image, identical for all participants, so class identity and image identity are the same variable: what crosses may be the response to that one picture, and pooling more people makes that confound stronger, not weaker. The fix is several images per class, with the scored image never seen in training.

And the pattern precedes the cue, so short-term memory, sustained attention and imagery begun early all predict it. The fix is to reveal which item to imagine only at the cue. If it holds, calibration inverts: show the images to a group once, and the new person contributes only unlabelled epochs.


---

## Part III · What the two halves say together

### Slide 26 · The two halves agree on three points

*Starts at 20:05 · about 51 s*

The two halves share no data, no feature, no model and no statistic, yet they agree on three points. Both read something coarse from the EEG: in Part one, five tokens out of 680 accept brain information; in Part two, one standardised dimension crosses people.

In both, the limit is the map from brain to target, not the machinery on either side: the true feature wins six of seven, a better retrieval encoder did not improve reconstruction, and no classifier or pretrained model moved the crossing. And in both, the evaluation decided what could be said: a flat colour disqualifies two metrics, holding a participant out still leaves the stimulus in training, and every result we later withdrew had first passed a pre-registered protocol.

### Slide 27 · What the thesis answers

*Starts at 20:56 · about 56 s*

So, the answers. Can a frozen generator use the part of the EEG that carries appearance? Only at its two coarsest scales, and only through a linear readout: the fine-grained signal exists, about two R-squared points, but no mechanism built here carried it into a generated image. Does the perceptual code persist in another person once the stimulus is gone? On one corpus, other people's perception reads a held-out participant's delay and imagery by about five points, and the design cannot yet say whether that is imagery or the trace of a fixed image.

In both halves the next experiment is a change of design, not a larger model: a per-participant alignment on a shared network, spatial conditioning scored against a permuted prefix, many images per class, and an imagery target revealed only at the cue.

### Slide 28 · Thank you · Questions

*Starts at 21:52 · about 22 s*

Thank you to my supervisors, Dr. Nataliya Kos'myna and Professor Mathieu Salzmann; to Ninon Lizé Masclef, who worked with me on the imagery project from its first day to its last; to my friends at the Media Lab; and to the MIT ORCD cluster, where every experiment ran. I'm happy to take your questions.
