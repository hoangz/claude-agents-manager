<script setup lang="ts">
// Types
interface SuggestionTarget {
  type: 'agent' | 'command' | 'skill'
  slug: string
}

interface Suggestion {
  type: 'missing-skill' | 'missing-description' | 'empty-body' | 'orphan-skill'
  severity: 'warning' | 'info'
  message: string
  target: SuggestionTarget
}

// Filter tabs
type FilterTab = 'all' | 'warning' | 'info'
const activeTab = ref<FilterTab>('all')

// Fetch suggestions
const { data: suggestions, pending: loading, refresh } = await useFetch<Suggestion[]>('/api/suggestions', {
  default: () => [],
})

// Filter by tab
const filtered = computed(() => {
  if (activeTab.value === 'all') return suggestions.value
  return suggestions.value.filter((s) => s.severity === activeTab.value)
})

// Counts
const warningCount = computed(() => suggestions.value.filter((s) => s.severity === 'warning').length)
const infoCount = computed(() => suggestions.value.filter((s) => s.severity === 'info').length)
const totalCount = computed(() => suggestions.value.length)
const healthScore = computed(() => {
  if (totalCount.value === 0) return 100
  // Warnings penalize more than infos
  const penalty = warningCount.value * 8 + infoCount.value * 2
  return Math.max(0, Math.min(100, 100 - penalty))
})

// Gauge ring math (SVG circle, r=36)
const CIRCUMFERENCE = 2 * Math.PI * 36
const strokeDash = computed(() => {
  const filled = (healthScore.value / 100) * CIRCUMFERENCE
  return `${filled} ${CIRCUMFERENCE}`
})
const gaugeColor = computed(() => {
  if (healthScore.value >= 90) return '#34d399'   // green
  if (healthScore.value >= 70) return '#fbbf24'   // yellow
  return '#f87171'                                  // red
})

// Group suggestions by type label
const typeLabel: Record<string, string> = {
  'missing-description': 'Missing Description',
  'missing-skill': 'Missing Skill Reference',
  'empty-body': 'Empty Body',
  'orphan-skill': 'Orphan Skills',
}

const grouped = computed(() => {
  const map: Record<string, Suggestion[]> = {}
  for (const s of filtered.value) {
    const key = s.type
    if (!map[key]) map[key] = []
    map[key]!.push(s)
  }
  return map
})

// Affected targets stats
const affectedStats = computed(() => {
  const agents = new Set<string>()
  const commands = new Set<string>()
  const skills = new Set<string>()
  for (const s of suggestions.value) {
    if (s.target.type === 'agent') agents.add(s.target.slug)
    else if (s.target.type === 'command') commands.add(s.target.slug)
    else if (s.target.type === 'skill') skills.add(s.target.slug)
  }
  return { agents: agents.size, commands: commands.size, skills: skills.size }
})

// Route to fix page
function fixRoute(target: SuggestionTarget): string {
  const map: Record<string, string> = { agent: 'agents', skill: 'skills', command: 'commands' }
  return `/${map[target.type]}/${target.slug}`
}

useHead({ title: 'System Health — Claude Code Agent Manager' })
</script>

<template>
  <div class="page-container">
    <!-- Header -->
    <div class="page-header">
      <div class="header-left">
        <div class="header-icon-wrap">
          <UIcon name="i-lucide-activity" class="header-icon" />
        </div>
        <div>
          <h1 class="page-title">System Health</h1>
          <p class="page-subtitle">Warnings and improvement suggestions</p>
        </div>
      </div>
      <div class="header-actions">
        <!-- Summary badges -->
        <span v-if="warningCount > 0" class="badge badge-warning">
          <UIcon name="i-lucide-alert-triangle" class="size-3" />
          {{ warningCount }} warning{{ warningCount !== 1 ? 's' : '' }}
        </span>
        <span v-if="infoCount > 0" class="badge badge-info">
          <UIcon name="i-lucide-info" class="size-3" />
          {{ infoCount }} info
        </span>
        <button class="refresh-btn" :disabled="loading" @click="refresh">
          <UIcon name="i-lucide-refresh-cw" :class="['btn-icon', { 'animate-spin': loading }]" />
          Refresh
        </button>
      </div>
    </div>

    <!-- Loading -->
    <div v-if="loading" class="loading-state">
      <UIcon name="i-lucide-loader-2" class="size-5 animate-spin" style="color: var(--text-disabled);" />
      <span>Checking system health...</span>
    </div>

    <template v-else>
      <!-- All healthy -->
      <div v-if="totalCount === 0" class="healthy-state">
        <div class="healthy-icon-wrap">
          <UIcon name="i-lucide-check-circle-2" class="healthy-icon" />
        </div>
        <h2 class="healthy-title">All systems healthy</h2>
        <p class="healthy-sub">No warnings or suggestions found. Everything looks great!</p>
      </div>

      <template v-else>
        <!-- Score + stats row -->
        <div class="summary-row">
          <!-- Gauge ring -->
          <div class="gauge-card">
            <div class="gauge-wrap">
              <svg viewBox="0 0 80 80" class="gauge-svg">
                <!-- Track -->
                <circle cx="40" cy="40" r="36" fill="none" stroke="var(--border-subtle)" stroke-width="6" />
                <!-- Fill -->
                <circle
                  cx="40" cy="40" r="36"
                  fill="none"
                  :stroke="gaugeColor"
                  stroke-width="6"
                  stroke-linecap="round"
                  :stroke-dasharray="strokeDash"
                  stroke-dashoffset="0"
                  transform="rotate(-90 40 40)"
                  style="transition: stroke-dasharray 0.6s ease;"
                />
              </svg>
              <div class="gauge-label">
                <span class="gauge-score" :style="{ color: gaugeColor }">{{ healthScore }}</span>
                <span class="gauge-unit">/ 100</span>
              </div>
            </div>
            <p class="gauge-desc">Health Score</p>
          </div>

          <!-- Stats -->
          <div class="stats-grid">
            <div class="stat-card">
              <UIcon name="i-lucide-cpu" class="stat-icon" />
              <span class="stat-num">{{ affectedStats.agents }}</span>
              <span class="stat-label">Agents</span>
            </div>
            <div class="stat-card">
              <UIcon name="i-lucide-terminal" class="stat-icon" />
              <span class="stat-num">{{ affectedStats.commands }}</span>
              <span class="stat-label">Commands</span>
            </div>
            <div class="stat-card">
              <UIcon name="i-lucide-sparkles" class="stat-icon" />
              <span class="stat-num">{{ affectedStats.skills }}</span>
              <span class="stat-label">Skills</span>
            </div>
            <div class="stat-card">
              <UIcon name="i-lucide-alert-circle" class="stat-icon stat-icon--total" />
              <span class="stat-num">{{ totalCount }}</span>
              <span class="stat-label">Total Issues</span>
            </div>
          </div>
        </div>

        <!-- Filter tabs -->
        <div class="filter-tabs">
          <button
            v-for="tab in (['all', 'warning', 'info'] as FilterTab[])"
            :key="tab"
            class="filter-tab"
            :class="{ 'filter-tab--active': activeTab === tab }"
            @click="activeTab = tab"
          >
            <UIcon
              v-if="tab === 'warning'"
              name="i-lucide-alert-triangle"
              class="size-3.5"
            />
            <UIcon
              v-else-if="tab === 'info'"
              name="i-lucide-info"
              class="size-3.5"
            />
            <UIcon v-else name="i-lucide-layers" class="size-3.5" />
            <span class="capitalize">{{ tab === 'all' ? `All (${totalCount})` : tab === 'warning' ? `Warnings (${warningCount})` : `Info (${infoCount})` }}</span>
          </button>
        </div>

        <!-- No results after filter -->
        <div v-if="!filtered.length" class="empty-filter">
          <UIcon name="i-lucide-filter-x" class="size-4" style="color: var(--text-disabled);" />
          <span>No {{ activeTab }} items</span>
        </div>

        <!-- Groups -->
        <div v-else class="groups">
          <div
            v-for="(groupItems, groupType) in grouped"
            :key="groupType"
            class="group"
          >
            <!-- Group header -->
            <div class="group-header">
              <UIcon
                :name="groupItems[0]?.severity === 'warning' ? 'i-lucide-alert-triangle' : 'i-lucide-info'"
                class="group-icon"
                :class="groupItems[0]?.severity === 'warning' ? 'group-icon--warning' : 'group-icon--info'"
              />
              <span class="group-title">{{ typeLabel[groupType] || groupType }}</span>
              <span class="group-count">{{ groupItems.length }}</span>
            </div>

            <!-- Suggestion items -->
            <div class="suggestion-list">
              <div
                v-for="(suggestion, idx) in groupItems"
                :key="`${suggestion.target.slug}-${idx}`"
                class="suggestion-item"
              >
                <div class="suggestion-left">
                  <UIcon
                    :name="suggestion.severity === 'warning' ? 'i-lucide-alert-triangle' : 'i-lucide-info'"
                    class="sug-icon"
                    :class="suggestion.severity === 'warning' ? 'sug-icon--warning' : 'sug-icon--info'"
                  />
                  <div class="suggestion-body">
                    <p class="suggestion-msg">{{ suggestion.message }}</p>
                    <div class="suggestion-tags">
                      <span class="type-badge" :class="`type-badge--${suggestion.target.type}`">
                        {{ suggestion.target.type }}
                      </span>
                      <span class="slug-text">{{ suggestion.target.slug }}</span>
                    </div>
                  </div>
                </div>
                <NuxtLink :to="fixRoute(suggestion.target)" class="fix-btn">
                  Fix
                  <UIcon name="i-lucide-arrow-right" class="size-3" />
                </NuxtLink>
              </div>
            </div>
          </div>
        </div>
      </template>
    </template>
  </div>
</template>

<style scoped>
/* ─── Layout ──────────────────────────────────────────── */
.page-container {
  max-width: 860px;
  margin: 0 auto;
  padding: 2rem 1.5rem 4rem;
}

/* ─── Header ──────────────────────────────────────────── */
.page-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 1.5rem;
  gap: 1rem;
  flex-wrap: wrap;
}
.header-left {
  display: flex;
  align-items: center;
  gap: 0.875rem;
}
.header-icon-wrap {
  width: 2.25rem;
  height: 2.25rem;
  border-radius: 0.625rem;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, rgba(79, 142, 247, 0.18) 0%, rgba(79, 142, 247, 0.06) 100%);
  border: 1px solid rgba(79, 142, 247, 0.2);
}
.header-icon {
  width: 1rem;
  height: 1rem;
  color: var(--accent);
}
.page-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: var(--text-primary);
  font-family: var(--font-display);
  line-height: 1.2;
}
.page-subtitle {
  font-size: 0.75rem;
  color: var(--text-tertiary);
  margin-top: 0.125rem;
}
.header-actions {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex-wrap: wrap;
}

/* ─── Badges ──────────────────────────────────────────── */
.badge {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
  font-size: 0.6875rem;
  font-weight: 500;
  padding: 0.2rem 0.55rem;
  border-radius: 9999px;
  font-family: var(--font-sans);
}
.badge-warning {
  background: rgba(251, 191, 36, 0.12);
  color: #fbbf24;
  border: 1px solid rgba(251, 191, 36, 0.25);
}
.badge-info {
  background: rgba(79, 142, 247, 0.1);
  color: rgba(79, 142, 247, 0.9);
  border: 1px solid rgba(79, 142, 247, 0.2);
}

/* ─── Refresh btn ─────────────────────────────────────── */
.refresh-btn {
  display: flex;
  align-items: center;
  gap: 0.375rem;
  padding: 0.375rem 0.75rem;
  font-size: 0.75rem;
  border-radius: 0.5rem;
  border: 1px solid var(--border-default);
  color: var(--text-secondary);
  background: var(--surface-raised);
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
  font-family: var(--font-sans);
}
.refresh-btn:hover:not(:disabled) {
  background: var(--surface-hover);
  color: var(--text-primary);
}
.refresh-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
.btn-icon {
  width: 0.875rem;
  height: 0.875rem;
}

/* ─── Loading ─────────────────────────────────────────── */
.loading-state {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 3rem;
  justify-content: center;
  font-size: 0.875rem;
  color: var(--text-tertiary);
}

/* ─── All healthy ─────────────────────────────────────── */
.healthy-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 4rem 2rem;
  text-align: center;
}
.healthy-icon-wrap {
  width: 5rem;
  height: 5rem;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(52, 211, 153, 0.1);
  border: 2px solid rgba(52, 211, 153, 0.3);
  margin-bottom: 1.25rem;
}
.healthy-icon {
  width: 2.5rem;
  height: 2.5rem;
  color: #34d399;
}
.healthy-title {
  font-size: 1.125rem;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 0.375rem;
  font-family: var(--font-display);
}
.healthy-sub {
  font-size: 0.875rem;
  color: var(--text-tertiary);
}

/* ─── Summary row ─────────────────────────────────────── */
.summary-row {
  display: flex;
  align-items: flex-start;
  gap: 1rem;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}

/* Gauge */
.gauge-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  padding: 1rem;
  border-radius: 0.875rem;
  border: 1px solid var(--border-subtle);
  background: var(--surface-raised);
  min-width: 120px;
}
.gauge-wrap {
  position: relative;
  width: 80px;
  height: 80px;
}
.gauge-svg {
  width: 100%;
  height: 100%;
}
.gauge-label {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
.gauge-score {
  font-size: 1.25rem;
  font-weight: 700;
  line-height: 1;
  font-family: var(--font-mono);
}
.gauge-unit {
  font-size: 0.5625rem;
  color: var(--text-disabled);
  font-family: var(--font-mono);
}
.gauge-desc {
  font-size: 0.6875rem;
  color: var(--text-tertiary);
  font-family: var(--font-sans);
}

/* Stats */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 0.5rem;
  flex: 1;
  min-width: 200px;
}
.stat-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.25rem;
  padding: 0.875rem 0.75rem;
  border-radius: 0.75rem;
  border: 1px solid var(--border-subtle);
  background: var(--surface-raised);
}
.stat-icon {
  width: 1rem;
  height: 1rem;
  color: var(--text-disabled);
}
.stat-icon--total {
  color: #f87171;
}
.stat-num {
  font-size: 1.25rem;
  font-weight: 700;
  color: var(--text-primary);
  font-family: var(--font-mono);
  line-height: 1;
}
.stat-label {
  font-size: 0.6875rem;
  color: var(--text-tertiary);
  font-family: var(--font-sans);
}

/* ─── Filter tabs ─────────────────────────────────────── */
.filter-tabs {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  padding: 0.25rem;
  border-radius: 0.625rem;
  background: var(--surface-raised);
  border: 1px solid var(--border-subtle);
  width: fit-content;
  margin-bottom: 1.25rem;
}
.filter-tab {
  display: flex;
  align-items: center;
  gap: 0.35rem;
  padding: 0.3rem 0.75rem;
  border-radius: 0.4rem;
  font-size: 0.75rem;
  color: var(--text-tertiary);
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
  font-family: var(--font-sans);
}
.filter-tab:hover {
  color: var(--text-secondary);
  background: var(--surface-hover);
}
.filter-tab--active {
  background: var(--surface-hover);
  color: var(--text-primary);
  font-weight: 500;
}

/* ─── Empty filter ────────────────────────────────────── */
.empty-filter {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 2rem;
  justify-content: center;
  font-size: 0.8125rem;
  color: var(--text-tertiary);
}

/* ─── Groups ──────────────────────────────────────────── */
.groups {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.group {
  border-radius: 0.875rem;
  border: 1px solid var(--border-subtle);
  background: var(--surface-raised);
  overflow: hidden;
}
.group-header {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1rem;
  border-bottom: 1px solid var(--border-subtle);
  background: var(--surface-base);
}
.group-icon {
  width: 0.875rem;
  height: 0.875rem;
  flex-shrink: 0;
}
.group-icon--warning { color: #fbbf24; }
.group-icon--info { color: rgba(79, 142, 247, 0.9); }
.group-title {
  font-size: 0.8125rem;
  font-weight: 500;
  color: var(--text-secondary);
  font-family: var(--font-sans);
  flex: 1;
}
.group-count {
  font-size: 0.6875rem;
  font-weight: 600;
  font-family: var(--font-mono);
  padding: 0.1rem 0.45rem;
  border-radius: 9999px;
  background: var(--badge-subtle-bg, rgba(255,255,255,0.06));
  color: var(--text-disabled);
}

/* ─── Suggestion items ────────────────────────────────── */
.suggestion-list {
  display: flex;
  flex-direction: column;
}
.suggestion-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0.75rem 1rem;
  border-bottom: 1px solid var(--border-subtle);
  transition: background 0.15s;
}
.suggestion-item:last-child {
  border-bottom: none;
}
.suggestion-item:hover {
  background: var(--surface-hover);
}
.suggestion-left {
  display: flex;
  align-items: flex-start;
  gap: 0.625rem;
  flex: 1;
  min-width: 0;
}
.sug-icon {
  width: 0.875rem;
  height: 0.875rem;
  flex-shrink: 0;
  margin-top: 0.15rem;
}
.sug-icon--warning { color: #fbbf24; }
.sug-icon--info { color: rgba(79, 142, 247, 0.8); }

.suggestion-body {
  flex: 1;
  min-width: 0;
}
.suggestion-msg {
  font-size: 0.8125rem;
  color: var(--text-secondary);
  font-family: var(--font-sans);
  margin-bottom: 0.3rem;
  line-height: 1.4;
}
.suggestion-tags {
  display: flex;
  align-items: center;
  gap: 0.375rem;
  flex-wrap: wrap;
}
.type-badge {
  font-size: 0.6rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  padding: 0.1rem 0.4rem;
  border-radius: 0.25rem;
  font-family: var(--font-mono);
}
.type-badge--agent {
  background: rgba(79, 142, 247, 0.1);
  color: rgba(79, 142, 247, 0.9);
  border: 1px solid rgba(79, 142, 247, 0.2);
}
.type-badge--skill {
  background: rgba(167, 139, 250, 0.1);
  color: rgba(167, 139, 250, 0.9);
  border: 1px solid rgba(167, 139, 250, 0.2);
}
.type-badge--command {
  background: rgba(52, 211, 153, 0.1);
  color: rgba(52, 211, 153, 0.9);
  border: 1px solid rgba(52, 211, 153, 0.2);
}
.slug-text {
  font-size: 0.6875rem;
  font-family: var(--font-mono);
  color: var(--text-disabled);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 200px;
}

/* Fix button */
.fix-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
  font-size: 0.75rem;
  font-weight: 500;
  padding: 0.3rem 0.625rem;
  border-radius: 0.4rem;
  border: 1px solid var(--border-default);
  color: var(--text-secondary);
  background: var(--surface-base);
  cursor: pointer;
  transition: background 0.15s, color 0.15s, border-color 0.15s;
  text-decoration: none;
  white-space: nowrap;
  font-family: var(--font-sans);
  flex-shrink: 0;
}
.fix-btn:hover {
  background: var(--accent-muted);
  color: var(--accent);
  border-color: rgba(79, 142, 247, 0.3);
}
</style>
