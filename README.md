# Reinforcement Learning (RL) and State Representation Learning (SRL) with robotic arms (Baxter and Kuka)

## Requirements:

- Python 3 (python 2 not supported because of OpenAI baselines)
- [OpenAI Baselines](https://github.com/openai/baselines) (latest version, install from source (at least commit 3cc7df0))
- [OpenAI Gym](https://github.com/openai/gym/) (version >= 0.10.3)
- Install the dependencies using `environment.yml` file (for conda users)

Note: The save method of ACER of baselines is currently buggy, you need to manually add an import (see [pull request #312](https://github.com/openai/baselines/pull/312))

[PyBullet Documentation](https://docs.google.com/document/d/10sXEhzFRSnvFcl3XxNGhnD4N2SedqwdAvK3dsihxVUA)

```
git clone git@github.com:araffin/robotics-rl-srl.git --recursive
```

## Kuka Arm \w PyBullet

Before you start a RL experiment, you have to make sure that a visdom server is running, unless you deactivate visualization.

Launch visdom server:
```
python -m visdom.server
```


To test the environment with random actions:
```
python -m environments.test_env
```
Can be as well used to render views (or dataset) with two cameras if `multi_view=True`.

To record data from the environment for SRL training, using random actions:
```bash
python -m environments.test_env --record-data --num-cpu 4 --save-name folder_name
```

## Reinforcement Learning

Note: All CNN policies normalize input, dividing it by 255.
By default, observations are not stacked.
For SRL, states are normalized using a running mean/std average.

About frame-stacking, action repeat (frameskipping) please read this blog post: [Frame Skipping and Pre-Processing for DQN on Atari](https://danieltakeshi.github.io/2016/11/25/frame-skipping-and-preprocessing-for-deep-q-networks-on-atari-2600-games/)

### OpenAI Baselines

Several algorithms from [Open AI baselines](https://github.com/openai/baselines) have been integrated along with some evolution strategies:

- DQN and variants (Double, Dueling, prioritized experience replay)
- ACER: Sample Efficient Actor-Critic with Experience Replay
- A2C: A synchronous, deterministic variant of Asynchronous Advantage Actor Critic (A3C) which gives equal performance.
- PPO2: Proximal Policy Optimization (GPU Implementation)
- DDPG: Deep Deterministic Policy Gradients
- ARS: Augmented Random Search
- CMA-ES: Covariance Matrix Adaptation Evolution Strategy

To train an agent:
```
python -m rl_baselines.train --algo ppo2 --log-dir logs/
```

To load a trained agent and see the result:
```
python -m replay.enjoy_baselines --log-dir path/to/trained/agent/ --render
```

Contiuous actions have been implemented for DDPG, PPO2, ARS, CMA-ES and random agent.
To use continuous actions in the position space:
```
python -m rl_baselines.train --algo ppo2 --log-dir logs/ -c
```

To use continuous actions in the joint space:
```
python -m rl_baselines.train --algo ppo2 --log-dir logs/ -c -joints
```

To run multiple enviroments with multiple SRL models for a given algorithm (you can use the same arguments as for training):
```
python  -m rl_baselines.pipeline --algo ppo2 --log-dir logs/ --env env1 env2 [...] --srl-model model1 model2 [...]
```

### Plot Learning Curve

To plot a learning curve from logs in visdom, you have to pass path to the experiment log folder:
```
python -m replay.plots --log-dir /logs/raw_pixels/ppo2/18-03-14_11h04_16/
```

To aggregate data from different experiments (different seeds) and plot them (mean + standard error).
You have to pass path to rl algorithm log folder (parent of the experiments log folders):
```
python -m replay.aggregate_plots --log-dir /logs/raw_pixels/ppo2/ --shape-reward --timesteps --min-x 1000 -o logs/path/to/output_file
```
Here it plots experiments with reward shaping and that have a minimum of 1000 data points (using timesteps on the x-axis), the plot data will be saved in the file `output_file.npz`.

To create a comparison plots from saved plots (.npz files), you need to pass a path to folder containing .npz files:
```
python -m replay.compare_plots -i logs/path/to/folder/ --shape-reward --timesteps
```

## Environments

When starting a baseline, you can choose from different environments:
```bash
python -m rl_baselines.train --algo ppo2 --log-dir logs/ --env env_name
```

the available environments are:
- KukaButtonGymEnv-v0 (default): a single button in front of the arm
- Kuka2ButtonGymEnv-v0: 2 buttons next to each others, they must be pressed in order
- KukaRandButtonGymEnv-v0: a single button in front of the arm, with some randomly positioned objects
- KukaMovingButtonGymEnv-v0: a single button in front of the arm, slowly moving left to right

## State Representation Learning Models

Please look the [SRL Repo](https://github.com/araffin/srl-robotic-priors-pytorch) to learn how to train a state representation model.
Then you must edit `config/srl_models.yaml` and set the right path to use the learned state representations.

To train the Reinforcement learning baselines on a specific SRL model:
```
python -m rl_baselines.train --algo ppo2 --log-dir logs/ --srl-model model_name
```

the available state representation model are:
- autoencoder: an autoencoder from the raw pixels
- ground_truth: the arm's x,y,z position
- srl_priors: SRL priors model
- supervised: a supervised model from the raw pixels to the arm's x,y,z position
- pca: pca applied to the raw pixels
- vae: a variational autoencoder from the raw pixels
- joints: the arm's joints angles
- joints_position: the arm's x,y,z position and joints angles

In the case of a custom_cnn (SRL priors model) trained with views of two cameras,
set the global variable N_CHANNELS to 6 in srl_priors/preprocessing/preprocess.py
to perform the inference.

## Baxter Robot with Gazebo and ROS
Gym Wrapper for baxter environment, more details in the dedicated README (environments/gym_baxter/README.md).

1. Start ros nodes (Python 2):
```
roslaunch arm_scenario_simulator baxter_world.launch
rosrun arm_scenario_simulator spawn_objects_example

python -m gazebo.gazebo_server
```

Then, you can either try to teleoperate the robot (python 3):
```
python -m gazebo.teleop_client
```
or test the environment with random actions (using the gym wrapper):

```
python -m environments.gym_baxter.test_baxter_env
```

If the port is already used, you can see the program pid using the following command:
```
sudo netstat -lpn | grep :7777
```
and then kill it (with `kill -9 program_pid`)

## Troubleshooting
If a submodule is not downloaded:
```
git submodule update --init
```
If you have troubles installing mpi4py, make sure you the following installed:
```
sudo apt-get install libopenmpi-dev openmpi-bin openmpi-doc
```

## Known issues

The inverse kinematics function has trouble finding a solution when the arm is fully straight and the arm must bend to reach the requested point.
