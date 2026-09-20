# Svelte

Every example on this page is a self-contained Svelte component using `fetch`, no BootForm SDK
to install.

## Basic submission

The plain HTML case works unchanged in a Svelte template, no reactive state needed at all:

```svelte
<form action="https://f.bootform.com/{form_id}" method="POST">
  <input name="email" type="email" required />
  <textarea name="message"></textarea>
  <button type="submit">Send</button>
</form>
```

## AJAX submission

Intercept the submit, post as JSON, and track status with a reactive `let` instead of navigating
away:

```svelte
<script>
  let status = 'idle' // 'idle' | 'sending' | 'sent' | 'error'
  let error = null

  async function handleSubmit(event) {
    status = 'sending'

    const response = await fetch(event.target.action, {
      method: 'POST',
      headers: { Accept: 'application/json' },
      body: new FormData(event.target),
    })
    const result = await response.json()

    if (result.ok) {
      event.target.reset()
      status = 'sent'
    } else {
      error = result.error
      status = 'error'
    }
  }
</script>

<form action="https://f.bootform.com/{form_id}" method="POST" on:submit|preventDefault={handleSubmit}>
  <input name="email" type="email" required />
  <textarea name="message"></textarea>
  <button type="submit" disabled={status === 'sending'}>
    {status === 'sending' ? 'Sending…' : 'Send'}
  </button>
  {#if status === 'sent'}<p>Thanks - your message was sent.</p>{/if}
  {#if status === 'error'}<p role="alert">{error}</p>{/if}
</form>
```

## File upload

`FormData` picks up a file input the same way as text fields, nothing Svelte-specific needed
beyond the state you're already tracking above:

```svelte
<script>
  let status = 'idle'

  async function handleSubmit(event) {
    status = 'sending'

    const response = await fetch(event.target.action, {
      method: 'POST',
      headers: { Accept: 'application/json' },
      body: new FormData(event.target), // includes the selected file automatically
    })
    const result = await response.json()
    status = result.ok ? 'sent' : 'error'
  }
</script>

<form action="https://f.bootform.com/{form_id}" method="POST" on:submit|preventDefault={handleSubmit}>
  <input name="email" type="email" required />
  <input name="resume" type="file" required />
  <button type="submit" disabled={status === 'sending'}>Send</button>
</form>
```

A file upload requires the form to be claimed first (`app.bootform.com/claim/{form_id}`), a
`422 FORM_NOT_CLAIMED` otherwise.

## Moderated content feed

Load a form's approved submissions on mount, and let visitors react. The feed endpoint is public
and unauthenticated, no API key involved:

```svelte
<script>
  export let formId

  let items = []

  async function loadFeed() {
    const response = await fetch(`https://f.bootform.com/forms/${formId}/feed?per_page=20`)
    const data = await response.json()
    items = data.items
  }

  async function react(submissionId, type) {
    await fetch(`https://f.bootform.com/forms/${formId}/feed/${submissionId}/react`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ type }),
    })
    loadFeed() // refetch to show the updated count
  }

  loadFeed()
</script>

<ul>
  {#each items as item (item.id)}
    <li>
      <p>{item.fields.message}</p>
      <button on:click={() => react(item.id, 'up')}>👍 {item.reactions.up}</button>
      <button on:click={() => react(item.id, 'down')}>👎 {item.reactions.down}</button>
    </li>
  {/each}
</ul>
```

`loadFeed()` called directly in `<script>` (rather than inside `onMount`) runs once when the
component is created, which is fine for a component that's only ever rendered client-side; use
`onMount(loadFeed)` from `svelte` instead if this component might render during SSR (SvelteKit).
