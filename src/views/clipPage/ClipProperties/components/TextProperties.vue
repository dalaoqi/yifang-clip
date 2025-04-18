<!--
 * Text Attribute Settings Component
 * Used to set text content, font, style, color, etc.
-->
<template>
  <div class="mb-6">
    <div class="text-base font-medium text-white mb-4">Text Attributes</div>
    <!-- Text Content -->
    <div class="mb-2">
      <el-input
        v-model="clip.textConfig.content"
        type="textarea"
        :rows="3"
        placeholder="Enter text content"
      />
    </div>

    <!-- Font Settings -->
    <div class="mb-4">
      <div class="text-base text-white mb-2">Font</div>
      <el-select v-model="clip.textConfig.fontFamily" class="w-full mb-2">
        <el-option label="DM Sans" value="DM Sans" />
        <el-option label="Microsoft YaHei" value="Microsoft YaHei" />
        <el-option label="SimSun" value="SimSun" />
        <el-option label="SimHei" value="SimHei" />
      </el-select>
      <div class="text-base text-white mb-2">Rotation Angle</div>
      <div class="flex items-center gap-4 mb-3">
        <div class="w-full">
          <el-input-number
            class="w-full"
            v-model="displayInputAngle"
            :min="0"
            :max="360"
            :step="1"
            size="small"
            controls-position="right"
            @change="handleAngleInput"
          />
        </div>
        <div
          class="relative flex items-center justify-center"
          @mousedown="startRotate"
          @mousemove="handleRotate"
          @mouseup="stopRotate"
          @mouseleave="stopRotate"
        >
          <div class="w-8 h-8 rounded-full border border-[#af24ff] relative">
            <!-- Rotation Pointer -->
            <div
              class="absolute w-[1px] h-[13px] bg-[#af24ff]"
              :style="{
                transform: `rotate(${displayAngle}deg)`,
                left: 'calc(50% - 0.5px)',
                top: '50%',
                transformOrigin: 'top',
              }"
            >
              <div
                class="absolute bottom-0 left-1/2 transform -translate-x-1/2 w-1.5 h-1.5 rounded-full bg-[#af24ff]"
              ></div>
            </div>
          </div>
        </div>
      </div>
      <div class="flex justify-start items-center gap-1">
        <el-button
          :class="{ 'text-purple-500': clip.textConfig.bold }"
          size="small"
          @click="clip.textConfig.bold = !clip.textConfig.bold"
        >
          <Icon icon="material-symbols:format-bold" />
        </el-button>
        <el-button
          class="!ml-0"
          :class="{ 'text-purple-500': clip.textConfig.italic }"
          size="small"
          @click="clip.textConfig.italic = !clip.textConfig.italic"
        >
          <Icon icon="material-symbols:format-italic" />
        </el-button>
        <el-dropdown trigger="click" @command="handleAlign">
          <el-button size="small">
            <Icon :icon="alignIcon" />
          </el-button>
          <template #dropdown>
            <el-dropdown-menu>
              <el-dropdown-item command="left">
                <Icon icon="material-symbols:format-align-left" />
                <span class="text-sm text-white ml-2">Left Align</span>
              </el-dropdown-item>
              <el-dropdown-item command="center">
                <Icon icon="material-symbols:format-align-center" />
                <span class="text-sm text-white ml-2">Center Align</span>
              </el-dropdown-item>
              <el-dropdown-item command="right">
                <Icon icon="material-symbols:format-align-right" />
                <span class="text-sm text-white ml-2">Right Align</span>
              </el-dropdown-item>
            </el-dropdown-menu>
          </template>
        </el-dropdown>
        <!-- Color Picker -->
        <el-color-picker
          v-model="clip.textConfig.color"
          size="small"
          show-alpha
        />
      </div>
    </div>

    <!-- Advanced Settings -->
    <el-collapse v-model="activeCollapse">
      <el-collapse-item name="advanced">
        <template #title>
          <span class="text-base text-white">Advanced</span>
        </template>
        <!-- Opacity Setting -->
        <div class="py-2">
          <div class="flex justify-between items-center mb-2">
            <span class="text-base text-white">Opacity</span>
            <div class="flex-1 mx-4">
              <el-slider v-model="clip.opacity" :min="0" :max="100" :step="1" />
            </div>
          </div>
          <!-- Line Height -->
          <div class="flex justify-between items-center mb-2">
            <span class="text-base text-white">Line Height</span>
            <el-input-number
              v-model="clip.textConfig.lineSpacing"
              :min="0"
              :step="1"
              controls-position="right"
              class="w-30"
            />
          </div>
          <!-- Letter Spacing -->
          <div class="flex justify-between items-center mb-2">
            <span class="text-base text-white">Letter Spacing</span>
            <el-input-number
              v-model="clip.textConfig.letterSpacing"
              :step="1"
              controls-position="right"
              class="w-30"
            />
          </div>
          <!-- Stroke -->
          <div class="flex justify-between items-center mb-2">
            <span class="text-base text-white">Stroke</span>
            <el-switch v-model="clip.textConfig.showStroke" />
          </div>
          <template v-if="clip.textConfig.showStroke">
            <div class="flex justify-between items-center mb-2 pl-4">
              <span class="text-base text-white">Stroke Color</span>
              <el-color-picker
                v-model="clip.textConfig.strokeColor"
                show-alpha
                size="small"
              />
            </div>
            <div class="flex justify-between items-center mb-2 pl-4">
              <span class="text-base text-white">Stroke Width</span>
              <el-input-number
                v-model="clip.textConfig.strokeWidth"
                :min="0"
                :step="0.5"
                controls-position="right"
                class="w-30"
              />
            </div>
          </template>
          <!-- Shadow -->
          <div class="flex justify-between items-center mb-2">
            <span class="text-base text-white">Shadow</span>
            <el-switch v-model="clip.textConfig.showShadow" />
          </div>
          <template v-if="clip.textConfig.showShadow">
            <div class="flex justify-between items-center mb-2 pl-4">
              <span class="text-base text-white">Shadow Color</span>
              <el-color-picker
                v-model="clip.textConfig.shadowColor"
                show-alpha
                size="small"
              />
            </div>
            <div class="flex justify-between items-center mb-2 pl-4">
              <span class="text-base text-white">Shadow Blur</span>
              <el-input-number
                v-model="clip.textConfig.shadowBlur"
                :min="0"
                :step="1"
                controls-position="right"
                class="w-30"
              />
            </div>
            <div class="flex justify-between items-center mb-2 pl-4">
              <span class="text-base text-white">Horizontal Offset</span>
              <el-input-number
                v-model="clip.textConfig.shadowOffsetX"
                :step="1"
                controls-position="right"
                class="w-30"
              />
            </div>
            <div class="flex justify-between items-center mb-2 pl-4">
              <span class="text-base text-white">Vertical Offset</span>
              <el-input-number
                v-model="clip.textConfig.shadowOffsetY"
                :step="1"
                controls-position="right"
                class="w-30"
              />
            </div>
          </template>
        </div>
      </el-collapse-item>
    </el-collapse>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue';
import type { TrackClip } from '@/types/track';
import { Icon } from '@iconify/vue';

const props = defineProps<{
  clip: TrackClip;
}>();

const emit = defineEmits(['update']);

const isDragging = ref(false);
const activeCollapse = ref([]);

// Radians to degrees
const toDegree = (radian: number) => {
  let degree = radian * (180 / Math.PI);
  degree = degree % 360;
  if (degree < 0) degree += 360;
  return Math.round(degree);
};

// Degrees to radians
const toRadian = (degree: number) => {
  return degree * (Math.PI / 180);
};

// Display angle
const displayAngle = computed(() => {
  if (props.clip?.angle === undefined) return 0;
  let angle = toDegree(props.clip.angle);
  angle = (angle + 180) % 360;
  return angle;
});

// Angle displayed in input box
const displayInputAngle = computed({
  get: () => {
    if (props.clip?.angle === undefined) return 0;
    return toDegree(props.clip.angle);
  },
  set: (val) => {
    if (!props.clip) return;
    props.clip.angle = toRadian(val);
  },
});

const alignIcon = computed(() => {
  switch (props.clip?.textConfig?.align) {
    case 'center':
      return 'material-symbols:format-align-center';
    case 'right':
      return 'material-symbols:format-align-right';
    default:
      return 'material-symbols:format-align-left';
  }
});

const handleAlign = (command: 'left' | 'center' | 'right') => {
  if (props.clip) {
    props.clip.textConfig.align = command;
    emit('update', props.clip);
  }
};

const handleAngleInput = (val: number | null) => {
  if (!props.clip) return;
  if (val === null || isNaN(val)) {
    props.clip.angle = 0;
    return;
  }
  let degree = val % 360;
  if (degree < 0) degree += 360;
  props.clip.angle = toRadian(degree);
  emit('update', props.clip);
};

const startRotate = (e: MouseEvent) => {
  isDragging.value = true;
};

const handleRotate = (e: MouseEvent) => {
  if (!isDragging.value || !props.clip) return;

  const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
  const centerX = rect.left + rect.width / 2;
  const centerY = rect.top + rect.height / 2;

  let angle = Math.atan2(e.clientY - centerY, e.clientX - centerX);
  angle = angle + Math.PI / 2;
  if (angle < 0) angle += Math.PI * 2;

  props.clip.angle = angle;
  emit('update', props.clip);
};

const stopRotate = () => {
  isDragging.value = false;
};
</script>

<style scoped>
:deep(.el-slider__runway) {
  background-color: #2a2a2a;
}

:deep(.el-slider__bar) {
  background-color: theme('colors.purple.500');
}

:deep(.el-slider__button) {
  border-color: theme('colors.purple.500');
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
</style>
