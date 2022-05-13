# Viewport Prediction Overview
This will go over many different viewport prediction methods, organized by method of approach.

# Basic Regression Models
Many systems to perform VP rely on simple regression-based approaches rather than more complicated neural network ones. On mobile devices, computational resources are limited, so neural networks could incur a high overhead.


## Towards Gaze-based Prediction of the Intent to Interact in Virtual Reality 
They train logistic regression models to predict the moment of interaction based on point-and-click events. 

H1: "Natural gaze dynamics from eye-tracking can be used to predict the onset of interaction in VR." This is probably based on the eye stopping at a particular region.
H2: "There is a consistent set of gaze features across individuals that reflect eye movements related to interaction."

Gaze data was filtered with a median filter with a width of 7 samples, which smoothed data due to sensor noise. I-VT saccade detection was used on this data to identify samples that moved more than 70 degrees per second within a time window. When movement didn't exceed 1 degree in a time window (100-2000ms), that was marked as eye fixation (no citation for this, but seems fine).

61 features were explored and features were extracted from each fixation and saccade.

Feature selection used some weird iterative approach... a random set was used to train a logistic model with 10-fold crossval and 3 repeats (whatever "repeats" means....). Mostly retained about 20 features. But why not just do PCA? 

![Alt Text](images/towards-gaze-based-arch.jpg?raw=true)

They then explore whether logreg models used in a couple other papers could predict intent. Basically, they claim yes, while blatantly overfitting their models. Runtime of the logreg inference isn't reported either, although for a simple model, it's probably negligible.

# Clustering Approaches
## Trajectory-Based Viewport Prediction for 360-Degree Virtual Reality Videos 
This works based on past users. Viewing trajectory behaviour from past users on the same video will be extracted, and that will be used to predict what the new user will do; so they group past users based on similar viewing trajectories, and create a model of the viewport movement, using roll/pitch/yaw.

A modified version of spectral clustering is used to cluster together trajectories. After clusters have been identified, a trend trajectory over all past users in a cluster is computed. They aim to predict the viewport very far into the future, like 5s+, based on independent predictions of roll, pitch, and yaw. In the interval [0; T_n], a trajectory is built for the new user. An affinity score is computed between the user trajectory and the trend trajectory for each cluster, for the same time window, and use the highest trend trajectory, R, and then use this to predict.


# Neural Network Approaches

## Analyzing Viewport Prediction Under Different VR Interactions
To collect data, they selected10 popular 360 degree youtube videos, downloaded at 4K resolution. User's viewport movement traces were collected from viewers as they watched the videos with a Gear VR headset, which doesn't have eye tracking. They find that viewport movement speed is much higher on PC than using an HMD (without eye tracking, at least), but viewports move less often than on HMD's. CDF's for these interactions are shown below.

An RNN, LSTM (Long Short-Term Memory) does not use Markov assumptions. Long-term trends can, in theory, be exposed using this, so it is used. It starts with a subtraction layer performing 1st-point normalization. Then, a 64-neuron layer and an Add layer denormalizes values before output, but they do not fine-tune. Mean Absolute Error is used as the evaluator, and ADAM for optimizing.

![Alt Text](images/cdf.jpg?raw=true)

## Predictive View Generation to Enable Mobile 360-degree and VR Experiences
Another LSTM approach, this paper uses deep learning to predict where a user is looking based on past behaviour. Video tiles are used, and head motion files encode timestamps, pitch/raw/roll, and "user information." Timestamps appear sparsely, every 200ms.

![Alt Text](images/predictive-view-generation.jpg?raw=true)

Each grid size is 30 by 30 degrees, which divides a 360 degree view into 72 tiles. They aim to predict 2s out. Cross-entropy in the LSTM layers is minimized, and mini-batches of size 30 are trained. 128 LSTM units are used. 

The model will want to minimize bandwidth consumption (pixels/bitrate) while maximizing the probability that the user view will be within the predicted FOV. To generate the FOV, they select m tiles with highest probabilities predicted by the LSTM model, and transmit predicted FOV with a high quality while leaving the other tiles blank. It takes longer to train than KNN models, but they claim higher accuracy while saving heavily on bandwidth.

![Alt Text](images/predictive-view2.jpg?raw=true)


## LiveROI: Region of Interest Analysis for Viewport Prediction in Live Mobile Virtual Reality Streaming
This one takes a different approach than others; it targets "semantics of actions" to predict v iewports.

First, they figure out user preference for video content. They used two videos with 48 users, with one consistent 360 degree video scene shot by one fixed camera "without dramatic scene switches". User traces were analyzed. Users look around the videos a lot at the beginning of videos. However, most of the time (97%), users spend watching one region that takes 20% of the size of the frame, but was the meaningful part of the scene.

From this preliminary study, they  use a tile-based selective streaming concept to use CV to assess the content of each tile. A 3D CNN is used to recognize meaningful actions in the video.

2D CNN's, which are often used on 2D images for feature extraction, presents challenges when considering motion information. To account for spatiotemporal features, a 3D CNN is used. Multiple sequential frames are stacked with 3D kernels. ECO lite is used for action recognition. They use a pre-trained ECO lite model, which is not trained for their videos, but could serve for a general purpose usage. They also do word analysis stuff, which may be less interesting for the purpose of this.

![Alt Text](images/liveroi-arch.jpg?raw=true)

Since this is one of two CNN papers I could find on this, there might be some weaknesses that can be improved upon. Maybe some different CV method - YOLO?

## Lightweight Neural Network-Based Viewport Prediction for Live VR Streaming in Wireless Video Sensor Network
This uses C-GhostNet, another CNN (https://blog.paperspace.com/ghostnet-cvpr-2020/). C-GhostNet will use random wheights to predict a user's viewport and transmit the tiles for the first video, since the first video isn't labeled. *This might suggest a weakness in the CNN architecture in general, unless some other method can be used to auto label scenes*.  The predictive viewport is compared with the actual viewport, and a loss value is calculated. After the first video segment is done, the corresponding viewport trajectory is generated and is input into GRU-ECA (https://towardsdatascience.com/understanding-gru-networks-2ef37df6c9be) for training and prediction. This is combined with the result of C-GhostNet for all subsequent videos until all video sequences are predicted.

![Alt Text](images/ghostnet-training.jpg?raw=true)

GhostNet uses a light Ghost module to generate feature maps from a few parameters, which should reduce the cost of each convolutional layer. The output feature map can be generated with

Y' = X * f', where f' is the convolutional kernel.

A Ghost Bottleneck is a small CNN in ghostnet. It integrates many convlutional layers; this uses an expansion layer to increase the number of channels, and a second ghost module reduces the number of channels to match the shortcut path.

The first layer in C-GhostNet is a convolutional layer with 16 convolution kernels. Some ghost bottlenecks with increasing channels are next, and the global pooling and convolutional layer convert the feature map into a 1280 dim feature vector. They remove the global pooling layer and reduce the layers per channel; this reduces the parameter space by about a third.

![Alt Text](images/ghostnet-layers.jpg?raw=true)

The GRU-ECA Modules allegedly improve prediction accuracy over LSTM, at the cost of a small overhead.

Batch training and inference were set to 200, and was trained for 10 epochs. They achieve a high (close to 95%) prediction accuracy, but the time for inference becomes close to half a second, if not higher.