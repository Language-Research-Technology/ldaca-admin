<script setup lang="ts">
import { ref } from 'vue'

interface Collection {
  id: string
  name: string
  path: string
}

const apiKey = ref('')
const collections = ref<Collection[]>([])
const errorMessage = ref('')
const statusMessage = ref('')

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
  }
}

const indexAll = async () => {
  statusMessage.value = ''

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
  } catch (error) {
    statusMessage.value = 'Indexing failed!'
  }
}

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
  } catch (error) {
    statusMessage.value = 'Delete failed!'
  }
}

</script>

<template>
  <div>
    <h1>LDACA Admin</h1>

    <div style="margin-top:20px">
      <button @click="indexAll">
        Index All
      </button>

      <button @click="deleteAll" style="margin-left:10px">
        Delete All
      </button>
    </div>

    <p v-if="statusMessage">
      {{ statusMessage }}
    </p>
    
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

    <ul>
      <li v-for="collection in collections" :key="collection.id">
        {{ collection.name }}
      </li>
    </ul>
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