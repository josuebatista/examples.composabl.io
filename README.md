# examples.composabl.io

---
sidebar_position: 1
---

# Creating a Simulator

A simulator (or environment) can be easily integrated with the Composabl SDK. To do so, we have to follow the following outline:

1. Create a new Simulation Package
2. Implement the Composabl Communication Interface
3. Build a docker container

## The Composabl Specification

> 💡 The below gives a highlight of how the Composabl SDK communicates with the SDK. To create your own simulator, you can follow the steps in **"Creating a Simulator"**

To interact with the simulators, Composabl depends on the gRPC protocol.

> We use a gRPC communication layer due its low overhead, but practical implementation

Currently we support the following calls:

```protobuf
service Composabl {
    rpc Make(MakeRequest) returns (MakeResponse) {}
    rpc State(StateRequest) returns (StateResponse) {}
    rpc Step(stream StepRequest) returns (stream StepResponse) {}
    rpc Reset(ResetRequest) returns (ResetResponse) {}
    rpc Close(CloseRequest) returns (CloseResponse) {}
    rpc ActionSpaceSample(ActionSpaceSampleRequest) returns (ActionSpaceSampleResponse) {}
    rpc ActionSpaceInfo(ActionSpaceInfoRequest) returns (ActionSpaceInfoResponse) {}
    rpc ObservationSpaceInfo(ObservationSpaceInfoRequest) returns (ObservationSpaceInfoResponse) {}
    rpc SetScenario(SetScenarioRequest) returns (SetScenarioResponse) {}
    rpc GetScenario(GetScenarioRequest) returns (GetScenarioResponse) {}
    rpc SetRewardFunc(SetRewardFuncRequest) returns (SetRewardFuncResponse) {}
    rpc GetRewardFunc(GetRewardFuncRequest) returns (Stringified) {}
}
```

Due to the complicated nature of how gRPC works, we package all of this into our core SDK and make it available for you to integrate through an easy to understand Python file.

## Creating a Simulator

The simulator can be written in python using the gym framework, or can be integrate to any other external simulator following the same structure.

You can find an example in the folder `simulation\server_impl.py`

```python
class SimEnv(gym.Env):
    def __init__(self):
        #  Define Observation Space
        self.obs_space_constraints = {
            'state1': {'low': 1, 'high': 10},
            'state2': {'low': -5, 'high': 0},
            'state3': {'low': 20, 'high': 100}
        }

        low_list = [x['low'] for x in self.obs_space_constraints.values()]
        high_list = [x['high'] for x in self.obs_space_constraints.values()]

        self.observation_space = gym.spaces.Box(low=np.array(low_list), high=np.array(high_list))

        #  Define Action Space
        self.action_constraints = {
            'action1': {'low': -0.1, 'high': 0.1},
            'action2': {'low': -0.5, 'high': 0.5},
        }

        low_act_list = [x['low'] for x in self.action_constraints.values()]
        high_act_list = [x['high'] for x in self.action_constraints.values()]

        self.action_space = gym.spaces.Box(low=np.array(low_act_list), high=np.array(high_act_list))

        self.scenario: Scenario = None

    def reset(self):
        self.cnt = 0
        """
        ****** Define scenario in the simulation ******
        """
        if isinstance(self.scenario, Scenario):
            sample = self.scenario.sample()

            self.obs = {
                'state1': sample["state1"],
                'state2': sample["state2"],
                'state3': sample["state3"]
                }
        else:
            self.obs = {
                'state1': random.uniform(self.obs_space_constraints['state1']['low'], self.obs_space_constraints['state1']['high']),
                'state2': random.uniform(self.obs_space_constraints['state2']['low'], self.obs_space_constraints['state2']['high']),
                'state3': random.uniform(self.obs_space_constraints['state3']['low'], self.obs_space_constraints['state3']['high'])
                }


        self.obs = np.array(list(self.obs.values()))
        info = {}
        return self.obs, info

    def set_scenario(self, scenario):
        self.scenario = scenario

    def step(self, action):
        done = False
        #  Increase time counting
        self.cnt += 1
        
        #  Run Simulation
        #  Update obs with new state values (dummy function)
        self.obs = {}
        for key in list(self.obs.keys()):
            self.obs[key] = np.clip(self.obs[key] + action[0] + action[1],
                                    self.obs_space_constraints[key]['low'],
                                    self.obs_space_constraints[key]['high'] )

        #  Reward variable definition
        reward = 0

        #  Define rules to end the simulation
        if self.cnt == 80:
            done = True

        self.obs = np.array(list(self.obs.values()))
        info = {}
        return self.obs, reward, done, False, info

    def render(self, mode='auto'):
        pass


```

### The Simulator Interface

The SDK can be found on the [PyPI registry](https://pypi.org/project/composabl/) and installed with `pip install composabl`. When downloaded, the simulator interface is available through the `composabl.server.server_composabl` path providing a `ServerComposabl` class to extend from which offers all the required methods.

```python
#!/usr/bin/env python3
from typing import Any, Dict, SupportsFloat, Tuple

import composabl.utils.logger as logger_util
from composabl.server.server_composabl import EnvSpec, Space, ServerComposabl

logger = logger_util.get_logger(__name__)

class ServerImpl(ServerComposabl):
    def __init__(self):
        #  Calls your SimEnv class
        self.env = SimEnv()

    def Make(self, env_id: str) -> EnvSpec:
        spec = {'id': 'simulation_example', 'max_episode_steps': 80}
        return spec

    def ObservationSpaceInfo(self) -> gym.Space:
        return self.env.observation_space

    def ActionSpaceInfo(self) -> gym.Space:
        return self.env.action_space

    def ActionSpaceSample(self) -> Any:
        return self.env.action_space.sample()

    def Reset(self) -> Tuple[Any, Dict[str, Any]]:
        obs, info = self.env.reset()
        return obs, info

    def Step(self, action) -> Tuple[Any, SupportsFloat, bool, bool, Dict[str, Any]]:
        return self.env.step(action)

    def Close(self):
        self.env.close()

    def SetScenario(self, scenario):
        scenario = Scenario.from_proto(scenario)
        self.env.scenario = scenario

    def GetScenario(self):
        if self.env.scenario is None:
            return Scenario({"dummy": 0})

        return self.env.scenario
```

### Creating a new Package

This is step is to create a new package that encapsulate all dependencies to run your simulator. To provide a easier way of creating a package, we recommend to:
1 . Create a `project.toml`file
TODO

## Running the Simulator

Once a simulator package has been created, you can run it either locally, or package it as a docker container and run it through docker.

### Locally

TODO

### As Docker Container

TODO