# Project plan/proposal for Not-Dumb Robot ~ [Robot JetRover + (decision model) Laya + LLM (Qwen) + CV (YOLO)]

**TL;DR**: Waypoints represent map discrete environments (e.g., Kitchen, Bathroom, etc.). A computer vision model identifies objects in the current environment and updates the robot's state. A [decision model](https://app.notion.com/p/Project-plan-proposal-for-Not-Dumb-Robot-Robot-JetRover-decision-model-Laya-LLM-Qwen-CV-4099be56aa8d803ab1d5c35f4c56c4f3?pvs=21) uses the current state to select the next robot action from a predefined set of actions. When a natural-language response is required, the state is passed to an LLM to generate a response for the user.

# Can do all in gazebo first?

### Why do it this way?

JetRover can already do simple actions like go to mapped location and report what the camera is seeing with built in CV and text to speech. Adding speech to text for input is trivial.  

To make more robust, we use decision model to handle ambiguous commands, and computer vision get new state each time it enter fixed background environments so objects change. 

It is not smart since no planning is used, but not hard-coded so not-dumb. 

## Key decision - navigation type

- **Waypoint** - go to this point in map. (Current decision)
- Visual servoing - see object → steer to object
- Pure vision -  self driving: camera → neural network →steering

#### Why not drone?

- Require ROS drone
- 3D space navigation
    - Visual SLAM - waypoint?
        - Object must be in place for interaction

## Timeline

```jsx
Task                            W1   W2   W3

Robot Setup                     ███
ROS Communication               ███
Laya Integration                ███
Waypoint Navigation             ███

YOLO Integration                     ██████
State Update System                  ██████
Qwen Integration                     ██████
Inspection Pipeline                  ██████

Whisper Integration                       ███
System Testing                            ███
Demo Preparation                          ███

Stretch: Pickup System               ░░░░░░░░░░
```

## Known problems/Needs testing/Might not be completed: Camera/Grabber Arm

- <u>**How well can camera do CV? ~ detection accuracy/distance/while moving**</u>
- <u>**Scan** environment for objects?</u>
    - assume all objects in same area?
    - Assumes discrete environments ~ e.g. if demo ground, assume flat sheet on floor divided into 2 with and assigned different labels, if no physical border CV will not see objects in “both” env
- **Naive grabbing - assumes objects on ground, can be grabbed**
    - **Need to figure out distance and angle**
        - Depth camera for distance/object position without grasp planning.

### Limitations:

- No planning (no multistep)
- We use Waypoint navigation (fixed background environments) and NOT CV [navigation](https://app.notion.com/p/Project-plan-proposal-for-Not-Dumb-Robot-Robot-JetRover-decision-model-Laya-LLM-Qwen-CV-4099be56aa8d803ab1d5c35f4c56c4f3?pvs=21)

### Additional requirements?

- Text-to-speech reply for LLM?

## Architecture

State is overwritten after every inspection. No memory. No history. No long-term storage.

```jsx
User (Whisper)
 ↓
Laya
 ├─→ chat
 │      ↓
 │     Qwen
 │
 └─→ robot_action
         ↓
       Robot
         ↓
         CV
         ↓
   Updated State
    ├─→ Laya
    └─→ Qwen
```

# Components (in order of architecture)

Component selection justification: choose fastest

## Whisper RT(speech to text) ~ PC

[https://github.com/tomer9080/WhisperRT-Streaming](https://github.com/tomer9080/WhisperRT-Streaming)

## Decision model for robot control  - Laya ~ PC

Decision model - stripped down llm that outputs choice of candidate labels rather than text.

[https://github.com/NandhaKishorM/laya](https://github.com/NandhaKishorM/laya)

**Input**:  

possible state (location and objects) and actions

**Actions**:

Non-terminal: Needs more info (assumes follow up action i.e. call Laya)

```jsx
inspect
navigate
pickup
drop
```

Terminal: task ended or requires user interaction

```jsx
report
ask_clarification
chat
stop
```

**Schema: (**[https://huggingface.co/spaces/convaiinnovations/laya-demo](https://huggingface.co/spaces/convaiinnovations/laya-demo))

![image.png](Project%20plan%20proposal%20for%20Not-Dumb%20Robot%20~%20%5BRobot%20/image.png)

## LLM - Qwen 1.7b  ~PC

[https://ollama.com/library/qwen3](https://ollama.com/library/qwen3) 

For `chat` or `report` , LLM replies to user

## Object detection (Yolo26n) ~ (onboard or PC?)

#### PC:

[https://docs.ultralytics.com/models/yolo26](https://docs.ultralytics.com/models/yolo26)

RTSP Realtime Yolo? (feels excessive) 

#### Onboard:

 [https://docs.hiwonder.com/projects/JetRover/en/jetson-orin-nano/docs/7.Robot_Arm_Control_Course.html#object-classification](https://docs.hiwonder.com/projects/JetRover/en/jetson-orin-nano/docs/7.Robot_Arm_Control_Course.html#object-classification)

#### Text-to-Speech

[https://github.com/debpalash/voicestudio](https://github.com/debpalash/voicestudio)

# Hardware requirements

- JetRover
- PC with GPU

# Prerequisite input to Laya

- Environment mapped
- Environment labels must be provided
- [Actions](https://app.notion.com/p/Project-plan-proposal-for-Not-Dumb-Robot-Robot-JetRover-decision-model-Laya-LLM-Qwen-CV-4099be56aa8d803ab1d5c35f4c56c4f3?pvs=21)

# Example Scenarios

**Given Environments:** kitchen, table

### 1: "What's cooking?”

Ambiguity: Environment and Action not explicit by user

```jsx
User
 ↓
"What's cooking?"

 ↓
Laya
 ↓
inspect(kitchen)

 ↓
Robot
 ↓
moves to kitchen

 ↓
CV
 ↓
state = {
  location: kitchen,
  objects: [pot, stove, bowl]
}

 ↓
Laya
 ↓
report()

 ↓
Qwen
 ↓
"I can see a pot on a stove and a bowl."
```

### 2: "Grab what's on the table”

Multiple objects detected

```jsx
User
 ↓
"Grab what's on the table"

 ↓
Laya
 ↓
inspect(table)

 ↓
CV
 ↓
state = {
  location: table,
  objects: [apple, cup]
}

 ↓
Laya
 ↓
ask_clarification()

 ↓
Qwen
 ↓
"I found an apple and a cup.Which one would you like?"
```

### 3: “Grab the apple”

Environment not given

```jsx
User
 ↓
"Grab the apple"

 ↓
Laya
 ↓
inspect(table)

 ↓
CV
 ↓
state = {
  objects: [apple]
}

 ↓
Laya
 ↓
pickup(apple)

 ↓
Robot
 ↓
grasp complete

 ↓
Laya
 ↓
report()

 ↓
Qwen
 ↓
"I picked up the apple."
```

## Stretch Goal: Tracking and Gripping

[https://docs.hiwonder.com/projects/JetRover/en/jetson-orin-nano/docs/7.Robot_Arm_Control_Course.html#tracking-and-gripping](https://docs.hiwonder.com/projects/JetRover/en/jetson-orin-nano/docs/7.Robot_Arm_Control_Course.html#tracking-and-gripping)