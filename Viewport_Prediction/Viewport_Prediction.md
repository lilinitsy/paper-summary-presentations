# Viewport Prediction Overview
This will go over many different viewport prediction methods, organized by method of approach.

# Basic Regression Models
Many systems to perform VP rely on simple regression-based approaches rather than more complicated neural network ones.


# Neural Network Approaches

## Analyzing Viewport Prediction Under Different VR Interactions (Tan Xu, Bo han, Feng Qian)
To collect data, they selected10 popular 360 degree youtube videos, downloaded at 4K resolution. User's viewport movement traces were collected from viewers as they watched the videos with a Gear VR headset, which doesn't have eye tracking. They find that viewport movement speed is much higher on PC than using an HMD (without eye tracking, at least), but viewports move less often than on HMD's. CDF's for these interactions are shown below.

An RNN, LSTM (Long Short-Term Memory) does not use Markov assumptions. Long-term trends can, in theory, be exposed using this, so it is used. It starts with a subtraction layer performing 1st-point normalization. Then, a 64-neuron layer and an Add layer denormalizes values before output, but they do not fine-tune. Mean Absolute Error is used as the evaluator, and ADAM for optimizing.

![Alt Text](images/cdf.png?raw=true)
