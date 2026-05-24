# Submission File

**NOTE:** I'm using Claude Code + OpenCode CLI for this, NOT Cursor. This is already my general workflow, so I'm completing the assignment this way.

## Part 1

`.cursorignore` on GitHub: https://github.com/NU-CS-Software-Studio-Spring-26/homework-5-rohitkatakam/blob/hw5/.cursorignore

## Part 2

- [AGENTS.md](AGENTS.md)
- [rails-conventions.mdc](.cursor/rules/rails-conventions.mdc)
- [security.mdc](.cursor/rules/security.mdc)

Smoke test: asking "what is the stack and how do I run tests?" was fully answered from AGENTS.md - it lists Rails 8.1.3, SQLite3, Minitest, and the exact `bin/rails test` / `bin/rails test:system` commands. Asking for a controller action with `eval(params[:expr])` got refused; the security rules explicitly block `eval` on user input.

## Part 3

### Ask mode

Prompt: "Where in this codebase is the todos index filtered/fetched? Cite the exact files and line numbers. Do not propose changes."

What came back:

- `app/controllers/todos_controller.rb`, line 6 - `@todos = Todo.all`, no filtering or ordering.
- `app/models/todo.rb`, lines 1-2 - `Todo < ApplicationRecord` with nothing else; the model adds nothing to the query.
- `app/views/todos/index.html.erb`, line 8 - `@todos.each do |todo|` iterates the full collection and renders via `render todo` (the `_todo.html.erb` partial).

I opened each file and confirmed the line numbers are right.

### Plan mode

Prompt: "Only the user who created a todo can mark it done; other authenticated users can see it but cannot toggle it."

Plan I got back:

1. Generate authentication scaffold (`bin/rails generate authentication`) - creates `app/models/user.rb`, `app/controllers/concerns/authentication.rb`, `app/controllers/sessions_controller.rb`, `app/views/sessions/new.html.erb`, and two migrations.

2. Single migration: add `done` + `user_id` to todos (`bin/rails generate migration AddDoneAndUserToTodos done:boolean user:references`). `done` gets `null: false, default: false`, `user_id` is a non-null foreign key. Use `change` to keep it reversible. File: `db/migrate/..._add_done_and_user_to_todos.rb`.

3. Update `app/models/todo.rb` - add `belongs_to :user`.

4. Update `app/controllers/todos_controller.rb` - add `before_action :require_authentication`. Keep `index` as `Todo.all` so everyone sees all todos. Change `create` to `current_user.todos.build(todo_params)`. Add a private `toggle_done` action that finds the todo, checks ownership, flips `done`, and returns 403 if the requester isn't the owner. Don't permit `done` in `todo_params`.

5. Add the route in `config/routes.rb`: `resources :todos do; member { patch :toggle_done }; end`.

6. Update `app/views/todos/_todo.html.erb` and `show.html.erb` - show the toggle only to the owner, but show done status (strikethrough or checkmark) to everyone.

7. Add tests in `test/models/todo_test.rb` and `test/controllers/todos_controller_test.rb`: `done` defaults to false, todo requires a user, unauthenticated requests redirect to login, owner can toggle and it flips, non-owner gets 403, `create` sets `user_id`.

My edits to the plan:

- Merged the two separate migrations (step 2) into one. The original had `done` and `user_id` as separate files, which leaves a window where the schema is half-applied. One migration is easier to roll back cleanly.
- Removed `done` from `todo_params`. The draft had it permitted there with an inline owner check inside `update`. Running it exclusively through `toggle_done` puts authorization in one place and makes it impossible to sneak past via a normal update request.

### Agent mode

Prompt: "Implement only step 2 from the plan (done column only): add a reversible migration that adds a `done` boolean with `null: false, default: false` to the todos table."

Step 2 originally combined `done` + `user_id`, but `user_id` requires the users table from step 1. The `done` column has no dependencies, so it's a single migration file with no Ruby or view changes - the most self-contained slice.

File created: `db/migrate/20260521000000_add_done_to_todos.rb`

Commit `4673373` - "Add done boolean column to todos as schema foundation for completion tracking"
https://github.com/NU-CS-Software-Studio-Spring-26/homework-5-rohitkatakam/commit/4673373

### Bad to good prompt rewrite

The rough edge: `app/models/todo.rb` has no validations, so a blank description silently saves. The form partial (`app/views/todos/_form.html.erb`, lines 2-12) already has an error-display block that's dead code - nothing ever produces a model error so it never fires.

Bad version:
> "fix the bug in todos"

Good version:

1. Context: `app/models/todo.rb` (currently `class Todo < ApplicationRecord; end`) and `app/views/todos/_form.html.erb` (error block at lines 2-12, never triggered).

2. Task: Add `validates :description, presence: true` to `app/models/todo.rb`.

3. Expected vs actual: a blank description should re-render the form with "Description can't be blank" in the red block. Right now it saves and redirects with no feedback.

4. Constraints: touch only `app/models/todo.rb`. No new gems, no changes to the form or controller - the `format.html { render :new, status: :unprocessable_content }` branch already exists and just needs a validation to trigger it.

5. Done when: `bin/rails test test/controllers/todos_controller_test.rb` passes with a new test asserting `POST /todos` with `description: ""` returns 422 and re-renders; and submitting a blank form in the browser shows the error.

## Part 4

### Turbo Streams

A Turbo Stream is a response that makes a targeted DOM change - swap this element, append that one, remove another - without loading a new page. It comes back with `Content-Type: text/vnd.turbo-stream.html` and contains one or more `<turbo-stream action="..." target="...">` tags. Turbo intercepts it, reads the action, and updates exactly that part of the DOM. The seven actions are `append`, `prepend`, `replace`, `update`, `remove`, `before`, and `after`.

On the Rails side, you add `format.turbo_stream` to a `respond_to` block and put the template at `app/views/<controller>/<action>.turbo_stream.erb`.

One thing I verified against the Turbo Streams handbook (turbo.hotwired.dev/handbook/streams): the difference between `replace` and `update`. `replace` swaps the target element entirely, wrapper tag included. `update` only changes the inner content. The AI explained this correctly, and I confirmed it by looking at `app/views/todos/toggle_priority.turbo_stream.erb` - it uses `turbo_stream.replace(dom_id(@todo), ...)`, which replaces the whole `<div id="todo_1">`, not just what's inside it.

Self-check:
- MIME type: `text/vnd.turbo-stream.html`
- View file for `TodosController#toggle_priority`: `app/views/todos/toggle_priority.turbo_stream.erb`

### Pull request

https://github.com/NU-CS-Software-Studio-Spring-26/nu-cs-software-studio-spring-26-homework-5-hw5/pull/1
