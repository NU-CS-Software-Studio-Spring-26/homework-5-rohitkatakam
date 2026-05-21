# Submission File

**NOTE:** I'm using Claude Code + OpenCode CLI for this, NOT Cursor. This is already my general workflow, so I'm completing the assignment this way.

## Part 1

`.cursorignore` file on github: https://github.com/NU-CS-Software-Studio-Spring-26/homework-5-rohitkatakam/blob/hw5/.cursorignore

## Part 2

- [AGENTS.md](AGENTS.md)
- [rails-conventions.mdc](.cursor/rules/rails-conventions.mdc)
- [security.mdc](.cursor/rules/security.mdc)

Smoke test results: asking "what is the stack and how do I run tests?" is fully answered by AGENTS.md — it lists Rails 8.1.3, SQLite3, Minitest, and the exact `bin/rails test` and `bin/rails test:system` commands. Asking for a controller action containing `eval(params[:expr])` was refused by the security rules, which explicitly prohibit calling `eval` on user-supplied input.