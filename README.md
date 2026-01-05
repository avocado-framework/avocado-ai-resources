# 🥑 Avocado AI Resources

Operational frameworks, set plans, and configurations for AI-assisted engineering within the Avocado ecosystem.

## 🎯 Overview
This repository serves as the "Instruction Manual" for AI agents (like **Cursor**, **Windsurf**, or **Claude**) when they are working on Avocado codebases. It standardizes how AI understands our logic, follows our plans, and executes tasks.

## 📂 Current Resources

### 📋 Set Plans (`/plans`)
*Structured blueprints for the AI to follow step-by-step.*

* **[Your-Plan-Name].md**: (Current active plan) - Use this to guide the AI through specific coding workflows or feature implementations.

---

## 🛠 Future Roadmap
These modules will be implemented as the Avocado AI ecosystem scales:

* **`.cursorrules`**: Global rulesets to automate context and coding style in the Cursor editor.
* **Personas**: System prompts that define specific "Expert Engineer" roles for different Avocado modules.
* **Knowledge Base**: Markdown-based snippets to give AI "long-term memory" of Avocado’s internal APIs.

## 🚀 How to Use
1. **Reference the Plan**: In your AI chat (e.g., Cursor Composer or Claude), reference the plan file:
   > *"@plans/[your-plan-name].md follow this plan to help me implement the next steps."*
2. **Stay Consistent**: Ensure any new AI logic or rules are added here to keep a single source of truth for "Avocado-style" coding.

---

## 🏗 Setup
To use these resources in a local project:
```bash
git clone [https://github.com/](https://github.com/)[your-username]/avocado-ai-resources.git
