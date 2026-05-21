# AGENTS.md

## Stack
- **Framework:** Rails 8.1.3
- **Database:** SQLite3 (stored in `storage/`), schema in `db/schema.rb`
- **Frontend:** Turbo (turbo-rails) + Stimulus (stimulus-rails) via importmap-rails; Propshaft for assets
- **Views:** ERB templates + jbuilder for JSON responses
- **Tests:** Minitest (unit/controller via `test/`), Capybara + Selenium for system tests

## Commands
- **Setup:** `bin/setup` — installs gems, prepares database, clears logs/tmp
- **Run:** `bin/dev` (development server) or `bin/rails server`
- **Test:** `bin/rails test` (unit + controller); `bin/rails test:system` (browser system tests)
- **Lint:** `bin/rubocop` (rubocop-rails-omakase style)
- **Security scan:** `bin/brakeman`

## Conventions
- RESTful resources only; routes defined with `resources :todos` in `config/routes.rb`
- Strong parameters use `params.expect(model: [:field])` — the Rails 8 API, not `params.require.permit`
- Controllers respond to both HTML and JSON; JSON views live in `.json.jbuilder` files
- Shared HTML partials go in `app/views/<resource>/_partial.html.erb` (e.g., `_form`, `_todo`)
- JavaScript behavior belongs in Stimulus controllers under `app/javascript/controllers/`

## Don'ts
- Do not access params directly with `params[:key]` — always use `params.expect` for strong parameters
- Do not write inline JavaScript or add `.js` files outside the importmap/Stimulus controller pattern
- Do not use `html_safe` or `raw` on any user-supplied or database-sourced string
- Do not disable CSRF protection or mass assignment protection
- Do not add gems without running `bundle install` to update `Gemfile.lock`
