<script lang="ts" setup>
const { value } = defineProps<{ value: number }>();
</script>
<template>
  <svg viewBox="0 0 105 5" xmlns="http://www.w3.org/2000/svg">
    <line x1="2.5" y1="2.5" x2="102.5" y2="2.5" class="stroke-slate-300" stroke-width="5" stroke-linecap="round"/>
    <line x1="2.5" y1="2.5" x2="102.5" y2="2.5" stroke="blue" stroke-width="5" stroke-linecap="round" :stroke-dashoffset="100 - value" stroke-dasharray="100 100" class="transition-all"/>
  </svg>
</template>
