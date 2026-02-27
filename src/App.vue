<script setup lang="ts">
import { ref } from 'vue'

interface Collection {
  id: string
  name: string
  path: string
  indexed?: boolean
}

const apiKey = ref('')
const collections = ref<Collection[]>([])
const errorMessage = ref('')
const statusMessage = ref('')
const statusType = ref<'success' | 'error'>('success')


// Fetch available repos from API
const fetchCollections = async () => {
  errorMessage.value = ''
  collections.value = []

  try {
    const response = await fetch('http://localhost:8080/admin/repository', {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${apiKey.value}`
      }
    })

    if (!response.ok) {
      throw new Error('Unauthorized or server error')
    }

    const data = await response.json()
    collections.value = data
  } catch (error) {
    errorMessage.value = 'Failed to fetch collections'
    statusType.value = 'success'
  }
}


// Index all fetched repos
const indexAll = async () => {
  statusMessage.value = ''

  if (collections.value.every(c => c.indexed)) {
    statusMessage.value = 'All repos have been indexed.'
    statusType.value = 'success'
    return
  }

  try {
    const response = await fetch('http://localhost:8080/admin/index/', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiKey.value}`
      }
    })

    if (!response.ok) {
      throw new Error('Index failed')
    }

    statusMessage.value = 'Indexing started successfully!'
    statusType.value = 'success'
    collections.value.forEach(c => c.indexed = true)
  } catch (error) {
    statusMessage.value = 'Indexing failed!'
    statusType.value = 'error'
  }
}


// Delete all fetched repos
const deleteAll = async () => {
  statusMessage.value = ''

  try {
    const response = await fetch('http://localhost:8080/admin/index/', {
      method: 'DELETE',
      headers: {
        'Authorization': `Bearer ${apiKey.value}`
      }
    })

    if (!response.ok) {
      throw new Error('Delete failed')
    }

    statusMessage.value = 'All indexes deleted successfully!'
    statusType.value = 'success'
  } catch (error) {
    statusMessage.value = 'Delete failed!'
    statusType.value = 'error'
  }
}


// Index specific fetched repos
const indexCollection = async (collectionId: string, collectionName: string) => {
  statusMessage.value = ''
  const collection = collections.value.find(c => c.id === collectionId)
  if (collection?.indexed) return

  try {
    const response = await fetch(`http://localhost:8080/admin/index/${encodeURIComponent(collectionId)}`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiKey.value}`
      }
    })

    if (!response.ok) {
      throw new Error('Index failed')
    }

    statusMessage.value = `Indexing started for ${collectionName}!`
    statusType.value = 'success'
    if (collection) collection.indexed = true
  } catch (error) {
    statusMessage.value = `Indexing failed for ${collectionName}!`
    statusType.value = 'error'
  }
}


// Delete specific fetched repos
const deleteCollection = async (collectionId: string, collectionName: string) => {
  statusMessage.value = ''
  
  try {
    const response = await fetch(`http://localhost:8080/admin/index/${encodeURIComponent(collectionId)}`, {
      method: 'DELETE',
      headers: {
        'Authorization': `Bearer ${apiKey.value}`
      }
    })

    if (!response.ok) {
      throw new Error('Delete failed')
    }

    statusMessage.value = `Deleted index for ${collectionName} successfully!`
    statusType.value = 'success'

    const collection = collections.value.find(c => c.id === collectionId)
    if (collection) collection.indexed = false
  } catch (error) {
    statusMessage.value = `Delete failed for ${collectionName}!`
    statusType.value = 'error'
  }
}

</script>

<template>
  <div>
    <h1>LDACA Admin</h1>

    <div>
      <input
        v-model="apiKey"
        placeholder="Enter your API key"
      />
      <button @click="fetchCollections">
        Load Collections
      </button>
    </div>

    <p v-if="errorMessage" style="color:red">
      {{ errorMessage }}
    </p>

    <div style="display:flex; justify-content:flex-end; margin-bottom:10px;">
      <button @click="indexAll">
        Index All
      </button>

      <button @click="deleteAll" style="margin-left:10px">
        Delete All
      </button>
    </div>

    <p v-if="statusMessage" :style="{ color: statusType === 'success' ? 'green' : 'red', fontWeight: 'bold' }">
      {{ statusMessage }}
    </p>

    <table v-if="collections.length" border="1" cellpadding="5" cellspacing="0" style="margin-top:20px; width: 100%;">
      <thead>
        <tr>
          <th>Name</th>
          <th>Action</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="collection in collections" :key="collection.id">
          <td>{{ collection.name }}</td>
          <td style="text-align: center;">
            <button @click="indexCollection(collection.id, collection.name)"
                    :disabled="collection.indexed">
              {{ collection.indexed ? 'Indexed' : 'Index' }}
            </button>
            <button @click="deleteCollection(collection.id, collection.name)"
                    :disabled="!collection.indexed"
                    style="margin-left:5px">
              Delete
            </button>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>


<style scoped>
input {
  margin-right: 10px;
  padding: 5px;
}
button {
  padding: 5px 10px;
}
</style>