# Hugo

Hugo has no client-side reactivity of its own; every example past the basic submission is plain
JavaScript in a `<script>` tag inside the same template. The form's own markup is plain Go
template syntax, kept in a partial so it can be reused across pages.

## Basic submission

Put the form in `layouts/partials/contact-form.html` and include it wherever it's needed with
`{{ partial "contact-form.html" . }}`. Reading the form ID from `hugo.toml`'s `params` instead of
hardcoding it means changing it in one place, not on every page that embeds the partial:

```html
{{/* layouts/partials/contact-form.html */}}
<form action="https://f.bootform.com/{{ .Site.Params.bootformId }}" method="POST">
  <input name="email" type="email" required>
  <textarea name="message"></textarea>
  <button type="submit">Send</button>
</form>
```

```toml
# hugo.toml
[params]
  bootformId = "{form_id}"
```

## AJAX submission

Intercept the submit, post as JSON, and swap the button/status text by hand. This still lives in
the same partial; Hugo ships the `<script>` tag as-is with the rest of the page's HTML:

```html
{{/* layouts/partials/contact-form.html */}}
<form id="contact-form" action="https://f.bootform.com/{{ .Site.Params.bootformId }}" method="POST">
  <input name="email" type="email" required>
  <textarea name="message"></textarea>
  <button type="submit">Send</button>
  <p class="status" hidden></p>
</form>

<script>
  const form = document.getElementById('contact-form')
  const button = form.querySelector('button')
  const status = form.querySelector('.status')

  form.addEventListener('submit', async (event) => {
    event.preventDefault()
    button.disabled = true
    button.textContent = 'Sending…'

    const response = await fetch(form.action, {
      method: 'POST',
      headers: { Accept: 'application/json' },
      body: new URLSearchParams(new FormData(form)), // see the note below the code
    })
    const result = await response.json()

    status.hidden = false
    if (result.ok) {
      form.reset()
      status.textContent = 'Thanks - your message was sent.'
    } else {
      status.setAttribute('role', 'alert')
      status.textContent = result.error
    }
    button.disabled = false
    button.textContent = 'Send'
  })
</script>
```

A raw `new FormData(form)` here would work as a request body, but `fetch` sets the
request's `Content-Type` from the *type* of value you pass as `body`, not from whether the form
actually has a file field: `FormData` always sends `multipart/form-data`, unconditionally, even
with zero files in it. That matters specifically because BootForm rejects any
`multipart/form-data` submission to a form that hasn't been claimed yet (`422
FORM_NOT_CLAIMED`), regardless of whether a real file is attached. Wrapping it as `new
URLSearchParams(new FormData(form))` keeps the convenient "read every field automatically"
behavior while forcing `application/x-www-form-urlencoded` instead, which an unclaimed form
accepts fine. The next section, which genuinely has a file input, needs the real `FormData` and
genuinely does need the form claimed first.

## File upload

`FormData` picks up a file input the same way as text fields, nothing Hugo-specific needed beyond
the script above:

```html
{{/* layouts/partials/upload-form.html */}}
<form id="upload-form" action="https://f.bootform.com/{{ .Site.Params.bootformId }}" method="POST">
  <input name="email" type="email" required>
  <input name="resume" type="file" required>
  <button type="submit">Send</button>
</form>

<script>
  const form = document.getElementById('upload-form')
  const button = form.querySelector('button')

  form.addEventListener('submit', async (event) => {
    event.preventDefault()
    button.disabled = true

    const response = await fetch(form.action, {
      method: 'POST',
      headers: { Accept: 'application/json' },
      body: new FormData(form), // includes the selected file automatically
    })
    const result = await response.json()
    button.disabled = false
    button.textContent = result.ok ? 'Sent' : 'Something went wrong'
  })
</script>
```

A file upload requires the form to be claimed first (`app.bootform.com/claim/{form_id}`), a
`422 FORM_NOT_CLAIMED` otherwise.

## Moderated content feed

Load a form's approved submissions on page load, and let visitors react. The feed endpoint is
public and unauthenticated, no API key involved:

```html
{{/* layouts/partials/feed.html */}}
<ul id="feed"></ul>

<script>
  const formId = {{ .Site.Params.bootformId | jsonify }}
  const list = document.getElementById('feed')

  async function loadFeed() {
    const response = await fetch(`https://f.bootform.com/forms/${formId}/feed?per_page=20`)
    const { items } = await response.json()
    list.innerHTML = items
      .map(
        (item) => `
          <li data-id="${item.id}">
            <p>${item.fields.message}</p>
            <button data-type="up">👍 ${item.reactions.up}</button>
            <button data-type="down">👎 ${item.reactions.down}</button>
          </li>
        `,
      )
      .join('')
  }

  list.addEventListener('click', async (event) => {
    const button = event.target.closest('button[data-type]')
    if (!button) return
    const id = button.closest('li').dataset.id
    await fetch(`https://f.bootform.com/forms/${formId}/feed/${id}/react`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ type: button.dataset.type }),
    })
    loadFeed() // refetch to show the updated count
  })

  loadFeed()
</script>
```

`{{ .Site.Params.bootformId | jsonify }}` renders the Go-template value as a properly-quoted
JavaScript string literal; interpolating it as raw text (`"{{ .Site.Params.bootformId }}"`) works
too as long as the ID never contains a character that would break out of the quotes, but
`jsonify` is the safe default.
