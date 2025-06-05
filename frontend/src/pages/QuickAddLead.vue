<template>
  <LayoutHeader>
    <template #left-header>
      <div class="text-2xl font-semibold">{{ __('Quick Add Lead') }}</div>
    </template>
  </LayoutHeader>
  <div class="p-5 flex flex-col gap-4">
    <FieldLayout v-if="tabs.data" :tabs="tabs.data" :data="lead" />
    <ErrorMessage v-if="error" class="mt-2" :message="__(error)" />
    <div class="flex flex-row-reverse gap-2">
      <Button
        variant="solid"
        :label="__('Create')"
        :loading="isLeadCreating"
        @click="createNewLead"
      />
    </div>
  </div>
</template>

<script setup>
import LayoutHeader from '@/components/LayoutHeader.vue'
import FieldLayout from '@/components/FieldLayout/FieldLayout.vue'
import { statusesStore } from '@/stores/statuses'
import { usersStore } from '@/stores/users'
import { Button, ErrorMessage, createResource } from 'frappe-ui'
import { ref, reactive, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'

const { getLeadStatus, statusOptions } = statusesStore()
const { getUser } = usersStore()
const router = useRouter()

const error = ref(null)
const isLeadCreating = ref(false)

const lead = reactive({
  first_name: '',
  status: '',
})

const tabs = createResource({
  url: 'crm.fcrm.doctype.crm_fields_layout.crm_fields_layout.get_fields_layout',
  cache: ['QuickAddLead', 'CRM Lead'],
  params: { doctype: 'CRM Lead', type: 'Required Fields' },
  auto: true,
  transform: (_tabs) => {
    _tabs.forEach((tab) => {
      tab.sections.forEach((section) => {
        section.columns.forEach((column) => {
          column.fields.forEach((field) => {
            if (field.fieldname == 'status') {
              field.fieldtype = 'Select'
              field.options = leadStatuses.value
              field.prefix = getLeadStatus(lead.status).color
            }
          })
        })
      })
    })
  },
})

const leadStatuses = computed(() => {
  let statuses = statusOptions('lead')
  if (!lead.status) {
    lead.status = statuses?.[0]?.value
  }
  return statuses
})

const createLead = createResource({
  url: 'frappe.client.insert',
  makeParams(values) {
    return {
      doc: {
        doctype: 'CRM Lead',
        ...values,
      },
    }
  },
})

function createNewLead() {
  createLead.submit(lead, {
    validate() {
      error.value = null
      if (!lead.first_name) {
        error.value = __('First Name is mandatory')
        return error.value
      }
      if (!lead.status) {
        error.value = __('Status is required')
        return error.value
      }
      isLeadCreating.value = true
    },
    onSuccess(data) {
      isLeadCreating.value = false
      router.push({ name: 'Lead', params: { leadId: data.name } })
    },
    onError(err) {
      isLeadCreating.value = false
      if (!err.messages) {
        error.value = err.message
        return
      }
      error.value = err.messages.join('\n')
    },
  })
}

onMounted(() => {
  if (!lead.lead_owner) {
    lead.lead_owner = getUser().name
  }
  if (!lead.status && leadStatuses.value[0]?.value) {
    lead.status = leadStatuses.value[0].value
  }
})
</script>
