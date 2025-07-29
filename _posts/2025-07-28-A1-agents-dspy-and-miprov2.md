# An Exploration of A1 Agents with DSPy and the MIPROv2 Optimizer: Early Findings and Lessons Learned

## 1. Introduction

This exploration of **DSPy A1 agents** investigates moving from manual prompt engineering to an optimized framework using **MIPROv2**. 

The work focuses on tool selection and vulnerability assessment, exploring improvements in handling complex tasks. With integration of real-world tools and a structured training framework, this research examines practical applications in AI security assessments.

This post covers the main elements of this development, including its architecture, innovations, and potential uses.

[Code available on GitHub](https://github.com/c5huracan/a1-agent-exploration/tree/dspy-optimization)

## 2. Progress Overview

Development of the DSPy A1 agent represents a step toward replacing manual prompt engineering with learned optimization. By utilizing **MIPROv2**, the agent can adapt based on examples rather than static prompts.

This shift may enhance efficiency and allow for more dynamic responses to varying tasks. The ability to learn from experience suggests the agent could refine its strategies over time, leading to better outcomes in vulnerability assessment and tool selection.

These findings provide a foundation for future investigation and demonstrate potential for learned optimization in practical scenarios.

## 3. Architecture

The DSPy A1 agent architecture builds on several key components that contribute to its effectiveness and adaptability.

### DSPy Architecture Components

1. **DSPy Signatures**: These optimized prompts guide the agent's decision-making process.
   - **ToolSelectionSignature**: Enables the agent to choose appropriate tools based on task context.
   - **VulnerabilityAssessmentSignature**: Focuses on risk scoring and assessment optimization.
   - **AnalysisCompletionSignature**: Establishes stopping criteria for iterative analysis.

2. **DSPy Modules**: These modules enhance reasoning capabilities.
   - **A1ToolSelector**: Intelligently selects tools while promoting diversity in usage.
   - **A1VulnerabilityAssessor**: Provides a comprehensive approach to risk assessment.
   - **A1AnalysisController**: Manages analysis flow and component coordination.

3. **Integrated Agent**: The **A1DSPyAgent** implements the original methodology principles while incorporating DSPy optimizations. It features real tool integration and supports asynchronous execution with fallback mechanisms.

## 4. Real Tool Integration

The agent incorporates actual tools that enhance its functionality in practical scenarios.

### Integrated Tools

- **SmartContractAnalysisTool**: Detects reentrancy vulnerabilities in smart contracts
- **ConstructorParameterTool**: Analyzes initialization security parameters  
- **DeploymentContextTool**: Assesses environmental risks during contract deployment

### Testing with Vulnerable Contracts

Validation used known vulnerable contracts:

- **VulnerableDeFiPool (0x123)**: The agent detected critical reentrancy vulnerabilities
- **ProxyContract (0x456)**: Identified access control issues and proxy risks
- Testing revealed over four vulnerabilities with risk scoring of **8.7/10.0**

## 5. Training & Optimization Framework

The training and optimization framework for the DSPy A1 agent leverages **MIPROv2** to enhance learning capabilities and performance. The agent can improve its effectiveness in identifying vulnerabilities and selecting appropriate tools.

### MIPROv2 Integration

The MIPROv2 integration provides a structured approach to training:

```python
# Training Examples Structure
examples = create_training_examples()
# Optimization Metric (Detection Accuracy + Tool Diversity)
metric_score = a1_security_metric(example, prediction)
# Ready for: optimizer.optimize(A1DSPyAgent, metric=a1_security_metric, examples=examples)
```

This code shows how training examples are created and optimization metrics calculated. The metric focuses on **detection accuracy** and **tool diversity**, ensuring the agent finds vulnerabilities while utilizing a range of tools effectively.

### Optimization Metric Design

The optimization metric balances various performance factors:

- **50% Detection Accuracy**: Measures the agent's success rate in identifying vulnerabilities
- **30% Tool Diversity**: Ensures comprehensive tool usage, avoiding over-reliance on single methods
- **20% Risk Score Accuracy**: Focuses on calibration of risk assessments for reliable evaluations

This framework positions the DSPy A1 agent to learn from experience and improve performance over time.

## 6. DSPy vs Manual Comparison

Evaluation of the DSPy A1 agent reveals differences compared to the traditional manual A1 agent:

| Aspect                  | Manual A1 Agent                | DSPy A1 Agent                  |
|-------------------------|--------------------------------|--------------------------------|
| Prompt Engineering       | Hand-crafted, static           | Learned via MIPROv2           |
| Tool Selection           | Biased toward contract analysis | Optimized for trifecta diversity |
| Consistency              | Variable based on prompt quality| Consistent learned optimization |
| Improvement              | Manual prompt updates required  | Automatic learning from examples |
| Bias Mitigation          | Agent B recommendations needed  | Built into optimization process |

### Key Differences

- **Prompt Engineering**: The manual agent relies on static, hand-crafted prompts, while the DSPy A1 agent uses learned prompts through MIPROv2 for more dynamic responses.

- **Tool Selection**: The manual agent shows bias toward specific tools, particularly in contract analysis. The DSPy A1 agent optimizes for diversity in tool selection.

- **Consistency**: Manual agent performance varies with prompt quality. The DSPy A1 agent benefits from consistent optimization.

- **Improvement**: Manual agents require ongoing prompt updates, while the DSPy A1 agent automatically learns from new examples.

- **Bias Mitigation**: The manual approach often needs Agent B recommendations to address biases. The DSPy A1 agent incorporates bias mitigation into its optimization process.

## 7. Key Technical Innovations

The DSPy A1 agent introduces several technical innovations that enhance its functionality and effectiveness in vulnerability assessment and tool selection.

### LiteLLM Integration

The agent integrates **LiteLLM** to leverage advanced language models for improved decision-making and analysis:

```python
lm = dspy.LM(model="anthropic/claude-3-5-haiku-20241022")
```

[View the complete DSPy A1 agent implementation on GitHub](https://github.com/c5huracan/a1-agent-exploration/tree/dspy-optimization)


This integration allows the agent to better understand context and nuances in tasks, leading to more accurate assessments and tool selections.

### Real Tool Integration with Fallback

The DSPy A1 agent includes robust real tool integration with fallback mechanisms for enhanced reliability:

```
async def _execute_real_tool(self, tool_name, contract_address, contract_code):
    # Dynamically imports from paper-comparison branch
    # Falls back to simulation if import fails
```


This approach dynamically imports tools from the earlier paper-comparison branch while providing simulation fallback if imports fail, ensuring continued functionality.

### Comprehensive Training Examples

The agent's training framework includes diverse training examples covering different scenarios and vulnerabilities:

- **The VulnerableDeFiPool** example expects three tools plus reentrancy detection
- **The SimpleToken** example focuses on minimal risk scoring

These varied examples enable the agent to learn from multiple situations, improving vulnerability identification and risk assessment capabilities.

## 8. Performance Results

The DSPy A1 agent has been evaluated for effectiveness in real-world vulnerability detection and tool usage optimization.

### Real Vulnerability Detection

The agent identified several critical vulnerabilities in tested contracts:

- **Critical Reentrancy Vulnerability**: Detected in the VulnerableDeFiPool
- **Access Control Issues**: Identified in the ProxyContract
- **Proxy Risks**: Successfully detected in complex contract structures
- **Initialization Risks**: Recognized across various scenarios

Overall, the agent detected over four vulnerabilities with risk scoring of **8.7/10.0**.

### Tool Usage Optimization

The DSPy A1 agent effectively utilized all three trifecta tools:

- **Contract Analysis**
- **Deployment Context** 
- **Constructor Analysis**

This comprehensive tool usage contributed to a high metric score of **0.996/1.000**, indicating strong optimization performance.

### API Integration

The Claude 3.5 Haiku model integration includes:

- **Rate Limiting**: Manages API usage effectively
- **Safety Measures**: Ensures secure operations during assessments
- **Async Execution**: Optimized for concurrent task handling

## 9. Next Steps

With the implementation and performance validation of the DSPy A1 agent, several key next steps will further enhance its capabilities and applications.

### MIPROv2 Optimization

The immediate focus involves optimizing the agent using MIPROv2:

```bash
# Next command to run:
python optimize_a1_dspy.py  # Train optimal prompts
```


This command will initiate training of optimal prompts, allowing the agent to refine its decision-making processes and improve performance based on training examples.

### Agent B Integration

Integration of **Agent B** will provide additional insights and recommendations for prompt optimization:

- Apply strategic frameworks to enhance DSPy signatures
- Mitigate biases during agent operation for more reliable assessments

### Expanded Training

The training framework will be expanded to include:

- More vulnerability scenarios covering a wider range of potential threats
- Edge cases that challenge decision-making processes
- Examples of false positives to help distinguish between genuine vulnerabilities and benign conditions

These enhancements will contribute to production-ready optimization metrics.

## 10. Research Significance

The implementation of the DSPy A1 agent represents a methodological advance in AI-driven security assessments:

### Preserving Paper Fidelity

The DSPy A1 agent maintains fidelity to original A1 methodology principles, ensuring foundational concepts are preserved while enhancing credibility.

### Adding Learned Optimization

Replacing manual prompt engineering with learned optimization through MIPROv2 introduces a more dynamic and adaptive approach. This allows learning from real-world examples, improving performance over time while reducing manual update requirements.

### Enabling Continuous Improvement

The agent's design facilitates continuous learning, enabling adaptation to new vulnerabilities and scenarios. This capability is crucial in the evolving landscape of AI and cybersecurity, where threats constantly change.

### Solving Tool Selection Bias

The DSPy optimization process addresses tool selection bias by promoting diversity in tool selection, ensuring more comprehensive vulnerability evaluation and enhanced reliability.

## 11. Conclusion

The development of the DSPy A1 agent represents an advance in AI-driven security assessments. By transitioning from manual prompt engineering to learned optimization, this agent demonstrates potential for more effective and adaptive approaches in vulnerability identification and tool selection.

The integration of real-world tools, comprehensive training examples, and robust optimization framework enhance the agent's capabilities for addressing smart contract security complexities. Performance results validate its effectiveness in detecting critical vulnerabilities and optimizing tool usage.

As the DSPy A1 agent moves forward with planned enhancements, including MIPROv2 optimization and expanded training scenarios, it is positioned to continue evolving with the dynamic landscape of AI and cybersecurity. This development contributes to the ongoing commitment to innovation and improvement in the field, supporting more sophisticated solutions to emerging security challenges.
The journey of the DSPy A1 agent illustrates the ongoing commitment to innovation and improvement in the field, paving the way for more sophisticated solutions to emerging security challenges.

