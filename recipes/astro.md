# Astro

Every example on this page is a self-contained `.astro` component. Astro components render on
the server with no client-side reactivity by default, so anything past the basic submission uses
a plain `<script>` tag, not a UI framework or Astro island.

## Basic submission

The plain HTML case works unchanged in an Astro component's template, no script needed at all:

```astro
---
---
<form action="https://f.bootform.com/{form_id}" method="POST">
  <input name="email" type="email" required />
  <textarea name="message"></textarea>
  <button type="submit">Send</button>
</form>
```

## AJAX submission

Intercept the submit in a plain `<script>` tag, post as JSON, and swap the button/status text by
hand instead of navigating away. Astro processes and bundles this script for you; it does not
need `client:load` or any other hydration directive, since it isn't a UI framework component:

```astro
---
---
<form id="contact-form" action="https://f.bootform.com/{form_id}" method="POST">
  <input name="email" type="email" required />
  <textarea name="message"></textarea>
  <button type="submit">Send</button>
  <p class="status" hidden></p>
</form>

<script>
  const form = document.querySelector<HTMLFormElement>('#contact-form')!
  const button = form.querySelector('button')!
  const status = form.querySelector('.status')!

  form.addEventListener('submit', async (event) => {
    event.preventDefault()
    button.setAttribute('disabled', '')
    button.textContent = 'Sending…'

    const response = await fetch(form.action, {
      method: 'POST',
      headers: { Accept: 'application/json' },
      body: new URLSearchParams(new FormData(form)), // see the note below the code
    })
    const result = await response.json()

    status.removeAttribute('hidden')
    if (result.ok) {
      form.reset()
      status.textContent = 'Thanks - your message was sent.'
    } else {
      status.setAttribute('role', 'alert')
      status.textContent = result.error
    }
    button.removeAttribute('disabled')
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

`FormData` picks up a file input the same way as text fields, nothing Astro-specific needed
beyond the script above:

```astro
---
---
<form id="contact-form" action="https://f.bootform.com/{form_id}" method="POST">
  <input name="email" type="email" required />
  <input name="resume" type="file" required />
  <button type="submit">Send</button>
</form>

<script>
  const form = document.querySelector<HTMLFormElement>('#contact-form')!
  const button = form.querySelector('button')!

  form.addEventListener('submit', async (event) => {
    event.preventDefault()
    button.setAttribute('disabled', '')

    const response = await fetch(form.action, {
      method: 'POST',
      headers: { Accept: 'application/json' },
      body: new FormData(form), // includes the selected file automatically
    })
    const result = await response.json()
    button.removeAttribute('disabled')
    button.textContent = result.ok ? 'Sent' : 'Something went wrong'
  })
</script>
```

A file upload requires the form to be claimed first (`app.bootform.com/claim/{form_id}`), a
`422 FORM_NOT_CLAIMED` otherwise.

## Moderated content feed

Load a form's approved submissions on page load, and let visitors react. The feed endpoint is
public and unauthenticated, no API key involved:

```astro
---
const { formId } = Astro.props
---
<ul id="feed"></ul>

<script define:vars={{ formId }}>
  const list = document.querySelector('#feed')

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

`Astro.props` here assumes this is a component receiving `formId` as a prop from its parent page;
pass a literal string instead if this is the only form on the site.
