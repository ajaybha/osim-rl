# NIPS2017: Learning to run

This repository contains software required for participation in the NIPS 2017 Challenge: Learning to Run. See more details about the challenge [here](https://www.crowdai.org/challenges/nips-2017-learning-to-run).

In this competition, you are tasked with developing a controller to enable a physiologically-based human model to navigate a complex obstacle course as quickly as possible. You are provided with a human musculoskeletal model and a physics-based simulation environment where you can synthesize physically and physiologically accurate motion. Potential obstacles include external obstacles like steps, or a slippery floor, along with internal obstacles like muscle weakness or motor noise. You are scored based on the distance you travel through the obstacle course in a set amount of time.

![HUMAN environment](https://github.com/kidzik/osim-rl/blob/master/demo/training.gif)

For modelling physics we use [OpenSim](https://github.com/opensim-org/opensim-core) - a biomechanical physics environment for musculoskeletal simulations. 

## Getting started

**Anaconda2** is required to run our simulation environment - you can get it from here https://www.continuum.io/downloads choosing version 2.7. In the following part we assume that Anaconda is successfully installed.

We support Windows, Linux or Mac OSX in 64-bit version. To install our simulator, you first need to create a conda environment with OpenSim package. On Windows open a command prompt and type:
    
    conda create -n opensim-rl -c kidzik opensim git
    activate opensim-rl

on Linux/OSX run:

    conda create -n opensim-rl -c kidzik opensim git
    source activate opensim-rl

These command will create a virtual environment on your computer with simulation libraries installed. Next, you need to install our python reinforcement learning environment. Type (independently on the system)

    conda install -c conda-forge lapack git
    pip install git+https://github.com/stanfordnmbl/osim-rl.git

If the command `python -c "import opensim"` runs smoothly you are done! Otherwise, please refer to our FAQ section.

## Basic usage

To run 200 steps of environment enter `python` interpreter and run:
```python
from osim.env import RunEnv

env = RunEnv(visualize=True)
observation = env.reset()
for i in range(200):
    observation, reward, done, info = env.step(env.action_space.sample())
```
![Random walk](https://github.com/stanfordnmbl/osim-rl/blob/master/demo/random.gif)

In this example muscles are activated randomly (red color indicates an active muscle and blue an inactive muscle). Clearly with this technique we won't go to far.

Your goal is to construct a controler, i.e. a function from the state space (current positions, velocities and accelerations of joints) to action space (muscles activations), to go as far as possible in limited time. Suppose you trained a neural network mapping observations (the current state of the model) to actions (muscles activations), i.e. you have a function `action = my_controler(observation)`, then 
```python
# ...
total_reward = 0.0
for i in range(200):
    # make a step given by the controler and record the state and the reward
    observation, reward, done, info = env.step(my_controler(observation)) 
    total_reward += reward
    if done:
        break

# Your reward is
print("Total reward %f" % total_reward)
```
There are many ways to construct the function `my_controler(observation)`. We will show how to do it with a DDPG algorithm, using keras-rl.

## Evaluation

Your task is to build a function `f` which takes current state `observation` (41 dimensional vector) and returns mouscle activations `action` (18 dimensional vector) in a way that maximizes the reward. See Section "Detalis of the environment" for more precise description.

The trial ends either if the pelvis of the model goes below `0.65` meter or if you reach `500` iterations (corresponding to `5` seconds in the virtual environment). Your total reward is the position of the pelvis on the `x` axis after the last iteration minus the penalty for using ligament forces. Ligaments are tissues which prevent your knee from bending too much - overusing these tissues leads to injuries, so we want to avoid it. The panalty in the total reward is equal to the sum of forces generated by ligaments over the trial, divided by `1000`.

After each iteration you get a reward equal to the change of the `x` axis of pelvis during this iteration minus the ligament force magnitude used in that iteration.

You can test your model on your local machine. For submission, you will need to interact with the remote environment: crowdAI sends you the current `observation` and you need to send back the action you take in the given state. You will be evaluated at three different levels of difficulty. For details please refer to "Detalis of the environment".

### Submission

```python
import opensim as osim
from osim.http.client import Client
from osim.env import *

# Settings
remote_base = "http://grader.crowdai.org"
crowdai_token = "[YOUR_CROWD_AI_TOKEN_HERE]"

client = Client(remote_base)

# Create environment
observation = client.env_create(crowdai_token)

# IMPLEMENTATION OF YOUR CONTROLLER
# my_controller = ...

# Run a single step
for i in range(500):
    [observation, reward, done, info] = client.env_step(my_controller(observation), True)
    print(observation)
    if done:
        break

client.submit()
```

### Rules

Additional rules:
* You are not allowed to use external datasets (ex. kinematics of people walking),
* Organizers reserve the right to modify challenge rules as required.

## Training in keras-rl

Below we present how to train a basic controller using [keras-rl](https://github.com/matthiasplappert/keras-rl). First you need to install extra packages

    conda install keras
    pip install git+https://github.com/matthiasplappert/keras-rl.git
    git clone http://github.com/stanfordnmbl/osim-rl.git
    
`keras-rl` is an excelent package compatible with OpenAi, which allows you to quickly build your first models!

Go to `scripts` subdirectory from this repository
    
    cd osim-rl/scripts

There are two scripts:
* `example.py` for training (and testing) an agent using DDPG algorithm. 
* `submit.py` for submitting the result to crowdAI.org

### Training

    python example.py --visualize --train --model sample
    
### Test

and for the gait example (walk as far as possible):

    python example.py --visualize --test --model sample
    
### Submission

After having trained your model you can submit it by using `/scripts/submit.py`:

    python submit.py

This script will interact with an environment on the crowdAI.org server.
    
## Datails of the environment

In order to create an environment use
```python
    env = RunEnv(visualize = True, max_obstacles = 3)
```
Parameters:

* `visualize` - turn the visualizer on and off
* `max_obstacles` - maximal number of obstacles in the scene

### Methods of `RunEnv`

#### `reset(difficulty, seed = None)`

* `difficulty` - `0` - no obstacles, `1` - 3 randomly positioned obstacles (balls fixed in the ground), `2` - as in `1` but also strength of psoas muscles varies. It is set to z * 100%, where z is a normal variable with the mean 1 and the standard deviation 0.1
* `seed` - starting seed for the random number generator. If the seed is `None`, generation from the previous seed is continued. 

Restart the enivironment with a given `difficulty` level and a `seed`.

#### `step(action)`

* `action` - a list of length `18` of continous values in `[0,1]` corresponding to excitation of muscles. 

The function returns:

* `observation` - a list of length `41` of real values corresponding to the current state of the model. Variables are explained in the section "Physics of the model".

* `reward` - reward gained in the last iteration. It is computed as a change in position of the pelvis along x axis minus penalty for the use of ligaments. See the "Physics of the model" section for details. 

* `done` - indicates if the move was the last step of the environment. This happens if either `500` iterations were reached or pelvis is below `0.65` meter.

* `info` - for compatibility with OpenAI, currently not used.

### Physics of the model

## Questions

**I'm getting 'version GLIBCXX_3.4.21 not defined in file libstdc++.so.6 with link time reference' error**

If you are getting this error

    ImportError: /home/deepart/anaconda2/envs/opensim-rl/lib/python2.7/site-packages/opensim/../../../libSimTKcommon.so.3.6: symbol _ZTVNSt7__cxx1119basic_istringstreamIcSt11char_traitsIcESaIcEEE, version GLIBCXX_3.4.21 not defined in file libstdc++.so.6 with link time reference
    
Try `conda install libgcc`

**Can I use different languages than python?**

Yes, you just need to set up your own python grader and interact with it
https://github.com/kidzik/osim-rl-grader. Find more details here [OpenAI http client](https://github.com/openai/gym-http-api)

## Credits

This challenge wouldn't be possible without:
* [OpenSim](https://github.com/opensim-org/opensim-core)
* Stanford NMBL group & Stanford Mobilize Center
* [OpenAI gym](https://gym.openai.com/)
* [OpenAI http client](https://github.com/openai/gym-http-api)
* [keras-rl](https://github.com/matthiasplappert/keras-rl)
* and many other teams, individuals and projects

For details please contact [Łukasz Kidziński](http://kidzinski.com/)
