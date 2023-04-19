<script setup lang="ts">
import { computed } from 'vue'
import { handleBackground } from '../layoutHelper'
const props = defineProps({
  image: {
    type: String,
  },
})
const style = computed(() => handleBackground(props.image))
</script>

<template>
  <div class="slidev-layout intro" :style="style">
    <div class="my-auto">
      <slot />
    </div>
  </div>
</template>
