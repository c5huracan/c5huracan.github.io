# Initial Thoughts on DSPy's MIPRO Optimizer v2

## Introduction

In the world of language model pipelines, hyper-parameter tuning is often the difference between a system that delivers and one that leaves you scratching your head. DSPy, for all its thoughtful design, still comes with its share of levers and switches that need attention. Under the hood, those configuration choices can quietly shape your results—and your compute bill.

This post takes a practical look at tuning DSPy’s `max_iters`, `num_trials`, and their close relatives. Expect a few stories from the trenches, some advice on avoiding common missteps, and a clear-eyed take on when the defaults will do the job—and when it’s worth rolling up your sleeves.

## Setting the Stage: The Role of `max_iters`

Not every setting in DSPy is a hyper-parameter, even if it looks important. Take `max_iters`: this one is more of a configuration knob. When you set it on a `dspy.ReAct` agent, you’re just telling the agent how many steps it can take to reason or use tools before it needs to wrap things up ([ReAct - DSPy](https://dspy.ai/api/modules/ReAct/)).

It’s a subtle but important distinction. Configuration parameters like `max_iters` set the boundaries for your agent’s behavior, while hyper-parameters such as `num_trials` or `num_candidates` guide the optimizer’s search for the best prompts or weights. If `max_iters` is too low, your agent might not get far enough to solve the problem; too high, and you could end up burning cycles on unnecessary steps. The trick is to pick a value that fits your task’s complexity without overdoing it.

## Diving into `num_trials`, `num_candidates`, and the Interplay with `auto`

When tuning DSPy optimizers, the relationship between `auto`, `num_trials`, and `num_candidates` is worth understanding. Setting `auto` to `"light"`, `"medium"`, or `"heavy"` lets DSPy pick sensible defaults for you, balancing optimization thoroughness and resource usage. This is great for quick experiments or when you want to avoid decision fatigue. However, if you set `auto=None`, you’re in the driver’s seat: you must specify both `num_trials` and `num_candidates` yourself ([MIPROv2 API](https://dspy.ai/api/optimizers/MIPROv2/#dspy.MIPROv2.compile)).

The interplay here is important. If you forget to set one of these when `auto=None`, DSPy will remind you—sometimes with a less-than-friendly error. More trials and candidates can improve results, but they also increase compute time and cost. The trick is to start with moderate values, observe the optimizer’s behavior, and scale up only if you see clear gains.

```python
import dspy
from dspy.datasets import HotPotQA

# Example 1: Using auto to let DSPy pick defaults
tp_auto = dspy.MIPROv2(metric=dspy.evaluate.answer_exact_match, auto="light")
optimized_program_auto = tp_auto.compile(
    dspy.ReAct("question -> answer"),
    trainset=[x.with_inputs('question') for x in HotPotQA(train_seed=2024, train_size=500).train]
)
# Example 2: Manually specifying num_trials and num_candidates
tp_manual = dspy.MIPROv2(
    metric=dspy.evaluate.answer_exact_match,
    auto=None,
    num_trials=20,
    num_candidates=10
)
optimized_program_manual = tp_manual.compile(
    dspy.ReAct("question -> answer"),
    trainset=[x.with_inputs('question') for x in HotPotQA(train_seed=2024, train_size=500).train]
)
```

## The Cascade: Adjusting Related Hyper-Parameters

Tuning one hyper-parameter often means you’ll need to revisit others. For example, after setting `num_trials` and `num_candidates`, you might encounter errors related to `minibatch_size`—especially if your validation set is too small. DSPy expects the `minibatch_size` to be less than or equal to the size of your validation set, and will throw an error if this isn’t the case ([MIPROv2 API](https://dspy.ai/api/optimizers/MIPROv2/#dspy.MIPROv2.compile)).

This is where manually splitting your data into `trainset` and `valset` becomes useful. By controlling the size of each, you can ensure your validation set is large enough to support your chosen batch sizes and optimization rounds ([Optimization Overview - DSPy](https://dspy.ai/learn/optimization/overview/)). A common approach is to shuffle your data and allocate a larger portion to validation than you might in traditional machine learning, since prompt optimizers often benefit from more validation examples. This hands-on approach gives you better control and helps avoid frustrating runtime errors.

```python
# Example 3: Manually splitting data into trainset and valset for better control
import random

data = [x.with_inputs('question') for x in HotPotQA(train_seed=2024, train_size=500).train]
random.shuffle(data)
split_idx = int(0.2 * len(data))  # 20% train, 80% validation
trainset = data[:split_idx]
valset = data[split_idx:]

tp_split = dspy.MIPROv2(
    metric=dspy.evaluate.answer_exact_match,
    auto=None,
    num_trials=20,
    num_candidates=10
)
optimized_program_split = tp_split.compile(
    dspy.ReAct("question -> answer"),
    trainset=trainset,
    valset=valset,
    minibatch_size=35  # Ensure this does not exceed len(valset)
)
```

In practice, expect to iterate: adjust your splits, tweak batch sizes, and revisit your hyper-parameters as you observe how the optimizer performs. This process, while sometimes tedious, pays off in more stable and effective DSPy pipelines.

## Lessons Learned and Recommendations

A few patterns emerge after spending time with DSPy’s hyper-parameters. First, the defaults are a reasonable starting point, but real progress comes from hands-on tuning—especially as your tasks grow in complexity. Adjusting `max_iters`, `num_trials`, and `num_candidates` can make a noticeable difference, but always keep an eye on your validation set size and batch configuration to avoid subtle pitfalls.

Treat hyper-parameter tuning as an ongoing process rather than a one-time setup. Review your results, make small adjustments, and don’t hesitate to revisit earlier choices if something isn’t working as expected. With a bit of patience and iteration, you’ll land on a configuration that balances performance, efficiency, and reliability—setting your DSPy pipeline up for success.

## Conclusion

Tuning DSPy’s configuration and hyper-parameters is an iterative process that rewards careful experimentation and attention to detail. By understanding the roles of parameters like `max_iters`, `num_trials`, and `num_candidates`, and by managing related settings such as validation set size and batch configuration, you can significantly improve both the performance and reliability of your language model pipelines. Applying these best practices—and making use of the code examples provided—will help you get the most out of DSPy, whether you’re building quick prototypes or production-ready systems.

## Further Reading

- [DSPy Optimization Overview](https://dspy.ai/learn/optimization/overview/)
- [MIPROv2 API Reference](https://dspy.ai/api/optimizers/MIPROv2/#dspy.MIPROv2.compile)
- [DSPy Tutorials](https://dspy.ai/tutorials/)
