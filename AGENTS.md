# Working in this repo

Copy-paste recipes for wiring a BootForm contact form into a static site generator. One Markdown
file per generator under `recipes/`, listed in the README. No code to build or run here: the
recipes are the product, so each one has to work when pasted into a real project.

## Every recipe has the same shape

Four sections, with these headings, in this order, so a reader can jump between generators:

1. `## Basic submission`: a plain HTML `<form>` that posts to `https://f.bootform.com/{form_id}`,
   no JavaScript.
2. `## AJAX submission`: intercept the submit, send `new FormData(form)` with `fetch` and the
   header `Accept: application/json`, then show the result on the page. The body is multipart,
   not JSON: the `Accept` header is what makes BootForm answer with JSON instead of a redirect.
3. `## File upload`: the same AJAX code with a file input, plus the note that files need a
   claimed form (`422 FORM_NOT_CLAIMED` otherwise).
4. `## Moderated content feed`: load a form's approved submissions from the public feed endpoint
   and let visitors react. The reaction request is the one place a recipe posts JSON.

Open with a short paragraph on what is specific to that generator (where files go, how it ships
a `<script>`, what its template syntax gets in the way of). Show the file path as a comment at the
top of each snippet. Start a new recipe by copying an existing one; `recipes/hugo.md` is the
usual starting point.

## The rules every snippet follows

- **`{form_id}` stays a literal placeholder.** Never a real-looking UUID, not even an example
  one: a form ID is the whole claim credential, and whoever claims a printed ID first receives
  everything sent to it.
- **Keep the honeypot field in every form**, exactly as the other recipes have it: `name="honeypot"`,
  positioned off screen with `style="position:absolute;left:-9999px"`, `tabindex="-1"` and
  `autocomplete="off"`. Not `hidden` and not `display:none`. In JSX the style becomes an object.
- **Test it in a real project before it goes in.** A fresh project on the generator's current
  version, every snippet in place, a build that passes, and the basic and AJAX forms sent once to
  a form ID you generated yourself (never committed). The PR says which version was tested. The
  Eleventy recipe once printed its Nunjucks comments as text on Markdown pages: only a real build
  shows that.
- **A new recipe gets a line in the README's list.**

## Writing style

- Second person, present tense, short sentences.
- **No em dashes or en dashes**, in prose or in snippet text. Use a comma, a colon, a full stop,
  or brackets. Don't use a spaced hyphen (` - `) as a stand-in either.
- No AI attribution in commits or pull requests, here or anywhere else in this organisation.
