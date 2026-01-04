# From Telemetry to Video: Bridging comma.ai Datasets with the Power of Physics and Agents

## Introduction

Imagine if an AI agent could find the moments where a self-driving car struggled, then track down similar video footage without you manually sifting through millions of rows of data?

That's what I built. Two open datasets from comma.ai, a physics insight, and a few lines of Python. An agentic system that hunts for edge cases in steering control data and matches them to video segments.

The rub? The datasets have no direct link between them. None. So I made one, using the power of physics.

**A note on process:** This project was developed iteratively in SolveIt, with Claude as a coding and writing partner. Small steps, test often, refine. The same way the agent works, actually.

**Code**: [github.com/c5huracan/agentic-av-sim](https://github.com/c5huracan/agentic-av-sim)

---

## The Problem

comma.ai has released some fantastic open datasets. I have been exploring two in particular:

**commaSteeringControl**: Millions of rows of telemetry from real drives. Steering angles, lateral acceleration, vehicle speed, whether openpilot was in control. The raw guts of how a self-driving system behaves.

**commavq**: Video segments encoded as tokens, with pose data (speed, orientation, yaw rate). Visual context for what the car was seeing.

I did start with **comma2k19**, which has synchronized video and telemetry. But it's older now and limited to 2019 segments on one highway corridor. To find more diverse edge cases, I needed the scale of commaSteeringControl. That meant losing the video link.

You'd think these would be linked. Same company, complementary data. But no. Different drives, different timestamps, no shared IDs. They're strangers.

So if you find an interesting edge case in the steering data, you can't just look up the video. You're stuck with numbers on a screen, wondering what actually happened.

Unless you get creative.

---

## The Insight

Here's the thing about cars: physics doesn't care about dataset IDs.

When a car turns, it experiences lateral acceleration. The relationship between lateral acceleration, speed, and yaw rate is simple kinematics:

```
yaw_rate = latAccel / speed
```

commaSteeringControl gives me `latAccelLocalizer` and `vEgo`. commavq gives me yaw rate in the pose data. Different datasets, same physics.

So I can find an edge case in the steering telemetry, compute what the yaw rate would have been, and search commavq for segments with similar dynamics. Not the same drive. Not the same moment. But kinematically similar.

It's not a perfect match. It's a physics-based approximation. And that's good enough to find relevant video.

---

## The Tools

I used claudette, a lightweight wrapper around Claude's API that makes building agentic systems simple. The `@tool` decorator turns any Python function into something an LLM can call.

First, a helper to check if a segment matches our target dynamics:

```python
def matches_edge_case_tight(pose, target_speed=20, target_yaw=0.05, max_yaw=0.1, tolerance=0.2):    
    speeds = np.array(pose)[:, 0]
    yaws = np.abs(np.array(pose)[:, 5])
    if np.nanmax(yaws) >= max_yaw: return False
    high_speed = speeds > target_speed * (1 - tolerance)
    in_yaw_range = (yaws > target_yaw * (1 - tolerance)) & (yaws < max_yaw)
    return np.any(high_speed & in_yaw_range)
```

A tool to find edge cases in the steering data:

```python
@tool
def find_edge_cases(n: int = 10):
    "Stream through control data and find edge cases where openpilot was in control."
    ds = load_dataset("commaai/commaSteeringControl", streaming=True)
    samples = []
    for i, s in enumerate(ds['train']):
        if i % 1000 != 0: continue
        gap = abs(s['latAccelDesired'] - s['latAccelLocalizer'])
        if gap > 1.0 and s['vEgo'] > 15 and s['latActive'] and not s['steeringPressed']:
            samples.append(s)
            if len(samples) >= n: break
    return samples
```

The logic: I'm looking for moments where there's a big gap between what openpilot wanted to do (`latAccelDesired`) and what actually happened (`latAccelLocalizer`). High speed, openpilot in control, driver not intervening. These are the interesting moments.

A tool to match those dynamics to video:

```python
@tool
def match_video(speed: float, latAccel: float, n: int = 3):
    "Find commavq segments matching speed and latAccel signature."
    yaw_rate = abs(latAccel / speed)
    ds_vq = load_dataset("commaai/commavq", streaming=True)
    keys = []
    for sample in ds_vq['train']:
        if matches_edge_case_tight(sample['pose.npy'], target_speed=speed, target_yaw=yaw_rate):
            keys.append(sample['__key__'])
            if len(keys) >= n: break
    return keys
```

The physics bridge in action. Compute the expected yaw rate, scan for video segments with similar dynamics.

---

## The Agent

Once the tools are defined, wiring up the agent is almost anticlimactic:

```python
sp = "You are an AV simulation assistant. Find edge cases where openpilot struggled and match them to video segments."
chat = Chat(model="claude-sonnet-4-20250514", sp=sp, tools=[find_edge_cases, match_video])

r = chat.toolloop("Find 2 edge cases and match the first one to video segments.")
```

That's it. The `toolloop` method lets Claude decide when to call which tool, chain the results together, and report back. I ask it to find edge cases, it calls `find_edge_cases`. It sees the results, extracts the speed and lateral acceleration, calls `match_video`. Done.

No manual orchestration. No state machine. Just a system prompt, some tools, and a task.

---

## Results

The agent found edge cases where openpilot's desired lateral acceleration diverged significantly from what the car actually did. High speed, moderate curves, no driver intervention. Exactly the kind of moments worth studying.

Here's what matched segments look like:

![Matched segments showing speed and yaw rate](/images/matched_segments.png)

The first segment shows acceleration with a significant yaw event mid-drive. The second shows variable speed through a curve. Both have dynamics similar to the edge cases the agent found in the steering data.

Not the same drive. Not the same car. But the same physics.

---

## What's Next

The agent finds edge cases and matches them to video segments. But right now, those segments are just tokenized data. To actually see the video, you need to decode it.

I have a video decoder working in Colab. The next step is bringing that into the agent's toolset. Imagine asking: "Find an edge case, match it to video, and show me what happened." One command, full pipeline.

After that: synthetic variants. Once I can identify interesting scenarios and retrieve matching video, the goal is to generate variations. Different lighting. Different weather. Wet roads. Glare. A pedestrian at the edge of the frame. Edge cases the car hasn't seen yet, built from ones it struggled with.

Self-driving systems fail on the long tail. The rare scenarios that don't show up often enough in real driving data. You can drive for millions of miles and still miss the weird stuff. But if you can automatically find the moments where the system struggled, and then generate hundreds of variations of those specific scenarios, you're targeting exactly where the model is weak.

That's not random data augmentation. That's directed training.

---

## Takeaways

The specific datasets don't matter. The pattern does.

**Beyond comma.ai**: Any AV platform has the same problem. Telemetry lives in one place, video in another, lidar in a third. Different sensors, different sample rates, different storage systems. But physics connects them. If you know the relationship between lateral acceleration and yaw rate, you can bridge datasets that were never designed to talk to each other.

**Robotics**: Same story, different equations. A robot arm has joint encoders, force sensors, and cameras. They're rarely synced perfectly. But kinematics and dynamics give you the bridge. End effector velocity is the Jacobian times joint velocities. Contact force is mass times acceleration. You can find similar manipulation scenarios across different recordings using the same approach.

**Beyond AV and robotics**: Any domain with multimodal data that wasn't designed to link up. Medical imaging and clinical notes. Financial transactions and news feeds. IoT sensor logs and maintenance records. The trick is finding the domain-specific "physics" that connects them. Sometimes it's actual physics. Sometimes it's a shared business process. Sometimes it's time and geography.

The real takeaway? Datasets are more connected than they look. You just have to find the bridge.

**Code**: [github.com/c5huracan/agentic-av-sim](https://github.com/c5huracan/agentic-av-sim)
