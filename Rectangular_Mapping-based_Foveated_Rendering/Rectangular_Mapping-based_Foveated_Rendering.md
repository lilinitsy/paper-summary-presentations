# Rectangular Mapping-based Foveated Rendering
https://3dvar.com/Ye2022Rectangular.pdf

## Background
### Foveated Rendering
In normal desktop rendering, the entire screen is rendered at as high of a resolution as the user desires (subject to computational limitations). The edges of the screen will be equally high fidelity to the center or any other segment of the screen.

Our eyes do not observe and process every part of its vision with equal fidelity - our periphery is much harder to see in detail than what's in front of us, or what's in front of where our eyes are looking. The absence of periphery (tunnel vision) is very noticeable and a hinderance, so despite its lower level of detail, it is still important. But, since it won't appear at the same quality to our eyes as what our eyes focus on, it need not be rendered in such an equivalent quality. With **foveated rendering**, the render quality of the periphery will be lowered, while maintaining a high visual quality where the eyes are looking. Thus, we do not have a uniformly sampled screen, as the sampling depends on the level of foveation.

![Alt Text](images/fov.jpg?raw=true)

In systems without eye tracking, foveated rendering is usually fixed, rendering the fovea at the center of the  screen and the lower quality periphery around it. In eye-tracking systems, dynamic foveated rendering renders the high-fidelity fovea at the focus of the two eyes, with the rest using whatever sort of mapping is implemented.


### Log-Polar Foveation
The log-polar coordinate system is 2D, with one axis being the logarithm of a distance to a point, and the other being an angle. Our eyes generally follow an approximately log-polar mapping for how we view scenes outside of the focus. There are a lot of ways to do this. One way could be to take an image rendered in cartesian (xy) coordinates, map it to log-polar space, and then reproject to the cartesian plane. There are tons of different ways to implement this for different types of renderers, with nuances and other effects, and all have varying degrees of foveation.

Other models also exist, but they lack widespread use.

### Deferred Rendering
Typically, things are rendered in some kind of "forward" renderer, and this is very common in VR. In forward renderers, we will render an object and then light it based on all the lights that are applicable to it. Additionally, depth information may not be completely leveraged to lower rendering costs.

Deferred rendering defers parts of the rendering to later stages. First, a geometry pass will occur, where positions, normals, and materials are rendered into a G-buffer (geometry buffer). Then, a fragment shader will execute utilizing the G-buffer.

![Alt Text](images/learnopengl_deferred1.PNG?raw=true)

![Alt Text](images/learnopengl_deferred2.PNG?raw=true)

## Paper's Approach
Pixels exist in 2D space. They discuss redistributing these to exist in 1 dimensional space, but then don't do it... In one dimension, where the pixel distribution is flattened into 1D space, the new sample points d<sub>i</sub> can be calculated with:

![Alt Text](images/1d-mapping.PNG?raw=true)

Essentially, more samples can be taken around the origin of the eye focus, and fewer beyond there. This is fairly obvious, and how other foveated approaches work anyway, but now they use a 2D version of this it seems.


The vertex and fragment shader passes are the same as in a regular deferred shading pipeline, but they introduce the rectangular mapping and inverse rectangular mapping steps. The G-Buffer is constructed with W x H resolution. After that, the rectangular mapping transformation is performed and stored in a Transformed Geometry Buffer (TG-Buffer), with resolution w x h. For each foveal point (what's this mean exactly? They don't clarify), each pixel (x', y') is mapped in the G-buffer accordingly.

![Alt Text](images/2d-mapping.PNG?raw=true)

This is still the G-Buffer. It is mapped into the TG-Buffer (u, v) with:

![Alt Text](images/2d-mapping2.PNG?raw=true)

Where Nx and Ny are normalization functions.

![Alt Text](images/2d-mapping3.PNG?raw=true)

Now, a lighting pass must occur. Information in this TG-Buffer is used to do the lighting pass, and rendered to a reduced-resolution buffer (RS-Buffer). If we let $s = \frac{W}{w} = \frac{H}{h}$, we see that shading cost is linear to the number of pixels. This will shade $w x h < W x H$ pixels, and so the maximum theoretical speedup of using this rectangular mapping is $s^2$.

One more pass must occur: the inverse rectangular mapping, or mapping the RS-Buffer to the full-res screen.

## Improvements of RMFR
They posit that images with vertical patterns can be sensitive to x-axis compression, and similarily for horizontally patterned imagery. For scenes with more of either, more or less foveation could be applied accordingly, while this is not possible in log-polar space. This does seem like sort of a stretch to argue as a use case, though.

They measure re Foveated Peak Signal-to-Noise Ratio (FPSN), Peak Signal-to-Noise Ratio (PSNR), and Structural Similarity (SSI), with a compression parameter s = 2.2, with a resolution of 1440x1600.

![Alt Text](images/figure5.PNG?raw=true)

A claim that *does* seem valid is robustness to antialiasing. Log-Polar foveation may not necessarily perform well without higher sampling rates, most likely, and so aliasing will be rather curved along image gradients.	

![Alt Text](images/figure7.PNG?raw=true)

## User Study
First, a pilot study was conducted to get a coarse estimate of the optimal sampling distribution for the parameters.

### Equipment
- GTX 980
- Vive Pro Eye (120hz eye tracking, 2880x1600)
- Unity

There were 12 participants (9M 3F), aged 21-32 (m = 23.9, std = 2.9)

The s's tried were 2.2, 2.5, and 2.8. Tests were done on 6 scenes with 48 trials each. Participants were shown either the full-resolution scene, or the rectangular-mapping based scene with the given parameters for 2 seconds before a black interval of .75s, and then were shown the other image. They then scored the images they saw on a scale of 1 to 3 (3: perceptually identical, 2: minimally different: 1: significantly different).

![Alt Text](images/figure8.PNG?raw=true)

After the pilot study, participants do much the same thing, but score the compression parameter instead.

Then, a comparison study was run as a 2-Alternate Force experiment. One image shown was the rectangular-mapping generated image, and the other was either another type of foveated rendering produced image, or the ground-truth, no foveation image. They run on the same 6 scenes, with 4 trials each (comparing RMFR vs 3 other methods and the ground truth). They also selected 30% of the trials as "validation" trials as in Meng et. al. In these validation trials, the user was presented with two identical images of the full-resolution image. If they said that the two images were not identical, they were asked to take a break for at least 30 seconds. If a user makes 3 or more errors, their data is discarded.

For each participant, the average perceptual scores $S_k(m, s, f_x, f_y)$ across $s_i$ is taken. Among "most" participants, a consensus and similar "hotspots" are found, so those averages are then averaged across participants.

![Alt Text](images/figure10.PNG?raw=true)

They also have some nifty heatmaps of the pixel density across different foveation methods.

![Alt Text](images/figure11.PNG?raw=true)

People seemed to prefer rectangular fall-off.
![Alt Text](images/table1.PNG?raw=true)

RMFR performs quicker than the full resolution, but they do not compare runtime against other foveation methods.
![Alt Text](images/table3.PNG?raw=true)
