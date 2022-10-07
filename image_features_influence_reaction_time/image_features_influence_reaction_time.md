# Image Features Influence Reaction Time: A Learned Probabilistic Perceptual Model for Saccade Latency
https://d1qx31qr3h6wln.cloudfront.net/publications/final_paper_draft_authorversion.pdf

## Motivation
In FPS games, players may find themselves shooting at differently constructed models. In QL for instance, you can choose which playermodel enemies appear as, as well as their colour; a certain subset of colours may be optimal (most players seem to choose bright green, bright blue, or bright pink). For non-arena shooters, colours are often duller.

## Background
Much work has been done on modeling temporal acurity of the HVS, but not on reaction, which is highly important for FPS's and other types of games. Eye fixation changes 3-4 times per second with saccades, which are rapid exploratory movements. These help us understand our surroundings.
Gaze-contingent rendering, where eyes are tracked at a high frequency and gaze locations are found, have been well developed. Dynamic foveated rendering is one such method, but others could be applying other effects such as DoF.
Saccade amplitude, velocity, and duration are not linear, but often follow a bell-shaped profile; so, a high speed eye tracker could predict where an ongoing saccade could end, which could help accelerate rendering techniques. Additionally, "saccadic suppression" is temporary perceptual blindness (failure to notice objects), which occurs during and shortly after a saccade.

## Pilot Study to measure saccadic latency
The first experiment was a pilot study. They parameterize stimuli and observe / measure the correlation between image features and the time it takes to trigger a saccade in participants. They also looked to see if the correlation is different from visual acuity.
They use a Vive Pro Eye with seated participants. 5 participants (22-28, 3 female) underwent a 2AFC. 10 blocks of 225 trials occured over 2.5 hours, with breaks between blocks. Users would fixate at the center of the display and be presented some Gabor pattern at 10-20 degree eccentricity in the visual field, or at the fixation point. They then would make a saccade towards that pattern. Participants were told to state whether the Gabor pattern was oriented 45 degrees clockwise from vertical, or 45 degrees counter clockwise. The eccentricity, contrast, and frequency of the patterns were varied so that each combination was shown 5 times in each of the 10 blocks.

They noticed that saccidic movements were asymmetric. The shapes were fairly consistent, but the saccade time could vary by 25% (100ms). The contrast of stimuli decreases reaction latency, but increasing the frequency of the pattern increases the latency. The eccentricity caused a U-shaped effect, with the lowest average latency values of 265ms at around 10 degrees.

What they learned from the pilot study is that:

1. Visual differences below perceivable thresholds can still result in changing saccidic latency significantly; therefore, perception of visual differences is not an entire explanation for the change in latencies.
2. The probability distribution of saccidic latencies follows prior work on visual-oculomotor reactive latencies.
3. At a given eccentricity, as the visibility improves for the stimuli, the latency decreases (as we would expect).



## Probabilistic Model for Saccadic Latencies
They use the Drift Diffusion Model, which presents an amount of "evidence" which quantifies how much confidence a subject needs to reach a decision. Commonly used in econ and psych. Accumulating this evidence is done randomly / stochastically due to reaction uncertaninties. The evidence distribution will be a Gaussian distribution.

When implementing the model, data is first normalized in order to account for wide variations across individuals. They used Unity and the Vive Pro Eye to run the study.
They define a "primary" saccade to be the movement that moves the gaze towards the target location, and is within 3 degrees of the target location.

A Radial Basis Function Neural Network (eqn 8), which takes in some configurations from the pilot study blocks, was trained with Pytorch. 2000 epochs, and only 3 minutes to train on a 3090.

## Evaluation
Partitioning the test set is important. Two types were used: First, a random selection from all data points, and secondly, all data from each individual (n = 5) subject. The Kolmogorov-Smirnov goodness-of-fit test between the test data and the prediction is performed. Predictions below the line (where accuracy is perfect) means that the model underpredicted saccade latencies; values above the line indicate the opposite.
This is just measuring how the model, trained on data from the pilot test, performs against the test set. They also use it and evaluate how it performs on real-world applications, or "natural-task stimuli". 14 participants (3 female) were recruited for a 2AFC experiment for this, but 2 were excluded, so 12 total; but, 2 participated in the preliminary study as well.
2 Questions:
1. Can the model predict saccade latencies in natural tasks, and
2. Can objects be imperceptibly altered  while still influencing reaction times?
3 sets of images were used:
1. Soccer scene
2. FPS view
3. Photographs of an indoor shelf.


Each group had 51 images, with a target stimulus appearing at different locations.
The users were shown an image for 1-1.5 seconds. Then, they were either shown a target, or a non-target. If they were shown a target, they should saccade to it. If they were shown a non-target, they should remain fixated at wherever their eyes were. This lets them measure the visual-oculomotor latency for when a subject dentifies the discernible features of interest from a stimulus.

In each set of images, three variations of target stimulus were changed: Higher contrast / decreased frequency was one (Accelerated),  decreased contrast / increased frequency was another (Deferred), and unfiltered was the Control. Each participant did 51 images x 3 conditions x 3 scenes = 459 total trials, giving 5508 distinct data points.
" Measuring the precise frequen-
cies affecting saccade latency is a complex task requiring pooling
from multiband. Investigating a comprehensive pooling strategy
is beyond the scope of this work. Therefore, we approximate the
representative frequency as the Laplacian pyramid layer with the
highest corresponding contrast, for those images without a uniform
frequency pattern."
Using K-S, and the FovVideoVDP metric as a method to measure perceptible differences, they find p = 0.79-1.0 for measuring saccades...

## Extension to Foveal-Peripheral Dual Tasks
Foveal and peripheral pathways can operate independently and in parallel. Saccades occur after both have finished processing. They define the peripheral stimulus for this task to be at a 10 degree eccentricity. Then, the saccade time is max(Tfoveal, Tperipheral), although each of these can't be individually measured, only the total saccade time can be. But, they come up with a fancy MLE way to estimate them? There's a derivation for it in the appendix.

12 Participants (3 female) were recruited to perform a 2AFC experiment. Gabor patches were used again, and three were shown: One at each side of the periphery at a 10 degree eccentricity, and one at the fovea. The ones shown at the sides would be tilted +-45 degrees. All three were displayed together. The users were asked to saccade towards whichever peripheral gabor patch had the same orientation as the foveal gabor. The contrast of the patches is sampled from a select set ([0.05, 0.22, 0.53, 1.0]) and drawn independently with a fixed frequency.

This massively failed to predict saccade latencies however. The model was only built on foveal saccades, and gave p = 0.002...


## E-sports
They downloaded a half hour of CSGO footage and then sampled 95 frames from there. YOLO was used to identify CT or T side players. Presumably, the POV gaze is on the crosshair, and predict the time when the viewer reorients their gaze to each target. They find that the average normalized latency for searching for CT players is 0.92 +- 0.02, while searching for Tside players is 0.95 +- 0.04. The mean saccade latency for pro CSGO players has been estimated to be 282ms, and so this results in a 9.3ms difference.