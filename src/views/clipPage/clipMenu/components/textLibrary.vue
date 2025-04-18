<template>
  <div class="p-4">
    <div class="grid grid-cols-2 gap-4">
      <!-- Basic Text -->
      <div
        class="relative group cursor-pointer bg-[#2a2a2a] hover:bg-[#3a3a3a] rounded-lg p-4 flex flex-col items-center"
        @click="handleAddText(basicTemplate)"
        draggable="true"
        @dragstart="handleDragStart(basicTemplate)"
      >
        <!-- Preview Effect -->
        <div class="h-8 text-white text-xl mb-2">
          {{ basicTemplate.preview }}
        </div>
        <!-- Template Name -->
        <div class="text-[#666] text-sm">{{ basicTemplate.name }}</div>
        <!-- Add Button on Hover -->
        <div
          class="absolute inset-0 bg-purple-500/20 opacity-0 group-hover:opacity-100 flex items-center justify-center transition-opacity rounded-lg"
        >
          <el-icon class="text-white text-2xl">
            <Plus />
          </el-icon>
        </div>
      </div>

      <!-- Fancy Text Templates -->
      <div
        v-for="template in textTemplates"
        :key="template.name"
        class="relative group cursor-pointer bg-[#2a2a2a] hover:bg-[#3a3a3a] rounded-lg p-4 flex flex-col items-center"
        @click="handleAddText(template)"
        draggable="true"
        @dragstart="handleDragStart(template)"
      >
        <div class="h-8 text-white text-xl mb-2">
          {{ basicTemplate.preview }}
        </div>
        <!-- Template Name -->
        <div class="text-[#666] text-sm">{{ basicTemplate.name }}</div>
        <!-- Add Button on Hover -->
        <div
          class="absolute inset-0 bg-purple-500/20 opacity-0 group-hover:opacity-100 flex items-center justify-center transition-opacity rounded-lg"
        >
          <el-icon class="text-white text-2xl">
            <Plus />
          </el-icon>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useTrackStore } from '@/store/modules/track';
import { v4 } from 'uuid';
import { Plus } from '@element-plus/icons-vue';
import type { TrackClip } from '@/types/track';
import { textTemplates } from '@/constants/textTemplates';

interface TextTemplate {
  preview: string;
  name: string;
  style: {
    fontSize: number;
    fontFamily: string;
    color: string;
    animation?: string;
  };
}

const trackStore = useTrackStore();
const addClip = inject('addClip') as
  | ((clip: TrackClip, createNewTrack?: boolean) => void)
  | undefined;

// Basic Text Template
const basicTemplate = ref<TextTemplate>({
  preview: 'Aa',
  name: 'Basic Text',
  style: {
    fontSize: 72,
    fontFamily: 'Microsoft YaHei',
    color: '#ffffff',
  },
});

// Handle Drag Start
const handleDragStart = (template: TextTemplate) => {
  const textClip = createTextClip(template);
  trackStore.setDragData(textClip);
};

// Handle Add Text Click
const handleAddText = (template: TextTemplate) => {
  const textClip = createTextClip(template);
  // Use injected addClip method
  addClip?.(textClip as TrackClip, true);
};

// Create Text Clip
const createTextClip = (template: TextTemplate) => {
  return {
    id: v4(),
    type: 'text',
    name: template.name,
    duration: 5,
    startTime: 0,
    endTime: 5,
    opacity: 100,
    angle: 0,
    textConfig: {
      content: template.name || 'Basic Text',
      lineSpacing: 0,
      letterSpacing: 0,
      showStroke: false,
      strokeColor: '#ffffff',
      strokeWidth: 0,
      showShadow: false,
      shadowColor: '#ffffff',
      shadowBlur: 0,
      shadowOffsetX: 0,
      shadowOffsetY: 0,
      ...template.style,
    },
  };
};
</script>
