# SwinEEG and FREUD: routing a hierarchical EEG encoder into a coarse-to-fine image generator

*A companion text to Part I of the thesis (Chapter 3 and Section 5.2). FREUD is the project whose ideas led to HINT.*

## Where the idea comes from

Visual cortex is organised as a hierarchy. Simple cells in the primary visual cortex respond to oriented edges in a small patch of the visual field, a code matched to natural images, and later stages build increasingly task-specific features. The hierarchy is also a timetable: the gist of a scene reaches the higher areas first and is refined by feedback, and electrical recordings show coarse information arriving before fine detail. A scalp recording resolves little in space but resolves time, so for EEG, time indexes coarseness.

Image generators are ordered the same way. A next-scale autoregressive transformer (VAR) draws an image as ten token maps of growing resolution, from a single token that fixes the global colour to a 16×16 grid that adds texture, each map predicted from the coarser ones. Yet the EEG-to-image pipelines of early 2026 compressed each trial into one global vector and conditioned the generator through it: the generator was hierarchical, but the neural conditioning was flat.

FREUD asked what happens when the conditioning is made hierarchical as well. An EEG encoder that computes a hierarchy should be able to hand the generator the whole succession of its stages, each at the scale where it belongs, rather than only the vector at the top. That turns the analogy between cortical and generative hierarchies into something that can be tested.

## SwinEEG: a multi-scale EEG encoder

The encoder is SwinEEG, a four-stage variant of the Swin Transformer V2 built for this project. A trial is laid out as a grid of electrodes against short slices of time. Every stage merges neighbouring time slices but never merges electrodes: electrodes lie on a curved surface, and no ordering along one axis keeps neighbours adjacent, so the electrode axis is preserved at every depth. Position enters through a Fourier encoding of each electrode's coordinates and of the time of its slice.

The result is an encoder whose stages are themselves a multi-scale readout of the whole scalp. Figure 3.1 of the thesis shows it for two participants: every stage is still defined at every electrode, and what changes with depth is the pattern, frontal at the second-to-last stage and posterior at the last, in both participants. SwinEEG was pretrained by masked reconstruction of EEG patches (SimMIM) and then aligned contrastively to CLIP image embeddings (ViT-H/14). Trained with the same recipe, it matched LaBraM, a public EEG foundation model pretrained on 2,500 hours of recordings, at about 0.21 top-1 retrieval among 200 test images.

## FREUD: routing each stage to a generator scale

FREUD pairs SwinEEG with the VAR generator through a fixed routing table: the deepest, most semantic stage drives the three coarsest scales, and the shallowest stage drives the two finest. Each stage is summarised into one vector, and a small adaptor at each scale turns it into that scale's conditioning signal. Every adaptor starts at zero, so a routed model begins exactly where an unrouted one does, and any difference has to be learned.

The design comes with its own controls. A baseline sends one global vector to every scale, as the previous system on this generator did. A reversed routing sends the shallow stages to the coarse scales, which tests whether direction matters. And a learned router is free to choose its own assignment. All conditions were trained on THINGS-EEG2, where ten participants viewed thousands of natural images.

## What FREUD revealed

In training, the routed models ended up behaving like their global-vector baseline. Working out why produced the three insights on which HINT is built.

**Teacher forcing hides the brain signal during training.** A next-scale generator is trained with teacher forcing: to predict scale k, it receives the upsampled ground truth of every coarser scale, which is a blurred version of the answer. Given that context, any additional condition carries almost no information about the next token, I(x_k ; c | coarser scales) ≈ 0, so it receives almost no gradient and its path stays near zero. This is structural rather than specific to EEG: even the true CLIP embedding of the image, given in place of the EEG, leaves the validation loss unchanged (6.047 against 6.046 nats). Only the very first scale has no coarser context, and it is one token out of 680. The consequence is that brain information should be written where the generator cannot see the answer: at inference.

**The information lives in the coarsest scales.** Writing the true tokens of the viewed image into chosen scales of the frozen generator measures where the headroom is. The single coarsest token carries global appearance: forcing it raises an appearance score from 0.69 to 0.85. The second scale, four tokens, carries the spatial layout. Together these five tokens reach 0.95, while the four finest scales add pixel detail but no appearance. Twelve numbers, the image pooled to 2×2, predict the coarse latents better than the 1024-dimensional CLIP embedding that EEG encoders are usually trained on, almost five times better on layout.

**One good readout is enough.** Predicting 42 image targets, from coarse pixels to CLIP features, from every stage of two staged encoders, the deepest stage turned out to be the best source for every target, fine ones included. Brain readouts also decode best at 40 to 60% of the depth of image networks. So rather than one readout per stage, a single readout from the encoder's final representation can feed the coarse scales.

## From FREUD to HINT

These three insights define HINT (Hierarchical Injection of Neural Tokens). The generator is a VAR fine-tuned on image features only and then frozen. A closed-form ridge regression maps the EEG representation to the generator's coarse feature maps; the prediction is quantised and written into the coarsest scales at inference, and the generator completes the fine scales on its own. No gradient from EEG ever reaches the generator.

FREUD's own routing, moved to inference in this way, already raised the count against the published reference from three to five of seven metrics, and its direction control carried over: forcing the finest scales instead lost semantic content in every participant. HINT then added a second knob that prior work collapses into the first: besides how many scales are forced (K), what fraction of the positions within them is forced (q). Freeing half of the positions improves four metrics in every participant, because an unforced position returns to the generator's own prior instead of taking a noisy estimate. The deployed system beats the published reference on five of seven metrics (six of seven with a deeper decoder), with a generator about ten times smaller and under a minute of adaptation per new subject.

## What FREUD contributes

FREUD's contribution is the question and the design: treating the correspondence between the brain's coarse-to-fine hierarchy and a generator's coarse-to-fine schedule as something to measure, scale by scale. Its encoder showed that EEG can be read out at several depths without collapsing the scalp, and its experiments located where brain information enters a scale-wise generator and why training-time conditioning cannot reveal it. Moved to inference, its central idea became HINT.
