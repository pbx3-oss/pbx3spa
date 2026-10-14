<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { getApiClient } from '@/api/client'
import { useToastStore } from '@/stores/toast'
import { loadTenantOptions } from '@/utils/loadTenantOptions'
import { firstErrorMessage } from '@/utils/formErrors'
import FormField from '@/components/forms/FormField.vue'
import FormReadonly from '@/components/forms/FormReadonly.vue'
import DeleteConfirmModal from '@/components/DeleteConfirmModal.vue'
import PanelBackLink from '@/components/PanelBackLink.vue'
import { useUnsavedForm } from '@/composables/useUnsavedForm'

const route = useRoute()
const router = useRouter()
const toast = useToastStore()
const { markDirty, beginHydrate, markClean } = useUnsavedForm()

const name = computed(() => decodeURIComponent(String(route.params.name || '')))
const cluster = computed(() => String(route.query.cluster || ''))

const row = ref(null)
const tenants = ref([])
const loading = ref(true)
const error = ref('')
const editBody = ref('')
const editNotes = ref('')
const warnings = ref([])
const saving = ref(false)
const saveError = ref('')
const deleting = ref(false)
const deleteError = ref('')
const confirmDeleteOpen = ref(false)
const confirmForceOpen = ref(false)
const confirmEmptyOpen = ref(false)
const copyTo = ref('')
const copying = ref(false)

const isSystem = computed(() => row.value?.source === 'system' || row.value?.read_only === true)

const tenantPkeyLabel = computed(() => {
  const cl = cluster.value
  if (!cl) return '—'
  for (const t of tenants.value) {
    if (String(t.shortuid) === cl || String(t.pkey) === cl) return t.pkey ?? cl
  }
  return cl
})

async function fetchTenants() {
  try {
    tenants.value = await loadTenantOptions()
  } catch {
    tenants.value = []
  }
}

async function fetchRow() {
  if (!name.value || !cluster.value) {
    error.value = 'Missing name or cluster'
    loading.value = false
    return
  }
  beginHydrate()
  loading.value = true
  error.value = ''
  try {
    const enc = encodeURIComponent(name.value)
    row.value = await getApiClient().get(`provision-streams/${enc}`, {
      params: { cluster: cluster.value }
    })
    editBody.value = row.value.body ?? ''
    editNotes.value = row.value.notes ?? ''
    warnings.value = []
    copyTo.value = name.value.startsWith('site.') ? name.value : `site.${name.value}`
    markClean()
  } catch (err) {
    error.value = firstErrorMessage(err, 'Failed to load stream')
    row.value = null
  } finally {
    loading.value = false
  }
}

function goBack() {
  router.push({ name: 'provision-streams', query: { cluster: cluster.value } })
}

async function doSave() {
  saving.value = true
  saveError.value = ''
  try {
    const enc = encodeURIComponent(name.value)
    const q = `?cluster=${encodeURIComponent(cluster.value)}`
    const res = await getApiClient().put(`provision-streams/${enc}${q}`, {
      body: editBody.value,
      notes: editNotes.value
    })
    row.value = res
    warnings.value = res.warnings ?? []
    markClean()
    toast.show('Provision stream updated')
    if (warnings.value.length) {
      toast.show(warnings.value[0])
    }
  } catch (err) {
    saveError.value = firstErrorMessage(err, 'Failed to save')
  } finally {
    saving.value = false
  }
}

function onSaveSubmit(e) {
  e.preventDefault()
  if (String(editBody.value || '').trim() === '') {
    confirmEmptyOpen.value = true
    return
  }
  doSave()
}

function askConfirmDelete() {
  deleteError.value = ''
  confirmForceOpen.value = false
  confirmDeleteOpen.value = true
}

function cancelConfirmDelete() {
  confirmDeleteOpen.value = false
  confirmForceOpen.value = false
  deleteError.value = ''
}

async function doDelete(force = false) {
  deleting.value = true
  deleteError.value = ''
  try {
    const enc = encodeURIComponent(name.value)
    const forceQ = force ? '&force=1' : ''
    await getApiClient().delete(
      `provision-streams/${enc}?cluster=${encodeURIComponent(cluster.value)}${forceQ}`
    )
    toast.show('Customer fragment deleted')
    router.push({ name: 'provision-streams', query: { cluster: cluster.value } })
  } catch (err) {
    const status = err?.response?.status
    if (status === 409 && !force) {
      deleteError.value = firstErrorMessage(err, 'Still referenced')
      confirmDeleteOpen.value = false
      confirmForceOpen.value = true
      return
    }
    deleteError.value = firstErrorMessage(err, 'Failed to delete')
  } finally {
    deleting.value = false
  }
}

async function copyFromSystem() {
  if (!copyTo.value) return
  copying.value = true
  saveError.value = ''
  try {
    const res = await getApiClient().post('provision-streams/copy-from-system', {
      cluster: cluster.value,
      from: name.value,
      to: copyTo.value
    })
    toast.show('Copied to customer fragment')
    router.push({
      name: 'provision-stream-detail',
      params: { name: res.name },
      query: { cluster: cluster.value, source: 'customer' }
    })
  } catch (err) {
    saveError.value = firstErrorMessage(err, 'Copy failed')
  } finally {
    copying.value = false
  }
}

onMounted(async () => {
  await fetchTenants()
  await fetchRow()
})
watch(() => [route.params.name, route.query.cluster], fetchRow)
</script>

<template>
  <div class="detail-view" @input="markDirty" @change="markDirty">
    <PanelBackLink :to="{ name: 'provision-streams', query: { cluster } }" label="Provision streams">
      <h1>Provision stream</h1>
    </PanelBackLink>

    <p v-if="loading" class="loading">Loading…</p>
    <p v-else-if="error" class="error">{{ error }}</p>

    <template v-else-if="row">
      <div class="detail-content">
        <!-- System (read-only) -->
        <form v-if="isSystem" class="edit-form" @submit.prevent>
          <p v-if="saveError" class="error" role="alert">{{ saveError }}</p>

          <div class="edit-actions edit-actions-top">
            <button type="button" class="secondary" @click="goBack">Cancel</button>
          </div>

          <h2 class="detail-heading">Identity</h2>
          <div class="form-fields">
            <FormReadonly id="name" label="Name" :value="name" />
            <FormReadonly id="tenant" label="Tenant" :value="tenantPkeyLabel" />
            <FormReadonly id="source" label="Source" value="System (read-only)" />
            <FormReadonly
              id="refcount"
              label="Refcount"
              :value="row.refcount != null ? String(row.refcount) : '0'"
            />
          </div>

          <h2 class="detail-heading">Source</h2>
          <div class="form-fields body-field">
            <FormField
              id="body"
              :model-value="row.body ?? ''"
              label="Body"
              multiline
              :rows="32"
              disabled
              hint="Package stock — copy to a customer fragment to customize."
            />
          </div>

          <h2 class="detail-heading">Copy to customer</h2>
          <div class="form-fields">
            <FormField
              id="copy-to"
              v-model="copyTo"
              label="Customer name"
              hint="Prefer site.… — must not match a System filename."
            />
          </div>

          <div class="edit-actions">
            <button type="button" :disabled="copying || !copyTo" @click="copyFromSystem">
              {{ copying ? 'Copying…' : 'Copy to customer' }}
            </button>
            <button type="button" class="secondary" @click="goBack">Cancel</button>
          </div>
        </form>

        <!-- Customer (editable) -->
        <form v-else class="edit-form" @submit="onSaveSubmit">
          <p v-if="saveError" class="error" role="alert">{{ saveError }}</p>
          <p v-if="deleteError && !confirmForceOpen" class="error" role="alert">{{ deleteError }}</p>

          <div class="edit-actions edit-actions-top">
            <button type="submit" :disabled="saving">{{ saving ? 'Saving…' : 'Save' }}</button>
            <button type="button" class="secondary" @click="goBack">Cancel</button>
            <button
              type="button"
              class="action-delete"
              :disabled="deleting"
              @click="askConfirmDelete"
            >
              {{ deleting ? 'Deleting…' : 'Delete' }}
            </button>
          </div>

          <h2 class="detail-heading">Identity</h2>
          <div class="form-fields">
            <FormReadonly id="name" label="Name" :value="name" />
            <FormReadonly id="tenant" label="Tenant" :value="tenantPkeyLabel" />
            <FormReadonly id="source" label="Source" value="Customer" />
            <FormReadonly
              id="refcount"
              label="Refcount"
              :value="row.refcount != null ? String(row.refcount) : '0'"
            />
            <FormField id="notes" v-model="editNotes" label="Notes" />
          </div>

          <h2 class="detail-heading">Source</h2>
          <div class="form-fields body-field">
            <FormField
              id="body"
              v-model="editBody"
              label="Body"
              multiline
              :rows="32"
              hint="Lines to add or override. Prefer #INCLUDE System first, then site.… fragments."
            />
          </div>

          <ul v-if="warnings.length" class="warnings">
            <li v-for="(w, i) in warnings" :key="i">{{ w }}</li>
          </ul>

          <div class="edit-actions">
            <button type="submit" :disabled="saving">{{ saving ? 'Saving…' : 'Save' }}</button>
            <button type="button" class="secondary" @click="goBack">Cancel</button>
            <button
              type="button"
              class="action-delete"
              :disabled="deleting"
              @click="askConfirmDelete"
            >
              {{ deleting ? 'Deleting…' : 'Delete' }}
            </button>
          </div>
        </form>
      </div>

      <DeleteConfirmModal
        :show="confirmDeleteOpen"
        title="Delete customer fragment?"
        :loading="deleting"
        @confirm="doDelete(false)"
        @cancel="cancelConfirmDelete"
      >
        <template #body>
          <p>
            Delete fragment <strong>{{ name }}</strong> for tenant
            <strong>{{ tenantPkeyLabel }}</strong
            >? If extensions still <code>#INCLUDE</code> it, you will be asked to force delete.
          </p>
        </template>
      </DeleteConfirmModal>

      <DeleteConfirmModal
        :show="confirmForceOpen"
        title="Force delete?"
        confirm-label="Force delete"
        :loading="deleting"
        @confirm="doDelete(true)"
        @cancel="cancelConfirmDelete"
      >
        <template #body>
          <p>{{ deleteError }}</p>
          <p>Force delete will remove the fragment even if extensions still reference it.</p>
        </template>
      </DeleteConfirmModal>

      <DeleteConfirmModal
        :show="confirmEmptyOpen"
        title="Save empty body?"
        body-text="Body is empty. Save anyway?"
        confirm-label="Save empty"
        @cancel="confirmEmptyOpen = false"
        @confirm="
          () => {
            confirmEmptyOpen = false
            doSave()
          }
        "
      />
    </template>
  </div>
</template>

<style scoped>
.detail-view {
  max-width: 52rem;
}
.loading,
.error {
  margin-top: 1rem;
}
.error {
  color: #dc2626;
}
.detail-content {
  margin-top: 1rem;
}
.edit-form {
  margin-bottom: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  max-width: 52rem;
}
.detail-heading {
  font-size: 1rem;
  font-weight: 600;
  margin: 0.5rem 0 0;
  color: #334155;
}
.form-fields {
  display: flex;
  flex-direction: column;
}
.edit-actions {
  display: flex;
  gap: 0.5rem;
}
.edit-actions-top {
  margin-top: 0;
}
.edit-actions button {
  padding: 0.375rem 0.75rem;
  font-size: 0.875rem;
  font-weight: 500;
  border-radius: 0.375rem;
  cursor: pointer;
}
.edit-actions button[type='submit'],
.edit-actions button:not(.secondary):not(.action-delete) {
  color: #fff;
  background: #2563eb;
  border: none;
}
.edit-actions button[type='submit']:disabled,
.edit-actions button:not(.secondary):not(.action-delete):disabled {
  opacity: 0.7;
  cursor: not-allowed;
}
.edit-actions button.secondary {
  color: #64748b;
  background: transparent;
  border: 1px solid #e2e8f0;
}
.edit-actions button.secondary:hover {
  background: #f1f5f9;
}
.edit-actions button.action-delete {
  color: #fff;
  background: #dc2626;
  border: none;
}
.edit-actions button.action-delete:hover:not(:disabled) {
  background: #b91c1c;
}
.edit-actions button.action-delete:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}
.warnings {
  color: #a16207;
  margin: 0;
  padding-left: 1.25rem;
}
/* Deep source editor — taller than default FormField textarea */
.body-field :deep(.form-input-textarea) {
  min-height: 28rem;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 0.85rem;
  line-height: 1.45;
}
</style>
