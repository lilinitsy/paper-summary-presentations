# Gaze-Contingent Perceptual Level of Detail Prediction

https://diglib.eg.org/handle/10.2312/sr20231130

This paper presents a geometry-space approach go foveated rendering using LOD reduction.
They present two perceptual models for mesh simplification and end up achieving  a 33% performance speedup when combined with Nanite.

Contributions:
- Introduction of a perceptually inspired visibility model for LoD and using them to select the most simple LOD while keeping changes to geometry below a selected visibility threshold.
- A spatial model focuses on the visibilty of changes in the object silhouette and body.
- A temporal model aims to measure the visibility of switching between LOD's in a dynamic scene.

I'll skip background work because I've covered some of this in other talks.

## Methodology
The authors point out that low quality meshes can result in visible geometry distortions, and mesh inaccuracy can result in normals and texture coords being wrong.
Poppping may also arise - based on their video and study design, they don't really address this.

These are the factors they list for visibility changes:
- Number of polygons (mesh complexity)
- Size of the shape
- Eccentricity of the shape to the gaze position
- Position / type of light sources
- Texture info
- Object and background motion
- Luminance of the display

Their model doesn't account for luminance and... motion... which is where popping becomes important.

## Spatial Model
Their spatial model takes a reference input image of the full object and the test image of the model using different LoD's (R0 and Ri respectively).
It also takes in an eccentricity, so three params.
The model will predict the probability that an observer will detect the spatial differences by *LODi* at eccentricity *ecc*
They write, "Our temporal model is designed to handle popping artifacts when the LOD level changes which switching to a coraser or more detailed mesh."
Watch the video to assess whether this is true...

They assume that geometric distortions are most visible around object boundaries.
These are referred to as the distortions between silhouettes of objects; S(R0) and S(Ri), or S0 and Si.
The geometric distortion is measured as the screen space area of all pixels that are in either silhouette, but not in both; an xor.
They compute a weighted area with weights decreasing based on eccentricity.
Finally, they account for area differences due to the difference in size across objects, and so they normalize the area by the perimeter of the reference silhouette.
This gives us:

![Alt Text](images/eqn1.JPG?raw=true)

p is an image pixel, eccp is the "retinal eccentricity" of the pixel p, Ap is the area of one pixel, p(S0) is the silhouette's perimeter, and w is the weighting function.
They are "inspired" by a softplus function which avoids negative values and has exponential and linear portions.
The weighting function then is:

![Alt Text](images/eqn2.JPG?raw=true)

alpha, beta, s, and b are free parameters determined later.

### Accounting for shading distortions from low quality models
LOD's could have incorrect normals and texture coordinates, so FovVideoVDP is used to detect differences due to shading.
They replace the (S0 x S1) region with a gray background colour before giving them to FovVideoVDP.

Finally, they estimate distortion magnitude using something from FovVideoVDP as:

![Alt Text](images/eqn3.JPG?raw=true)

where Wg, Wvdp, and ks are optimized parameters. 
The probability of detection is a sigmoid function, with many parameters calibrated to fit perceptual data:

![Alt Text](images/eqn4.JPG?raw=true)


## Temporal Model
They adapt a model from Tursen et. al 

(TD22] TURSUN, CARA and DIDYK, PIOTR. “Perceptual Visibility Model
for Temporal Contrast Changes in Periphery”. ACM Trans. Graph. 42.2
(Nov. 2022). ISSN: 0730-0301. DOI: 10.1145/3564241 4–6.))

The model takes Ri and Ri+1 (consecutive LOD's) and the eccentricity of the objects to predict the probablity that the transition will be detected.
They did a few other things to modify the model to get "pooled" probability for the entire object; this might be across all LOD's?
LOD changes are instantaneous, which makes me skeptical of this model's effectiveness. 
Pooled values are converted to a probability using a sigmoid, once again.

![Alt Text](images/eqn5.JPG?raw=true)


## Calibrating the models


Many of the free parameters are calibrated using images with different amounts of LOD, ie varying distortion levels, and the probabilities of detecting these at varying eccentricities.

Eight meshes from Stanford, Blender, and Unity's repo's were used, all with grayscale, so no texture information -- which is **quite** important in affecting visual perception.

![Alt Text](images/modellist.JPG?raw=true)

Each model Geometries were used corresponding to four LOD levels, and each of the resulting geometries was rendered in three sizes spanning between 1.22 and 12.22 degrees on the screen.

Two environments were used: An outdoor and indoor one, although I'm not familiar with either of their examples, and the rendering quality is low.

A 2AFC procedure was used. In each trial, participants were shown 3 images; the reference was shown in the center, and two test images were shown on either side in the center.
One test image was an exact copy of the reference, while the ot her was the LOD.
A cross was shown in the center as the fixation target. 
Gaze position was monitored using an eye tracker (I don't think this was done in a headset?)

Test images weere shown at 7.88, 15.47, and 22.56 degrees eccentricity. 
In the smallest eccentricity, the largest size of the objects wasn't used since they'd overlap the reference image.
256 stimuli were shown (8 meshes * 4 LOD's * 3 sizes * 3 eccentricities - 1 size * 8 meshes * 4LODs), and each trial was repeated twice; ie, each participant underwent 512 trials, and stimuli order was randomized.

They used a Tobii Pro Spectrum eye tracker at 600Hz connected to a 27 inch Acer Predator 4K monitor at 120hz, peak luminance of 170 cd/m^2.
15 participants were used.

The training dataset is a set of tuples

![Alt Text](images/tuples.JPG?raw=true)

where vk is a binary 1 if the observer detected visual distortions from the LOD.
Optimal parameters are then estimated by minimizing this function:

![Alt Text](images/argmin.JPG?raw=true)

The model is evaluated by the correlation and mean absolute error between the predicted probabilities given by equation 4 and P(R0, Ri, eccj).
In a 4-fold cross-validation, the Pearson correlation coefficient is 0.74 +- 0.0051 and the MAE is 0.10 +- 0.006.
When the entire dataset is used, the Pearson coefficient is 0.81 and MAE is 0.09.

Here are the optimized parameters:

![Alt Text](images/table1.JPG?raw=true)

## Gaze-Contingent LOD Prediction
The spatial and temporal models are used to select LOD at varying eccentricities; a mapping L(d, ecc) where d is the distance to the rendered object.
They use a  predefined detection threshold; therefore, the probability that the model change is detected should be below this threshold.
They use a threshold of 0.75, which they note is the midpoint betweenst seeing and not seeing distortions.

![Alt Text](images/eqn9.JPG?raw=true)

This is for the spatial model.
They use the same detectability threshold to avoid visible temporal changes when LOD's change.

![Alt Text](images/eqn10.JPG?raw=true)

Actually evaluating these is expensive in real time, so the mappings are precomupted and stored in a lookup table.
The conditions are evaluated for a varying set of object orientations.

## Evaluation
The model is evaluated using UE5 and Nanite.

In Cathedral, 16 objects were placed on the sides of the main aisle.
Objects were always chosen from a set of 5 geometries, separate for what was used in the calibration method.
Each object was rendered with a different material, but no textures.

![Alt Text](images/fig7.JPG?raw=true)

The lighting came from an environment map.

The second scene came from Courtyard illuminated with a directional light source.
40 randomly drawn objects from 8 meshes previously used in the calibration scene were used.

10 participants were asked to start.
Experimenters were given an explanation of popping artifacts so participants "took it into account in their evaluation."
In Cathedral, users could move around, but in Courtyard, they could only rotate the camera.
They used a 3090, 12900K, and had 128GB of RAM. UE5.1 was used.

Users were exposed in both VR and desktop setups.
Here are the results of preference for their method for both scenes using a binomial test.
These don't seem like strong results.

![Alt Text](images/table3.JPG?raw=true)


They compare against Nanite to showcase how the two techniques compare, and work with eachother; they don't argue to replace Nanite, but Nanite doesn't take eccentricity into account.
To compare, they tested rendering performance at 1k = 1024x540, 2k = 2048x1080, and 4k= 4096x2160 -- odd resolutions.
They used only two reference objects in four configurations; Nanite only, Ours only, Nanite + Ours, All Off.

2184 copies of a reference object were used in each scene, with 2.20 bilion triangles for Suzanne and 2.36 billion triangles for Asian Dragon total.

![Alt Text](images/fig8.JPG?raw=true)

They find that Nanite + theirs achieved a speedup of 33% at 2K resolution and 24% at 4K resolution compared to Nanite only.