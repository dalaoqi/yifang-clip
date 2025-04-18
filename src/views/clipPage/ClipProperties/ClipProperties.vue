<!--
 * 片段属性设置组件
 * 根据片段类型显示不同的属性设置界面
-->
<template>
  <div class="p-6 text-white">
    <template v-if="clip">
      <!-- Video Properties -->
      <template v-if="clip.type === 'video'">
        <VideoProperties :clip="clip" @update="handleUpdate" />
      </template>

      <!-- Audio Properties -->
      <template v-if="clip.type === 'audio'">
        <AudioProperties :clip="clip" @update="handleUpdate" />
      </template>

      <!-- Image Properties -->
      <template v-if="clip.type === 'image'">
        <ImageProperties :clip="clip" @update="handleUpdate" />
      </template>

      <!-- Text Properties -->
      <template v-if="clip.type === 'text'">
        <TextProperties :clip="clip" @update="handleUpdate" />
      </template>

      <!-- Filter Properties -->
      <template v-if="clip.type === 'filter'">
        <FilterProperties
          :clip="clip as unknown as FilterTrackClip"
          @update="handleUpdate"
        />
      </template>
    </template>
    <div v-else class="text-center text-[#999] text-base">Select a clip</div>
  </div>
</template>

<script setup lang="ts">
import { watch } from 'vue';
import { useTrackStore } from '@/store/modules/track';
import type { TrackClip, FilterTrackClip } from '@/types/track';
import VideoProperties from './components/VideoProperties.vue';
import AudioProperties from './components/AudioProperties.vue';
import ImageProperties from './components/ImageProperties.vue';
import TextProperties from './components/TextProperties.vue';
import FilterProperties from './components/FilterProperties.vue';

const props = defineProps<{
  clip: TrackClip;
  minClipDuration: number;
}>();

const emit = defineEmits(['update']);

const trackStore = useTrackStore();

// 处理更新
const handleUpdate = (updatedClip: TrackClip) => {
  emit('update', updatedClip);
};

// 处理去片头
const handleChangeHead = (val: number | null) => {
  if (!props.clip) return;
  const oldSourceStartTime = Number(props.clip.sourceStartTime);
  const value = val === null || isNaN(val) ? 0 : val;
  const newDuration =
    props.clip.originalDuration - value - props.clip.sourceEndTime;
  if (newDuration < props.minClipDuration) return;

  props.clip.sourceStartTime = value;
  const diff = props.clip.sourceStartTime - oldSourceStartTime;
  // 调整startTime和duration
  if (props.clip.startTime + diff < 0) {
    props.clip.startTime = 0;
    props.clip.duration =
      props.clip.originalDuration -
      props.clip.sourceStartTime -
      props.clip.sourceEndTime;
    props.clip.endTime = props.clip.startTime + props.clip.duration;
  } else {
    props.clip.startTime = props.clip.startTime + diff;
    props.clip.duration =
      props.clip.originalDuration -
      props.clip.sourceStartTime -
      props.clip.sourceEndTime;
    props.clip.endTime = props.clip.startTime + props.clip.duration;
  }

  emit('update', props.clip);
};

// 处理去片尾
const handleChangeTail = (val: number | null) => {
  if (!props.clip) return;
  const oldSourceEndTime = Number(props.clip.sourceEndTime);
  const value = val === null || isNaN(val) ? 0 : val;
  const newDuration =
    props.clip.originalDuration - value - props.clip.sourceStartTime;
  if (newDuration < props.minClipDuration) return;

  props.clip.sourceEndTime = value;
  const diff = props.clip.sourceEndTime - oldSourceEndTime;
  // 调整endTime和duration
  props.clip.endTime = props.clip.endTime - diff;
  props.clip.duration = props.clip.duration - diff;

  emit('update', props.clip);
};

// 提供给子组件使用
provide('handleChangeHead', handleChangeHead);
provide('handleChangeTail', handleChangeTail);

// 监听属性变化
watch(
  () => props.clip,
  (newClip) => {
    if (!newClip) return;
    trackStore.publishClipUpdate(newClip);
  },
  { deep: true },
);
</script>

<style lang="scss">
:deep(.el-input-number) {
  width: 100%;
}

:deep(.el-input-number .el-input__wrapper) {
  background-color: #1a1a1a;
  border-color: #333;
  &:hover {
    border-color: #666;
  }
}

:deep(.el-textarea__inner) {
  background-color: #1a1a1a;
  border-color: #333;
  color: white;
  &:hover {
    border-color: #666;
  }
}

:deep(.el-input__inner) {
  background-color: #1a1a1a;
  border-color: #333;
  color: white;
  &:hover {
    border-color: #666;
  }
}

:deep(.el-select) {
  width: 100%;
}

:deep(.el-select .el-input__wrapper) {
  background-color: #1a1a1a;
  border-color: #333;
  &:hover {
    border-color: #666;
  }
}

:deep(.el-slider) {
  width: 100%;
}

:deep(.el-color-picker) {
  width: 100%;
}

:deep(.el-color-picker .el-color-picker__trigger) {
  width: 100%;
  border-color: #333;
  &:hover {
    border-color: #666;
  }
}

:deep(.el-button) {
  background-color: transparent;
  border-color: #333;
  color: white;
  &:hover {
    background-color: #333;
    border-color: #666;
  }
}

:deep(.el-input-number.is-controls-right .el-input-number__decrease),
:deep(.el-input-number.is-controls-right .el-input-number__increase) {
  background-color: transparent;
  border-color: #333;
  color: white;
  &:hover {
    background-color: #333;
  }
}

:deep(.el-switch) {
  --el-switch-on-color: #8b5cf6;
}

:deep(.el-collapse) {
  border: none;
  background-color: transparent;
}

:deep(.el-collapse-item__header) {
  background-color: transparent;
  border: none;
  color: white;
}

:deep(.el-collapse-item__content) {
  background-color: transparent;
  color: white;
  padding: 0;
}

:deep(.el-collapse-item__arrow) {
  color: white;
}

:deep(.el-dropdown-menu) {
  background-color: #1a1a1a;
  border-color: #333;
}

:deep(.el-dropdown-menu__item) {
  color: white;
  &:hover {
    background-color: #333;
    color: white;
  }
}

:deep(.el-dropdown-menu__item:not(.is-disabled):hover) {
  background-color: #333;
  color: white;
}
</style>
