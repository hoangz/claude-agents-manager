<script setup lang="ts">
// Types
interface ClaudeCodeProject {
  name: string
  path: string
  displayName: string
  lastActivity?: string
  sessionCount: number
}

interface ClaudeCodeSession {
  id: string
  summary: string
  messageCount: number
  lastActivity: string
  cwd: string
  model?: string
}

// State
const searchQuery = ref('')
const expandedProject = ref<string | null>(null)
const projectSessions = ref<Record<string, ClaudeCodeSession[]>>({})
const loadingSessions = ref<Record<string, boolean>>({})

// Fetch projects
const { data: projects, pending: loadingProjects, refresh } = await useFetch<ClaudeCodeProject[]>('/api/projects', {
  default: () => [],
})

// Filtered projects
const filteredProjects = computed(() => {
  if (!searchQuery.value.trim()) return projects.value
  const q = searchQuery.value.toLowerCase()
  return projects.value.filter(
    (p) => p.displayName.toLowerCase().includes(q) || p.path.toLowerCase().includes(q)
  )
})

// Toggle project expand
async function toggleProject(project: ClaudeCodeProject) {
  if (expandedProject.value === project.name) {
    expandedProject.value = null
    return
  }
  expandedProject.value = project.name
  if (!projectSessions.value[project.name]) {
    loadingSessions.value[project.name] = true
    try {
      const sessions = await $fetch<ClaudeCodeSession[]>(`/api/projects/${project.name}`)
      projectSessions.value[project.name] = sessions
    } catch {
      projectSessions.value[project.name] = []
    } finally {
      loadingSessions.value[project.name] = false
    }
  }
}

// Format relative time
function formatRelativeTime(dateStr?: string): string {
  if (!dateStr) return 'Never'
  const date = new Date(dateStr)
  if (isNaN(date.getTime())) return dateStr
  const now = Date.now()
  const diff = now - date.getTime()
  const minutes = Math.floor(diff / 60000)
  if (minutes < 1) return 'Just now'
  if (minutes < 60) return `${minutes}m ago`
  const hours = Math.floor(minutes / 60)
  if (hours < 24) return `${hours}h ago`
  const days = Math.floor(hours / 24)
  if (days < 7) return `${days}d ago`
  return date.toLocaleDateString()
}

// Short model label
function modelLabel(model?: string): string {
  if (!model) return ''
  if (model.includes('sonnet')) return 'Sonnet'
  if (model.includes('haiku')) return 'Haiku'
  if (model.includes('opus')) return 'Opus'
  return model.split('-').slice(0, 2).join('-')
}

useHead({ title: 'History — Claude Code Agent Manager' })
</script>

<template>
  <div class="page-container">
    <!-- Header -->
    <div class="page-header">
      <div class="header-left">
        <div class="header-icon-wrap">
          <UIcon name="i-lucide-clock" class="header-icon" />
        </div>
        <div>
          <h1 class="page-title">History</h1>
          <p class="page-subtitle">
            <template v-if="!loadingProjects">
              {{ projects.length }} project{{ projects.length !== 1 ? 's' : '' }} tracked
            </template>
            <template v-else>Loading...</template>
          </p>
        </div>
      </div>
      <button class="refresh-btn" :disabled="loadingProjects" @click="refresh">
        <UIcon name="i-lucide-refresh-cw" :class="['btn-icon', { 'animate-spin': loadingProjects }]" />
        Refresh
      </button>
    </div>

    <!-- Search -->
    <div class="search-wrap">
      <UIcon name="i-lucide-search" class="search-icon" />
      <input
        v-model="searchQuery"
        class="search-input"
        placeholder="Search by project name or path..."
        type="text"
      />
      <button v-if="searchQuery" class="search-clear" @click="searchQuery = ''">
        <UIcon name="i-lucide-x" class="size-3.5" />
      </button>
    </div>

    <!-- Loading skeleton -->
    <div v-if="loadingProjects" class="skeleton-list">
      <div v-for="i in 5" :key="i" class="skeleton-card">
        <div class="skeleton-line skeleton-title" />
        <div class="skeleton-line skeleton-sub" />
      </div>
    </div>

    <!-- Empty state -->
    <div v-else-if="!filteredProjects.length" class="empty-state">
      <div class="empty-icon-wrap">
        <UIcon name="i-lucide-folder-open" class="empty-icon" />
      </div>
      <p class="empty-title">{{ searchQuery ? 'No matching projects' : 'No history yet' }}</p>
      <p class="empty-sub">
        {{ searchQuery ? 'Try a different search term.' : 'Start using Claude Code CLI to see projects here.' }}
      </p>
    </div>

    <!-- Project list -->
    <div v-else class="project-list">
      <div
        v-for="project in filteredProjects"
        :key="project.name"
        class="project-card"
        :class="{ 'project-card--expanded': expandedProject === project.name }"
      >
        <!-- Project row -->
        <button class="project-row" @click="toggleProject(project)">
          <div class="project-info">
            <div class="project-name-row">
              <UIcon name="i-lucide-folder" class="folder-icon" />
              <span class="project-name">{{ project.displayName }}</span>
            </div>
            <div class="project-path">{{ project.path }}</div>
          </div>
          <div class="project-meta">
            <div class="meta-item">
              <UIcon name="i-lucide-message-square" class="meta-icon" />
              <span>{{ project.sessionCount }} session{{ project.sessionCount !== 1 ? 's' : '' }}</span>
            </div>
            <div v-if="project.lastActivity" class="meta-item">
              <UIcon name="i-lucide-clock" class="meta-icon" />
              <span>{{ formatRelativeTime(project.lastActivity) }}</span>
            </div>
          </div>
          <UIcon
            :name="expandedProject === project.name ? 'i-lucide-chevron-up' : 'i-lucide-chevron-down'"
            class="chevron-icon"
          />
        </button>

        <!-- Sessions drawer -->
        <Transition name="drawer">
          <div v-if="expandedProject === project.name" class="sessions-drawer">
            <!-- Loading sessions -->
            <div v-if="loadingSessions[project.name]" class="sessions-loading">
              <UIcon name="i-lucide-loader-2" class="size-4 animate-spin" style="color: var(--text-disabled);" />
              <span>Loading sessions...</span>
            </div>

            <!-- No sessions -->
            <div v-else-if="!projectSessions[project.name]?.length" class="sessions-empty">
              <UIcon name="i-lucide-inbox" class="size-4" style="color: var(--text-disabled);" />
              <span>No sessions found</span>
            </div>

            <!-- Session list -->
            <div v-else class="session-list">
              <div
                v-for="session in projectSessions[project.name]"
                :key="session.id"
                class="session-item"
              >
                <div class="session-summary">
                  <UIcon name="i-lucide-message-circle" class="session-icon" />
                  <span class="session-text">{{ session.summary || 'No summary available' }}</span>
                </div>
                <div class="session-footer">
                  <div class="session-meta-row">
                    <span class="session-count">
                      <UIcon name="i-lucide-hash" class="size-3" />
                      {{ session.messageCount }} msgs
                    </span>
                    <span class="session-time">
                      <UIcon name="i-lucide-clock" class="size-3" />
                      {{ formatRelativeTime(session.lastActivity) }}
                    </span>
                    <span v-if="session.cwd" class="session-cwd font-mono">
                      {{ session.cwd.split('/').slice(-2).join('/') }}
                    </span>
                  </div>
                  <span v-if="session.model" class="model-badge">
                    {{ modelLabel(session.model) }}
                  </span>
                </div>
              </div>
            </div>
          </div>
        </Transition>
      </div>
    </div>
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

/* ─── Search ──────────────────────────────────────────── */
.search-wrap {
  position: relative;
  margin-bottom: 1.25rem;
}
.search-icon {
  position: absolute;
  left: 0.75rem;
  top: 50%;
  transform: translateY(-50%);
  width: 0.875rem;
  height: 0.875rem;
  color: var(--text-disabled);
  pointer-events: none;
}
.search-input {
  width: 100%;
  padding: 0.5rem 2.25rem 0.5rem 2.25rem;
  font-size: 0.8125rem;
  background: var(--input-bg);
  border: 1px solid var(--border-subtle);
  border-radius: 0.625rem;
  color: var(--text-primary);
  outline: none;
  font-family: var(--font-sans);
  transition: border-color 0.15s;
}
.search-input::placeholder {
  color: var(--text-disabled);
}
.search-input:focus {
  border-color: var(--border-default);
}
.search-clear {
  position: absolute;
  right: 0.625rem;
  top: 50%;
  transform: translateY(-50%);
  display: flex;
  align-items: center;
  justify-content: center;
  width: 1.25rem;
  height: 1.25rem;
  border-radius: 0.25rem;
  color: var(--text-disabled);
  cursor: pointer;
  transition: color 0.15s;
}
.search-clear:hover {
  color: var(--text-secondary);
}

/* ─── Skeleton ──────────────────────────────────────────── */
.skeleton-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}
.skeleton-card {
  padding: 1rem;
  border-radius: 0.75rem;
  border: 1px solid var(--border-subtle);
  background: var(--surface-raised);
}
.skeleton-line {
  border-radius: 0.25rem;
  background: var(--surface-hover);
  animation: pulse 1.5s ease-in-out infinite;
}
.skeleton-title {
  height: 0.875rem;
  width: 40%;
  margin-bottom: 0.5rem;
}
.skeleton-sub {
  height: 0.75rem;
  width: 65%;
}
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}

/* ─── Empty state ──────────────────────────────────────── */
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 4rem 2rem;
  text-align: center;
}
.empty-icon-wrap {
  width: 3.5rem;
  height: 3.5rem;
  border-radius: 1rem;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--surface-raised);
  border: 1px solid var(--border-subtle);
  margin-bottom: 1rem;
}
.empty-icon {
  width: 1.5rem;
  height: 1.5rem;
  color: var(--text-disabled);
}
.empty-title {
  font-size: 0.9375rem;
  font-weight: 500;
  color: var(--text-secondary);
  margin-bottom: 0.375rem;
}
.empty-sub {
  font-size: 0.8125rem;
  color: var(--text-tertiary);
}

/* ─── Project list ──────────────────────────────────────── */
.project-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.project-card {
  border-radius: 0.75rem;
  border: 1px solid var(--border-subtle);
  background: var(--surface-raised);
  overflow: hidden;
  transition: border-color 0.15s;
}
.project-card:hover {
  border-color: var(--border-default);
}
.project-card--expanded {
  border-color: rgba(79, 142, 247, 0.3);
}

.project-row {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.875rem 1rem;
  cursor: pointer;
  text-align: left;
  transition: background 0.15s;
}
.project-row:hover {
  background: var(--surface-hover);
}

.project-info {
  flex: 1;
  min-width: 0;
}
.project-name-row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.25rem;
}
.folder-icon {
  width: 0.875rem;
  height: 0.875rem;
  color: var(--accent);
  flex-shrink: 0;
}
.project-name {
  font-size: 0.875rem;
  font-weight: 500;
  color: var(--text-primary);
  font-family: var(--font-sans);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.project-path {
  font-size: 0.6875rem;
  font-family: var(--font-mono);
  color: var(--text-tertiary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-left: 1.375rem;
}

.project-meta {
  display: flex;
  align-items: center;
  gap: 0.875rem;
  flex-shrink: 0;
}
.meta-item {
  display: flex;
  align-items: center;
  gap: 0.3rem;
  font-size: 0.75rem;
  color: var(--text-tertiary);
  font-family: var(--font-sans);
}
.meta-icon {
  width: 0.75rem;
  height: 0.75rem;
}
.chevron-icon {
  width: 1rem;
  height: 1rem;
  color: var(--text-disabled);
  flex-shrink: 0;
  transition: color 0.15s;
}
.project-row:hover .chevron-icon {
  color: var(--text-tertiary);
}

/* ─── Sessions drawer ──────────────────────────────────── */
.sessions-drawer {
  border-top: 1px solid var(--border-subtle);
  background: var(--surface-base);
  padding: 0.75rem;
}

.sessions-loading,
.sessions-empty {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 0.5rem;
  font-size: 0.8125rem;
  color: var(--text-tertiary);
}

.session-list {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}
.session-item {
  padding: 0.75rem;
  border-radius: 0.5rem;
  border: 1px solid var(--border-subtle);
  background: var(--surface-raised);
  transition: border-color 0.15s;
}
.session-item:hover {
  border-color: var(--border-default);
}

.session-summary {
  display: flex;
  align-items: flex-start;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}
.session-icon {
  width: 0.875rem;
  height: 0.875rem;
  color: var(--accent);
  flex-shrink: 0;
  margin-top: 0.1rem;
}
.session-text {
  font-size: 0.8125rem;
  color: var(--text-secondary);
  line-height: 1.4;
  font-family: var(--font-sans);
}

.session-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  padding-left: 1.375rem;
}
.session-meta-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex-wrap: wrap;
}
.session-count,
.session-time,
.session-cwd {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  font-size: 0.6875rem;
  color: var(--text-disabled);
}
.session-cwd {
  font-size: 0.625rem;
  font-family: var(--font-mono);
}
.model-badge {
  font-size: 0.625rem;
  font-weight: 500;
  font-family: var(--font-mono);
  padding: 0.1rem 0.45rem;
  border-radius: 9999px;
  background: rgba(79, 142, 247, 0.1);
  color: rgba(79, 142, 247, 0.9);
  border: 1px solid rgba(79, 142, 247, 0.2);
  white-space: nowrap;
}

/* ─── Drawer transition ─────────────────────────────────── */
.drawer-enter-active,
.drawer-leave-active {
  transition: max-height 0.25s ease, opacity 0.2s ease;
  overflow: hidden;
  max-height: 800px;
}
.drawer-enter-from,
.drawer-leave-to {
  max-height: 0;
  opacity: 0;
}
</style>
