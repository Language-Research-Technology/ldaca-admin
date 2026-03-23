<script setup lang="ts">
import NavBar from '@/components/Nav.vue'
import FooterBar from '@/components/Footer.vue'
</script>

<template>
  <div class="layout">
    <NavBar />

    <div class="content">
      <router-view />
    </div>

    <FooterBar />
  </div>
</template>

<style scoped>
.layout {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.content {
  flex: 1;
  padding: 16px;
}
</style>