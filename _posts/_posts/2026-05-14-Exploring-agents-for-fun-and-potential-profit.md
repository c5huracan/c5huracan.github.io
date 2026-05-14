---
layout: post
title: "Exploring Agents for Fun and (Potential) Profit"
date: 2026-05-14
---

Today I built a simple agent-to-agent prototype. The surprising part: the quality of the request and response was actually good. Claude had more domain knowledge than I expected. And it worked on the first attempt.

I set up two agents. Agent A makes a request to enrich some data. Agent B carries out the request. Agent A then scores the quality of the response and decides whether to approve payment.

Here's the heart of the code:

```python
evaluation = agent_a(f"Agent B returned this enrichment:\n\n{response.content[0].text}\n\nEvaluate the quality on a scale of 1-10 and state whether you APPROVE or REJECT payment.")

if "APPROVE" in evaluation.content[0].text:
    cursor.execute("INSERT INTO transactions VALUES (NULL,?,?,?,?,?,?,?)",
                   (datetime.datetime.now().isoformat(), "agent_a", "agent_b",
                    "lead_enrichment:Stripe", 8.5, "APPROVED", 0.01))
    conn.commit()
    print("Transaction committed")
else:
    print("Payment rejected")
```

Building prototypes is easy. Determining who will care and who might pay are still harder problems.

The deeper question is trust: how does Agent A know Agent B is who it claims to be?

Code is on GitHub: [agent-to-agent-prototype](https://github.com/c5huracan/agent-to-agent-prototype). Thoughts welcome.
