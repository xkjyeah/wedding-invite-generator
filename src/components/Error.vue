<template>
  <div class="error" v-if="message">
    {{ message }}
  </div>
</template>

<script setup lang="ts">
const props = defineProps(['message'])
</script>
<style scoped>
.error {
  background-color: #FCC;
  border: dashed 1px #966;
}
</style>
