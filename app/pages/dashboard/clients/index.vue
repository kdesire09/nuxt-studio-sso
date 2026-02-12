<script setup lang="ts">
interface OAuthClient {
  id: string
  name: string
  websiteUrl: string
  previewUrlPattern: string | null
  callbackUrl: string
  isActive: boolean
  createdAt: string
  ownerName: string | null
}

useSeoMeta({
  title: 'OAuth Clients - Nuxt Studio SSO',
  description: 'Manage your registered OAuth client applications.'
})

definePageMeta({
  middleware: 'auth'
})

const { user } = useUserSession()

// Redirect non-admins to dashboard
if (!user.value?.isAdmin) {
  await navigateTo('/dashboard')
}

const { data: clients, refresh } = await useFetch<OAuthClient[]>('/api/clients')

const showCreateModal = ref(false)
const newClient = ref({
  name: '',
  websiteUrl: '',
  previewUrlPattern: ''
})
const createdSecret = ref<string | null>(null)
const createdClientId = ref<string | null>(null)
const creating = ref(false)
const copied = ref(false)

async function createClient() {
  creating.value = true
  try {
    const result = await $fetch<OAuthClient & { secret: string }>('/api/clients', {
      method: 'POST',
      body: {
        name: newClient.value.name,
        websiteUrl: newClient.value.websiteUrl,
        previewUrlPattern: newClient.value.previewUrlPattern || undefined
      }
    })

    createdSecret.value = result.secret
    createdClientId.value = result.id
    await refresh()

    newClient.value = {
      name: '',
      websiteUrl: '',
      previewUrlPattern: ''
    }
  } catch (error) {
    console.error('Failed to create client:', error)
  } finally {
    creating.value = false
  }
}

function closeModal() {
  showCreateModal.value = false
  createdSecret.value = null
  createdClientId.value = null
  copied.value = false
}

async function copySecret() {
  if (createdSecret.value) {
    await navigator.clipboard.writeText(createdSecret.value)
    copied.value = true
    setTimeout(() => copied.value = false, 2000)
  }
}

async function deleteClient(id: string) {
  if (!confirm('Are you sure you want to delete this client? This action cannot be undone.')) {
    return
  }

  try {
    await $fetch(`/api/clients/${id}`, { method: 'DELETE' })
    await refresh()
  } catch (error) {
    console.error('Failed to delete client:', error)
  }
}
</script>

<template>
  <div class="py-8">
    <!-- Page header -->
    <div class="flex flex-col sm:flex-row justify-between items-start sm:items-center gap-4 mb-8">
      <div>
        <h1 class="text-2xl font-bold text-highlighted">
          OAuth Clients
        </h1>
        <p class="text-sm text-muted mt-1">
          Manage your registered OAuth client applications
        </p>
      </div>
      <UButton icon="i-heroicons-plus" @click="showCreateModal = true">
        New Client
      </UButton>
    </div>

    <!-- Empty state -->
    <UEmpty
      v-if="clients?.length === 0"
      icon="i-heroicons-key"
      title="No clients yet"
      description="Create your first OAuth client to enable authentication for your Nuxt Studio sites."
      :actions="[{ label: 'Create Your First Client', icon: 'i-heroicons-plus', onClick: () => { showCreateModal = true } }]"
    />

    <!-- Client list -->
    <div v-else class="grid gap-4">
      <UCard v-for="client in clients" :key="client.id">
        <div class="flex flex-col lg:flex-row lg:items-center justify-between gap-4">
          <div class="flex-1 min-w-0">
            <div class="flex items-center gap-3 mb-2">
              <div class="p-2 bg-primary/10 rounded-lg inline-flex shrink-0">
                <UIcon name="i-heroicons-key" class="size-5 text-primary" />
              </div>
              <div class="min-w-0">
                <div class="flex items-center gap-2">
                  <h3 class="font-semibold text-highlighted truncate">
                    {{ client.name }}
                  </h3>
                  <UBadge
                    v-if="client.isActive"
                    color="success"
                    variant="subtle"
                    size="sm"
                  >
                    Active
                  </UBadge>
                  <UBadge
                    v-else
                    color="error"
                    variant="subtle"
                    size="sm"
                  >
                    Inactive
                  </UBadge>
                </div>
                <p class="text-xs text-muted font-mono truncate">
                  {{ client.id }}
                </p>
              </div>
            </div>

            <div class="ml-11 space-y-1">
              <UButton
                :to="client.websiteUrl"
                :label="client.websiteUrl"
                target="_blank"
                external
                size="xs"
                trailing-icon="i-lucide-arrow-up-right"
                variant="link"
                color="neutral"
                square
              />
              <p v-if="client.previewUrlPattern" class="text-xs text-dimmed truncate px-1">
                Preview: {{ client.previewUrlPattern }}
              </p>
              <p v-if="client.ownerName" class="text-xs text-dimmed px-1">
                Created by {{ client.ownerName }}
              </p>
            </div>
          </div>

          <div class="flex items-center gap-2 ml-11 lg:ml-0">
            <UButton
              :to="`/dashboard/clients/${client.id}`"
              color="neutral"
              variant="soft"
              size="sm"
              leading-icon="i-heroicons-pencil"
            >
              Edit
            </UButton>
            <UButton
              color="error"
              variant="ghost"
              size="sm"
              icon="i-heroicons-trash"
              @click="deleteClient(client.id)"
            />
          </div>
        </div>
      </UCard>
    </div>

    <!-- Create Modal -->
    <UModal
      v-model:open="showCreateModal"
      :title="createdSecret ? 'Client Created Successfully' : 'Create New Client'"
      @close="closeModal"
    >
      <template #body>
        <!-- Success state -->
        <div v-if="createdSecret" class="space-y-4">
          <UAlert
            color="warning"
            icon="i-heroicons-exclamation-triangle"
            title="Save your client secret"
            description="This secret will only be shown once. Make sure to copy and store it securely."
          />

          <UFormField label="STUDIO_SSO_CLIENT_ID">
            <UInput :model-value="createdClientId" readonly class="font-mono" />
          </UFormField>

          <UFormField label="STUDIO_SSO_CLIENT_SECRET">
            <div class="flex gap-2">
              <UInput
                :model-value="createdSecret"
                readonly
                class="font-mono flex-1"
                type="password"
              />
              <UButton
                :icon="copied ? 'i-heroicons-check' : 'i-heroicons-clipboard-document'"
                :color="copied ? 'success' : 'neutral'"
                variant="soft"
                @click="copySecret"
              />
            </div>
          </UFormField>
        </div>

        <!-- Create form -->
        <form v-else class="space-y-4" @submit.prevent="createClient">
          <UFormField label="Client Name" required>
            <UInput v-model="newClient.name" placeholder="My Nuxt Studio Site" class="w-full" />
          </UFormField>

          <UFormField label="Website URL" description="The callback path will be added automatically." required>
            <UInput
              v-model="newClient.websiteUrl"
              placeholder="https://docs.example.com"
              type="url"
              class="w-full"
            />
          </UFormField>

          <UFormField label="Preview URL Pattern" description="Use * as wildcard for preview deployments." hint="Optional">
            <UInput
              v-model="newClient.previewUrlPattern"
              placeholder="https://my-docs-*.vercel.app"
              class="w-full"
            />
          </UFormField>
        </form>
      </template>

      <template #footer>
        <template v-if="createdSecret">
          <UButton block @click="closeModal">
            Done
          </UButton>
        </template>
        <template v-else>
          <UButton color="neutral" variant="outline" @click="closeModal">
            Cancel
          </UButton>
          <UButton :loading="creating" @click="createClient">
            Create Client
          </UButton>
        </template>
      </template>
    </UModal>
  </div>
</template>
