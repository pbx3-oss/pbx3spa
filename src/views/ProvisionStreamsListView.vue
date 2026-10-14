<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { getApiClient } from '@/api/client'
import { useToastStore } from '@/stores/toast'
import { useStickyFilter, useStickySort } from '@/composables/useStickyFilter'
import { loadTenantOptions } from '@/utils/loadTenantOptions'
import { firstErrorMessage } from '@/utils/formErrors'
import DeleteConfirmModal from '@/components/DeleteConfirmModal.vue'
import ListLoadingState from '@/components/ListLoadingState.vue'

const route = useRoute()
const router = useRouter()
const toast = useToastStore()
const { filterText } = useStickyFilter('provision-streams')
const { sortKey, sortOrder } = useStickySort('provision-streams', { defaultKey: 'name' })

const streams = ref([])
const tenants = ref([])
const cluster = ref('')
const loading = ref(true)
const error = ref('')
const deleteError = ref('')
const deletingName = ref(null)
const confirmDeleteName = ref(null)

const tenantOptions = computed(() =>
  tenants.value
    .map((t) => ({
      shortuid: t.shortuid,
      pkey: t.pkey ?? t.shortuid
    }))
    .filter((t) => t.shortuid)
)

const tenantPkeyLabel = computed(() => {
  const hit = tenantOptions.value.find((t) => t.shortuid === cluster.value)
  return hit?.pkey ?? cluster.value ?? '—'
})

/** Hide docs / junk that scandir may expose as "system" files. */
function isListableStream(row) {
  const name = String(row?.name ?? '')
  if (!name) return false
  if (/\.md$/i.test(name)) return false
  if (/^readme(\.|$)/i.test(name)) return false
  return true
}

const listableStreams = computed(() => streams.value.filter(isListableStream))

const filteredRows = computed(() => {
  const list = listableStreams.value
  const q = (filterText.value || '').trim().toLowerCase()
  if (!q) return list
  return list.filter((s) => {
    const name = (s.name || '').toLowerCase()
    const source = (s.source || '').toLowerCase()
    const notes = (s.notes || '').toLowerCase()
    const updated = (s.updated_at || '').toLowerCase()
    return name.includes(q) || source.includes(q) || notes.includes(q) || updated.includes(q)
  })
})

function sortValue(row, key) {
  if (key === 'refcount') return String(row.refcount ?? 0)
  if (key === 'source') return row.source === 'system' ? 'System' : 'Customer'
  const v = row[key]
  return v == null || v === '' ? '' : String(v)
}

const sortedRows = computed(() => {
  const list = [...filteredRows.value]
  const key = sortKey.value
  const order = sortOrder.value
  list.sort((a, b) => {
    if (key === 'refcount') {
      const na = Number(a.refcount)
      const nb = Number(b.refcount)
      const va = Number.isFinite(na) ? na : -Infinity
      const vb = Number.isFinite(nb) ? nb : -Infinity
      const cmp = va === vb ? 0 : va < vb ? -1 : 1
      return order === 'asc' ? cmp : -cmp
    }
    const va = sortValue(a, key).toLowerCase()
    const vb = sortValue(b, key).toLowerCase()
    let cmp = 0
    if (va < vb) cmp = -1
    else if (va > vb) cmp = 1
    return order === 'asc' ? cmp : -cmp
  })
  return list
})

function setSort(k) {
  if (sortKey.value === k) sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc'
  else {
    sortKey.value = k
    sortOrder.value = 'asc'
  }
}

function sortClass(k) {
  if (sortKey.value !== k) return ''
  return sortOrder.value === 'asc' ? 'sort-asc' : 'sort-desc'
}

function sourceLabel(row) {
  return row.source === 'system' ? 'System' : 'Customer'
}

function detailQuery(row) {
  return { cluster: cluster.value, source: row.source }
}

async function fetchTenants() {
  try {
    tenants.value = await loadTenantOptions()
    const fromQuery = String(route.query.cluster || '')
    if (fromQuery && tenantOptions.value.some((t) => t.shortuid === fromQuery)) {
      cluster.value = fromQuery
    } else if (!cluster.value && tenantOptions.value.length) {
      cluster.value = String(tenantOptions.value[0].shortuid)
    }
  } catch {
    tenants.value = []
  }
}

async function loadStreams() {
  if (!cluster.value) {
    streams.value = []
    loading.value = false
    return
  }
  loading.value = true
  error.value = ''
  deleteError.value = ''
  try {
    const res = await getApiClient().get('provision-streams', {
      params: { cluster: cluster.value }
    })
    streams.value = res.streams ?? []
  } catch (err) {
    error.value = firstErrorMessage(err, 'Failed to load provision streams')
    streams.value = []
  } finally {
    loading.value = false
  }
}

function askConfirmDelete(name) {
  confirmDeleteName.value = name
  deleteError.value = ''
}

function cancelConfirmDelete() {
  confirmDeleteName.value = null
}

async function confirmAndDelete(name) {
  if (confirmDeleteName.value !== name) return
  deleteError.value = ''
  deletingName.value = name
  try {
    const enc = encodeURIComponent(name)
    await getApiClient().delete(
      `provision-streams/${enc}?cluster=${encodeURIComponent(cluster.value)}`
    )
    await loadStreams()
    toast.show('Customer fragment deleted')
  } catch (err) {
    deleteError.value = firstErrorMessage(err, 'Failed to delete')
  } finally {
    confirmDeleteName.value = null
    deletingName.value = null
  }
}

onMounted(async () => {
  await fetchTenants()
  if (!cluster.value) {
    loading.value = false
  }
})

watch(cluster, (next, prev) => {
  const q = { ...route.query }
  if (next) q.cluster = next
  else delete q.cluster
  if (String(route.query.cluster || '') !== String(next || '')) {
    router.replace({ query: q })
  }
  if (next === prev) return
  if (!next) {
    streams.value = []
    loading.value = false
    return
  }
  loadStreams()
})
</script>

<template>
  <div class="list-view">
    <header class="list-header">
      <h1>Provision streams</h1>
      <p class="list-legend">
        System = package stock (read-only). Customer = this tenant’s fragments (survive upgrade). Prefer
        <code>#INCLUDE</code> System first, then <code>site.…</code> Customer.
      </p>
      <p class="toolbar">
        <router-link
          :to="{ name: 'provision-stream-create', query: { cluster } }"
          class="add-btn"
          :class="{ 'add-btn-disabled': !cluster }"
          :aria-disabled="!cluster ? 'true' : undefined"
          @click="
            (e) => {
              if (!cluster) e.preventDefault()
            }
          "
        >
          Create
        </router-link>
        <span class="toolbar-end">
          <label class="tenant-filter">
            Tenant
            <select v-model="cluster" aria-label="Tenant">
              <option v-for="t in tenantOptions" :key="t.shortuid" :value="t.shortuid">
                {{ t.pkey }}
              </option>
            </select>
          </label>
          <input
            v-model="filterText"
            type="search"
            autocomplete="off"
            autocapitalize="off"
            autocorrect="off"
            spellcheck="false"
            class="filter-input"
            placeholder="Filter by name, source, or notes"
            aria-label="Filter provision streams"
          />
        </span>
      </p>
    </header>

    <section
      v-if="loading || error || deleteError || !cluster || listableStreams.length === 0"
      class="list-states"
    >
      <ListLoadingState v-if="loading" message="Loading provision streams…" />
      <p v-else-if="error" class="error">{{ error }}</p>
      <p v-if="deleteError" class="error">{{ deleteError }}</p>
      <div v-else-if="!cluster" class="empty">No tenants available.</div>
      <div v-else-if="listableStreams.length === 0" class="empty">
        No provision streams for this tenant.
      </div>
    </section>

    <section v-else class="list-body">
      <p v-if="filterText && filteredRows.length === 0" class="empty">No streams match the filter.</p>
      <table v-else class="table">
        <thead>
          <tr>
            <th
              class="th-sortable"
              :class="sortClass('name')"
              title="Click to sort"
              @click="setSort('name')"
            >
              Name
            </th>
            <th
              class="th-sortable"
              :class="sortClass('source')"
              title="Click to sort"
              @click="setSort('source')"
            >
              Source
            </th>
            <th
              class="th-sortable"
              :class="sortClass('refcount')"
              title="Click to sort"
              @click="setSort('refcount')"
            >
              Refcount
            </th>
            <th
              class="th-sortable"
              :class="sortClass('notes')"
              title="Click to sort"
              @click="setSort('notes')"
            >
              Notes
            </th>
            <th
              class="th-sortable"
              :class="sortClass('updated_at')"
              title="Click to sort"
              @click="setSort('updated_at')"
            >
              Updated
            </th>
            <th class="th-actions" title="Edit">
              <span class="action-icon" aria-hidden="true">
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  width="1em"
                  height="1em"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                >
                  <path d="M17 3a2.85 2.85 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z" />
                </svg>
              </span>
            </th>
            <th class="th-actions" title="Delete">
              <span class="action-icon" aria-hidden="true">
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  width="1em"
                  height="1em"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                >
                  <path d="M3 6h18" />
                  <path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6" />
                  <path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2" />
                  <line x1="10" x2="10" y1="11" y2="17" />
                  <line x1="14" x2="14" y1="11" y2="17" />
                </svg>
              </span>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="row in sortedRows" :key="row.source + ':' + row.name">
            <td class="cell-immutable" title="Immutable">{{ row.name }}</td>
            <td>{{ sourceLabel(row) }}</td>
            <td>{{ row.refcount ?? 0 }}</td>
            <td>{{ row.notes || '—' }}</td>
            <td>{{ row.updated_at || '—' }}</td>
            <td>
              <router-link
                :to="{
                  name: 'provision-stream-detail',
                  params: { name: row.name },
                  query: detailQuery(row)
                }"
                class="cell-link cell-link-icon"
                title="Edit"
                aria-label="Edit"
              >
                <span class="action-icon" aria-hidden="true">
                  <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="1em"
                    height="1em"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  >
                    <path d="M17 3a2.85 2.85 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z" />
                  </svg>
                </span>
              </router-link>
            </td>
            <td>
              <button
                v-if="row.source === 'customer'"
                type="button"
                class="cell-link cell-link-delete cell-link-icon"
                :disabled="deletingName === row.name"
                :title="deletingName === row.name ? 'Deleting…' : 'Delete'"
                :aria-label="`Delete ${row.name}`"
                @click="askConfirmDelete(row.name)"
              >
                <span class="action-icon" aria-hidden="true">
                  <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="1em"
                    height="1em"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  >
                    <path d="M3 6h18" />
                    <path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6" />
                    <path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2" />
                    <line x1="10" x2="10" y1="11" y2="17" />
                    <line x1="14" x2="14" y1="11" y2="17" />
                  </svg>
                </span>
              </button>
              <span v-else class="cell-link cell-link-icon" title="System streams are read-only" style="opacity: 0.5"
                >—</span
              >
            </td>
          </tr>
        </tbody>
      </table>
    </section>

    <DeleteConfirmModal
      :show="!!confirmDeleteName"
      title="Delete customer fragment?"
      :loading="deletingName === confirmDeleteName"
      @confirm="confirmDeleteName && confirmAndDelete(confirmDeleteName)"
      @cancel="cancelConfirmDelete"
    >
      <template #body>
        <p>
          Delete fragment <strong>{{ confirmDeleteName }}</strong> for tenant
          <strong>{{ tenantPkeyLabel }}</strong
          >? System stock is unchanged.
        </p>
      </template>
    </DeleteConfirmModal>
  </div>
</template>

<style scoped>
.list-view {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.list-header {
  margin: 0;
}
.list-legend {
  margin: 0.25rem 0 0;
  max-width: 48rem;
  color: var(--color-muted, #64748b);
  font-size: 0.95rem;
  line-height: 1.4;
}
.list-states {
  margin: 0;
}
.list-body {
  margin: 0;
}
.error,
.empty {
  margin-top: 0;
}
.error {
  color: #dc2626;
}
.table {
  margin-top: 0;
  width: 100%;
  border-collapse: collapse;
  font-size: 0.9375rem;
}
.table th,
.table td {
  padding: 0.5rem 0.75rem;
  text-align: left;
  border-bottom: 1px solid #e2e8f0;
}
.table th {
  font-weight: 600;
  color: #475569;
  background: #f8fafc;
}
.cell-immutable {
  color: var(--pbx-text-muted);
  background: transparent;
}
.th-sortable {
  cursor: pointer;
  user-select: none;
  white-space: nowrap;
}
.th-sortable::before {
  content: '\21C5';
  font-size: 0.7em;
  color: #94a3b8;
  margin-left: 0.2em;
  font-weight: normal;
}
.th-sortable.sort-asc::before,
.th-sortable.sort-desc::before {
  content: none;
}
.th-sortable:hover {
  background: #f1f5f9;
}
.th-sortable.sort-asc::after {
  content: ' \2191';
  font-size: 0.75em;
  color: #64748b;
}
.th-sortable.sort-desc::after {
  content: ' \2193';
  font-size: 0.75em;
  color: #64748b;
}
.th-actions {
  cursor: default;
  white-space: nowrap;
}
.th-actions .action-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  color: #64748b;
}
.action-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
}
.cell-link-icon {
  padding: 0.25rem;
}
.cell-link-icon .action-icon {
  color: inherit;
}
.table tbody tr:hover {
  background: #f8fafc;
}
.cell-link {
  color: #2563eb;
  text-decoration: none;
}
.cell-link:hover {
  text-decoration: underline;
}
.cell-link-delete {
  color: #dc2626;
  background: none;
  border: none;
  padding: 0;
  font: inherit;
  cursor: pointer;
}
.cell-link-delete:hover:not(:disabled) {
  text-decoration: underline;
}
.cell-link-delete:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}
.toolbar {
  margin: 0.75rem 0 0 0;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
}
.toolbar-end {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex-wrap: wrap;
  margin-left: auto;
}
.tenant-filter {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  font-size: 0.9rem;
  color: #475569;
}
.tenant-filter select {
  padding: 0.45rem 0.6rem;
  font-size: 0.9375rem;
  border: 1px solid #e2e8f0;
  border-radius: 0.375rem;
  background: #fff;
}
.add-btn {
  display: inline-block;
  padding: 0.5rem 1rem;
  font-size: 0.9375rem;
  font-weight: 500;
  color: #fff;
  background: #2563eb;
  border-radius: 0.375rem;
  text-decoration: none;
}
.add-btn:hover {
  background: #1d4ed8;
}
.add-btn-disabled {
  opacity: 0.5;
  pointer-events: none;
}
.filter-input {
  padding: 0.5rem 0.75rem;
  font-size: 0.9375rem;
  border: 1px solid #e2e8f0;
  border-radius: 0.375rem;
  min-width: 16rem;
}
.filter-input:focus {
  outline: none;
  border-color: #2563eb;
}
</style>
