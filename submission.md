# Submission File

**NOTE:** I'm using Claude Code + OpenCode CLI for this, NOT Cursor. This is already my general workflow, so I'm completing the assignment this way.

## Part 1

`.cursorignore` file on github: https://github.com/NU-CS-Software-Studio-Spring-26/homework-5-rohitkatakam/blob/hw5/.cursorignore

## Part 2

- [AGENTS.md](AGENTS.md)
- [rails-conventions.mdc](.cursor/rules/rails-conventions.mdc)
- [security.mdc](.cursor/rules/security.mdc)

Smoke test results: asking "what is the stack and how do I run tests?" is fully answered by AGENTS.md — it lists Rails 8.1.3, SQLite3, Minitest, and the exact `bin/rails test` and `bin/rails test:system` commands. Asking for a controller action containing `eval(params[:expr])` was refused by the security rules, which explicitly prohibit calling `eval` on user-supplied input.

## Part 3

### Ask Mode

**Prompt used:** "Where in this codebase is the todos index filtered/fetched? Cite the exact files and line numbers. Do not propose changes."

**Findings:**

- `app/controllers/todos_controller.rb`, line 6 — `@todos = Todo.all` is the fetch; there is no filtering, ordering, or scope applied.
- `app/models/todo.rb`, lines 1–2 — `Todo < ApplicationRecord` with no scopes, validations, or callbacks defined; the model adds nothing to the query.
- `app/views/todos/index.html.erb`, line 8 — `@todos.each do |todo|` iterates the full unfiltered collection and renders each record via `render todo` (the `_todo.html.erb` partial).

I opened each of the three files above and confirmed the cited line numbers are accurate.

### Plan Mode

**Prompt used:** "Only the user who created a todo can mark it done; other authenticated users can see it but cannot toggle it."

**Full numbered plan:**

1. **Generate authentication scaffold** (`bin/rails generate authentication`)
   - Creates `app/models/user.rb` (has_secure_password, email), `app/controllers/concerns/authentication.rb` (Current.user, require_authentication), `app/controllers/sessions_controller.rb`, `app/views/sessions/new.html.erb`, and two migrations (create_users, create_sessions).

2. **Single migration: add `done` + `user_id` to todos** (`bin/rails generate migration AddDoneAndUserToTodos done:boolean user:references`)
   - `done`: `null: false, default: false`
   - `user_id`: foreign key to `users`, `null: false`
   - Use `change` method to keep migration reversible.
   - File: `db/migrate/..._add_done_and_user_to_todos.rb`

3. **Update Todo model** (`app/models/todo.rb`)
   - Add `belongs_to :user`.

4. **Update TodosController** (`app/controllers/todos_controller.rb`)
   - Add `before_action :require_authentication` (from the authentication concern).
   - `index`: keep `Todo.all` — all authenticated users can see every todo.
   - `create`: change `Todo.new(todo_params)` to `current_user.todos.build(todo_params)`.
   - Add private `toggle_done` action: find todo, check `current_user == @todo.user`, flip `done`, save; return 403 if not owner.
   - Do NOT permit `done` in `todo_params` — done is only reachable through `toggle_done`.

5. **Add route for toggle_done** (`config/routes.rb`)
   - `resources :todos do; member { patch :toggle_done }; end`

6. **Update views** (`app/views/todos/_todo.html.erb`, `app/views/todos/show.html.erb`)
   - Render the toggle button only when `current_user == todo.user`.
   - Show done status (strikethrough or checkmark) to all authenticated users.

7. **Add tests** (`test/models/todo_test.rb`, `test/controllers/todos_controller_test.rb`)
   - Model: `done` defaults to `false`; todo requires a user.
   - Controller: unauthenticated requests redirect to login.
   - Controller: owner can call `toggle_done` and `done` flips.
   - Controller: non-owner `PATCH toggle_done` returns 403.
   - Controller: `create` sets `user_id` to current user.

**Self-review edits:**

- **Merged two migrations into one (Step 2).** The original draft had separate migrations for `done` and `user_id`. Merging them is atomic — no window where the schema is half-applied and easier to roll back as a unit.
- **Removed `done` from `todo_params` strong parameters.** Initially `done` was permitted in `todo_params` with an inline owner check inside `update`. Routing it exclusively through `toggle_done` puts the authorization in exactly one place and makes it impossible to bypass via a regular `update` request.

### Agent Mode

**Prompt acted on:** "Implement only step 2 from the plan (done column only): add a reversible migration that adds a `done` boolean with `null: false, default: false` to the todos table."

Step 2 originally combined `done` + `user_id`, but `user_id` requires the users table from Step 1. The `done` column has no dependencies — one migration file, no Ruby or ERB changes — making it the most self-contained piece.

**File created:** `db/migrate/20260521000000_add_done_to_todos.rb`

**Commit:** `4673373` — "Add done boolean column to todos as schema foundation for completion tracking"
(Full URL: GitHub repo URL + `/commit/4673373`)

### Bad to Good Prompt Rewrite

**The rough edge found:** `app/models/todo.rb` has no validations, so submitting a blank description silently creates a todo with an empty string. The form partial (`app/views/todos/_form.html.erb` lines 2–12) already has a full error-display block that is dead code — it never renders because no model errors are ever generated.

---

**Bad version:**
"fix the bug in todos"

---

**Good version:**

1. **Context:** `app/models/todo.rb` (currently `class Todo < ApplicationRecord; end` with no validations) and `app/views/todos/_form.html.erb` (has an error-display block at lines 2–12 that is never triggered).

2. **Task:** Add `validates :description, presence: true` to `app/models/todo.rb` so that submitting a blank description fails validation and the existing error UI activates.

3. **Expected vs actual:** Expected — submitting the new/edit form with a blank description re-renders the form and displays "Description can't be blank" in the red error block. Actual — the form saves successfully and redirects to the show page with no feedback.

4. **Constraints:** Touch only `app/models/todo.rb`. Do not add gems, do not change the form partial or controller — the error-display infrastructure and the `format.html { render :new, status: :unprocessable_content }` branch in the controller already exist and just need the validation to trigger them.

5. **Done when:** `bin/rails test test/controllers/todos_controller_test.rb` passes with a new test asserting that `POST /todos` with `description: ""` returns HTTP 422 and re-renders the form; and manually submitting a blank form in the browser shows the red error message.