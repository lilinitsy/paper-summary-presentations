# A Log-Rectilinear Transformation for Foveated 360-Degree Video Streaming (Paper Summary)

Github code: https://github.com/AugmentariumLab/foveated-360-video

## Background
### Foveated Rendering
In normal desktop rendering, the entire screen is rendered at as high of a resolution as the user desires (subject to computational limitations). The edges of the screen will be equally high fidelity to the center or any other segment of the screen; thus, each portion of the monitor is approximately equally intensive to render.

Our eyes do not "render" every part of its vision with equal fidelity - our periphery is much harder to see in detail than what's in front of us, or what's in front of where our eyes are looking. The absence of periphery (tunnel vision) is very noticeable and a hinderance, so despite its lower level of detail, it is still important. But, since it won't appear at the same quality to our eyes as what our eyes focus on, it need not be rendered in such an equivalent quality. With **foveated rendering**, the render quality of the periphery will be lowered, while maintaining a high visual quality where the eyes are looking. 

![Alt Text](images/fov.jpg?raw=true)

#### Video Streaming
YouTube, Twitch, and Netflix are examples of services that provide real-time video streaming. Improved visual quality can be streamed (480p, 720p, 1080, even 4k) for more crisp displaying pleasure at the price of increased bandwidth usage -- many modern internet connections won't support downloading 4k video streams without buffering, and this is for a flat screen, let alone for a 360 degree field of view.

### 360-Degree Video Challenges
Most of the pixels in a 360 degre video are invisible, since your field of view is less than 180 degrees horizontally, and the HMD's is typically even less. But, streaming an entire 360 degree video will require a ton of bandwidth to transmit many pixels that won't even be seen. Additionally, they often have a much worse perceived visual quality than traditional videos.

One solution is to use a "multiple resolution" approach. A video can be divided into multiple video tils, and each tile will be encoded at a different resolution - this is like doing foveated rendering. Then, the tiles can be streamed independently as their encoding finishes, and combined into one single video frame. However, if there aren't enough tiles, there can be tearing in the reconstructed frame, and perhaps even colours that don't match. Of course, more tiles means more tiles being streamed, and more being reconstructed, and so computational time can increase.

![Alt Text](images/foveated-tiles.PNG?raw=true)

## Paper's Solution
In this paper, the authors decided on a log-rectilinear transformation which preserves the full resolution around the gaze position (where the eyes look), and implement a blur along the periphery.

Typically, a Log-polar transformation is used to emulate the falloff for the eye's visual perception. The authors argue that the sampling methods aruond the gaze position could end up undersampling as it may sample the same pixel multiple times, which can cause artifacting and flickering.

![Alt Text](images/log-polar.png?raw=true)

The log-rectilinear transformation proposed provides rectangular transformations, which allows for constant-time filtering. It also gives a 1-1 mapping from the full-resolution video frame to the reduced resolution buffer. The exact fidelity decay they use is described in **Section 3.1** of the exact paper. They also do not sample directly from the image, since that, with foveation, can produce artifacts. Instead, they use a summed-area table to sample from. This lets them get the average sampled colour value with only four memory reads for each pixel -- if multiple GPU's were available, this could be done in parallel, potentially.

![](images/log-polar2.png?raw=true)

The entire server pipeline works as follows: In the first stage, there is server-side video decoding. As is common, the video is converted from YCbCr to RGB (24 bits per pixel). They use FFmpeg for this, which can take advantage of hardware acceleration. In stage 2, the summed-area tables are computed, once again on the server. This is done per-image, and is done in an OpenCL kernel. In the third stage, the log-rectilinear buffer is generated. For 1080p images, it can be downsampled to 1072x608, as that has been found to be conformant to a ratio of at least 1.8 which is indistinguishable for users using a log-polar foveated VR implementation. This is again done using an OpenCL kernel. In the fourth stage, this is actually encoded.

The client's pipeline is largely dependent on the server. First it will decode the packet that the server sent; this packet is the encoded log-rectilinear buffer. Then, on the GPU, it's converted into a full-resolution foveated video frame using bilinear interpolation.

![](images/pipeline.png?raw=true)


## Measurements
- Server had a 2080 Ti
- Client had a 1050
- Kernels were done in OpenCL rather than CUDA.

We can see that the video quality of the log-rectilinear approach is superior to the traditional log-polar one -- even without the summed area tables (SAT's).
![](images/bitrate-quality.png?raw=true)


The packets transmitted are also decently smaller than the log-polar packets when using the summed are tables. This is important for low-bandwidth systems.
![](images/bitrate-packetsize.png?raw=true)

One thing to note is that the total time to process is higher than for other approaches (12.44ms or the SAT log-rectilinear vs 11.46ms, so about 1ms more -- which is not trivial).

The log-rectilinear approach has less flickering than the log-polar implementation. 
![](images/flicker.png?raw=true)


#### Micro-optimizing out those branch statements... Actually very important in GPU pipelines that don't have branch prediction / good brandh prediction!
```cpp
float top_left_value = source_buffer[top_left_coord + color] * (rect_x > 0);
float top_right_value =
	source_buffer[top_right_coord + color] * (rect_y > 0);
float bottom_left_value =
	source_buffer[bottom_left_coord + color] * (rect_x > 0 && rect_y > 0);
float bottom_right_value =
	source_buffer[bottom_right_coord + color] * (rect_x > 0 || rect_y > 0);
```
