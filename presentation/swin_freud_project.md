# SwinEEG and FREUD: injecting Swin-transformer EEG features, scale by scale, into an image generator

*A companion text to Part I of the thesis (Chapter 3 and Section 5.2).*

## The idea

In the EEG-to-image pipelines of early 2026, an encoder mapped one trial to one vector, and everything it computed on the way was discarded. The image generator on the other side was hierarchical: a next-scale autoregressive transformer (VAR) draws a picture as ten token maps of growing resolution, from a single token that fixes the global colour and layout to a 16×16 grid that fixes texture. Visual cortex is ordered the same way, and its hierarchy is also a timetable: coarse information arrives before fine detail.

The project started from one premise: an EEG encoder that computes a hierarchy should hand a generator that consumes a hierarchy the whole succession of its stages, not only the vector at the top. The encoder was SwinEEG; the routing from its stages into the generator's scales was FREUD.

## SwinEEG: a hierarchical encoder that never merges electrodes

SwinEEG, built between March and May 2026, is a four-stage variant of the Swin Transformer V2. A trial is laid out as a grid of electrodes against short slices of time. Every stage merges neighbouring time slices and leaves the electrode axis intact, because electrodes lie on a curved surface and no ordering along one axis keeps neighbours adjacent. Position enters through a Fourier encoding of each electrode's coordinates and of the time of its slice. Each stage is therefore a readout of the whole montage at a different depth, which is what a stage-to-scale routing needs; Figure 3.1 of the thesis shows the electrode axis surviving intact to the deepest stage.

Training had two phases. A self-supervised phase reconstructed masked EEG patches, following SimMIM. It ran on the training and test recordings together, so the encoder saw the EEG of the 200 test concepts (never their labels), a transductive step that neither comparison system takes. A contrastive phase then aligned the encoder to frozen CLIP image embeddings (ViT-H/14). Top-1 retrieval among the 200 test images reached 0.214, against 0.300 for AVDE, the published system built on the same generator; both figures are upper bounds, selected on the test data.

Two measurements qualify the design. The attention windows hold nine consecutive electrodes of a front-to-back list rather than patches of scalp, and a trained encoder attends almost uniformly across the scalp (Figure 3.2), so the claim that the windows separate occipital from frontal channels holds for only one window. And the retrieval gap was first attributed to a smaller pretraining corpus, but AVDE's own encoder, LaBraM, fine-tuned in-house with our recipe, reached 0.2175, the same level as ours. The surviving explanation is the recipe: every one of the forty-seven contrastive launches was stopped by the cluster's time limit before the end of its schedule.

## FREUD: a fixed table from encoder depth to generator scale

FREUD paired SwinEEG with the VAR generator. A fixed table sends the deepest, most semantic stage to the three coarsest scales and the shallowest stage to the two finest. Each stage is averaged over electrodes and time into one vector, and a small adaptor at each scale turns that vector into the scale's conditioning signal. Every adaptor starts at zero, so at the first training step a routed model equals an unrouted one.

Four conditions were trained, differing only in the table: a baseline that sends one global vector to every scale (as AVDE does), the depth-aligned routing (the proposal), a reversed routing (the negative control for direction) and a learned router. Each was trained once, at one seed. Because training scores every token equally, a scale's share of the learning signal is its share of the token positions: the single-token scale driven by the "most semantic" stage carries 0.147% of the objective under the proposed routing, and no condition separates the effect of direction from the effect of that share.

The paper was submitted to the ICML 2026 workshop on foundation models for generation and rejected. The area chair asked for more subjects, a seed analysis and evidence on the learned routing.

## What reading the code against the paper found

The main claim rested on one table: four conditions, one participant, one seed, and a difference of 0.013 between proposal and baseline on one metric, with no standard error. The code that produced it differs from the paper in at least seven places. The per-scale signal is described as entering each block through adaptive layer normalisation; the code adds it to the residual stream, and the normalisation layers see only the global vector. The decoder is described as frozen; every decoder weight is in the optimiser in all four conditions. The adaptors are described as containing a nonlinearity; they have none, and apply undeclared normalisations and clamps instead. The learned router, said to converge on the aligned assignment, started with seven tenths of its preference there. And one line of the setup code disables the default weight initialisation "for speed" before the injection machinery is attached.

That line decided the outcome. Nine trained models survive. In six of them, the stage normalisation held zeros (two of these are the no-routing arms, which never built the path), so the adaptor weights and both optimiser moments were still exactly zero after 103,375 steps: every scale received a constant, independent of the EEG, and another image's EEG produced bit-identical pictures. The other three ran on stray uninitialised memory, and there the brain-dependent part of the signal moves the teacher-forced loss by at most 3×10⁻⁴ nat at any scale. All nine score zero of seven metrics against the published reference.

The four variants of one participant are effectively one network: their weights differ by about one part per thousand, and their validation losses coincide at every epoch. Of the ninety routed-minus-baseline differences in the paper's own tables, none exceeds two standard errors, where chance alone would produce four or five. Repairing the initialisation gave every adaptor a gradient and changed nothing: fifty more epochs of training cut the generator's free-running pixel correlation by more than half, routed and unrouted alike.

## Why it could not have worked in training

The null is not only a bug. Rebuilt correctly on another encoder, the per-scale signal matched its no-conditioning twin to three decimals, and seven further ways of injecting EEG scale by scale did the same, including one that replaced the EEG with the true CLIP embedding of the image (validation loss 6.047 against 6.046 nats).

One structural property explains all of them. A next-scale generator is trained with teacher forcing: the input at scale k is the upsampled ground truth of every coarser scale, a blurred version of the answer. Given that context, the condition carries almost no information about the next token, I(x_k ; c | coarser scales) ≈ 0, so its path receives no gradient and decays to zero; this is the posterior-collapse pattern of powerful autoregressive decoders. Only the first scale has no coarser context, and it is one token of 680, 0.15% of the loss. Pinning it down takes 7.74 bits; the best participant's EEG supplies 1.56.

## The premise, measured

The routing also rests on an analogy that the data do not support. Predicting 42 image targets, from coarse pixels to CLIP features, from every stage of two staged encoder families, the deepest stage beats the three shallower ones in all 126 cells, fine targets included, while the two shallowest stages sit at their shuffled controls. Across image networks, brain readouts decode best at 40 to 60% of relative depth (seven of eight networks). And at inference, a routing shifted one scale finer beats the depth-aligned one in ten of ten participants. Scale k does not belong to stage k.

## What survived

The idea of writing brain information into the generator's scales survived once it moved from training to inference. On a frozen VAR that never saw EEG, a ridge regression predicts the generator's coarse feature map from a frozen EEG encoder, and the quantised prediction is forced into the two or three coarsest scales. That raised the count against the published reference from three of seven metrics to five, while forcing the finest scales instead lost meaning in every participant. The paper version of this method, HINT, adds that forcing half of the positions within those scales beats forcing all of them on four metrics, in every participant.

## What the project taught

Four rules carried into the rest of the thesis. A conditioning mechanism is judged on generated images, by the gap between correct and shuffled EEG, never on the teacher-forced loss. A difference measured on one participant at one seed is not evidence. A described system is checked against the code that produced its numbers. And every abandoned direction is reported with the measurement that closed it, which is why SwinEEG and FREUD appear in the thesis in full rather than as a footnote.
