<template>
  <aside class="sidebar-doc" role="complementary" aria-label="AI Suggest Sidebar">
    <header class="header">
      <h3 class="title">AI Suggest</h3>
      <div class="header-actions">
        <div class="counter" aria-live="polite">
          <span v-if="total > 0">{{ displayIndex }} of {{ total }}</span>
          <span v-else>0 of 0</span>
        </div>
        <div class="nav">
          <button
            class="nav-btn"
            :disabled="total <= 1 || busy"
            @click="$emit('prev')"
            aria-label="Previous suggestion"
            type="button"
          >
            &lt;
          </button>
          <button
            class="nav-btn"
            :disabled="total <= 1 || busy"
            @click="$emit('next')"
            aria-label="Next suggestion"
            type="button"
          >
            &gt;
          </button>
        </div>
        <button
          class="primary-btn"
          :disabled="busy"
          @click="$emit('request-suggest')"
          type="button"
        >
          {{ busy ? 'Getting Suggestion…' : 'AI Suggest' }}
        </button>
      </div>
    </header>

    <section class="content">
      <div v-if="current" class="compare">
        <div class="col">
          <div class="label">Original</div>
          <div class="panel original" :title="originalJoined">
            <pre class="text">{{ originalJoined }}</pre>
          </div>
        </div>
        <div class="col">
          <div class="label">Suggestion</div>
          <div class="panel suggested" :title="suggestedJoined">
            <pre class="text">{{ suggestedJoined }}</pre>
          </div>
        </div>
      </div>

      <div v-else class="placeholder">
        Select text in the document (or leave empty to use full content), then click “AI Suggest”.
      </div>

      <p v-if="error" class="error" role="alert">{{ error }}</p>
    </section>

    <footer class="footer">
      <button
        class="accept"
        :disabled="!current || busy"
        @click="onAccept"
        type="button"
      >
        Accept
      </button>
      <button
        class="reject"
        :disabled="!current || busy"
        @click="onReject"
        type="button"
      >
        Reject
      </button>
    </footer>
  </aside>
</template>

<script setup lang="ts">
/**
 * Lightweight view-only Sidebar for AI suggestions.
 * - Receives SuggestionBundle list via props
 * - Emits user intents (request-suggest/accept/reject/prev/next)
 * - Does not mutate the editor directly
 */

export interface BlockSuggestion {
  blockId: string
  originalText: string
  suggestedText: string
  type: 'paragraph' | 'heading' | 'list_item'
  attrs: Record<string, any>
  marksMap?: Array<{ from: number; to: number; marks: any[] }>
  anchor: { from: number; to: number }
}

export interface SuggestionBundle {
  id: string
  blocks: BlockSuggestion[]
  createdAt: number
  selectionSummary: string
}

const props = withDefaults(
  defineProps<{
    suggestions: SuggestionBundle[]
    currentIndex: number
    busy?: boolean
    error?: string | null
  }>(),
  {
    suggestions: () => [],
    currentIndex: 0,
    busy: false,
    error: null,
  },
)

const emit = defineEmits<{
  (e: 'accept', id: string): void
  (e: 'reject', id: string): void
  (e: 'prev'): void
  (e: 'next'): void
  (e: 'request-suggest'): void
}>()

const total = $computed(() => props.suggestions.length)
const current = $computed(() => props.suggestions[props.currentIndex] ?? null)
const displayIndex = $computed(() => (total > 0 ? props.currentIndex + 1 : 0))

const originalJoined = $computed(() =>
  current ? current.blocks.map(b => b.originalText).join('\n\n') : '',
)
const suggestedJoined = $computed(() =>
  current ? current.blocks.map(b => b.suggestedText).join('\n\n') : '',
)

function onAccept() {
  if (!current?.id && !current?.value) return
  const id = (current as any).value ? (current as any).value.id : (current as any).id
  if (!id) return
  emit('accept', id)
}

function onReject() {
  if (!current?.id && !current?.value) return
  const id = (current as any).value ? (current as any).value.id : (current as any).id
  if (!id) return
  emit('reject', id)
}
</script>

<style scoped>
.sidebar-doc {
  display: flex;
  flex-direction: column;
  gap: 16px;
  width: 50%;
  height: 100%;
  box-sizing: border-box;
  padding: 16px;
  background: var(--umo-container-background, #fafafa);
  border-left: 1px solid var(--umo-border-color, #e5e7eb);
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.title {
  margin: 0;
  font-size: 16px;
  font-weight: 700;
  color: var(--umo-text-color, #111827);
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-left: auto;
}

.counter {
  font-size: 12px;
  color: #6b7280;
  min-width: 56px;
  text-align: right;
}

.nav {
  display: flex;
  gap: 4px;
}

.nav-btn {
  width: 28px;
  height: 28px;
  border: 1px solid var(--umo-border-color, #d1d5db);
  background: #f9fafb;
  color: #374151;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.15s ease;
}
.nav-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
.nav-btn:not(:disabled):hover {
  background: #eef2f7;
}

.primary-btn {
  height: 32px;
  padding: 0 12px;
  border: none;
  border-radius: 6px;
  background: var(--umo-primary-color, #3b82f6);
  color: var(--umo-color-white, #fff);
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.15s ease;
}
.primary-btn:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}
.primary-btn:not(:disabled):hover {
  background: var(--umo-primary-color-hover, #2563eb);
}

.content {
  flex: 1;
  min-height: 180px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.compare {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.col {
  display: flex;
  flex-direction: column;
  gap: 6px;
  min-width: 0;
}

.label {
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.4px;
  color: #6b7280;
  text-transform: uppercase;
  padding-left: 2px;
}

.panel {
  border: 1px solid var(--umo-border-color, #e5e7eb);
  border-radius: 8px;
  background: var(--umo-color-white, #fff);
  padding: 10px;
  max-height: 260px;
  overflow: auto;
}

.panel.original {
  background: #fff7f7;
  border-left: 4px solid #dc2626;
}

.panel.suggested {
  background: #f4fff6;
  border-left: 4px solid #16a34a;
}

.text {
  margin: 0;
  white-space: pre-wrap;
  word-break: break-word;
  font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Ubuntu,
    Cantarell, Noto Sans, sans-serif, Apple Color Emoji, Segoe UI Emoji;
  font-size: 12px;
  line-height: 1.55;
  color: #111827;
}

.placeholder {
  font-size: 12px;
  color: #6b7280;
  padding: 12px;
  border: 1px dashed var(--umo-border-color, #e5e7eb);
  border-radius: 8px;
  background: var(--umo-color-white, #fff);
}

.error {
  margin: 0;
  color: #dc2626;
  font-size: 12px;
}

.footer {
  display: flex;
  gap: 8px;
}

.accept,
.reject {
  flex: 1;
  height: 34px;
  border: none;
  border-radius: 6px;
  font-weight: 700;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.accept {
  background: #16a34a;
  color: #fff;
}
.accept:disabled {
  background: #9ae6b4;
  cursor: not-allowed;
}
.accept:not(:disabled):hover {
  background: #15803d;
}

.reject {
  background: #dc2626;
  color: #fff;
}
.reject:disabled {
  background: #fca5a5;
  cursor: not-allowed;
}
.reject:not(:disabled):hover {
  background: #b91c1c;
}

@media (max-width: 1360px) {
  .compare {
    grid-template-columns: 1fr;
  }
}
</style>


