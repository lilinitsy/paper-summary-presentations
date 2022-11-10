# FoV-NeRF: Foveated Neural Radiance Fields for VR

https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9872532

This paper presents a gaze-contindent neural radiance approach with foveated rendering. Human vision's stereoscopic acuity and temporal sensitivity are encoded, providing improved responsiveness from neural methods without losing visual fidelity. 3D radiance fields with spherical coordinates are used.

Contributions:
- lowlatency/high-fidelity application
- 3D NeRF representation for egocentric viewing
- human visual system neural synthesis method, considering visual and stereo acuity
- spatio-temporal model for optimizing system latency with perceptual quality

## Background
A deep neural network can store the environment as an implicit function, allowing it to approximate an object's appearance given a pose. Signed distance functions can represent point clouds efficiently from 2D images, 3D contexts, and temporal vector fields. Many of these approaches have a narrow viewing window though, with a high latency and low resolution on top.

The other important background is in gaze-contingent rendering, for which recent nvidia works provide a more adequate background than the described section.

## Methodology
The first main rendering step is synthesizing visual/stereoscopic acuity images with raymarching for foveal, mid, and far peripheries. Then, IBR composites frames.

Egocentric coordinates can be used. Panormaic disocclusion methods can show the FPS views with concentric spherical coordinates, which will allow for real-time rendering and 6DoF navigation. This is parameterized with the number of concentric spheres per NN (N) and their radii; thus, given a 3D position q = q(r, theta, phi), we have a 3D spatial representation. The goal is to predict an rgbd vector for each q and then ray march for each pixel (d is density, not depth).

Egocentric neural representations might encode fewer viewing changes but more spatial variations, so the network should infer an array of vectors for each viewing ray.

![Alt Text](images/eqn1.png?raw=true)

L is encoded as an MLP network. NeRF predicts each intersection point before calculating the final colour, but that causes N network inference operations per ray. The egocentric view representation in Eqn 1 feeds the N coordinates from a vector to the MLP with one inference call instead of N operations. Nm fully connected layers have Nc channels in each layer.

Neural rendering has been historically slow; however, multiple elemental images are synthesized at once to lower overall runtime. "Elemental images", though they don't say what this means.

When a camera position, direction, and gaze position are tracked, these three elemental images cover different eccentricity ranges; the fovea, mid eceentricity, and the entire visual field. If, Im, and Ip for each left and right eye provide parallax depth cues; since we are doing stereoscopic rendering, the computational overhead doubles. Some blending and shifting occurs to prevent image disparities.

![Alt Text](images/fov-nerf-fig3.png?raw=true)

![Alt Text](images/fov-nerf-fig4.png?raw=true)

With this egocentric representation, a q point is reprojected to the nearest point on a sphere that can connect it to the origin. These points can cause some sparsity that introduces error from the approximation quality.

![Alt Text](images/fov-nerf-eqn2.png?raw=true)

All q's (vertices) and camera positions can be integerated, coming up with the scene. When expanding to mimage space, there is an error given a camera's projection matrix.

![Alt Text](images/fov-nerf-eqn6.png?raw=true)

Higher representation densities correlates to higher image quality, and higher latency.

Spatial-temporal joint modeling (also see Temporally Stable Real-Time Joint Neural Denoising and Supersampling for more info, it's not cited but does this better) will determine the optimal coordinates ystem (N) related to precision and network complexity.

![Alt Text](images/fov-nerf-eqn7.png?raw=true)

Lnm,nc(q) are the RGBA output L1 distance between a network setting and highest values.

They compute that since the latency for a foveated system has to be below 50ms to avoid undetectable artifacts, and their device has an eye tracking latency of 12ms and photon submission latency of 14ms, this must conclude within 24ms.

## Evaluation

### Experiment 1
They use a Blender-Unity rendering engine type thing. Four data sets were used from 4 scenes. Two separate datasets for each scene were used to train separate foveal and peripheral networks. They implement their system with OpenGL and CUDA with TensorRT. A xeon and 3090 wer used. 
IMPORTANT: Training happens per scene. This is a major weakness of this approach as of now.

A psychophysical experiment comparing the proposed solution with NeRF and ground-truth images was conducted. Static images from the same views were presented, generated across the same and randomly defined gaze direction across conditions. 1440x1600 resolution per eye.

![Alt Text](images/fov-nerf-fig7-1.png?raw=true)

![Alt Text](images/fov-nerf-fig7-2.png?raw=true)

The models were retrained with different parameters. Users wore the Vive Pro Eye and were seated. 6F, 6M = 12 users, corrected to normal vision, etc. The experiment used a 2AFC task. Each trial had a stimuli from 2/3 methods with the view and gaze position. Static gaze was enforced to avoid gaze motion variances. Each image appeared for 500ms. A black screen then appears for 300ms, and the other image appears next. Users were told to fixate on a green cross on the display, and gazes were tracked. Whenever a gaze was more than 5 degres away from the target, the trial was dropped immediately via the black screen. After each trial, participants selected which of the two stimuli appeared higher quality. 6 trials were done as a break-in / trial period before data collection began. 36 trials, 6 trials eper pair of ordered conditions, counterbalanced, and breaks after each scene of 1 minute occurred to avoid fatigue. They could also take as much time as they wanted to choose which one was better. They don't seem to say whether they could say they were *equivalent* quality however, which as studied in a few other papers, might make more sense... They *shouldn't* say that either of the NeRF implementations are better than ground truth, they would ideally say they're indistinguishable; although, presumably, a 50/50 GT-Theirs is also somewhat like that.

For results, they compare ground truth to theirs, theirs to NeRF, and GT over NeRF.

GT vs theirs: 42.4% preferred theirs to GT, SD = 0.23. No p value listed, so presumably it's not a significant difference.
GT vs NeRF: 91% preferred GT, p < 0.005, SD = 0.11
NeRF vs theirs: 89.6% preferred theirs, SD = 0.09, p < 0.0005

They also calculate the effect size when the sample size was 12, for a .8 power. For a .8 power, the effect size is at least 1.69 and the effect size Cohen' sd = 3.5 for theirs vs NeRF and 4.5 for GT vs NeRF.

They claim that the close-to-random and lack of significance between GT vs theirs suggests that theirs was seen as statistically perpetually similar.  NeRF also picks identical image quality (as a goal)  throughout the entire visual field, as opposed to concentrating on the foveal regions.

LPIPS is a method to compare deep perceptual similarity. It uses deep neural networks to estimate perceptual simularities between images. I'm not sure why they use this over SSIM or other metrics, but whatever.

![Alt Text](images/fov-nerf-fig7-3.png?raw=true)

![Alt Text](images/fov-nerf-fig7-4.png?raw=true)
