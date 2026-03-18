<script setup lang="ts">
import { ref } from 'vue'
import { ElTree } from 'element-plus'

interface Collection {
  id: string
  name: string
  path: string
  indexed?: boolean
  children?: Collection[]
}

const apiBase = import.meta.env.VITE_API_BASE_URL
const apiKey = import.meta.env.VITE_API_TOKEN
const collections = ref<Collection[]>([])
const errorMessage = ref('')
const statusMessage = ref('')
const statusType = ref<'success' | 'error'>('success')

const getIndexedRepos = (): string[] => {
  const data = localStorage.getItem('indexedRepos')
  return data ? JSON.parse(data) : []
}

const saveIndexedRepos = (ids: string[]) => {
  localStorage.setItem('indexedRepos', JSON.stringify(ids))
}

const buildTree = (items: Collection[]): Collection[] => {
  const map = new Map<string, Collection>()
  const roots: Collection[] = []

  items.forEach(item => {
    map.set(item.id, { ...item, children: [] })
  })

  items.forEach(item => {
    const parts = item.id.split('/')
    if (parts.length <= 3) {
      roots.push(map.get(item.id)!)
      return
    }

    const parentId = parts.slice(0, -1).join('/')
    const parent = map.get(parentId)

    if (parent) {
      parent.children!.push(map.get(item.id)!)
    } else {
      roots.push(map.get(item.id)!)
    }
  })

  return roots
}

// Fetch available repos from API
const fetchCollections = async () => {
  errorMessage.value = ''
  collections.value = []

  try {
    const response = await fetch(`${apiBase}/admin/repository`, {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${apiKey}`
      }
    })

    if (!response.ok) {
      throw new Error('Unauthorized or server error')
    }

    const data = await response.json()
    const indexedRepos = getIndexedRepos()
    const list = data.map((c: Collection) => ({
      ...c,
      indexed: indexedRepos.includes(c.id)
    }))

    collections.value = buildTree(list)
      } catch (error) {
    errorMessage.value = 'Failed to fetch collections'
    statusType.value = 'success'
  }
}

const markIndexedRecursively = (node: Collection, value: boolean) => {
  node.indexed = value
  if (node.children) {
    node.children.forEach(child => markIndexedRecursively(child, value))
  }
}

const getAllIds = (node: Collection): string[] => {
  return [node.id, ...(node.children ? node.children.flatMap(getAllIds) : [])]
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
    const response = await fetch(`${apiBase}/admin/index/`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiKey}`
      }
    })

    if (!response.ok) {
      throw new Error('Index failed')
    }

    collections.value.forEach(c => 
      markIndexedRecursively(c, true)
    )
    const allIds = collections.value.flatMap(getAllIds)
    saveIndexedRepos(allIds)
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
    const response = await fetch(`${apiBase}/admin/index/`, {
      method: 'DELETE',
      headers: {
        'Authorization': `Bearer ${apiKey}`
      }
    })

    if (!response.ok) {
      throw new Error('Delete failed')
    }

    collections.value.forEach(c => 
      markIndexedRecursively(c, false)
    )
    saveIndexedRepos([])
    statusMessage.value = 'All indexes deleted successfully!'
    statusType.value = 'success'
  } catch (error) {
    statusMessage.value = 'Delete failed!'
    statusType.value = 'error'
  }
}

const findCollectionById = (id: string, nodes: Collection[]): Collection | undefined => {
  for (const node of nodes) {
    if (node.id === id) return node
    if (node.children) {
      const found = findCollectionById(id, node.children)
      if (found) return found
    }
  }
  return undefined
}

// Index specific fetched repos
const indexCollection = async (collectionId: string, collectionName: string) => {
  statusMessage.value = ''
  const collection = findCollectionById(collectionId, collections.value)
  if (collection?.indexed) return

  try {
    const response = await fetch(`${apiBase}/admin/index/${encodeURIComponent(collectionId)}`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiKey}`
      }
    })

    if (!response.ok) {
      throw new Error('Index failed')
    }

    statusMessage.value = `Indexing started for ${collectionName}!`
    statusType.value = 'success'
    if (collection) {
      markIndexedRecursively(collection, true)

      const indexedRepos = getIndexedRepos()
      const allIds = getAllIds(collection)
      saveIndexedRepos([...indexedRepos, ...allIds])
    }
  } catch (error) {
    statusMessage.value = `Indexing failed for ${collectionName}!`
    statusType.value = 'error'
  }
}


// Delete specific fetched repos
const deleteCollection = async (collectionId: string, collectionName: string) => {
  statusMessage.value = ''
  
  try {
    const response = await fetch(`${apiBase}/admin/index/${encodeURIComponent(collectionId)}`, {
      method: 'DELETE',
      headers: {
        'Authorization': `Bearer ${apiKey}`
      }
    })

    if (!response.ok) {
      throw new Error('Delete failed')
    }

    statusMessage.value = `Deleted index for ${collectionName} successfully!`
    statusType.value = 'success'
    const collection = findCollectionById(collectionId, collections.value)
    if (collection) {
      markIndexedRecursively(collection, false)
      const allIds = getAllIds(collection)
      const indexedRepos = getIndexedRepos().filter(id => !allIds.includes(id))
      saveIndexedRepos(indexedRepos)
    }
    
  } catch (error) {
    statusMessage.value = `Delete failed for ${collectionName}!`
    statusType.value = 'error'
  }
}

</script>

<template>
  <div>
    <div class="navbar">
      <span class="navbar-title">LDACA Admin</span>
    </div>

    <p v-if="errorMessage" style="color:red">
      {{ errorMessage }}
    </p>

    <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:16px;">
      <el-button type="primary" @click="fetchCollections">
        Load Collections
      </el-button>

      <div style="display:flex; gap:10px;">
        <el-button type="success" @click="indexAll">
          Index All
        </el-button>

        <el-button type="danger" @click="deleteAll">
          Delete All
        </el-button>
      </div>
    </div>

    <p v-if="statusMessage" :style="{ color: statusType === 'success' ? 'green' : 'red', fontWeight: 'bold' }">
      {{ statusMessage }}
    </p>

    <el-tree
      v-if="collections.length"
      :data="collections"
      node-key="id"
      default-expand-all
    >
      <template #default="{ data }">
        <div class="tree-row">
          <span class="tree-label">
            {{ data.name }}
          </span>

          <span class="tree-actions">
            <el-button type="success"
              @click="indexCollection(data.id, data.name)"
              :disabled="data.indexed"
            >
              {{ data.indexed ? 'Indexed' : 'Index' }}
            </el-button>

            <el-button type="danger"
              @click="deleteCollection(data.id, data.name)"
              :disabled="!data.indexed"
            >
              Delete
            </el-button>
          </span>
        </div>
      </template>
    </el-tree>
  </div>
</template>


<style scoped>
.navbar {
  height: 50px;
  display: flex;
  align-items: center;
  padding: 0 16px;
  background-color: #146472;
  color: white;
  font-size: 18px;
  font-weight: bold;
  margin-bottom: 16px;
}
.navbar-title {
  user-select: none;
}
input {
  margin-right: 10px;
  padding: 5px;
}
button {
  padding: 5px 10px;
}
.tree-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.tree-label {
  flex: 1;
}

.tree-actions {
  display: flex;
  gap: 8px;
}
.el-button {
  min-width: 80px;
}
</style>