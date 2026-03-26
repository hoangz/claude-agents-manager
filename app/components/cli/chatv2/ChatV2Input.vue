<script setup lang="ts">
const props = defineProps<{
  modelValue: string
  disabled?: boolean
  isStreaming?: boolean
}>()

const emit = defineEmits<{
  (e: 'update:modelValue', value: string): void
  (e: 'send'): void
  (e: 'abort'): void
}>()

const textareaRef = ref<HTMLTextAreaElement | null>(null)
const isFocused = ref(false)

// Local value for v-model
const localValue = computed({
  get: () => props.modelValue,
  set: (value) => emit('update:modelValue', value),
})

// Auto-resize textarea
function autoResize() {
  if (!textareaRef.value) return

  textareaRef.value.style.height = 'auto'
  const newHeight = Math.min(textareaRef.value.scrollHeight, 200)
  textareaRef.value.style.height = `${newHeight}px`
}

// Handle keydown
function handleKeydown(e: KeyboardEvent) {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault()
    if (!props.disabled && localValue.value.trim()) {
      emit('send')
    }
  }
}

// Watch value changes to resize
watch(localValue, () => {
  nextTick(() => autoResize())
})

// Focus on mount
onMounted(() => {
  textareaRef.value?.focus()
})
</script>

<template>
  <div class="px-4 py-3" style="background: var(--surface);">
    <!-- Input container -->
    <div
      class="relative flex items-end gap-2 p-2 rounded-xl transition-all duration-200"
      :style="{
        background: 'var(--surface-raised)',
        border: isFocused ? '1px solid var(--accent)' : '1px solid var(--border-subtle)',
        boxShadow: isFocused ? '0 0 0 3px rgba(229, 169, 62, 0.1)' : 'none',
      }"
    >
      <!-- Textarea -->
      <textarea
        ref="textareaRef"
        v-model="localValue"
        :disabled="disabled"
        rows="1"
        class="flex-1 px-2 py-1.5 bg-transparent text-[13px] resize-none focus:outline-none"
        :style="{
          color: disabled ? 'var(--text-disabled)' : 'var(--text-primary)',
          maxHeight: '160px',
        }"
        placeholder="Send a message..."
        @keydown="handleKeydown"
        @input="autoResize"
        @focus="isFocused = true"
        @blur="isFocused = false"
      />

      <!-- Action buttons -->
      <div class="flex items-center gap-1 shrink-0 pb-0.5">
        <!-- Abort button when streaming -->
        <button
          v-if="isStreaming"
          class="p-2 rounded-lg transition-all hover:opacity-80"
          style="background: rgba(239, 68, 68, 0.1); color: #ef4444;"
          title="Stop generating"
          @click="emit('abort')"
        >
          <UIcon name="i-lucide-square" class="size-4" />
        </button>

        <!-- Send button -->
        <button
          v-else
          class="p-2 rounded-lg transition-all"
          :style="{
            background: disabled || !localValue.trim() ? 'transparent' : 'var(--accent)',
            color: disabled || !localValue.trim() ? 'var(--text-disabled)' : 'white',
            cursor: disabled || !localValue.trim() ? 'not-allowed' : 'pointer',
          }"
          :disabled="disabled || !localValue.trim()"
          title="Send message"
          @click="emit('send')"
        >
          <UIcon name="i-lucide-arrow-up" class="size-4" />
        </button>
      </div>
    </div>

    <!-- Bottom hints -->
    <div class="flex items-center justify-between mt-2 px-1">
      <div class="flex items-center gap-3 text-[10px]" style="color: var(--text-tertiary);">
        <span>
          <kbd class="px-1 py-0.5 rounded text-[9px]" style="background: var(--surface-raised);">Enter</kbd>
          send
        </span>
        <span>
          <kbd class="px-1 py-0.5 rounded text-[9px]" style="background: var(--surface-raised);">Shift+Enter</kbd>
          newline
        </span>
      </div>

      <div class="flex items-center gap-2">
        <!-- Character counter -->
        <span
          v-if="localValue.length > 0"
          class="text-[10px] font-mono"
          :style="{
            color: localValue.length > 10000 ? '#ef4444' : 'var(--text-tertiary)',
          }"
        >
          {{ localValue.length.toLocaleString() }}
        </span>

        <!-- Streaming indicator -->
        <span
          v-if="isStreaming"
          class="text-[10px] flex items-center gap-1"
          style="color: var(--accent);"
        >
          <UIcon name="i-lucide-loader-2" class="size-3 animate-spin" />
          Generating...
        </span>
      </div>
    </div>
  </div>
</template>
