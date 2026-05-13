# Reveng

A Claude Code plugin and CLI for reverse-engineering legacy applications.

## Context

Organisations often maintain large estates of legacy applications built across decades using a variety of technologies. Reveng helps engineers understand, document, and modernise these systems regardless of language, framework, or database stack.

## Conventions

- Use British English in all generated documentation
- Treat all legacy code as potentially undocumented — never assume documentation exists
- Do not assume any particular technology stack — agents and skills should adapt to whatever is present in the target codebase
- When creating a new skill, add an entry to the Skills table in README.md
- When creating a new agent, add an entry to the Agents table in README.md
