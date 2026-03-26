<script setup lang="ts">
// CLI page - handles all navigation internally without URL routing
const { executionOptions } = useCliExecution()

useHead({
  title: 'Claude Code',
})
</script>

<template>
  <div class="h-full flex flex-col">
    <div class="flex-1 flex min-h-0 relative overflow-hidden">
      <div class="flex-1 flex flex-col">
        <ChatV2Interface :execution-options="executionOptions" />
      </div>
    </div>
  </div>
</template>
