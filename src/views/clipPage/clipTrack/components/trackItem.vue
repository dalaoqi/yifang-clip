<!--
 * Track Item Component
 * Responsible for displaying various media items (video, audio, image, text, etc.) in the track
 * Supports previews such as thumbnails, waveforms, etc.
-->
<template>
  <div
    class="w-full h-full p-1 flex flex-col justify-center items-center pointer-events-none select-none"
  >
    <!-- Video Clip -->
    <template v-if="clip.type === 'video'">
      <div
        class="flex flex-nowrap gap-0.5 items-center overflow-hidden px-1 h-full"
      >
        <template
          v-if="displayThumbnails.length > 0"
          v-loading="displayThumbnails.length === 0"
        >
          <img
            v-for="item in displayThumbnails"
            :key="item.timestamp"
            :src="item.url"
            class="h-[90%] shrink-0 grow-0 object-cover rounded"
            :style="{ width: `${70}px` }"
          />
        </template>
      </div>
    </template>

    <!-- Image Clip -->
    <template v-if="clip.type === 'image'">
      <div
        class="flex flex-nowrap gap-0.5 items-center overflow-hidden px-1 h-full"
      >
        <template
          v-if="displayThumbnails.length > 0"
          v-loading="displayThumbnails.length === 0"
        >
          <img
            v-for="item in displayThumbnails"
            :key="item.timestamp"
            :src="item.url"
            class="h-[90%] shrink-0 grow-0 object-cover rounded"
            :style="{ width: `${70}px` }"
          />
        </template>
      </div>
    </template>

    <!-- Audio Clip -->
    <template v-else-if="clip.type === 'audio'">
      <div class="w-full h-full flex flex-col relative rounded">
        <!-- Waveform -->
        <div class="absolute inset-0">
          <canvas ref="waveformCanvas" class="w-full h-full"></canvas>
        </div>
        <!-- Name -->
        <div
          v-if="hovered"
          class="absolute inset-0 w-auto h-full flex items-center bg-[#ffffff] bg-opacity-10 text-[#000000]"
        >
          <span class="text-base inline-block truncate px-5 z-10">{{
            clip.name
          }}</span>
        </div>
      </div>
    </template>

    <!-- Text Clip -->
    <template v-else-if="clip.type === 'text'">
      <!-- Type Indicator Icon -->
      <div
        class="flex items-center text-bigger text-white whitespace-nowrap overflow-hidden text-ellipsis w-full"
      >
        <Icon
          icon="material-symbols-light:text-fields"
          class="w-6 h-6 absolute left-1 text-white"
        >
        </Icon>
        <div class="pl-6.5">
          {{ clip.textConfig.content }}
        </div>
      </div>
    </template>

    <!-- Filter Clip -->
    <template v-else-if="clip.type === 'filter'">
      <div
        class="flex items-center text-bigger text-white whitespace-nowrap overflow-hidden text-ellipsis w-full"
      >
        <Icon
          icon="material-symbols-light:magnify-fullscreen-outline-rounded"
          class="w-6 h-6 absolute left-1 text-white"
        >
        </Icon>
        <div class="pl-6.5">
          <span>{{ getFilterName(clip.filterType) }}</span>
          <span v-if="clip.intensity !== undefined">
            {{ Math.round(clip.intensity) }}%
          </span>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, computed, onMounted, nextTick } from 'vue';
import type { TrackClip, FilterType } from '@/types/track';
import { Icon } from '@iconify/vue';

interface Thumbnail {
  timestamp: number;
  url: string;
}

// Props
const props = defineProps({
  clip: {
    type: Object as () => TrackClip,
    required: true,
  },
  zoom: {
    type: Number,
    default: 1,
  },
  frameLength: {
    type: Number,
    default: 20,
  },
  hovered: {
    type: Boolean,
    default: false,
  },
});

// Computed properties
const THUMBNAIL_WIDTH_DURATION = computed(() => 70 / props.frameLength);

// State variables
const displayThumbnails = ref<Thumbnail[]>([]);
const waveformCanvas = ref<HTMLCanvasElement | null>(null);

// Thumbnail-related methods
const updateDisplayThumbnails = () => {
  try {
    if (!props.clip.thumbnail || !Array.isArray(props.clip.thumbnail)) {
      displayThumbnails.value = [];
      return;
    }

    const duration = Number(props.clip.duration);
    if (isNaN(duration) || duration <= 0) {
      console.warn('Invalid clip duration:', duration);
      return;
    }

    const thumbnailCount = Math.ceil(duration / THUMBNAIL_WIDTH_DURATION.value);
    const result: Thumbnail[] = [];

    if (props.clip.type === 'image' && props.clip.thumbnail.length > 0) {
      // Image type: Repeat the first frame
      const thumbnail = props.clip.thumbnail[0] as Thumbnail;
      for (let i = 0; i < thumbnailCount; i++) {
        result.push(thumbnail);
      }
    } else if (props.clip.type === 'video' && props.clip.thumbnail.length > 0) {
      // Video type: Select appropriate thumbnails based on time distribution
      const availableThumbnails = props.clip.thumbnail as Thumbnail[];
      let lastThumbnail = availableThumbnails[0];

      for (let i = 0; i < thumbnailCount; i++) {
        const targetTime = i * THUMBNAIL_WIDTH_DURATION.value;
        const closestThumbnail =
          availableThumbnails.find(
            (item) => item.timestamp / 1e6 >= targetTime,
          ) ||
          lastThumbnail ||
          availableThumbnails[availableThumbnails.length - 1];

        if (closestThumbnail) {
          result.push(closestThumbnail);
          lastThumbnail = closestThumbnail;
        }
      }
    }

    displayThumbnails.value = result;
  } catch (error) {
    console.error('Error updating thumbnails:', error);
    displayThumbnails.value = [];
  }
};

// Waveform-related methods
const drawWaveform = () => {
  try {
    if (!waveformCanvas.value || !props.clip.volumeData) return;

    const canvas = waveformCanvas.value;
    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    // Set canvas dimensions
    const rect = canvas.getBoundingClientRect();
    canvas.width = rect.width * window.devicePixelRatio;
    canvas.height = rect.height * window.devicePixelRatio;

    // Clear canvas
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Prepare waveform data
    if (!props.clip.originalDuration) {
      console.warn('Missing originalDuration for audio clip');
      return;
    }

    const data = props.clip.volumeData.slice(
      (props.clip.sourceStartTime / props.clip.originalDuration) *
        props.clip.volumeData.length,
      ((props.clip.originalDuration - props.clip.sourceEndTime) /
        props.clip.originalDuration) *
        props.clip.volumeData.length,
    );

    if (!data.length) {
      console.warn('No volume data available');
      return;
    }

    // Set drawing parameters
    const barWidth = 2;
    const gap = 2;
    const barCount = Math.floor(canvas.width / (barWidth + gap));
    const maxHeight = canvas.height * 0.8;
    const samplingInterval = data.length / barCount;

    // Set styles
    ctx.fillStyle = '#ffffff80';

    // Draw waveform
    for (let i = 0; i < barCount; i++) {
      const dataIndex = Math.floor(i * samplingInterval);
      const value = data[dataIndex] || 0;
      const height = value * maxHeight;
      const x = i * (barWidth + gap);
      const y = (canvas.height - height) / 2;

      ctx.fillRect(x, y, barWidth, height);
    }
  } catch (error) {
    console.error('Error drawing waveform:', error);
  }
};

// Filter-related methods
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

// Watchers
watch(
  [
    () => props.frameLength,
    () => props.zoom,
    () => props.clip.duration,
    () => props.clip.sourceStartTime,
    () => props.clip.sourceEndTime,
    () => props.clip.thumbnail,
  ],
  () => {
    if (!props.clip) return;
    if (props.clip.type === 'video' || props.clip.type === 'image') {
      nextTick(() => {
        updateDisplayThumbnails();
      });
    }
  },
  { deep: true, immediate: true },
);

watch(
  () => props.clip.thumbnail,
  () => {
    if (!props.clip) return;
    if (props.clip.type === 'video' || props.clip.type === 'image') {
      nextTick(() => {
        updateDisplayThumbnails();
      });
    }
  },
  { deep: true, immediate: true },
);

// Lifecycle hooks
onMounted(() => {
  if (props.clip.type === 'audio') {
    const resizeObserver = new ResizeObserver(() => {
      drawWaveform();
    });
    if (waveformCanvas.value) {
      resizeObserver.observe(waveformCanvas.value);
      // Initial draw
      drawWaveform();
    }
  }
});
</script>

<style scoped>
canvas {
  image-rendering: pixelated;
}
</style>
