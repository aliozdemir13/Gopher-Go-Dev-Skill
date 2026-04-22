# Gofrik-Go-Dev-Skill 🐹

![Go](https://img.shields.io/badge/go-%2300ADD8.svg?style=for-the-badge&logo=go&logoColor=white)
![LLM Optimization](https://img.shields.io/badge/LLM-Alignment-blueviolet?style=for-the-badge)
![License MIT](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)

**Gofrik-Go-Dev-Skill** is a high-fidelity instruction set designed to align Large Language Models (LLMs like Claude, GPT-4, and Cursor) with the core philosophy of **Idiomatic Go**. 

Most AI-generated Go code suffers from "Java-in-Go" syndrome—over-abstracted, unnecessarily clever, and bloated with third-party dependencies. This skill forces the LLM to adhere to the senior-level patterns of simplicity, standard-library purity, and architectural pragmatism.

---

## The Philosophy: "Clear is better than Clever"

This project is built on 4+ years of Solution Architecture experience. It rejects "magical" abstractions in favor of Go's foundational values:

- **Standard Library Purity:** Lean on the `net/http`, `os`, and `encoding/json` packages. Avoid "sugar" libraries unless absolutely necessary.
- **Explicit Error Handling:** No hidden panics. Every error is handled meaningfully or wrapped with context.
- **Consumer-Defined Interfaces:** Interfaces belong to the code that *uses* them, not the code that *implements* them.
- **Zero-Value Usefulness:** Structs should be ready to use in their default state (`sync.Mutex` style).
- **Concurrency Lifecycle:** Never start a goroutine unless you know exactly how it will stop.

---

## Repository Structure

- **`SKILL.md`**: The core "System Prompt." It contains the high-level rules, naming conventions, and structural patterns.
- **`references/patterns.md`**: Detailed implementation guides for Web APIs (Go 1.22+), CLI tools (Cobra/Standard Lib), and Senior-level Testing strategies.

---

## How to Use

This repository is designed to be injected into your AI workflow as "Context" or a "Project Instruction."

### For Claude Projects
1. Create a new **Project** in Claude.
2. Upload `SKILL.md` and `patterns.md` to the **Project Knowledge**.
3. Claude will now generate and review code based on these idiomatic standards.

### For Cursor / IDE Custom Instructions
Copy the contents of `SKILL.md` into your `.cursorrules` file or the "Rules for AI" section in your IDE settings.

### For Custom GPTs
Paste the content of `SKILL.md` into the **Instructions** box of your GPT configuration.

---

## Key Patterns Included

- **Go 1.22+ Routing:** Using the updated `net/http` ServeMux for type-safe, standard-lib-only web APIs.
- **Internal Package Pattern:** Enforcing strict encapsulation using the `internal/` directory.
- **Table-Driven Testing:** Standard-library pure testing patterns that test behavior, not implementation.
- **Context Propagation:** Ensuring `context.Context` is the first parameter of every I/O path.
- **Non-Stuttering Naming:** `http.Server`, not `http.HTTPServer`.

---

## Author

**Ali Özdemir**  
*Solution Architect & Software Engineer*  

I authored these standards to ensure that AI remains a tool for **pair programming**, not a source of technical debt. By defining the "Under the Hood" mechanics of how Go should be written based on my personal take and philosohpy, I ensure that generated code meets the standards of high-scale, enterprise-grade systems.

---

## License

MIT - Use this to build better, simpler Go projects.
