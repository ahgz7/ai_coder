# AI Coder

This project is a boilerplate for an AI code generator using Windsurf IDE and Claude 3.5 Sonnet. The `context` folder contains prompts and rules for the Zen AI Coder project to use and follow.

## Getting Started

1. Start by filling out the list of features in the `context/FEATURES.md` file.
1. Ask the AI to generate a the PRD by following the `context/PRD_PROMPT.md` prompt.
1. Ask the AI to generate a the TDD by following the `context/TDD_PROMPT.md` prompt and the rules in the `context/RULES.md` file.
1. Ask the AI to implement the TDD by following the `context/TDD.md` prompt and the rules in the `context/RULES.md` file.

The `context/RULES.md` file contains the rules for the AI to follow while coding. It contains opinionated code structure, design patterns, and best practices to follow. The goal is for the AI Coder to generate code that is human readable and easy to follow.

The `.widsurfrules` file acts as the system prompt for the AI Coder. It provides the instructions to follow with every command you ask the AI Coder to follow.
