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

![Alt Text](images/towards-gaze-based-arch.png?raw=true)

They then explore whether logreg models used in a couple other papers could predict intent. Basically, they claim yes, while blatantly overfitting their models. Runtime of the logreg inference isn't reported either, although for a simple model, it's probably negligible.


# Trajectory-Based Viewport Prediction for 360-Degree Virtual Reality Videos 
This paper isn't really a regression paper, just a basic clustering one, and works based on past users. Viewing trajectory behaviour from past users on the same video will be extracted, and that will be used to predict what the new user will do; so they group past users based on similar viewing trajectories, and create a model of the viewport movement, using roll/pitch/yaw.

A modified version of spectral clustering is used to cluster together trajectories. After clusters have been identified, a trend trajectory over all past users in a cluster is computed. They aim to predict the viewport very far into the future, like 5s+, based on independent predictions of roll, pitch, and yaw. In the interval [0; T_n], a trajectory is built for the new user. An affinity score is computed between the user trajectory and the trend trajectory for each cluster, for the same time window, and use the highest trend trajectory, R, and then use this to predict.


# Neural Network Approaches

## Analyzing Viewport Prediction Under Different VR Interactions
To collect data, they selected10 popular 360 degree youtube videos, downloaded at 4K resolution. User's viewport movement traces were collected from viewers as they watched the videos with a Gear VR headset, which doesn't have eye tracking. They find that viewport movement speed is much higher on PC than using an HMD (without eye tracking, at least), but viewports move less often than on HMD's. CDF's for these interactions are shown below.

An RNN, LSTM (Long Short-Term Memory) does not use Markov assumptions. Long-term trends can, in theory, be exposed using this, so it is used. It starts with a subtraction layer performing 1st-point normalization. Then, a 64-neuron layer and an Add layer denormalizes values before output, but they do not fine-tune. Mean Absolute Error is used as the evaluator, and ADAM for optimizing.

![Alt Text](images/cdf.png?raw=true)
