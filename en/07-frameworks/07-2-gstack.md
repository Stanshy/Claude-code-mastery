# 07-2 gstack Role System
> How Y Combinator President Garry Tan uses 18 Skills to simulate a complete engineering team.

## Learning Objectives
- Understand why gstack uses "role-based division of labor" rather than a "single monolithic CLAUDE.md" to organize workflows
- Master the design logic behind the two most critical mechanisms: `/office-hours` and `/investigate`
- Learn three practices from gstack that you can directly transplant into your own system
- Understand gstack's limitations and determine which scenarios are suitable for adopting this system

---

## Content

### 1. Background: An Engineering Experiment by a YC President

Garry Tan is the current President of Y Combinator. He open-sourced gstack (github.com/garrytan/gstack) and used this system to deliver 600,000 lines of production code within 60 days. The number itself is already noteworthy, but what is even more noteworthy is the design philosophy he chose.

gstack is not a massive configuration file stuffed with rules, but rather a system that simulates the organizational structure of an engineering team. Its core assumption is: **Different types of problems require different thinking frameworks, and these frameworks should not interfere with each other within the same context window.**

---

### 2. Design Philosophy: Role-Based Division of Labor, Not Rule Stacking

Most people build their CLAUDE.md by continuously adding rules until the file becomes a list spanning hundreds of lines. This approach has a fundamental problem: when you ask Claude to write code, rules about "how to do a security audit" are also occupying space and attention in the context.

gstack's solution is to split different types of work into independent Skill files, with each Skill simulating an engineering team role. When you need a particular role's judgment, only then do you load that role's context.

gstack's 18 Skills cover the complete engineering lifecycle:

| Role | Function | Corresponding Skill |
|------|----------|-------------------|
| CEO | Rethink product direction | `/rethink` |
| Eng Manager | Lock down architecture decisions | `/lock` |
| Designer | Detect AI slop (low-quality AI output) | `/design-review` |
| Reviewer | Find production bugs | `/review` |
| QA Lead | Browser-based testing | `/qa` |
| Security Officer | OWASP + STRIDE audit | `/security` |
| Release Engineer | PR release | `/ship` |

These seven roles cover the entire process from product decisions to deployment. Each role has clear responsibility boundaries and does not encroach on others.

---

### 3. Core Workflow: Think → Plan → Build → Review → Test → Ship → Reflect

gstack enforces a complete cycle and does not allow skipping steps. The design logic behind this cycle is: **The output of each step is the input for the next step, and any skipped step will appear later at a higher cost.**

- **Think**: Clarify what the problem is and why it needs to be solved
- **Plan**: Decide how to solve it, break it down into steps
- **Build**: Implement
- **Review**: Examine implementation quality from multiple angles (functionality, security, performance, maintainability)
- **Test**: Verify functional correctness
- **Ship**: Release and document decisions
- **Reflect**: Extract learnings from this round of work and update the system

---

### 4. Two Mechanisms Worth Understanding in Depth

#### /office-hours: Forcing Answers to Six Questions Before Taking Action

**Why this mechanism exists:** The problem with most engineering decisions is not "how to do it," but rather that "what to do" and "why to do it" have not been sufficiently thought through. Jumping straight into writing code is an impulse — it feels like progress, but it is often rapid movement in the wrong direction.

`/office-hours` forces you to answer six questions before starting any major task. These six questions are designed to surface hidden assumptions and overlooked considerations:

1. What specific user problem does this feature solve?
2. What is the definition of success? How do you verify it?
3. What technical constraints or dependencies exist?
4. Where is the most likely point of failure?
5. Is there a simpler solution?
6. Is this decision reversible? If wrong, what is the cost?

Until you have answered all six questions, gstack does not allow you to enter the Build phase. This forced pause is essentially about front-loading the "cost of thinking" rather than letting it appear later during implementation as a much higher "cost of correction."

#### /investigate: Finding the Root Cause Before Fixing a Bug

**Why this mechanism exists:** The most common mistake pattern in bug fixing is "seeing a symptom and immediately making changes." This approach has two problems: first, you may be fixing the symptom rather than the cause, and the problem will resurface in a different form; second, without understanding the root cause, you cannot know whether your fix will introduce new problems.

`/investigate` enforces root cause analysis before any bug fix. The process is:

1. Reproduce the problem (confirm that the problem actually exists)
2. Isolate variables (confirm the boundaries of the problem)
3. Form a hypothesis (why does this problem occur)
4. Verify the hypothesis (confirm the hypothesis is correct using the most minimal approach)
5. Propose a fix (fix the root cause, not the symptom)

Only after completing the first four steps do you move into the phase of actually modifying code.

---

### 5. Three Practices You Can Directly Transplant

You do not need to adopt the entire gstack system, but these three practices can be used independently, and each one can bring immediate, tangible benefits:

**Practice One: Interrogation Mode**

Before starting any task estimated to take more than 4 hours, force yourself to answer 5 to 6 questions. You can use gstack's six questions as a starting point, or adjust them for your own domain. The key point is "surfacing hidden assumptions before taking action."

**Practice Two: Find the Root Cause First**

Establish a personal rule: for any bug fix, you must first write down "The root cause is" on paper or in CLAUDE.md before making any changes. This act of putting it in writing forces you to switch from "feeling" to "analysis."

**Practice Three: Complete the Full Cycle**

Do not just Build — walk through the full cycle of Think → Plan → Build → Review → Test → Ship → Reflect. Even if each step takes only 5 minutes, it is more reliable than skipping straight to Build. The Reflect step is especially worth emphasizing — it is the only opportunity to extract reusable knowledge from each round of work.

---

### 6. Limitations of gstack

Before adopting any system, understand its boundaries first:

**Learning Curve:** 18 Skills take time to become familiar with. When you are still unsure which Skill corresponds to which scenario, the friction of switching roles will reduce efficiency rather than improve it.

**Scale Assumptions:** gstack's design assumes you are working on a project of a certain scale — with real security concerns, architecture decisions that require cross-team coordination, and release processes that need to be formalized. If you are building a weekend prototype, `/security` (OWASP + STRIDE audit) is likely over-engineering.

**Customization Required:** gstack was designed by Garry Tan for his own workflow. His product decision cycles, tech stack, and team structure are all reflected in the design of these 18 roles. You need to adjust it to your own situation rather than copying it directly.

---

## Teaching Example

### Scenario: Using /office-hours to Avoid a Costly Wrong Technical Decision

Suppose you are an independent developer building a "report export" feature for a B2B SaaS product. Your first instinct is: "This is simple — use `exceljs` to generate an Excel file and let the user download it." You are about to open a Claude Code session, provide the task description, and let it start writing code.

But you remember gstack's `/office-hours` mechanism. Before starting the session, you force yourself to answer six questions.

**Question One: What specific user problem does this feature solve?**

Your answer: "Users need to take data off the platform and do further analysis in Excel."

This answer seems reasonable, but you continue.

**Question Two: What is the definition of success? How do you verify it?**

You start thinking: is success "the user can download an Excel file"? Or is it "the user can correctly analyze data in Excel"? If a report has 500,000 records and the user downloads a 300MB Excel file, does that count as success?

You realize you have not thought through the data volume issue.

**Question Three: What technical constraints or dependencies exist?**

You check: your API server is stateless, running in a container with 512MB of memory. If a large customer has 1 million records, generating Excel in memory will cause the container to OOM crash. You had not thought of this before.

**Question Four: Where is the most likely point of failure?**

Now it is clear: the memory issue with large data volumes is the most likely failure point. Second is whether "Excel format" is truly what users need, or whether they actually just need CSV because they will eventually convert the Excel file to another format anyway.

**Question Five: Is there a simpler solution?**

You think about it: if the user's real need is "take data away to analyze," then CSV streaming might be a more lightweight solution. The server does not need to construct the entire file in memory; instead, it outputs line by line using streaming, with near-zero memory usage.

**Question Six: Is this decision reversible? If wrong, what is the cost?**

If you build the Excel solution first and later discover problems with large data volumes, you will need to rewrite the entire export logic. The cost is high. If you build CSV streaming first and later need to add Excel support, you can extend from this foundation. The cost is low.

**Conclusion**

You spent 15 minutes answering six questions and arrived at a conclusion completely different from your original instinct: implement CSV streaming first, with a clear memory cap (e.g., return a maximum of 100,000 records; beyond that, prompt the user to export in batches), and then evaluate whether Excel format is needed based on user demand later.

This decision changed the task description you later gave to Claude:

Original version: "Use exceljs to implement report export, letting users download an Excel file."

Updated version: "Implement CSV streaming export using Node.js `stream.Transform`, querying from the database and writing to the response stream line by line. Query a maximum of 1,000 records at a time, with a total cap of 100,000 records. When the cap is exceeded, return HTTP 413 with an error message. Please write a unit test for the streaming behavior first, then implement the main logic."

With this more precise task description, the code Claude produced was of higher quality because the task boundaries and acceptance criteria had been clearly defined. And all of this required only 15 minutes of interrogative thinking.

This is the core value of `/office-hours`: it is not about making you spend more time thinking, but about making you think before you start rather than after problems emerge.

---

## Key Takeaways
- gstack's role-based division of labor addresses the problem of "a single monolithic CLAUDE.md causing different types of work to interfere with each other"
- The six interrogation questions in `/office-hours` force hidden assumptions to be surfaced upfront, preventing rapid progress in the wrong direction
- `/investigate` requires completing four root cause analysis steps before fixing a bug, preventing the common pattern of "fixing symptoms rather than causes"
- Every step in the complete cycle (Think → Plan → Build → Review → Test → Ship → Reflect) is the foundation for the next step; the cost of skipping steps will appear later
- gstack is suitable for projects of a certain scale; smaller projects should selectively adopt its mechanisms rather than importing the entire system

---

## Self-Assessment
- The last time you started a new feature, how much time did you spend thinking about "why should we do this" and "where is the most likely failure point" before writing the first line of code?
- Which of gstack's roles (CEO, Eng Manager, Designer, Reviewer, QA Lead, Security Officer, Release Engineer) is most lacking in your current workflow? How are you currently filling (or not filling) that gap?
- If you had to pick one of gstack's three transplantable practices to try this week, which one would you choose? Why?
