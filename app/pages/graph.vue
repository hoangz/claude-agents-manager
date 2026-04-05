<script setup lang="ts">
import { VueFlow, useVueFlow } from '@vue-flow/core'
import { Controls } from '@vue-flow/controls'
import { MiniMap } from '@vue-flow/minimap'
import '@vue-flow/core/dist/style.css'
import '@vue-flow/core/dist/theme-default.css'
import '@vue-flow/controls/dist/style.css'
import '@vue-flow/minimap/dist/style.css'
import type { Relationship } from '~/types'
import { getAgentColor } from '~/utils/colors'
import { getModelBadgeStyle } from '~/utils/models'

const { agents } = useAgents()
const { commands } = useCommands()
const { skills } = useSkills()
const { plugins } = usePlugins()
const { servers: mcpServers } = useMCP()
const router = useRouter()

const relationships = ref<Relationship[]>([])
const loading = ref(true)
const showLegend = ref(true)

// Search & filter state
const searchQuery = ref('')
const activeFilters = ref<Set<string>>(new Set())

// Active only toggle
const showActiveOnly = ref(false)

const { workingDir } = useWorkingDir()

onMounted(async () => {
  try {
    relationships.value = await $fetch<Relationship[]>('/api/relationships', {
      query: { workingDir: workingDir.value }
    })
  } finally {
    loading.value = false
  }
})

// --- Layout constants ---
const NODE_WIDTH = 230
const COL_GAP = NODE_WIDTH + 80
const Y_GAP = 88
const HEADER_Y = -32

// --- Column labels ---
const columnLabels: Record<string, { label: string; icon: string; color: string }> = {
  command: { label: 'Commands', icon: '>_', color: 'var(--text-disabled)' },
  agent: { label: 'Agents', icon: 'cpu', color: 'var(--accent)' },
  skill: { label: 'Skills', icon: 'zap', color: 'var(--model-haiku)' },
  plugin: { label: 'Plugins', icon: 'puzzle', color: 'var(--model-sonnet)' },
  mcp: { label: 'MCP Servers', icon: 'server', color: 'var(--accent)' },
}

// --- Filter types available ---
const allFilterTypes = computed(() => {
  const types: string[] = []
  if (commands.value.length > 0) types.push('command')
  if (agents.value.length > 0) types.push('agent')
  if (skills.value.length > 0) types.push('skill')
  if (plugins.value.length > 0) types.push('plugin')
  if (mcpServers.value.length > 0) types.push('mcp')
  return types
})

function toggleFilter(type: string) {
  if (activeFilters.value.has(type)) {
    activeFilters.value.delete(type)
  } else {
    activeFilters.value.add(type)
  }
  // trigger reactivity
  activeFilters.value = new Set(activeFilters.value)
}

// --- Connected node IDs (for orphan detection) ---
const connectedNodeIds = computed(() => {
  const ids = new Set<string>()
  for (const r of relationships.value) {
    ids.add(`${r.sourceType}-${r.sourceSlug}`)
    ids.add(`${r.targetType}-${r.targetSlug}`)
  }
  return ids
})

// --- Filter items by search query ---
function matchesSearch(name: string, description?: string): boolean {
  if (!searchQuery.value) return true
  const q = searchQuery.value.toLowerCase()
  return (
    name.toLowerCase().includes(q) ||
    (description?.toLowerCase().includes(q) ?? false)
  )
}

// --- Filter items by active state ---
function matchesActiveFilter(nodeId: string, item: any, type: string): boolean {
  if (!showActiveOnly.value) return true
  if (type === 'plugin') return item.enabled === true
  if (type === 'mcp') return true
  // agents, skills, commands: phải có ít nhất 1 connection
  return connectedNodeIds.value.has(nodeId)
}

// --- Build columns: Commands → Agents → Skills → Plugins → MCP ---
const columns = computed(() => {
  const cols: { type: string; items: any[] }[] = []
  // Fixed left-to-right order: source → target
  if (commands.value.length > 0) cols.push({ type: 'command', items: commands.value })
  if (agents.value.length > 0) cols.push({ type: 'agent', items: agents.value })
  if (skills.value.length > 0) cols.push({ type: 'skill', items: skills.value })
  if (plugins.value.length > 0) cols.push({ type: 'plugin', items: plugins.value })
  if (mcpServers.value.length > 0) cols.push({ type: 'mcp', items: mcpServers.value })
  return cols
})

const nodes = computed(() => {
  const result: any[] = []

  // Apply type filter
  const filteredCols = activeFilters.value.size > 0
    ? columns.value.filter(c => activeFilters.value.has(c.type))
    : columns.value

  filteredCols.forEach((col, colIndex) => {
    const x = colIndex * COL_GAP + 40

    // Filter items by search and active state
    const filteredItems = col.items.filter((item) => {
      const nodeId = col.type === 'plugin' ? `plugin-${item.id}` : (col.type === 'mcp' ? `mcp-${item.name}` : `${col.type}-${item.slug}`)
      const name = col.type === 'plugin' ? item.name : (col.type === 'mcp' ? item.name : item.frontmatter?.name ?? item.slug)
      const desc = col.type === 'plugin' ? item.description : (col.type === 'mcp' ? '' : item.frontmatter?.description)
      if (!matchesSearch(name, desc)) return false
      return matchesActiveFilter(nodeId, item, col.type)
    })

    if (filteredItems.length === 0) return

    // Column header node (non-interactive)
    result.push({
      id: `header-${col.type}`,
      type: 'columnHeader',
      position: { x, y: HEADER_Y },
      data: {
        label: columnLabels[col.type]?.label ?? col.type,
        count: filteredItems.length,
        color: columnLabels[col.type]?.color ?? 'var(--text-disabled)',
      },
      selectable: false,
      draggable: false,
      connectable: false,
    })

    filteredItems.forEach((item, i) => {
      const y = i * Y_GAP + 40
      const nodeId = col.type === 'plugin' ? `plugin-${item.id}` : (col.type === 'mcp' ? `mcp-${item.name}` : `${col.type}-${item.slug}`)
      const isOrphan = !connectedNodeIds.value.has(nodeId)

      if (col.type === 'agent') {
        result.push({
          id: nodeId,
          type: 'agent',
          position: { x, y },
          class: isOrphan ? 'graph-orphan' : '',
          data: {
            label: item.frontmatter.name,
            description: item.frontmatter.description,
            color: getAgentColor(item.frontmatter.color),
            model: item.frontmatter.model,
            slug: item.slug,
            orphan: isOrphan,
          },
        })
      }
      else if (col.type === 'command') {
        result.push({
          id: nodeId,
          type: 'command',
          position: { x, y },
          class: isOrphan ? 'graph-orphan' : '',
          data: {
            label: item.frontmatter.name,
            slug: item.slug,
            directory: item.directory,
            description: item.frontmatter.description,
            orphan: isOrphan,
          },
        })
      }
      else if (col.type === 'skill') {
        result.push({
          id: nodeId,
          type: 'skill',
          position: { x, y },
          class: isOrphan ? 'graph-orphan' : '',
          data: {
            label: item.frontmatter.name,
            description: item.frontmatter.description,
            slug: item.slug,
            orphan: isOrphan,
          },
        })
      }
      else if (col.type === 'plugin') {
        result.push({
          id: nodeId,
          type: 'plugin',
          position: { x, y },
          class: isOrphan ? 'graph-orphan' : '',
          data: {
            label: item.name,
            description: item.description,
            id: item.id,
            enabled: item.enabled,
            skillCount: item.skills.length,
            orphan: isOrphan,
          },
        })
      }
      else if (col.type === 'mcp') {
        result.push({
          id: nodeId,
          type: 'mcp',
          position: { x, y },
          class: isOrphan ? 'graph-orphan' : '',
          data: {
            label: item.name,
            name: item.name,
            scope: item.scope,
            orphan: isOrphan,
          },
        })
      }
    })
  })

  return result
})

const edgeRelationshipLabels: Record<string, string> = {
  spawns: 'spawns',
  'agent-frontmatter': 'uses',
  'spawned-by': 'invokes',
}

// Edge colors per relationship type — more vivid
const edgeColors: Record<string, string> = {
  spawns: 'var(--accent)',
  'agent-frontmatter': '#34d399',
  'spawned-by': '#818cf8',
}

const edges = computed(() => {
  // Only show edges where both endpoints are visible
  const visibleNodeIds = new Set(nodes.value.map(n => n.id))
  return relationships.value
    .filter(r =>
      visibleNodeIds.has(`${r.sourceType}-${r.sourceSlug}`) &&
      visibleNodeIds.has(`${r.targetType}-${r.targetSlug}`)
    )
    .map((r, i) => {
      const color = edgeColors[r.type] ?? 'var(--border-emphasis)'
      return {
        id: `edge-${i}`,
        source: `${r.sourceType}-${r.sourceSlug}`,
        target: `${r.targetType}-${r.targetSlug}`,
        type: 'smoothstep',
        animated: r.type === 'spawns',
        label: edgeRelationshipLabels[r.type] ?? r.type,
        labelStyle: { opacity: 0 },
        labelBgStyle: { opacity: 0 },
        data: { relType: r.type },
        style: {
          stroke: color,
          strokeWidth: r.type === 'spawns' ? 2.5 : 1.5,
          opacity: r.type === 'spawns' ? 0.85 : 0.65,
        },
      }
    })
})

// --- Hover highlighting ---
const hoveredNodeId = ref<string | null>(null)
const graphCanvasRef = ref<HTMLElement | null>(null)

const neighborMap = computed(() => {
  const map = new Map<string, Set<string>>()
  for (const r of relationships.value) {
    const src = `${r.sourceType}-${r.sourceSlug}`
    const tgt = `${r.targetType}-${r.targetSlug}`
    if (!map.has(src)) map.set(src, new Set())
    if (!map.has(tgt)) map.set(tgt, new Set())
    map.get(src)!.add(tgt)
    map.get(tgt)!.add(src)
  }
  return map
})

const highlightedNodeIds = computed(() => {
  if (!hoveredNodeId.value) return new Set<string>()
  const neighbors = neighborMap.value.get(hoveredNodeId.value)
  const set = new Set<string>([hoveredNodeId.value])
  if (neighbors) neighbors.forEach(n => set.add(n))
  return set
})

const highlightedEdgeIds = computed(() => {
  if (!hoveredNodeId.value) return new Set<string>()
  const set = new Set<string>()
  relationships.value.forEach((r, i) => {
    const src = `${r.sourceType}-${r.sourceSlug}`
    const tgt = `${r.targetType}-${r.targetSlug}`
    if (src === hoveredNodeId.value || tgt === hoveredNodeId.value) {
      set.add(`edge-${i}`)
    }
  })
  return set
})

function hasClientXY(e: unknown): e is { clientX: number; clientY: number } {
  return (
    typeof e === 'object'
    && e != null
    && 'clientX' in e
    && 'clientY' in e
    && typeof (e as { clientX: unknown }).clientX === 'number'
    && typeof (e as { clientY: unknown }).clientY === 'number'
  )
}

function parseVueFlowNodeArgs(first: unknown, second?: unknown): {
  event: { clientX: number; clientY: number } | undefined
  node: any
} {
  if (second != null && second !== undefined) {
    return {
      event: hasClientXY(first) ? first : undefined,
      node: second,
    }
  }
  const p = first as Record<string, unknown> | undefined
  if (p && typeof p === 'object' && p.node != null) {
    return {
      event: hasClientXY(p.event) ? p.event : undefined,
      node: p.node,
    }
  }
  if (p && typeof p === 'object' && typeof p.id === 'string' && (p.data != null || p.type != null)) {
    return { event: undefined, node: p }
  }
  return { event: undefined, node: undefined }
}

function onNodeMouseEnter(first: unknown, second?: unknown) {
  const { node } = parseVueFlowNodeArgs(first, second)
  if (!node?.id) return
  if (node.id.startsWith('header-')) return
  hoveredNodeId.value = node.id
  applyHighlightClasses()
}

function onNodeMouseLeave() {
  hoveredNodeId.value = null
  clearHighlightClasses()
}

function applyHighlightClasses() {
  const el = graphCanvasRef.value
  if (!el) return

  el.classList.add('graph-dimmed')

  nextTick(() => {
    el.querySelectorAll('.vue-flow__node').forEach((nodeEl) => {
      const id = nodeEl.getAttribute('data-id')
      if (id && highlightedNodeIds.value.has(id)) {
        nodeEl.classList.add('graph-highlighted')
      } else {
        nodeEl.classList.remove('graph-highlighted')
      }
    })

    el.querySelectorAll('.vue-flow__edge').forEach((edgeEl) => {
      const id = edgeEl.getAttribute('data-id')
      if (id && highlightedEdgeIds.value.has(id)) {
        edgeEl.classList.add('graph-edge-highlighted')
        const labelEl = edgeEl.querySelector('.vue-flow__edge-text') as HTMLElement | null
        const labelBgEl = edgeEl.querySelector('.vue-flow__edge-textbg') as HTMLElement | null
        if (labelEl) labelEl.style.opacity = '1'
        if (labelBgEl) labelBgEl.style.opacity = '1'
      } else {
        edgeEl.classList.remove('graph-edge-highlighted')
        const labelEl = edgeEl.querySelector('.vue-flow__edge-text') as HTMLElement | null
        const labelBgEl = edgeEl.querySelector('.vue-flow__edge-textbg') as HTMLElement | null
        if (labelEl) labelEl.style.opacity = '0'
        if (labelBgEl) labelBgEl.style.opacity = '0'
      }
    })
  })
}

function clearHighlightClasses() {
  const el = graphCanvasRef.value
  if (!el) return

  el.classList.remove('graph-dimmed')
  el.querySelectorAll('.graph-highlighted').forEach(e => e.classList.remove('graph-highlighted'))
  el.querySelectorAll('.graph-edge-highlighted').forEach(e => e.classList.remove('graph-edge-highlighted'))

  el.querySelectorAll('.vue-flow__edge-text').forEach((e) => {
    ;(e as HTMLElement).style.opacity = '0'
  })
  el.querySelectorAll('.vue-flow__edge-textbg').forEach((e) => {
    ;(e as HTMLElement).style.opacity = '0'
  })
}

// --- Tooltip ---
const tooltip = ref<{ text: string; x: number; y: number } | null>(null)

function showTooltip(event: { clientX: number; clientY: number }, description: string | undefined) {
  if (!description) return
  tooltip.value = {
    text: description,
    x: event.clientX + 14,
    y: event.clientY + 14,
  }
}

function hideTooltip() {
  tooltip.value = null
}

function handleNodeMouseEnter(first: unknown, second?: unknown) {
  const { event, node } = parseVueFlowNodeArgs(first, second)
  onNodeMouseEnter(first, second)
  if (node?.data?.description && event) showTooltip(event, node.data.description)
}

function handleNodeMouseLeave() {
  onNodeMouseLeave()
  hideTooltip()
}

function onNodeClick(first: unknown, second?: unknown) {
  const { node } = parseVueFlowNodeArgs(first, second)
  if (!node?.type || !node.data) return
  if (node.type === 'agent') router.push(`/agents/${node.data.slug}`)
  else if (node.type === 'command') router.push(`/commands/${node.data.slug}`)
  else if (node.type === 'skill') router.push(`/skills/${node.data.slug}`)
  else if (node.type === 'plugin') router.push(`/plugins/${node.data.id}`)
  else if (node.type === 'mcp') router.push({ path: `/mcp/${encodeURIComponent(node.data.name)}`, query: { scope: node.data.scope } })
}

// Stats
const visibleNodeCount = computed(() => nodes.value.filter(n => n.type !== 'columnHeader').length)
const visibleEdgeCount = computed(() => edges.value.length)
const totalNodeCount = computed(() => {
  return (agents.value.length + commands.value.length + skills.value.length + plugins.value.length + mcpServers.value.length)
})
const activeNodeCount = computed(() =>
  nodes.value.filter(n => n.type !== 'columnHeader' && !n.data?.orphan).length
)
</script>

<template>
  <div class="relative h-screen flex flex-col">
    <!-- Floating header -->
    <div
      class="absolute top-0 left-0 right-0 z-10 flex flex-col gap-0 px-5"
      style="background: color-mix(in srgb, var(--surface-base) 90%, transparent); backdrop-filter: blur(14px); border-bottom: 1px solid var(--border-subtle);"
    >
      <!-- Top row: title + stats + legend toggle -->
      <div class="h-12 flex items-center gap-3">
        <h1 class="text-page-title">Graph</h1>

        <!-- Stats -->
        <div class="flex items-center gap-3 ml-2">
          <span class="font-mono text-[11px]" style="color: var(--text-disabled);">
            <span style="color: var(--text-secondary);">{{ visibleNodeCount }}</span>
            <span v-if="visibleNodeCount !== totalNodeCount" style="color: var(--text-disabled);">/{{ totalNodeCount }}</span>
            <span class="ml-1">nodes</span>
          </span>
          <span class="font-mono text-[11px]" style="color: var(--text-disabled);">
            <span style="color: var(--text-secondary);">{{ visibleEdgeCount }}</span>
            <span class="ml-1">edges</span>
          </span>
          <span v-if="!showActiveOnly" class="font-mono text-[11px]" style="color: var(--text-disabled);">
            <span style="color: var(--success);">{{ activeNodeCount }}</span> active
          </span>
        </div>

        <div class="flex-1" />

        <!-- Legend toggle -->
        <button
          class="font-mono text-[11px] px-2.5 py-1 rounded focus-ring flex items-center gap-1.5"
          :style="{
            color: showLegend ? 'var(--accent)' : 'var(--text-tertiary)',
            background: showLegend ? 'rgba(229,169,62,0.08)' : 'var(--surface-raised)',
            border: `1px solid ${showLegend ? 'rgba(229,169,62,0.25)' : 'var(--border-default)'}`,
          }"
          @click="showLegend = !showLegend"
        >
          <UIcon name="i-lucide-map" class="size-3" />
          Legend
        </button>
      </div>

      <!-- Filter row -->
      <div class="h-10 flex items-center gap-2 pb-1">
        <!-- Search -->
        <div class="relative flex items-center">
          <UIcon name="i-lucide-search" class="absolute left-2.5 size-3.5 pointer-events-none" style="color: var(--text-disabled);" />
          <input
            v-model="searchQuery"
            type="text"
            placeholder="Search nodes..."
            class="font-mono text-[11px] pl-8 pr-3 py-1 rounded-md outline-none w-44 focus:w-56 transition-all"
            style="background: var(--surface-raised); border: 1px solid var(--border-default); color: var(--text-primary);"
          />
          <button
            v-if="searchQuery"
            class="absolute right-2 size-3.5 flex items-center justify-center"
            style="color: var(--text-disabled);"
            @click="searchQuery = ''"
          >
            <UIcon name="i-lucide-x" class="size-3" />
          </button>
        </div>

        <!-- Active Only toggle -->
        <button
          class="flex items-center gap-1.5 font-mono text-[10px] px-2.5 py-1 rounded-full transition-all shrink-0"
          :style="{
            background: showActiveOnly ? 'rgba(34, 197, 94, 0.12)' : 'var(--surface-raised)',
            border: `1px solid ${showActiveOnly ? 'rgba(34, 197, 94, 0.35)' : 'var(--border-subtle)'}`,
            color: showActiveOnly ? 'var(--success)' : 'var(--text-tertiary)',
          }"
          @click="showActiveOnly = !showActiveOnly"
        >
          <span
            class="size-1.5 rounded-full shrink-0"
            :style="{ background: showActiveOnly ? 'var(--success)' : 'var(--text-disabled)' }"
          />
          Active only
        </button>

        <!-- Divider -->
        <div class="h-4 w-px mx-0.5" style="background: var(--border-subtle);" />

        <!-- Type filters -->
        <div class="flex items-center gap-1.5">
          <button
            v-for="type in allFilterTypes"
            :key="type"
            class="font-mono text-[10px] px-2.5 py-0.5 rounded-full transition-all"
            :style="{
              background: activeFilters.has(type) ? `${columnLabels[type]?.color}18` : 'var(--surface-raised)',
              border: `1px solid ${activeFilters.has(type) ? `${columnLabels[type]?.color}40` : 'var(--border-subtle)'}`,
              color: activeFilters.has(type) ? columnLabels[type]?.color : 'var(--text-tertiary)',
            }"
            @click="toggleFilter(type)"
          >
            {{ columnLabels[type]?.label ?? type }}
          </button>
        </div>

        <!-- Clear filters -->
        <button
          v-if="activeFilters.size > 0 || searchQuery || showActiveOnly"
          class="font-mono text-[10px] px-2 py-0.5 rounded"
          style="color: var(--text-disabled);"
          @click="activeFilters = new Set(); searchQuery = ''; showActiveOnly = false"
        >
          Clear
        </button>
      </div>
    </div>

    <div v-if="loading" class="flex-1 flex items-center justify-center" style="background: var(--surface-base);">
      <UIcon name="i-lucide-loader-2" class="size-6 animate-spin" style="color: var(--text-disabled);" />
    </div>

    <!-- Empty state -->
    <div
      v-else-if="visibleNodeCount === 0"
      class="flex-1 flex flex-col items-center justify-center gap-3"
      style="background: var(--surface-base); margin-top: 88px;"
    >
      <UIcon name="i-lucide-search-x" class="size-8" style="color: var(--text-disabled);" />
      <p class="font-mono text-sm" style="color: var(--text-tertiary);">No nodes match your search</p>
      <button
        class="font-mono text-xs px-3 py-1.5 rounded-md"
        style="background: var(--surface-raised); color: var(--text-secondary); border: 1px solid var(--border-default);"
        @click="activeFilters = new Set(); searchQuery = ''; showActiveOnly = false"
      >
        Clear filters
      </button>
    </div>

    <div v-else ref="graphCanvasRef" class="flex-1 graph-canvas" style="margin-top: 88px;">
      <VueFlow
        :nodes="nodes"
        :edges="edges"
        fit-view-on-init
        :default-edge-options="{ type: 'smoothstep' }"
        :min-zoom="0.2"
        :max-zoom="2"
        @node-click="onNodeClick"
        @node-mouse-enter="handleNodeMouseEnter"
        @node-mouse-leave="handleNodeMouseLeave"
      >
        <!-- Column header -->
        <template #node-columnHeader="{ data }">
          <div class="graph-col-header" :style="{ '--col-color': data.color }">
            <span>{{ data.label }}</span>
            <span class="graph-col-count">{{ data.count }}</span>
          </div>
        </template>

        <!-- Agent node -->
        <template #node-agent="{ data }">
          <div
            class="graph-node graph-node--agent"
            :class="{ 'graph-node--orphan': data.orphan }"
            :style="{
              '--node-glow': `${data.color}30`,
              borderColor: data.orphan ? undefined : `${data.color}45`,
              borderLeftColor: data.color,
              borderLeftWidth: '3px',
            }"
          >
            <div class="flex items-center gap-2">
              <div class="size-2 rounded-full shrink-0" :style="{ background: data.color }" />
              <span class="font-mono text-[11px] font-semibold truncate" style="color: var(--text-primary); flex:1; min-width:0;">
                {{ data.label }}
              </span>
              <span
                v-if="data.model"
                class="text-[9px] font-mono font-medium px-1.5 py-px rounded-full shrink-0"
                :style="getModelBadgeStyle(data.model)"
              >
                {{ data.model }}
              </span>
            </div>
            <div v-if="data.description" class="mt-1 text-[10px] truncate" style="color: var(--text-tertiary);">
              {{ data.description }}
            </div>
          </div>
        </template>

        <!-- Command node -->
        <template #node-command="{ data }">
          <div class="graph-node graph-node--command" :class="{ 'graph-node--orphan': data.orphan }">
            <div class="flex items-center gap-2">
              <span class="font-mono text-[10px] font-bold shrink-0" style="color: var(--text-disabled); line-height:1;">&gt;_</span>
              <span class="font-mono text-[11px] font-medium truncate" style="color: var(--text-secondary); flex:1; min-width:0;">
                /{{ data.label }}
              </span>
            </div>
            <div v-if="data.description" class="mt-0.5 text-[10px] truncate" style="color: var(--text-tertiary);">
              {{ data.description }}
            </div>
          </div>
        </template>

        <!-- Skill node -->
        <template #node-skill="{ data }">
          <div class="graph-node graph-node--skill" :class="{ 'graph-node--orphan': data.orphan }">
            <div class="flex items-center gap-1.5">
              <UIcon name="i-lucide-zap" class="size-3.5 shrink-0" style="color: var(--model-haiku);" />
              <span class="font-mono text-[11px] font-medium truncate" style="color: var(--text-secondary); flex:1; min-width:0;">
                {{ data.label }}
              </span>
            </div>
            <div v-if="data.description" class="mt-0.5 text-[10px] truncate" style="color: var(--text-tertiary);">
              {{ data.description }}
            </div>
          </div>
        </template>

        <!-- Plugin node -->
        <template #node-plugin="{ data }">
          <div class="graph-node graph-node--plugin" :class="{ 'graph-node--orphan': data.orphan }">
            <div class="flex items-center gap-1.5">
              <UIcon name="i-lucide-puzzle" class="size-3.5 shrink-0" style="color: var(--model-sonnet);" />
              <span class="font-mono text-[11px] font-medium truncate" style="color: var(--text-secondary); flex:1; min-width:0;">
                {{ data.label }}
              </span>
              <span
                class="text-[9px] font-mono px-1.5 py-px rounded-full shrink-0"
                :style="{
                  background: data.enabled ? 'rgba(74,222,128,0.15)' : 'var(--badge-subtle-bg)',
                  color: data.enabled ? 'var(--success)' : 'var(--text-disabled)',
                }"
              >
                {{ data.enabled ? 'on' : 'off' }}
              </span>
            </div>
            <div v-if="data.skillCount" class="text-[10px] mt-0.5" style="color: var(--text-tertiary);">
              {{ data.skillCount }} skill{{ data.skillCount !== 1 ? 's' : '' }}
            </div>
          </div>
        </template>

        <!-- MCP node -->
        <template #node-mcp="{ data }">
          <div class="graph-node graph-node--mcp" :class="{ 'graph-node--orphan': data.orphan }">
            <div class="flex items-center gap-1.5">
              <UIcon name="i-lucide-server" class="size-3.5 shrink-0" style="color: var(--accent);" />
              <span class="font-mono text-[11px] font-medium truncate" style="color: var(--text-secondary); flex:1; min-width:0;">
                {{ data.label }}
              </span>
              <span
                class="text-[8px] font-mono px-1 py-px rounded-full shrink-0 uppercase border"
                :style="{
                  borderColor: data.scope === 'global' ? 'rgba(229,169,62,0.3)' : 'var(--border-subtle)',
                  color: data.scope === 'global' ? 'var(--accent)' : 'var(--text-disabled)',
                }"
              >
                {{ data.scope }}
              </span>
            </div>
          </div>
        </template>

        <Controls position="bottom-right" />
        <MiniMap position="top-right" :style="{ marginTop: '8px' }" />
      </VueFlow>

      <!-- Hover tooltip -->
      <div
        v-if="tooltip"
        class="graph-tooltip"
        :style="{ left: tooltip.x + 'px', top: tooltip.y + 'px' }"
      >
        {{ tooltip.text }}
      </div>

      <!-- Legend -->
      <Transition name="page">
        <div
          v-if="showLegend"
          class="absolute bottom-4 left-4 z-10 rounded-xl p-3 text-[11px] space-y-1.5"
          style="background: color-mix(in srgb, var(--surface-base) 94%, transparent); backdrop-filter: blur(14px); border: 1px solid var(--border-default); min-width: 160px;"
        >
          <div class="font-mono text-[10px] font-semibold mb-2 uppercase tracking-wider" style="color: var(--text-disabled);">Legend</div>

          <div class="font-mono text-[9px] uppercase tracking-wider mb-1" style="color: var(--text-disabled);">Node Types</div>
          <div class="flex items-center gap-2">
            <div class="size-2 rounded-full" style="background: var(--accent);" />
            <span style="color: var(--text-tertiary);">Agent</span>
          </div>
          <div class="flex items-center gap-2">
            <span class="font-mono text-[9px] font-bold" style="color: var(--text-disabled);">&gt;_</span>
            <span style="color: var(--text-tertiary);">Command</span>
          </div>
          <div class="flex items-center gap-2">
            <UIcon name="i-lucide-zap" class="size-3" style="color: var(--model-haiku);" />
            <span style="color: var(--text-tertiary);">Skill</span>
          </div>
          <div class="flex items-center gap-2">
            <UIcon name="i-lucide-puzzle" class="size-3" style="color: var(--model-sonnet);" />
            <span style="color: var(--text-tertiary);">Plugin</span>
          </div>
          <div class="flex items-center gap-2">
            <UIcon name="i-lucide-server" class="size-3" style="color: var(--accent);" />
            <span style="color: var(--text-tertiary);">MCP Server</span>
          </div>

          <hr class="my-1.5" style="border-color: var(--border-subtle);" />
          <div class="font-mono text-[9px] uppercase tracking-wider mb-1" style="color: var(--text-disabled);">Connections</div>
          <div class="flex items-center gap-2">
            <div class="w-5 h-[2px] rounded-full" style="background: var(--accent);" />
            <span style="color: var(--text-tertiary);">Spawns</span>
          </div>
          <div class="flex items-center gap-2">
            <div class="w-5 h-[1.5px] rounded-full" style="background: #34d399;" />
            <span style="color: var(--text-tertiary);">Uses</span>
          </div>
          <div class="flex items-center gap-2">
            <div class="w-5 h-[1.5px] rounded-full" style="background: #818cf8;" />
            <span style="color: var(--text-tertiary);">Invokes</span>
          </div>

          <hr class="my-1.5" style="border-color: var(--border-subtle);" />
          <div class="flex items-center gap-2">
            <div class="size-3 rounded" style="border: 1px dashed var(--border-default); opacity: 0.55;" />
            <span style="color: var(--text-tertiary);">No connections</span>
          </div>
          <div class="mt-1.5 text-[10px]" style="color: var(--text-disabled);">
            Hover to highlight · Click to open
          </div>
        </div>
      </Transition>
    </div>
  </div>
</template>
