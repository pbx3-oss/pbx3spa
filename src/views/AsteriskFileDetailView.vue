<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { getApiClient } from '@/api/client'
import { useToastStore } from '@/stores/toast'
import { firstErrorMessage } from '@/utils/formErrors'
import PanelBackLink from '@/components/PanelBackLink.vue'
import FormField from '@/components/forms/FormField.vue'
import { useUnsavedForm } from '@/composables/useUnsavedForm'
import { refreshCommitStatusUi } from '@/utils/commitStatus'
const route = useRoute()
const router = useRouter()
const toast = useToastStore()
const { markDirty, beginHydrate, markClean } = useUnsavedForm()

const filename = computed(() => route.params.filename)
const content = ref('')
const readonly = ref(true)
const loading = ref(true)
const error = ref('')
const saving = ref(false)
const saveError = ref('')
const editContent = ref('')

async function loadFile() {
  if (!filename.value) return
  beginHydrate()
  loading.value = true
  error.value = ''
  saveError.value = ''
  try {
    const res = await getApiClient().get(`astfiles/${encodeURIComponent(filename.value)}`)
    content.value = res.content ?? ''
    readonly.value = res.readonly === true
    editContent.value = res.content ?? ''
  } catch (err) {
    error.value = firstErrorMessage(err, 'Failed to load file')
    content.value = ''
    editContent.value = ''
  } finally {
    loading.value = false
    await markClean()
  }
}

function goBack() {
  router.push({ name: 'asterisk-files' })
}

async function saveEdit(e) {
  e.preventDefault()
  if (readonly.value) return
  saveError.value = ''
  saving.value = true
  try {
    await getApiClient().put(`astfiles/${encodeURIComponent(filename.value)}`, {
      content: editContent.value
    })
    toast.show('File updated')
    refreshCommitStatusUi()
    content.value = editContent.value
  } catch (err) {
    saveError.value = firstErrorMessage(err, 'Failed to save file')
    toast.show(saveError.value, 'error')
  } finally {
    saving.value = false
  }
}

onMounted(loadFile)
watch(filename, loadFile)
</script>

<template>
  <div class="astfile-detail-view">
    <PanelBackLink :to="{ name: 'asterisk-files' }" label="Asterisk Files" class="detail-header">
      <h1>{{ filename || 'File' }}</h1>
    </PanelBackLink>

    <section v-if="loading || error" class="detail-states">
      <p v-if="loading" class="loading">Loading file…</p>
      <p v-else-if="error" class="error">{{ error }}</p>
    </section>

    <template v-else>
      <!-- Read-only: same disabled FormField treatment as Provision streams Body -->
      <form v-if="readonly" class="edit-form" @submit.prevent>
        <div class="edit-actions edit-actions-top">
          <button type="button" class="secondary" @click="goBack">Cancel</button>
        </div>
        <div class="form-fields body-field">
          <FormField
            id="astfile-content"
            :model-value="content"
            label="Contents"
            multiline
            :rows="24"
            disabled
            hint="Read-only — this file cannot be edited here."
          />
        </div>
        <div class="edit-actions">
          <button type="button" class="secondary" @click="goBack">Cancel</button>
        </div>
      </form>
      <form v-else class="edit-form" @submit="saveEdit" @input="markDirty" @change="markDirty">
        <div class="edit-actions edit-actions-top">
          <button type="submit" :disabled="saving">
            {{ saving ? 'Saving…' : 'Save' }}
          </button>
          <button type="button" class="secondary" @click="goBack">Cancel</button>
        </div>
        <p v-if="saveError" class="error">{{ saveError }}</p>
        <div class="form-fields body-field">
          <FormField
            id="astfile-content"
            v-model="editContent"
            label="Contents"
            multiline
            :rows="24"
          />
        </div>
        <div class="edit-actions">
          <button type="submit" :disabled="saving">
            {{ saving ? 'Saving…' : 'Save' }}
          </button>
          <button type="button" class="secondary" @click="goBack">Cancel</button>
        </div>
      </form>
    </template>
  </div>
</template>

<style scoped>
.astfile-detail-view {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  /* Left-align with other panels (no horizontal centering) */
  max-width: none;
}
.detail-header h1 {
  margin: 0;
}
.detail-states .loading,
.detail-states .error {
  margin: 0;
}
.loading {
  color: #64748b;
}
.error {
  color: #b91c1c;
}
.edit-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.form-fields {
  display: flex;
  flex-direction: column;
}
.body-field {
  max-width: 70rem;
}
.edit-actions {
  display: flex;
  gap: 0.5rem;
}
.edit-actions button {
  padding: 0.375rem 0.75rem;
  font-size: 0.875rem;
  font-weight: 500;
  border-radius: 0.375rem;
  cursor: pointer;
}
.edit-actions button[type='submit'] {
  color: #fff;
  background: #2563eb;
  border: none;
}
.edit-actions button[type='submit']:disabled {
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
/* Label above, full-width body — box centered with the panel (not right-column grid) */
.body-field :deep(.form-field) {
  grid-template-columns: 1fr;
  gap: 0.375rem;
}
.body-field :deep(.form-field-label) {
  padding-top: 0;
}
.body-field :deep(.form-input-textarea) {
  min-height: 28rem;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 0.85rem;
  line-height: 1.45;
}
</style>
