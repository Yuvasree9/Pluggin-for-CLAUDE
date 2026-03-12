# AI Architect Plugin

A comprehensive, production-grade Claude Code plugin for the full software engineering lifecycle. This plugin provides capabilities for designing, implementing, testing, reviewing, and fixing production-grade systems using specialized AI subagents and domain-specific skills.

---

## 🚀 Features & Commands

The plugin exposes several commands to assist throughout the development lifecycle:

- **`/design-system`**: Generates a deterministic, production-grade system design document based on requirements.
- **`/review-architecture`**: Analyzes and reviews architectural designs against established patterns and best practices.
- **`/implement`**: Translates system designs and specifications into working code using predefined implementation patterns.
- **`/review-code`**: Conducts thorough code reviews enforcing strict code review standards.
- **`/generate-tests`**: Automatically generates comprehensive test suites (unit, integration) based on the testing playbook.
- **`/fix`**: Identifies, debugs, and patches code issues or vulnerabilities.

---

## 🧠 Skills & Playbooks

The plugin leverages a rich set of embedded skills and playbooks to ensure high-quality outputs:

- **Architecture Patterns** (`skills/architecture-patterns.md`): Core engineering rules and architectural patterns for distributed systems.
- **Implementation Patterns** (`skills/implementation-patterns.md`): Best practices and coding standards for implementing scalable applications.
- **Code Review Standards** (`skills/code-review-standards.md`): Strict guidelines for reviewing code, ensuring maintainability and security.
- **Testing Playbook** (`skills/testing-playbook.md`): Strategies and patterns for generating effective tests.
- **Stack-Specific Playbook** (`skills/springboot-flutter-aws-playbook.md`): Directives tailored specifically for Spring Boot (Backend), Flutter (Frontend), and AWS (Infrastructure) environments.

---

## 🤖 Specialized Subagents

This plugin utilizes specialized subagents, each with a distinct persona and focus area, to handle different aspects of the engineering lifecycle:

- **System Designer** & **Architect**: Focus on high-level architecture, system design, and component interaction.
- **Implementer**: Focuses on writing clean, efficient, and pattern-compliant code.
- **Tester** & **Evaluator**: Focus on writing tests, ensuring code coverage, and validating functionality.
- **Code Reviewer**: Audits code for quality, performance, and security issues.
- **Fixer**: Diagnoses and resolves bugs, compilation errors, and failing tests.
- **DevOps**: Manages infrastructure, CI/CD pipelines, and deployment configurations.
- **Documenter**: Ensures code and systems are well-documented.
- **Researcher**: Investigates new technologies, tools, and best practices.

---

## 🛠️ Installation (Local)

1. Clone this repository:
   ```bash
   git clone <your-repo-url>
   cd My-Pluggin-for-CLAUDE
   ```
2. Open Claude Code and install the plugin from the local path. Ensure you are at the root of the project folder:
   ```bash
   claude plugins install .
   ```

---

## 📖 Usage Example

In Claude Code, invoke any of the available commands:

```bash
/design-system I need an event-driven architecture for a real-time chat application using Spring Boot and AWS.
```

```bash
/implement Create the core generic repository interface using the implementation-patterns playbook.
```

---

## 📁 Repository Structure

```text
My-Pluggin-for-CLAUDE/
├── .claude-plugin/            # Plugin configuration (plugin.json, marketplace.json)
├── commands/                  # Command definitions (.md files)
├── skills/                    # Engineering rules, standards, and playbooks
├── subagents/                 # Specialized AI subagent personas
└── templates/                 # Structural templates for standardized outputs
```

---

## 🏷️ Version

**v0.2.0**
