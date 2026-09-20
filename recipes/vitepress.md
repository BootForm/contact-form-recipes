# VitePress

Every example on this page goes directly inside a `.md` file's `<script setup>` block and Vue
template, the same as any other dynamic content in a VitePress page. No separate `.vue` file is
needed just for a form. See `github.com/BootForm/vitepress-marketing`,
`vitepress-portfolio`, and `vitepress-blog` for these patterns already wired into a full site.

## Basic submission

The plain HTML case works unchanged in the markdown body, no script needed at all:

```md
<form action="https://f.bootform.com/{form_id}" method="POST">
  <input name="email" type="email" required>
  <textarea name="message"></textarea>
  <button type="submit">Send</button>
</form>
```

`https://f.bootform.com/{form_id}` is an absolute, external URL, so it does not need VitePress's
own `withBase()` helper. `withBase()` only matters for a `src`/`href` pointing back into the site
itself, built dynamically rather than written as a literal markdown link; an external `action`
like this one is never subject to that.

## AJAX submission

Intercept the submit, post as JSON, and track status with `ref()` instead of navigating away:

```md
<script setup>
import { ref } from 'vue'

const status = ref('idle') // 'idle' | 'sending' | 'sent' | 'error'
const error = ref(null)

async function handleSubmit(event) {
  status.value = 'sending'

  const response = await fetch(event.target.action, {
    method: 'POST',
    headers: { Accept: 'application/json' },
    body: new FormData(event.target),
  })
  const result = await response.json()

  if (result.ok) {
    event.target.reset()
    status.value = 'sent'
  } else {
    error.value = result.error
    status.value = 'error'
  }
}
</script>

<form action="https://f.bootform.com/{form_id}" method="POST" @submit.prevent="handleSubmit">
  <input name="email" type="email" required>
  <textarea name="message"></textarea>
  <button type="submit" :disabled="status === 'sending'">
    {{ status === 'sending' ? 'Sending…' : 'Send' }}
  </button>
  <p v-if="status === 'sent'">Thanks - your message was sent.</p>
  <p v-if="status === 'error'" role="alert">{{ error }}</p>
</form>
```

## File upload

`FormData` picks up a file input the same way as text fields, nothing VitePress-specific needed
beyond the state you're already tracking above:

```md
<script setup>
import { ref } from 'vue'

const status = ref('idle')

async function handleSubmit(event) {
  status.value = 'sending'

  const response = await fetch(event.target.action, {
    method: 'POST',
    headers: { Accept: 'application/json' },
    body: new FormData(event.target), // includes the selected file automatically
  })
  const result = await response.json()
  status.value = result.ok ? 'sent' : 'error'
}
</script>

<form action="https://f.bootform.com/{form_id}" method="POST" @submit.prevent="handleSubmit">
  <input name="email" type="email" required>
  <input name="resume" type="file" required>
  <button type="submit" :disabled="status === 'sending'">Send</button>
</form>
```

A file upload requires the form to be claimed first (`app.bootform.com/claim/{form_id}`), a
`422 FORM_NOT_CLAIMED` otherwise.

## Moderated content feed

Load a form's approved submissions on mount, and let visitors react. The feed endpoint is public
and unauthenticated, no API key involved:

```md
<script setup>
import { onMounted, ref } from 'vue'

const formId = '{form_id}'
const items = ref([])

async function loadFeed() {
  const response = await fetch(`https://f.bootform.com/forms/${formId}/feed?per_page=20`)
  const { items: fetched } = await response.json()
  items.value = fetched
}

async function react(submissionId, type) {
  await fetch(`https://f.bootform.com/forms/${formId}/feed/${submissionId}/react`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ type }),
  })
  loadFeed() // refetch to show the updated count
}

onMounted(loadFeed)
</script>

<ul>
  <li v-for="item in items" :key="item.id">
    <p>{{ item.fields.message }}</p>
    <button @click="react(item.id, 'up')">👍 {{ item.reactions.up }}</button>
    <button @click="react(item.id, 'down')">👎 {{ item.reactions.down }}</button>
  </li>
</ul>
```
