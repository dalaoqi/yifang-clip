<!--
 * Filter Properties Component
 * Used to display and set the filter's name and intensity
-->
<template>
  <div class="text-white">
    <template v-if="clip">
      <div class="mb-6">
        <div class="text-base font-medium text-white mb-4">
          Filter Properties
        </div>
        <!-- Filter Name -->
        <div class="mb-4">
          <div class="text-base text-[#999] mb-1">Name</div>
          <div class="text-base text-white">
            {{ getFilterName(clip.filterType) }}
          </div>
        </div>
        <!-- Filter Intensity -->
        <div class="mb-4">
          <div class="text-base text-[#999] mb-1">Intensity</div>
          <el-slider
            v-model="clip.intensity"
            :min="0"
            :max="100"
            :step="1"
            @change="handleIntensityChange"
            class="filter-slider"
          />
        </div>
      </div>
    </template>
  </div>
</template>

<script setup lang="ts">
import type { FilterTrackClip, FilterType } from '@/types/track';
import { ref, watch } from 'vue';

const props = defineProps<{
  clip: FilterTrackClip;
}>();

const emit = defineEmits(['update']);

// Handle intensity change
const handleIntensityChange = (value: number) => {
  props.clip.intensity = value;
};

// Get filter name
const getFilterName = (filterType?: FilterType) => {
  if (!filterType) return 'Unknown Filter';
  const filterNames: Record<FilterType, string> = {
    grayscale: 'Grayscale',
    invert: 'Invert',
    brightness: 'Brightness',
    sepia: 'Sepia',
    blur: 'Blur',
  };
  return filterNames[filterType];
};
</script>

<style scoped>
.filter-slider {
  width: 100%;
}

:deep(.el-slider__runway) {
  background-color: #2a2a2a;
}

:deep(.el-slider__bar) {
  background-color: theme('colors.purple.500');
}

:deep(.el-slider__button) {
  border-color: theme('colors.purple.500');
}

:deep(.el-slider__button:hover) {
  transform: scale(1.2);
}
</style>
