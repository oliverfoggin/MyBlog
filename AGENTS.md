# Repository Guidelines

## Project Structure & Module Organization

This repository is a Zola static site configured by `zola.toml`. Keep site content and presentation in the standard Zola locations:

- `content/`: Markdown pages, sections, and posts. Use colocated page assets only when they belong to one page.
- `templates/`: Tera templates and layout partials.
- `sass/`: Sass stylesheets. `compile_sass = true` is enabled, so Zola compiles files from here during builds.
- `static/`: Static files copied directly into the built site, such as images, downloads, icons, and browser metadata.
- `themes/`: Third-party or local Zola themes, if added.

Do not commit generated output such as `public/` unless the deployment process explicitly requires it.

## Build, Test, and Development Commands

- `zola serve`: Start the local development server with live reload.
- `zola build`: Build the production site into `public/` using `base_url = "https://oliverfoggin.com"`.
- `zola check`: Validate pages, templates, internal links, and configuration without producing a full deployable build.

Run commands from the repository root. This project has no `package.json`, Makefile, or separate asset pipeline at present.

## Coding Style & Naming Conventions

Use concise, semantic Markdown and valid TOML/Tera syntax. Prefer lowercase, hyphenated filenames for content slugs, for example `content/posts/my-post.md`. Keep front matter explicit and consistent across similar content types.

Use two-space indentation in TOML and template blocks where indentation is needed. Keep Sass organized by responsibility, with descriptive class names and small partials once styles grow.

## Testing Guidelines

There is no dedicated automated test suite in this repository. Treat `zola check` as the required validation step before opening a pull request. For visual or layout changes, also run `zola serve` and inspect the affected pages locally in a browser.

When adding templates, verify empty-state behavior and pages with minimal front matter so new content does not break rendering.

## Commit & Pull Request Guidelines

This checkout does not include Git history, so no existing commit convention can be inferred. Use short, imperative commit messages such as `Add article template` or `Update site styles`.

Pull requests should include a brief description, the pages or templates changed, validation performed (`zola check`, local browser review), and screenshots for visible design changes. Link related issues when applicable.

## Agent-Specific Instructions

Keep edits scoped and avoid introducing a separate build system unless the repository already adopts one. Preserve Zola conventions and update this guide when project commands or structure change.
