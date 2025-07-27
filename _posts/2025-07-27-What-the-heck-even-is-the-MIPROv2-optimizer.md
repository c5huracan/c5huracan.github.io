# What the Heck Even is the MIPROv2 Optimizer?

## Intro:

MIPROv2 (Multiprompt Instruction Proposal Optimizer v2) is a powerful DSPy optimizer designed to jointly optimize both prompt instructions and few-shot examples for language model programs. It works by generating candidate instructions and demonstrations, then uses Bayesian Optimization to search for the best combination that maximizes your chosen evaluation metric, making it suitable for both few-shot and zero-shot prompt optimization ([MIPROv2 API](https://dspy.ai/api/optimizers/MIPROv2/), [How MIPROv2 works](https://dspy.ai/api/optimizers/MIPROv2/#how-miprov2-works)).

## Preliminary breakdown:

MIPROv2 operates in three main stages: (1) it bootstraps few-shot example candidates by sampling from your training set and retaining only those that yield correct outputs, (2) it proposes instruction candidates by generating summaries of your data, program code, and example traces, and (3) it uses Bayesian Optimization to search for the best combination of instructions and examples, evaluating candidate prompts on minibatches and selecting those that maximize your metric ([How MIPROv2 works](https://dspy.ai/api/optimizers/MIPROv2/#how-miprov2-works), [MIPROv2 API](https://dspy.ai/api/optimizers/MIPROv2/)).

This process allows MIPROv2 to jointly optimize both instructions and demonstrations, or just instructions for zero-shot settings, and is configurable for light, medium, or heavy optimization runs depending on your needs ([MIPROv2 API](https://dspy.ai/api/optimizers/MIPROv2/#dspy.MIPROv2.compile)).

## Step-by-step breakdown:

  1. **Bootstrap Few-Shot Examples**: MIPROv2 samples examples from your training set and runs them through your LM program, keeping only those that produce correct outputs as few-shot candidates.
  2. **Propose Instruction Candidates**: It generates instruction candidates using summaries of your dataset, program code, and the previously bootstrapped examples, plus random tips to diversify instructions.
  3. **Bayesian Optimization Search**: MIPROv2 evaluates combinations of instructions and examples on minibatches from your validation set, using Bayesian Optimization to iteratively select the best-performing prompt configuration ([How MIPROv2 works](https://dspy.ai/api/optimizers/MIPROv2/#how-miprov2-works)).

## Code example:

```
import dspy
from dspy.datasets import HotPotQA

# Configure your language model
dspy.configure(lm=dspy.LM("openai/gpt-4o-mini"))

# Prepare your training data
trainset = [x.with_inputs('question') for x in HotPotQA(train_seed=2024, train_size=500).train]

# Initialize MIPROv2 optimizer
tp = dspy.MIPROv2(
    metric=dspy.evaluate.answer_exact_match,
    auto="light"  # Can also use "medium" or "heavy" for more thorough optimization
)

# Optimize your DSPy program
optimized_program = tp.compile(
    dspy.ReAct("question -> answer"),
    trainset=trainset
)
```
This workflow allows you to jointly optimize instructions and few-shot examples for your LM pipeline ([MIPROv2 API](https://dspy.ai/api/optimizers/MIPROv2/#dspy.MIPROv2.compile)).
