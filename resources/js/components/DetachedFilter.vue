<template>
  <div class="relative flex detached-filter" :class="width">
    <component
      :resource-name="resourceName"
      :key="filter.name"
      :filter-key="filter.class"
      :is="filter.component"
      :lens="''"
      @input="$emit('handle-filter-changed', filter)"
      @change="$emit('handle-filter-changed', filter)"
    />

    <filter-icon v-if="filter.withReset" @click="$emit('reset-filter', filter)"/>
  </div>
</template>

<script>
import FilterIcon from "./icons/FilterIcon";

export default {
  components: {FilterIcon},
  props: ['filter', 'width', 'resourceName'],
};
</script>
