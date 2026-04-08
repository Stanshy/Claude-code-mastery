# 05-1 Agent Teams
> Let multiple Claude sessions collaborate to break through the capability boundaries of a single session.

## Learning Objectives
- Understand the collaborative architecture and applicable scenarios of Agent Teams
- Be able to enable and configure the Agent Teams experimental feature
- Learn to design task allocation strategies for Lead and Teammates
- Determine when to use Teams versus when Subagent is sufficient

## Content

### 1. Why Agent Teams Are Needed

Every Claude session has a context limit and capability boundary. When a task is large enough, a single session needs to remember "the feature being implemented" while simultaneously "analyzing test coverage" -- these two tasks compete for context, causing quality degradation in both.

The design philosophy of Agent Teams is division of labor, not stacking: one Lead session coordinates the big picture, while multiple Teammate sessions independently execute subtasks on a shared codebase, reporting results upon completion. Team members can communicate directly with each other without routing everything through the Lead. This enables truly parallel progress on large tasks, rather than simulated parallelism.

Agent Teams is currently an experimental feature, meaning the API and behavior may change across versions. Before using it in production systems, pin your Claude Code version and perform thorough testing.

### 2. Architecture

Agent Teams has three roles:

**Team Lead**: The commander of the overall task. Responsible for parsing user requirements, breaking down subtasks, assigning them to appropriate Teammates, monitoring progress, and ultimately integrating results. The Lead does not directly modify code -- its job is coordination.

**Teammates**: The executors. Each Teammate receives a clearly scoped subtask and makes independent decisions within their own scope. Teammates can communicate directly with each other, for example, "the Teammate responsible for the feature notifies the Teammate responsible for testing that the function signature has been finalized."

**How to enable**: Set an environment variable -- no code changes required:

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

Once enabled, describe a task requiring multi-agent collaboration in the Claude Code session, and the Lead will automatically determine whether to spawn Teammates.

### 3. Applicable Scenarios

The value of Agent Teams lies in "truly independent parallel work." The following scenarios meet this condition:

**Research and Review**: One Teammate dives deep into a problem (e.g., analyzing the existing authentication implementation), while another simultaneously reviews the related security specifications. The two workflows do not interfere with each other, and the Lead synthesizes the results at the end.

**New Module Development**: Feature implementation and test writing can be split apart. The Teammate responsible for implementation notifies the testing Teammate once the function signature is complete; the latter immediately begins writing tests while the former continues refining the implementation details.

**Multi-Hypothesis Debugging**: When a bug has three possible causes, the traditional approach is to verify them sequentially. Agent Teams can have three Teammates simultaneously verify three hypotheses, and the Lead selects the correct fix based on the results.

**Cross-Layer Collaboration**: A frontend Teammate modifies the API call logic while a backend Teammate simultaneously adjusts the corresponding endpoint. The two communicate to ensure interface consistency, rather than waiting for one to finish before the other can begin.

### 4. Agent-to-Agent Review

OpenAI developed a more aggressive pattern in practice: Agents review each other's PRs, and humans do not need to intervene every time.

The process (OpenAI calls it the "Ralph Wiggum Loop"):

1. Agent A completes the implementation and opens a PR
2. Agent A reviews its own PR
3. Agent B (local or cloud) performs an independent review of the PR
4. Agent A responds to the review feedback and makes changes
5. The loop iterates until all reviewer Agents are satisfied
6. Humans can review, but it is not required

This pattern makes sense in high-throughput environments: when each person produces 3.5 PRs per day, human review becomes the bottleneck. Agent-to-Agent review frees up human attention, allowing humans to focus on directional decisions rather than code details.

Current limitations: Requires a highly mature Harness (architectural constraints + automated testing + structured linters) as the underlying quality assurance layer; otherwise, the quality of Agent reviews is unreliable. Until your Harness has established sufficient architectural constraints, human review remains necessary.

---

### 5. Limitations and Considerations

**Risks of an experimental feature**: The `EXPERIMENTAL` in `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is not decorative. The feature itself may have unknown edge-case behaviors. Before using it on important projects, verify it on a test branch first.

**Compute budget**: Each Teammate consumes independent tokens. The cost of three Teammates running simultaneously is approximately three times that of a single session. When splitting tasks, ensure that the time savings from parallelism justify this cost.

**Scale recommendations**: A team size of 3-5 agents is optimal. Beyond five agents, the Lead's coordination overhead becomes the bottleneck -- it needs to handle more progress reports, conflict resolution, and result integration, making it less efficient than a smaller team.

**Task boundary design**: The most common issue with Teammates is modifying the same file, leading to merge conflicts. When designing tasks, file-level isolation is the most reliable strategy: Teammate A is responsible for `src/analyzer.ts`, Teammate B is responsible for `tests/analyzer.test.ts`, with no overlap.

**Pragmatic selection principle**: For simple tasks (single file, single goal), Subagent is sufficient. Only when a task genuinely requires "multiple people working independently in parallel to speed things up" is it worth incurring the coordination cost of Agent Teams. Do not use a new feature just for the sake of using it.

## Hands-On Example: 3-Agent Team Developing a New Module
> Continuing the main project: In the AI Dev Assistant project, use an Agent Team to develop a PR analysis module in parallel, with one Teammate implementing the feature and another writing tests.

**Goal**: Use an Agent Team to build a PR analysis module for ai-dev-assistant, including implementation and tests, and complete the collaboration through the Team mechanism.

**Prerequisites**:
- Completed 04-3 (Worktree setup), project root directory is `C:\projects\ai-dev-assistant`
- Node.js 18.x, TypeScript 5.x, Jest 29.x environment is ready
- Claude Code is installed and functioning normally

**Steps**:

1. Set the experimental feature environment variable (this step is required each time you open a new terminal):
   ```bash
   export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
   echo $CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS  # Confirm output is 1
   ```

2. Navigate to the project directory and start the Lead session:
   ```bash
   cd /c/projects/ai-dev-assistant
   claude
   ```

3. Enter the following task description in the Lead session (paste it in full so the Lead can correctly split the tasks):
   ```
   I need to build a PR analysis module for ai-dev-assistant.
   Please split this into the following two subtasks and assign them to Teammates:
   - Teammate A: Implement the analyzePR function in src/analyzer.ts,
     accepting a PR diff string and returning an object containing
     issues and suggestions arrays.
   - Teammate B: Write tests for analyzePR in tests/analyzer.test.ts,
     including normal input and empty string edge case tests.
   Once Teammate A completes the function signature, please notify Teammate B.
   ```

4. Observe the Lead's task allocation output. The Lead will display the Teammates it is spawning and their respective task scopes. Under normal circumstances, you will see a message similar to "Spawning Teammate for src/analyzer.ts implementation."

5. Observe the two Teammates working independently. After Teammate A creates `src/analyzer.ts` and defines the function signature, the Lead interface will display "Notifying Teammate B of function signature."

6. After both Teammates have finished, the Lead will integrate the results and report back. Confirm the files exist:
   ```bash
   ls /c/projects/ai-dev-assistant/src/analyzer.ts
   ls /c/projects/ai-dev-assistant/tests/analyzer.test.ts
   ```

7. Run the tests to confirm the integrated result is correct:
   ```bash
   cd /c/projects/ai-dev-assistant
   npx jest tests/analyzer.test.ts
   ```

8. If the tests fail, return to the Lead session and ask it to coordinate the fix:
   ```
   The tests failed with the following error message: [paste error]. Please coordinate Teammates to fix it.
   ```

**Expected Results**:
- The Lead session displays clear task allocation records and progress for both Teammates
- `src/analyzer.ts` contains the `analyzePR` function with complete TypeScript types
- `tests/analyzer.test.ts` contains at least three test cases (normal flow, empty string, malformed input)
- `npx jest tests/analyzer.test.ts` passes all tests

**Common Errors**:
- **Environment variable not set, Claude started in normal mode instead of Team mode**: The symptom is Claude immediately begins completing the task independently without displaying "Spawning Teammate." Solution: exit the session, run `export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, then restart `claude`. Note that export only applies to the current shell; you need to run it again when opening a new terminal.
- **Teammates modified the same file simultaneously, causing content conflicts**: The symptom is abnormal content in a file, or TypeScript compilation reporting errors that should not exist. The root cause is that the task description did not clearly delineate file responsibilities. Solution: when re-describing the task, explicitly specify "Teammate A only modifies src/analyzer.ts, Teammate B only modifies tests/analyzer.test.ts, neither touches the other's files."

## Key Takeaways
- Agent Teams solve the problem of "tasks too large that need truly parallel work," not "tasks too difficult"
- The Lead coordinates, Teammates execute -- clear division of labor is the prerequisite for a functioning Team
- After enabling `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, you must proactively ask the Lead to split tasks in your task description for it to spawn Teammates
- 3-5 Teammates is the optimal scale; beyond that, coordination costs rise faster than benefits
- Designing task boundaries with file-level isolation is the most reliable way to avoid Teammate conflicts

## Self-Assessment
- Can you describe the responsibilities of the Lead and Teammates, and how they communicate with each other?
- Given a task (e.g., "add a user authentication module to ai-dev-assistant"), can you design a reasonable 3-Teammate division of labor and specify each Teammate's file scope?
- Under what conditions would you choose Subagent over Agent Teams?

---

⬅️ [Previous: Parallel Development & Worktree](../04-orchestration/04-3-worktree.md) ｜ 📖 [Index](../00-index.md) ｜ ➡️ [Next: Headless Mode & CI/CD Integration](05-2-headless-cicd.md)

🌐 [繁體中文版](../../zh/05-自主層/05-1-Agent-Teams.md)
