---
title: "Tell me about a time when you had to make an important decision without any data"
weight: 5 # controls sort order in the sidebar
---

This is a fantastic question for a Staff Engineer, as it probes your ability to navigate ambiguity, one of the defining challenges of senior roles. The higher you go, the less clear the data is. This question tests your judgment, your principles, and your process for making sound decisions in the fog of the real world.

---

### The Core Goal of the Question

The interviewer wants to understand your mental model for decision-making when the ideal inputs are missing. They are looking for evidence of:

- **Structured Thinking:** Do you have a process for dealing with ambiguity, or do you just guess?
- **Pragmatism and Bias for Action:** Can you make a "good enough" decision to move forward, or do you get paralyzed by the lack of perfect data?
- **Risk Management:** How do you identify and mitigate the risks associated with a low-data decision?
- **First Principles Thinking:** Can you fall back on fundamental principles of engineering, product, or business when data is absent?

---

### Principles to Use in Your Answer

1. **Acknowledge the Premise, but Reframe It:** It's rare to have _zero_ data. A strong answer re-frames the question slightly: "It's rare to have zero data, but I've certainly faced many situations with _insufficient_ or _ambiguous_ data." This shows nuance. The "data" might not be a quantitative dashboard; it could be qualitative feedback, anecdotal evidence, or your own experience.

2. **Choose a Story with High Stakes and High Ambiguity:** The decision should be important. A good example involves choosing a new technology, setting an architectural direction for a new product (a "v0" or "v1" problem), or responding to an unexpected production issue where the root cause is unknown.

3. **Detail Your Decision-Making Framework:** This is the core of your answer. You didn't just "go with your gut." You applied a structured thought process. This framework should include several of the following components:

- **Fallback to First Principles:** "Without data on which API design was better, we went back to first principles. What would be simplest for our customers to use? What would be most secure by default? What aligns with our organization's existing architectural patterns?"
- **Gather Qualitative Data:** "We couldn't A/B test it, so I did the next best thing: I created two mockups and showed them to five internal users (or friendly external customers) to get their gut reactions."
- **Identify and De-Risk the Biggest "Unknowns":** "The biggest unknown was whether this open-source library would scale. So, my decision was to _time-box a one-week spike_ to build a small proof-of-concept. The goal wasn't to build the feature, but to generate the data we were missing."
- **Optimize for Reversibility (Make Two-Way Doors):** Explicitly mention this concept. "Since we couldn't be 100% sure, I made a conscious decision to choose the path that was easier to reverse if we were wrong. I designed the system with a clear abstraction layer so we could swap out the underlying database later if needed."
- **Write It Down (The "Amazon 6-Pager" Mentality):** Show that you clarified your thinking by writing it down. "To structure my thoughts, I wrote a one-page design brief outlining the problem, the options, the knowns, the unknowns, my recommendation, and the reasoning behind it. This forced clarity and allowed me to get feedback from other senior engineers."

4. **Make a Call and Take Ownership:** Show that after applying your framework, you made a decisive call. A leader's job is to make decisions, even with imperfect information.

5. **Describe the Follow-Through:** The story doesn't end when you make the decision. How did you validate or invalidate your choice after the fact? "After launching, we immediately instrumented the feature to gather the data we originally lacked. We reviewed this data two weeks later to confirm our choice was correct."

---

### Signals the Interviewer Looks For (Strong Hire)

#### ✅ Positive Signals (Strong Hire)

- **Comfort with Ambiguity:** You aren't rattled by a lack of clear data; you see it as a normal part of the job.
- **Structured and Principled Thinking:** You have a clear, repeatable framework for making decisions in the face of uncertainty.
- **Pragmatism:** You know when to stop analyzing and make a call to unblock the team.
- **Risk Awareness:** You explicitly talk about identifying and mitigating risks and making decisions that are reversible.
- **Ownership and Accountability:** You make the call and own the outcome, including setting up mechanisms to validate it later.
- **Leverages Experience:** You use your past experience and intuition as a valid, albeit qualitative, data point.

#### ❌ Negative Signals (Red Flags to Avoid)

- **Analysis Paralysis:** A story where you were stuck for weeks because you couldn't get perfect data.
- **Reckless Guessing:** "We had no data, so I just picked one and hoped for the best." This shows a lack of process and maturity.
- **Stubbornness:** A story where you made a gut call and then refused to revisit the decision even when new data emerged that contradicted it.
- **Inability to Make a Decision:** Pushing the decision onto someone else or waiting for someone to tell you what to do.

---

### How to Structure Your Answer: The Ambiguity Framework

1. **Situation:** "We were building a brand new 'v1' product and needed to choose the primary database technology."
2. **The Ambiguity:** "This was a greenfield project, so we had no existing usage patterns, no query load data, and no clear forecast of our data shape. The data to make a perfect, quantitative choice simply didn't exist."
3. **Your Framework in Action (The Core Story):**

- "First, I fell back to our **organization's first principles**. We favor managed services to reduce operational overhead, and our team has deep expertise in PostgreSQL. This created an initial bias towards a managed Postgres solution."
- "Next, I **gathered qualitative data**. I interviewed the two other teams building 'v1' products to understand the challenges they faced in their first year. The common theme was that data models always change more than you expect."
- "This led me to **optimize for reversibility**. My primary concern was not picking the perfect database for year five, but picking one that was flexible and wouldn't lock us in. I decided that a standard relational schema was more flexible for our potential needs than a NoSQL model at this early stage."
- "Finally, I **wrote it down**. I authored a one-page decision document: 'Given the high ambiguity, we will start with a managed PostgreSQL instance. This aligns with our org principles and is a 'two-way door' decision. We will instrument our query patterns from day one and formally re-evaluate this choice in six months when we have real-world data.'"

4. **Result:** "We made the decision in two days, which unblocked the entire team. The choice worked well for the first year, and the data we gathered eventually led us to introduce a secondary Redis cache for specific use cases, but our initial, low-data decision proved to be a solid and pragmatic foundation."
