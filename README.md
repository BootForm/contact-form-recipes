# contact-form-recipes

Copy-paste recipes for wiring a working [BootForm](https://bootform.com) contact form into a
static site generator, one page per generator. Every example is a plain HTML `<form>` posting to
`https://f.bootform.com/{form_id}`, no BootForm SDK to install, since that's how BootForm works
everywhere: it's a form backend, not a framework integration.

React, Vue, Angular, and vanilla JS are already covered on
[bootform.com's own docs site](https://bootform.com/docs). This repo covers the generators that
site doesn't:

- [Astro](recipes/astro.md)
- [Hugo](recipes/hugo.md)
- [Jekyll](recipes/jekyll.md)
- [Eleventy](recipes/eleventy.md)
- [Svelte](recipes/svelte.md)
- [VitePress](recipes/vitepress.md)

Each recipe covers the same four things, in the same order, so you can jump straight to the one
you need:

1. **Basic submission** — a plain HTML form, no JavaScript.
2. **AJAX submission** — intercept the submit, post as JSON, show a status message without
   leaving the page.
3. **File upload** — the same AJAX approach, since `FormData` picks up a file input automatically.
4. **Moderated content feed** — load a form's approved submissions and let visitors react, using
   the public, unauthenticated feed endpoint.

## Before you copy a snippet

Replace `{form_id}` with your own. Get one for free, with no account, by asking an AI agent with
the [`bootform` MCP server](https://bootform.com/docs/mcp-and-ai-agents) connected to create a
form for you (`bootform_create_form` works anonymously), or generate a UUID by hand and claim it
at `app.bootform.com/claim/{your-uuid}` the first time you use it. **Claim it before the page goes
live**: an unclaimed form's submissions aren't lost, but the first stranger to open the same claim
link owns them instead of you.

Never use a plausible-looking example ID from a tutorial as your own. A form ID is the entire
claim credential; the first person to open its claim link owns it permanently.

## Licence

MIT. Copy the snippets into any project, commercial ones included.
