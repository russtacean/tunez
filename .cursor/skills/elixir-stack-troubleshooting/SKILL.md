---
name: elixir-stack-troubleshooting
description: Guides development and troubleshooting in this Elixir project using Phoenix LiveView, Ash (AshPhoenix, AshPostgres), and Tailwind. Use when adding a new resource, new LiveView, or new feature; when debugging errors or fixing broken forms or reads; or when the user asks about the stack, conventions, data flow, or where to look for issues in this codebase.
---

# Elixir Stack Development and Troubleshooting

## When to use

- Adding a new Ash resource, LiveView (index/show/form), or feature.
- Debugging errors, fixing broken forms or reads, or tracing where a failure occurs.
- User asks about the stack, conventions, data flow, or where to look in this codebase.

For current request and form flows, see [dataflow.md](dataflow.md). Update that diagram when adding new domains, new LiveView entry points, or significant request paths.

---

## Stack overview

| Layer | Tech | Location |
|-------|------|----------|
| App | Elixir 1.19, Phoenix 1.8, LiveView | `lib/tunez_web/`, `lib/tunez/` |
| Domain / data | Ash 3, AshPhoenix, AshPostgres | Domain: `lib/tunez/music.ex`; resources: `lib/tunez/music/*.ex` |
| UI | Tailwind, heroicons | `assets/css/app.css`, `assets/css/theme.css`; components: `lib/tunez_web/components/core_components.ex` |
| Routes | Phoenix Router | `lib/tunez_web/router.ex` |
| LiveViews | By context | `lib/tunez_web/live/artists/`, `lib/tunez_web/live/albums/` |

No separate JSON API for the app UI; all UI goes through LiveView.

---

## Commands

Use these when building features or troubleshooting. Project aliases are in `mix.exs`.

| Purpose | Command |
|---------|---------|
| **Setup** (first time or after pull) | `mix setup` — deps, ash.setup, assets, seeds |
| **Run app** | `mix phx.server` or `iex -S mix phx.server` |
| **Ash** | `mix ash.setup` — ensure Ash/Postgres state; run before tests. `mix ash.codegen` — generate form helpers after changing domain. `mix ash_postgres.generate_migrations --name <name>` — generate migration after adding/changing resources (e.g. `--name add_tracks`) |
| **Ecto** | `mix ecto.create`, `mix ecto.migrate`, `mix ecto.rollback`, `mix ecto.reset`, `mix ecto.setup` |
| **Routes** | `mix phx.routes` — list routes |
| **Tests** | `mix test` — project alias runs `ash.setup --quiet` first |
| **Format** | `mix format` |
| **Seeds** | `mix seed` (project alias) or `mix run priv/repo/seeds.exs` |
| **Assets** | `mix assets.build` — compile Tailwind/JS; `mix assets.deploy` — minify + phx.digest |

After adding or changing an Ash resource: run `mix ash_postgres.generate_migrations --name <descriptive_name>`, then `mix ecto.migrate`. After changing domain interface (defines): run `mix ash.codegen`.

---

## Development patterns

### Adding a new Ash resource

1. **Resource module** in `lib/tunez/music/` (pattern: `lib/tunez/music/artist.ex`, `lib/tunez/music/album.ex`):
   - `use Ash.Resource`, `domain: Tunez.Music`, `data_layer: AshPostgres.DataLayer`
   - `postgres do table "name" repo Tunez.Repo end`
   - `attributes do` (e.g. `uuid_primary_key :id`, `attribute :name, :string`, timestamps)
   - `relationships do` (e.g. `belongs_to`, `has_many`)
   - `actions do` — either `defaults [:create, :read, :update, :destroy]` and `default_accept [...]` or explicit `create :create do accept [...] end`, etc.

2. **Domain** in `lib/tunez/music.ex`:
   - Add `resource YourModule` and `define :create_*, action: :create`, `define :read_*`, `define :get_*_by_id, action: :read, get_by: :id`, `define :update_*`, `define :destroy_*` as needed.
   - Run `mix ash.codegen` for form helpers (`form_to_create_*`, `form_to_update_*`) if the project uses codegen.

3. **Migration**: Run `mix ash_postgres.generate_migrations --name add_<resource>` (e.g. `add_tracks`), then `mix ecto.migrate`. See [Commands](#commands) above.

### Adding a new LiveView

- **Index** (pattern: `lib/tunez_web/live/artists/index_live.ex`): Module under `lib/tunez_web/live/<context>/index_live.ex`. In `mount`, assign `page_title`. In `handle_params`, load list via domain (e.g. `Tunez.Music.read_artists!()`) and assign. Render list; use `~p"/path"` for links.

- **Show** (pattern: `lib/tunez_web/live/artists/show_live.ex`): Load record in `mount` with `Tunez.Music.get_*_by_id!(id)`; assign record and `page_title`. Use `Layouts.app`, `.header`, and `~p"/path"` for edit/new links.

- **Form new/edit** (pattern: `lib/tunez_web/live/artists/form_live.ex`, `lib/tunez_web/live/albums/form_live.ex`): Two `mount` clauses — one with `%{"id" => id}` for edit (load record, `form_to_update_*(record)`), one without for new (`form_to_create_*()`). Assign `to_form(form)` and `page_title`. Use `.simple_form` with `phx-change="validate"` and `phx-submit="save"`. Validate: `update(socket, :form, fn form -> AshPhoenix.Form.validate(form, form_data) end)`. Save: `AshPhoenix.Form.submit(socket.assigns.form, params: form_data)`; on `{:ok, record}` use `put_flash` + `push_navigate`; on `{:error, form}` assign form and flash error.

### Adding routes

In `lib/tunez_web/router.ex`, under `scope "/", TunezWeb` and `pipe_through :browser`, add e.g. `live "/path", ModuleLive` or `live "/path/:id/edit", ModuleLive, :edit`. Use `~p"/path"` in LiveViews for links.

### UI and Tailwind

- Use `Layouts.app` and core components (`.header`, `.button_link`, `.input`, `.simple_form`, etc.) from `lib/tunez_web/components/core_components.ex`.
- Icons: `.icon name="hero-*"` (heroicons).
- Tailwind: use project theme and existing classes; phx-* variants are in `assets/css/app.css`.

### After adding new flows

Update the dataflow diagram in [dataflow.md](dataflow.md) when adding a new domain, new LiveView entry point, or significantly new request path.

---

## Troubleshooting

### Ash / domain

- **Form helpers missing**: `form_to_create_*` / `form_to_update_*` come from the domain’s AshPhoenix extension. Ensure `lib/tunez/music.ex` has the right `define` for the action, then run `mix ash.codegen`.
- **Table or column missing**: After adding/changing a resource, run `mix ash_postgres.generate_migrations --name <name>` and `mix ecto.migrate`. See [Commands](#commands).
- **Action not defined / not in interface**: Check `resources do ... define :action_name, action: :create` (or `:read`, etc.) and the resource’s `actions do ... create :create do ... end`.
- **Validation / allow_nil?**: Changeset errors often come from resource attributes or relationships; check `accept` and `allow_nil?` in the resource (e.g. `lib/tunez/music/album.ex`).
- **Relationships**: Loading associations may require `load!` or a read action that loads the relationship; missing-field errors often point to resource/action definition.

### AshPhoenix forms

- **Form source**: LiveView must use a form from `Tunez.Music.form_to_*` so `AshPhoenix.Form.validate`/`submit` work. Wrapping with `to_form(form)` is required (see `lib/tunez_web/live/artists/form_live.ex`).
- **Validate**: `update(socket, :form, fn form -> AshPhoenix.Form.validate(form, form_data) end)`; keep the result in assigns.
- **Submit**: `AshPhoenix.Form.submit(socket.assigns.form, params: form_data)` returns `{:ok, record}` or `{:error, form}`; on error, assign the returned form (with `to_form(form)` if needed) and set flash.

### LiveView

- **Data loading**: Use `handle_params` for URL-driven data (e.g. list in `lib/tunez_web/live/artists/index_live.ex`) so it re-runs on navigation; use `mount` for one-time setup.
- **Paths**: Use `~p"/artists/#{id}"` etc.; routes in `lib/tunez_web/router.ex`.
- **Flash**: `put_flash(:info, ...)` / `put_flash(:error, ...)`; success often uses `push_navigate(to: ...)`.

### Tailwind

- **Custom variants**: phx-* loading states in `assets/css/app.css` (`phx-click-loading`, `phx-submit-loading`, etc.); theme in `assets/css/theme.css`.
- **Heroicons**: Use `.icon` / `hero-*` as configured (see `assets/css/app.css`).
