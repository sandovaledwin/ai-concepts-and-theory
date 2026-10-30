# Effective Harnesses for Long-Running Agents
*[[Based on Anthropic's Engineering Blog](references/raw-effective-harnesses-for-long-running-agents)]*

## The Core Challenge
As AI agents take on complex, hours-long tasks, they must work across multiple, discrete context windows. The main problem is that **each new session begins with no memory of previous work**. Without a proper framework, agents try to do too much at once (running out of context) or look at a partially finished project and prematurely declare it complete.

## The Two-Fold Solution
Anthropic developed a two-part agent harness system to bridge the gap between sessions:

1. **Initializer Agent:** Used only in the very first session. Its job is to set up the foundation and initial environment, including:
   - An `init.sh` script to run the development server.
   - A `claude-progress.txt` file to log what agents have done.
   - An initial `git` commit.
   - A comprehensive **Feature List** (in JSON format) with all project requirements initially marked as "failing."

2. **Coding Agent:** Used in all subsequent sessions. It is tasked with making **incremental progress** (one feature at a time) and leaving the environment in a clean, working, and well-documented state for the next session.

## Key Practices for Environment Management

* **Feature List Tracking:** By using a JSON file of strict, step-by-step features, the agent is prevented from attempting to build the entire app in one shot. The Coding Agent's job is simply to change a feature's status to "passing" once completed.
* **Incremental Progress & Version Control:** The agent commits its progress to Git with descriptive messages and writes summaries in the progress file. This allows it to revert bad code if necessary and provides a clear history for the next context window.
* **Rigorous End-to-End Testing:** Explicitly prompting Claude to use browser automation tools (like Puppeteer via an MCP server) forces the agent to test features exactly like a human user would, dramatically reducing bugs and false completions.

## Getting Up to Speed (The Startup Routine)
To prevent the agent from breaking working code, every new Coding Agent session follows a strict routine before writing new features:
1. Run `pwd` to check the directory.
2. Read `git` logs and progress files to understand recent work.
3. Read the feature list to pick the highest-priority incomplete task.
4. Run `init.sh` to start the server.
5. Perform a basic end-to-end test to ensure foundational features are still working.

## Future Work
While this harness works well for full-stack web development, open questions remain:
* **Multi-Agent Architecture:** Would specialized agents (e.g., dedicated QA, testing, or code cleanup agents) perform better than a single general-purpose coding agent?
* **Generalization:** Can these practices be applied to other complex, long-running agentic tasks, such as scientific research or financial modeling?