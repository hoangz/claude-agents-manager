<script setup lang="ts">
const { skills, loading, error, remove, fetchAll: fetchSkills } = useSkills()
const router = useRouter()
const toast = useToast()
const { workingDir } = useWorkingDir()

const showCreateModal = ref(false)
const showImportModal = ref(false)
const searchQuery = ref('')

// --- Category/Filter state ---
const activeCategory = ref('All')
const viewMode = ref<'list' | 'grid'>('list')
const sortOrder = ref<'name-asc' | 'name-desc' | 'recent'>('name-asc')
const showSortMenu = ref(false)

// Auto-detect category from skill slug/name/description
function detectCategory(skill: any): string {
  const text = `${skill.slug} ${skill.frontmatter?.name ?? ''} ${skill.frontmatter?.description ?? ''}`.toLowerCase()
  if (/code|dev|program|debug|test|review|refactor|lint/.test(text)) return 'Development'
  if (/writ|content|copy|blog|email|doc|draft/.test(text)) return 'Writing'
  if (/data|analyz|research|report|sql|metric|stat/.test(text)) return 'Analysis'
  if (/design|ui|ux|figma|css|style|layout/.test(text)) return 'Design'
  if (/git|deploy|devops|docker|ci|cd|infra|server/.test(text)) return 'DevOps'
  if (/game|pusoy|big2|casino|card|poker/.test(text)) return 'Game'
  return 'General'
}

// Skills with category attached
const skillsWithCategory = computed(() =>
  skills.value.map(s => ({
    ...s,
    _category: (s.frontmatter?.category as string | undefined) || detectCategory(s),
  }))
)

// Category counts (only non-empty categories)
const categoryCounts = computed(() => {
  const counts: Record<string, number> = {}
  for (const s of skillsWithCategory.value) {
    counts[s._category] = (counts[s._category] ?? 0) + 1
  }
  return counts
})

// Ordered category list: All first, then existing categories
const CATEGORY_ORDER = ['Development', 'Writing', 'Analysis', 'Design', 'DevOps', 'Game', 'General']

const availableCategories = computed(() => {
  const existing = CATEGORY_ORDER.filter(c => (categoryCounts.value[c] ?? 0) > 0)
  // Include any categories not in predefined order (e.g. from frontmatter)
  const extra = Object.keys(categoryCounts.value).filter(c => !CATEGORY_ORDER.includes(c))
  return [...existing, ...extra]
})

// Sort helper
function sortSkills(list: typeof skillsWithCategory.value) {
  const copy = [...list]
  if (sortOrder.value === 'name-asc') return copy.sort((a, b) => a.frontmatter.name.localeCompare(b.frontmatter.name))
  if (sortOrder.value === 'name-desc') return copy.sort((a, b) => b.frontmatter.name.localeCompare(a.frontmatter.name))
  // 'recent': preserve original order (as returned by API)
  return copy
}

const filteredSkills = computed(() => {
  let list = skillsWithCategory.value

  // Search filter
  if (searchQuery.value) {
    const q = searchQuery.value.toLowerCase()
    list = list.filter(s =>
      s.frontmatter.name.toLowerCase().includes(q) ||
      s.frontmatter.description?.toLowerCase().includes(q) ||
      s.frontmatter.agent?.toLowerCase().includes(q)
    )
  }

  // Category filter
  if (activeCategory.value !== 'All') {
    list = list.filter(s => s._category === activeCategory.value)
  }

  return sortSkills(list)
})

const sortLabel = computed(() => {
  if (sortOrder.value === 'name-asc') return 'A → Z'
  if (sortOrder.value === 'name-desc') return 'Z → A'
  return 'Recent'
})

onMounted(() => {
  fetchSkills({ workingDir: workingDir.value })
})

// Close sort menu on outside click
if (import.meta.client) {
  onMounted(() => {
    document.addEventListener('click', () => { showSortMenu.value = false })
  })
}

// --- Bulk selection ---
const selectedSlugs = ref<Set<string>>(new Set())

const isSelected = (slug: string) => selectedSlugs.value.has(slug)

function toggleSelect(slug: string, event: MouseEvent) {
  event.preventDefault()
  event.stopPropagation()
  const next = new Set(selectedSlugs.value)
  if (next.has(slug)) next.delete(slug)
  else next.add(slug)
  selectedSlugs.value = next
}

function selectAll() {
  selectedSlugs.value = new Set(filteredSkills.value.map(s => s.slug))
}

function clearSelection() {
  selectedSlugs.value = new Set()
}

async function deleteSelected() {
  const slugs = Array.from(selectedSlugs.value)
  try {
    await Promise.all(slugs.map(slug => remove(slug)))
    toast.add({ title: `Deleted ${slugs.length} skill${slugs.length === 1 ? '' : 's'}`, color: 'success' })
    clearSelection()
  } catch (e: any) {
    toast.add({ title: 'Delete failed', description: e.data?.message || e.message, color: 'error' })
  }
}
</script>

<template>
  <div>
    <PageHeader title="Skills">
      <template #trailing>
        <span class="font-mono text-[12px] text-meta">{{ skills.length }}</span>
      </template>
      <template #right>
        <UButton label="Import" icon="i-lucide-upload" size="sm" variant="soft" @click="showImportModal = true" />
        <UButton label="New Skill" icon="i-lucide-plus" size="sm" @click="showCreateModal = true" />
      </template>
    </PageHeader>

    <div class="px-6 py-4">
      <p class="text-[13px] mb-4 leading-relaxed text-label">
        Specific capabilities that can be added to agents and invoked as slash commands.
      </p>

      <!-- Search + View/Sort controls -->
      <div class="flex items-center gap-2 mb-3">
        <input
          v-model="searchQuery"
          placeholder="Search skills..."
          class="field-search max-w-xs flex-1"
        />

        <!-- View toggle -->
        <div class="flex items-center gap-0.5 shrink-0 rounded-lg p-0.5" style="background: var(--input-bg); border: 1px solid var(--border-subtle);">
          <button
            class="view-toggle-btn"
            :class="{ 'view-toggle-btn--active': viewMode === 'list' }"
            title="List view"
            @click="viewMode = 'list'"
          >
            <UIcon name="i-lucide-list" class="size-3.5" />
          </button>
          <button
            class="view-toggle-btn"
            :class="{ 'view-toggle-btn--active': viewMode === 'grid' }"
            title="Grid view"
            @click="viewMode = 'grid'"
          >
            <UIcon name="i-lucide-layout-grid" class="size-3.5" />
          </button>
        </div>

        <!-- Sort dropdown -->
        <div class="relative shrink-0" @click.stop>
          <button
            class="flex items-center gap-1.5 px-2.5 py-1.5 rounded-lg text-[11px] transition-colors duration-150 focus-ring"
            style="color: var(--text-tertiary); background: var(--input-bg); border: 1px solid var(--border-subtle);"
            @mouseenter="($event.currentTarget as HTMLElement).style.borderColor = 'var(--border-default)'"
            @mouseleave="($event.currentTarget as HTMLElement).style.borderColor = 'var(--border-subtle)'"
            @click="showSortMenu = !showSortMenu"
          >
            <UIcon name="i-lucide-arrow-up-down" class="size-3" />
            <span>{{ sortLabel }}</span>
            <UIcon name="i-lucide-chevron-down" class="size-3" />
          </button>
          <!-- Sort menu -->
          <div
            v-if="showSortMenu"
            class="sort-menu"
          >
            <button
              v-for="opt in [{ value: 'name-asc', label: 'Name A → Z', icon: 'i-lucide-arrow-up-a-z' }, { value: 'name-desc', label: 'Name Z → A', icon: 'i-lucide-arrow-down-z-a' }, { value: 'recent', label: 'Recently added', icon: 'i-lucide-clock' }]"
              :key="opt.value"
              class="sort-menu-item"
              :class="{ 'sort-menu-item--active': sortOrder === opt.value }"
              @click="sortOrder = opt.value as typeof sortOrder; showSortMenu = false"
            >
              <UIcon :name="opt.icon" class="size-3.5" />
              {{ opt.label }}
            </button>
          </div>
        </div>
      </div>

      <!-- Category filter pills -->
      <div v-if="skills.length" class="category-pills mb-4">
        <!-- All pill -->
        <button
          class="category-pill"
          :class="{ 'category-pill--active': activeCategory === 'All' }"
          @click="activeCategory = 'All'"
        >
          All <span class="pill-count">{{ skills.length }}</span>
        </button>
        <!-- Category pills -->
        <button
          v-for="cat in availableCategories"
          :key="cat"
          class="category-pill"
          :class="{ 'category-pill--active': activeCategory === cat }"
          @click="activeCategory = cat"
        >
          {{ cat }} <span class="pill-count">{{ categoryCounts[cat] }}</span>
        </button>
      </div>

      <!-- Active filter info bar -->
      <div v-if="activeCategory !== 'All' || searchQuery" class="flex items-center gap-2 mb-3 text-[12px]" style="color: var(--text-tertiary);">
        <UIcon name="i-lucide-filter" class="size-3.5 shrink-0" />
        <span>
          <template v-if="activeCategory !== 'All' && searchQuery">
            {{ filteredSkills.length }} skills in <strong style="color: var(--text-secondary);">{{ activeCategory }}</strong> matching "{{ searchQuery }}"
          </template>
          <template v-else-if="activeCategory !== 'All'">
            {{ filteredSkills.length }} skill{{ filteredSkills.length === 1 ? '' : 's' }} in <strong style="color: var(--text-secondary);">{{ activeCategory }}</strong>
          </template>
          <template v-else>
            {{ filteredSkills.length }} result{{ filteredSkills.length === 1 ? '' : 's' }} for "{{ searchQuery }}"
          </template>
        </span>
        <button
          class="ml-auto text-[11px] focus-ring rounded px-1.5 py-0.5 transition-colors"
          style="color: var(--accent);"
          @mouseenter="($event.currentTarget as HTMLElement).style.background = 'var(--accent-muted)'"
          @mouseleave="($event.currentTarget as HTMLElement).style.background = 'transparent'"
          @click="activeCategory = 'All'; searchQuery = ''"
        >
          Clear filters
        </button>
      </div>

      <div
        v-if="error"
        class="rounded-xl px-4 py-3 mb-4 flex items-start gap-3"
        style="background: rgba(248, 113, 113, 0.06); border: 1px solid rgba(248, 113, 113, 0.12);"
      >
        <UIcon name="i-lucide-alert-circle" class="size-4 shrink-0 mt-0.5" style="color: var(--error);" />
        <span class="text-[12px]" style="color: var(--error);">{{ error }}</span>
      </div>

      <div v-if="loading" class="space-y-1">
        <SkeletonRow v-for="i in 5" :key="i" />
      </div>

      <!-- Grid view -->
      <div v-else-if="filteredSkills.length && viewMode === 'grid'" class="skills-grid">
        <NuxtLink
          v-for="skill in filteredSkills"
          :key="skill.slug"
          :to="`/skills/${skill.slug}`"
          class="skill-grid-card group focus-ring"
          :style="isSelected(skill.slug) ? 'outline: 1px solid var(--accent); background: rgba(var(--accent-rgb, 59, 130, 246), 0.08);' : ''"
        >
          <!-- Checkbox -->
          <button
            class="skill-grid-checkbox"
            :class="{ 'skill-grid-checkbox--visible': isSelected(skill.slug) || selectedSlugs.size > 0 }"
            :aria-label="isSelected(skill.slug) ? 'Deselect' : 'Select'"
            @click="(e) => toggleSelect(skill.slug, e)"
          >
            <UIcon
              :name="isSelected(skill.slug) ? 'i-lucide-check-square' : 'i-lucide-square'"
              class="size-3"
              :style="{ color: isSelected(skill.slug) ? 'var(--accent)' : 'var(--text-meta)' }"
            />
          </button>
          <!-- Category badge -->
          <span class="skill-grid-category-badge">{{ skill._category }}</span>
          <div class="flex items-start gap-2 mt-1">
            <UIcon name="i-lucide-sparkles" class="size-3.5 shrink-0 mt-0.5" style="color: var(--accent);" />
            <div class="min-w-0 flex-1">
              <div class="text-[12px] font-semibold truncate" style="color: var(--text-primary);">{{ skill.frontmatter.name }}</div>
              <div class="text-[11px] mt-0.5 line-clamp-2" style="color: var(--text-secondary);">{{ skill.frontmatter.description }}</div>
            </div>
          </div>
          <!-- Badges row -->
          <div class="flex flex-wrap gap-1 mt-2">
            <span v-if="skill.frontmatter.context" class="badge badge-subtle text-[10px] px-1.5 py-px font-mono">{{ skill.frontmatter.context }}</span>
            <span v-if="skill.source === 'plugin' && skill.pluginName" class="badge badge-accent text-[10px] px-1.5 py-px font-mono">plugin: {{ skill.pluginName }}</span>
            <span
              v-if="skill.mcpServer"
              class="badge text-[10px] px-1.5 py-px font-mono"
              style="background: rgba(99, 102, 241, 0.1); color: #818cf8; border: 1px solid rgba(99, 102, 241, 0.2);"
            >mcp: {{ skill.mcpServer.name }}</span>
            <span v-else-if="skill.frontmatter.agent" class="badge badge-agent text-[10px] px-1.5 py-px font-mono">agent: {{ skill.frontmatter.agent }}</span>
          </div>
        </NuxtLink>
      </div>

      <!-- List view (original layout, enhanced) -->
      <div v-else-if="filteredSkills.length" class="space-y-1">
        <div
          v-for="skill in filteredSkills"
          :key="skill.slug"
          class="relative group skill-row"
        >
          <!-- Checkbox overlay -->
          <button
            class="skill-checkbox"
            :class="{ 'skill-checkbox--visible': isSelected(skill.slug) || selectedSlugs.size > 0 }"
            :aria-label="isSelected(skill.slug) ? 'Deselect' : 'Select'"
            @click="(e) => toggleSelect(skill.slug, e)"
          >
            <UIcon
              :name="isSelected(skill.slug) ? 'i-lucide-check-square' : 'i-lucide-square'"
              class="size-3.5"
              :style="{ color: isSelected(skill.slug) ? 'var(--accent)' : 'var(--text-meta)' }"
            />
          </button>

          <NuxtLink
            :to="`/skills/${skill.slug}`"
            class="flex items-center gap-3 px-3 py-2.5 rounded-lg group focus-ring hover-row"
            :style="isSelected(skill.slug) ? 'background: rgba(var(--accent-rgb, 59, 130, 246), 0.06); outline: 1px solid var(--accent);' : ''"
          >
            <!-- Indent when checkbox is visible -->
            <span
              class="shrink-0 transition-all duration-150"
              :class="selectedSlugs.size > 0 ? 'w-5' : 'w-0'"
            />

            <!-- Icon -->
            <UIcon name="i-lucide-sparkles" class="size-3.5 shrink-0" style="color: var(--accent);" />

            <!-- Name -->
            <span class="text-[13px] font-medium w-44 shrink-0 truncate">
              {{ skill.frontmatter.name }}
            </span>

            <!-- Category badge -->
            <span class="skill-category-label shrink-0">
              {{ skill._category }}
            </span>

            <!-- Context badge -->
            <span
              v-if="skill.frontmatter.context"
              class="text-[10px] font-mono px-1.5 py-px rounded-full shrink-0 badge badge-subtle"
            >
              {{ skill.frontmatter.context }}
            </span>

            <!-- Plugin badge -->
            <span
              v-if="skill.source === 'plugin' && skill.pluginName"
              class="text-[10px] font-mono px-1.5 py-px rounded-full shrink-0 badge badge-accent"
            >
              plugin: {{ skill.pluginName }}
            </span>

            <!-- MCP badge -->
            <span
              v-if="skill.mcpServer"
              class="text-[10px] font-mono px-1.5 py-px rounded-full shrink-0 badge"
              style="background: rgba(99, 102, 241, 0.1); color: #818cf8; border: 1px solid rgba(99, 102, 241, 0.2);"
            >
              mcp: {{ skill.mcpServer.name }}
            </span>

            <!-- Agent badge -->
            <span
              v-else-if="skill.frontmatter.agent"
              class="text-[10px] font-mono px-1.5 py-px rounded-full shrink-0 badge badge-agent"
            >
              agent: {{ skill.frontmatter.agent }}
            </span>

            <!-- Preloaded by badge -->
            <div
              v-if="skill.agents?.length"
              class="flex items-center gap-1 shrink-0"
              :title="`Preloaded by: ${skill.agents.map((a: any) => a.name).join(', ')}`"
            >
              <span
                class="text-[10px] font-mono px-1.5 py-px rounded-full badge badge-subtle flex items-center gap-1"
              >
                <UIcon name="i-lucide-user" class="size-2.5" />
                <span v-if="skill.agents.length > 1">({{ skill.agents.length }})</span>
              </span>
            </div>

            <!-- GitHub badge -->
            <ImportBadge
              v-if="skill.source === 'github' && skill.githubRepo"
              :repo="skill.githubRepo"
            />

            <!-- Description -->
            <span class="flex-1 text-[12px] truncate text-label">
              {{ skill.frontmatter.description }}
            </span>

            <!-- Metadata -->
            <div class="flex items-center gap-3 shrink-0">
              <UIcon
                name="i-lucide-chevron-right"
                class="size-3.5 opacity-0 group-hover:opacity-100 transition-opacity text-meta"
              />
            </div>
          </NuxtLink>
        </div>
      </div>

      <!-- Empty state: search/filter miss -->
      <div v-else-if="searchQuery || activeCategory !== 'All'" class="flex flex-col items-center justify-center py-16 gap-3">
        <UIcon name="i-lucide-search-x" class="size-8" style="color: var(--text-disabled);" />
        <p class="text-[13px] text-label">No skills match your filters.</p>
        <button
          class="text-[12px] focus-ring rounded px-2 py-1"
          style="color: var(--accent);"
          @click="activeCategory = 'All'; searchQuery = ''"
        >
          Clear all filters
        </button>
      </div>

      <!-- Empty state: no skills -->
      <div v-else class="flex flex-col items-center justify-center py-12 space-y-5">
        <div class="rounded-lg p-4 bg-card max-w-sm w-full text-[12px] text-label leading-relaxed space-y-1">
          <div class="flex items-center gap-2">
            <UIcon name="i-lucide-cpu" class="size-3.5" style="color: var(--accent);" />
            <span>code-reviewer</span>
            <span class="text-meta">agent</span>
          </div>
          <div class="flex items-center gap-2 ml-5">
            <UIcon name="i-lucide-sparkles" class="size-3" style="color: var(--accent);" />
            <span>security-audit</span>
            <span class="text-meta">skill</span>
          </div>
          <div class="flex items-center gap-2 ml-5">
            <UIcon name="i-lucide-sparkles" class="size-3" style="color: var(--accent);" />
            <span>performance-check</span>
            <span class="text-meta">skill</span>
          </div>
        </div>
        <p class="text-[13px] text-label">Skills teach agents specific capabilities. Link a skill to an agent to extend what it can do.</p>
        <div class="flex items-center gap-2">
          <UButton label="Create a skill" size="sm" @click="showCreateModal = true" />
          <UButton label="Import from GitHub" size="sm" variant="outline" to="/explore?tab=imported" />
        </div>
      </div>
    </div>

    <UModal v-model:open="showCreateModal">
      <template #content>
        <SkillForm
          mode="create"
          @saved="(s) => { showCreateModal = false; router.push(`/skills/${s.slug}`) }"
          @cancel="showCreateModal = false"
        />
      </template>
    </UModal>

    <UModal v-model:open="showImportModal">
      <template #content>
        <div class="p-6 space-y-4 bg-overlay">
          <h3 class="text-page-title">Import Skill</h3>
          <FileImport
            type="skills"
            @imported="(s) => { showImportModal = false; fetchSkills(); router.push(`/skills/${s.slug}`) }"
          />
          <div class="flex justify-end">
            <UButton label="Cancel" variant="ghost" color="neutral" size="sm" @click="showImportModal = false" />
          </div>
        </div>
      </template>
    </UModal>

    <!-- Bulk action bar -->
    <BulkActionBar
      :selected-count="selectedSlugs.size"
      :total-count="filteredSkills.length"
      item-label="skill"
      @delete-selected="deleteSelected"
      @clear-selection="clearSelection"
      @select-all="selectAll"
    />
  </div>
</template>

<style scoped>
/* Row wrapper for checkbox positioning */
.skill-row {
  position: relative;
}

/* Checkbox shown on hover or when selection mode active */
.skill-checkbox {
  position: absolute;
  left: 6px;
  top: 50%;
  transform: translateY(-50%);
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  border-radius: 5px;
  background: var(--surface-overlay);
  border: 1px solid var(--border-subtle);
  cursor: pointer;
  opacity: 0;
  transition: opacity 0.15s;
  backdrop-filter: blur(4px);
}

.skill-row:hover .skill-checkbox,
.skill-checkbox--visible {
  opacity: 1;
}

/* Category filter pills */
.category-pills {
  display: flex;
  align-items: center;
  gap: 6px;
  overflow-x: auto;
  padding-bottom: 2px;
  scrollbar-width: none;
}
.category-pills::-webkit-scrollbar { display: none; }

.category-pill {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 10px;
  border-radius: 9999px;
  font-size: 11px;
  font-weight: 500;
  white-space: nowrap;
  cursor: pointer;
  transition: background 0.15s, color 0.15s, border-color 0.15s;
  color: var(--text-tertiary);
  background: transparent;
  border: 1px solid var(--border-subtle);
  font-family: var(--font-sans);
}

.category-pill:hover {
  color: var(--text-primary);
  border-color: var(--border-default);
  background: var(--surface-hover);
}

.category-pill--active {
  color: var(--accent) !important;
  background: var(--accent-muted) !important;
  border-color: var(--accent-glow) !important;
}

.pill-count {
  font-size: 10px;
  font-family: var(--font-mono);
  opacity: 0.7;
}

/* Category label in list row */
.skill-category-label {
  font-size: 10px;
  font-family: var(--font-mono);
  padding: 1px 6px;
  border-radius: 9999px;
  background: var(--badge-subtle-bg);
  color: var(--text-disabled);
  border: 1px solid var(--border-subtle);
}

/* View toggle buttons */
.view-toggle-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.1s, color 0.1s;
  color: var(--text-disabled);
}

.view-toggle-btn:hover {
  color: var(--text-secondary);
  background: var(--surface-hover);
}

.view-toggle-btn--active {
  color: var(--accent) !important;
  background: var(--accent-muted) !important;
}

/* Sort dropdown menu */
.sort-menu {
  position: absolute;
  top: calc(100% + 4px);
  right: 0;
  z-index: 30;
  min-width: 160px;
  border-radius: 8px;
  overflow: hidden;
  background: var(--surface-raised, var(--surface-overlay));
  border: 1px solid var(--border-subtle);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
}

.sort-menu-item {
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
  padding: 7px 12px;
  font-size: 12px;
  text-align: left;
  cursor: pointer;
  color: var(--text-secondary);
  transition: background 0.1s, color 0.1s;
  font-family: var(--font-sans);
}

.sort-menu-item:hover {
  background: var(--surface-hover);
  color: var(--text-primary);
}

.sort-menu-item--active {
  color: var(--accent) !important;
  background: var(--accent-muted) !important;
}

/* Grid view */
.skills-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  gap: 10px;
}

.skill-grid-card {
  position: relative;
  padding: 12px;
  border-radius: 10px;
  border: 1px solid var(--border-subtle);
  background: var(--surface-raised, var(--bg-card));
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s, box-shadow 0.15s;
  display: flex;
  flex-direction: column;
}

.skill-grid-card:hover {
  border-color: var(--border-default);
  background: var(--surface-hover);
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.06);
}

.skill-grid-checkbox {
  position: absolute;
  top: 8px;
  left: 8px;
  z-index: 10;
  width: 20px;
  height: 20px;
  border-radius: 4px;
  background: var(--surface-overlay);
  border: 1px solid var(--border-subtle);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transition: opacity 0.15s;
  backdrop-filter: blur(4px);
}

.skill-grid-card:hover .skill-grid-checkbox,
.skill-grid-checkbox--visible {
  opacity: 1;
}

.skill-grid-category-badge {
  align-self: flex-start;
  font-size: 10px;
  font-family: var(--font-mono);
  padding: 1px 6px;
  border-radius: 9999px;
  background: var(--accent-muted);
  color: var(--accent);
  border: 1px solid var(--accent-glow);
  margin-bottom: 6px;
}
</style>
