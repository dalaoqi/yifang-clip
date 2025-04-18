<!--
  * Main component of the video editing page
  * Includes the left menu, central player and properties panel, and bottom track editing area
  * Implements the core functionality of video editing
-->
<template>
  <!-- Main container: Styled using Tailwind CSS -->
  <div
    class="w-full h-full flex bg-[#1b1b1d] overflow-hidden"
    @click="closeContextMenu"
  >
    <!-- Left menu area -->
    <div id="left" class="h-screen">
      <ClipMenu />
    </div>
    <!-- Central main content area -->
    <div id="right" class="h-screen overflow-hidden">
      <!-- Top area: Contains player and properties panel -->
      <div id="top" class="flex justify-between">
        <!-- Player area -->
        <div id="topLeft" class="w-4/5">
          <Player
            ref="playerRef"
            :currentTime="currentTime"
            :tracks="tracks"
            :timelineDuration="timelineDuration"
            :playerDuration="playerDuration"
            @prevFrame="prevFrame"
            @nextFrame="nextFrame"
            @timeUpdate="timeUpdate"
            @captureImage="captureImage"
            @updateClipProps="updateClipProps"
          />
        </div>
        <!-- Right properties panel -->
        <div id="topRight" class="w-1/5 bg-[#303030]">
          <ClipProperties
            :clip="selectedClip"
            :min-clip-duration="minClipDuration"
            @update="updateClipProps"
          />
        </div>
      </div>
      <!-- Bottom track editing area -->
      <div id="bottom" class="bg-[#201f20]">
        <ClipTrack
          ref="clipTrackRef"
          v-model:tracks="tracks"
          :trackZoom="trackZoom"
          :maxTracksNum="MAX_TRACKS_NUM"
          :currentTime="currentTime"
          :timelineDuration="timelineDuration"
          :BASE_TICK_SPACING="BASE_TICK_SPACING"
          :MIN_CLIP_WIDTH="MIN_CLIP_WIDTH"
          :frame-length="frameLength"
          :min-clip-duration="minClipDuration"
          @stopPlay="handleStopPlay"
          @updateTrackZoom="updateTrackZoom"
          @timeUpdate="timeUpdate"
          @add-clip="addPlayerClip"
          @refreshPlayer="refreshPlayer"
          @delete-empty-track="deleteEmptyTrack"
          @activeClip="handleActiveClip"
          @handle-export="handleExport"
        />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
/**
 * Import required components and utilities
 */
import Player from './clipPlayer/Player.vue';
import { useTrackStore } from '@/store/modules/track';
import ClipMenu from './clipMenu/clipMenu.vue';
import Split from 'split.js';
import ClipTrack from './clipTrack/clipTrack.vue';
import ClipProperties from './ClipProperties/ClipProperties.vue';
import { useRoute } from 'vue-router';
import { db, Media } from '@/db/db';
import { Track, TrackClip } from '@/types/track';
import { getKeyframes, getVolume } from '@/components/js/webcodecs';
import { cloneDeep } from 'lodash';
import { ElMessage } from 'element-plus';
import { dataURLToBuffer } from '@/utils/opfs-file';
import { write } from 'opfs-tools';

/**
 * Disable default browser zoom behavior
 * Prevent Ctrl+wheel from triggering browser zoom, affecting the editing experience
 */
const disableZoom = (e: WheelEvent) => {
  if (e.ctrlKey) {
    e.preventDefault();
  }
};

onMounted(() => {
  window.addEventListener('wheel', disableZoom, { passive: false });
});

onUnmounted(() => {
  window.removeEventListener('wheel', disableZoom);
});

const route = useRoute();
const trackStore = useTrackStore();
const projectId = Number(route.params.id);
const MAX_TRACKS_NUM = 10; // Maximum number of tracks
const MIN_DURATION = 300; // Minimum duration of 5 minutes (seconds)
const BUFFER_DURATION = 5; // Buffer duration (seconds)

// Split layout related
const SPLIT_STORAGE_KEY = 'split-sizes';
const DEFAULT_HORIZONTAL_SIZES = [25, 75];
const DEFAULT_VERTICAL_SIZES = [65, 35];

// Track-related constants
const BASE_TICK_SPACING = 3; // Base tick spacing (px)
const MIN_CLIP_WIDTH = 30; // Minimum clip width (px)
const ZOOM_STORAGE_KEY = 'track-zoom-value';

// Retrieve zoom value from localStorage
const trackZoom = ref(
  parseFloat(localStorage.getItem(ZOOM_STORAGE_KEY) || '1'),
);

const updateTrackZoom = (zoom: number) => {
  trackZoom.value = zoom;
  localStorage.setItem(ZOOM_STORAGE_KEY, zoom.toString());
};

// Computed properties
const frameLength = computed(() => BASE_TICK_SPACING * trackZoom.value * 10); // Pixels per second
const minClipDuration = computed(() => MIN_CLIP_WIDTH / frameLength.value); // Minimum duration based on zoom

// Track data
const tracks = ref<Track[]>([]);
const currentTime = ref(0);
// Compute total timeline duration
const timelineDuration = computed(() => {
  // Maximum endTime plus buffer duration, not less than minimum duration
  return Math.max(playerDuration.value + BUFFER_DURATION, MIN_DURATION);
});
// Compute total player duration
const playerDuration = computed(() => {
  // Find the maximum endTime across all tracks
  let maxEndTime = 0;
  tracks.value.forEach((track) => {
    track.clips.forEach((clip) => {
      const endTime = Number(clip.endTime);
      if (endTime > maxEndTime) {
        maxEndTime = endTime;
      }
    });
  });
  return maxEndTime;
});

const playerRef = ref(null);
const clipTrackRef = ref(null);

const timeUpdate = (time: number) => {
  currentTime.value = time;
};

const captureImage = async (dataUrl: string) => {
  const buffer = dataURLToBuffer(dataUrl);
  await write('/capture/' + projectId + '.png', buffer, {
    overwrite: true,
  });
  const project = await db.projects.where({ id: projectId }).first();
  if (project) {
    db.projects.update(project, {
      thumbnail: '/capture/' + projectId + '.png',
    });
  }
};

const prevFrame = () => {
  currentTime.value -= 30 / 1000 / 1000;
};

const nextFrame = () => {
  currentTime.value += 30 / 1000 / 1000;
};

// Provide addClip method to all child components
provide('addClip', async (clip: TrackClip, createNewTrack?: boolean) => {
  // Set clip start time to current time
  clip.startTime = currentTime.value;
  clip.endTime = currentTime.value + clip.duration;

  // For video type, set source time-related properties
  if (clip.type === 'video') {
    clip.sourceStartTime = 0;
    clip.sourceEndTime = 0;
    clip.originalDuration = clip.duration;
  }

  // Fetch video keyframes or image thumbnails
  if (clip.type === 'video' || clip.type === 'image') {
    const res = await getKeyframes(clip as Media);
    clip.thumbnail = res.data as { url: string; timestamp: number }[];
  }

  // Fetch audio waveform data
  if (clip.type === 'audio') {
    const res = await getVolume(clip as Media);
    if (res.type === 'audio') {
      clip.volumeData = res.data;
    }
  }

  // Create new track if needed
  if (createNewTrack) {
    if (tracks.value.length >= MAX_TRACKS_NUM) {
      ElMessage.warning(
        `Track count has reached the maximum limit (${MAX_TRACKS_NUM})`,
      );
      return;
    }
    const newTrack: Track = {
      id: String(tracks.value.length),
      clips: [clip],
    };
    tracks.value.push(newTrack);
  } else {
    // Otherwise, add to the last track
    if (tracks.value.length === 0) {
      tracks.value.push({
        id: '0',
        clips: [],
      });
    }
    tracks.value[tracks.value.length - 1].clips.push(clip);
  }
  // Update player
  nextTick(() => {
    playerRef.value?.refreshPlayer();
  });
});

// Provide activeClip method to activate a clip in the track when needed
provide('activeClip', (id: string | null) => {
  if (id) {
    const clip = tracks.value
      .find((track) => track.clips.some((clip) => clip.id === id))
      ?.clips.find((clip) => clip.id === id);
    if (clip) {
      clipTrackRef.value?.activeClip(clip);
      selectedClip.value = clip;
    }
  } else {
    clipTrackRef.value?.activeClip(null);
    selectedClip.value = null;
  }
});

const playerWidth = ref(0);
const playerHeight = ref(0);
onMounted(async () => {
  // Retrieve Split layout data from localStorage
  const savedSizes = localStorage.getItem(SPLIT_STORAGE_KEY);
  const { horizontalSizes, verticalSizes } = savedSizes
    ? JSON.parse(savedSizes)
    : {
        horizontalSizes: DEFAULT_HORIZONTAL_SIZES,
        verticalSizes: DEFAULT_VERTICAL_SIZES,
      };

  // Initialize horizontal split
  Split(['#left', '#right'], {
    sizes: horizontalSizes,
    minSize: [250, 500],
    snapOffset: 0,
    onDragEnd: (sizes) => {
      // Save new horizontal split sizes
      const savedSizes = localStorage.getItem(SPLIT_STORAGE_KEY);
      const sizeData = savedSizes ? JSON.parse(savedSizes) : {};
      localStorage.setItem(
        SPLIT_STORAGE_KEY,
        JSON.stringify({
          ...sizeData,
          horizontalSizes: sizes,
        }),
      );
    },
  });

  // Initialize vertical split
  Split(['#top', '#bottom'], {
    direction: 'vertical',
    sizes: verticalSizes,
    minSize: [250, 150],
    snapOffset: 0,
    onDragEnd: (sizes) => {
      // Save new vertical split sizes
      const savedSizes = localStorage.getItem(SPLIT_STORAGE_KEY);
      const sizeData = savedSizes ? JSON.parse(savedSizes) : {};
      localStorage.setItem(
        SPLIT_STORAGE_KEY,
        JSON.stringify({
          ...sizeData,
          verticalSizes: sizes,
        }),
      );
    },
  });

  const project = await db.projects.where({ id: projectId }).first();
  if (project) {
    tracks.value = project.tracks;
    deleteEmptyTrack();
    trackStore.clearHistory(projectId);
    trackStore.saveHistoryState(projectId, tracks.value);
    refreshPlayer();
  }
});
onUnmounted(() => {
  releaseThumbnail();
});

const getThumbnail = async () => {
  tracks.value.forEach((track) => {
    track.clips.forEach((clip) => {
      if (clip.type === 'audio' && !clip.volumeData) {
        getVolume(clip as Media).then((res) => {
          if (res.type === 'audio') {
            clip.volumeData = res.data;
          }
        });
      } else if (clip.type === 'video' || clip.type === 'image') {
        if (clip.thumbnail && clip.thumbnail.length > 0) {
          const img = new Image();
          img.src = clip.thumbnail[0].url;
          img.onerror = () => {
            getKeyframes(clip as Media).then((res) => {
              clip.thumbnail = res.data as { url: string; timestamp: number }[];
            });
          };
        } else {
          getKeyframes(clip as Media).then((res) => {
            clip.thumbnail = res.data as { url: string; timestamp: number }[];
          });
        }
      }
    });
  });
};

const releaseThumbnail = () => {
  tracks.value.forEach((track) => {
    track.clips.forEach((clip) => {
      if (clip.thumbnail && clip.thumbnail.length > 0) {
        clip.thumbnail.forEach((item) => {
          URL.revokeObjectURL(item.url);
        });
      }
      clip.thumbnail = null;
    });
  });
};

const closeContextMenu = () => {
  trackStore.setShowContextMenu(false);
};

const updateClipProps = (updateClip: TrackClip) => {
  for (const track of tracks.value) {
    for (const clip of track.clips) {
      if (clip.id === updateClip.id) {
        Object.assign(clip, updateClip);
        break;
      }
    }
  }
};

watch(
  () => tracks.value,
  async (val) => {
    await saveProject();
  },
  {
    deep: true,
  },
);

// Update project when saving track data
const saveProject = async () => {
  const project = await db.projects.where({ id: projectId }).first();
  if (project) {
    // Deep clone tracks array to avoid IDB cloning errors
    const rawTracks = cloneDeep(tracks.value);
    rawTracks.forEach((track) => {
      track.clips.forEach((clip) => {
        clip.thumbnail = null;
      });
    });
    await db.projects.update(project, { tracks: rawTracks });
  }
};

const selectedClip = ref(null);

const handleActiveClip = (clip: TrackClip | null) => {
  selectedClip.value = clip;
  if (clip?.type !== 'audio') {
    playerRef.value?.activeClip(clip);
  }
};

const addPlayerClip = (clip: TrackClip) => {
  playerRef.value?.addClip(clip);
};

// Delete tracks with no clips
const deleteEmptyTrack = () => {
  tracks.value = tracks.value.filter((track) => track.clips.length > 0);
};

const handleStopPlay = () => {
  playerRef.value?.handleStopPlay();
};

const refreshPlayer = () => {
  getThumbnail();
  playerRef.value?.refreshPlayer();
};

const handleExport = () => {
  playerRef.value?.handleExport();
};
</script>

<style lang="scss">
#left + .gutter {
  position: relative;
  z-index: 100;
  cursor: col-resize;
  background-color: #303030;
  box-shadow: inset 2px 0 1px 0px #303030;
  transition: box-shadow 0.4s ease-in-out 0.2s;

  &:hover {
    box-shadow: inset -2px 0 1px 0px rgba(128, 0, 128, 0.7);
  }

  &.gutter-vertical {
    cursor: n-resize;
  }
}

#right .gutter {
  position: relative;
  z-index: 100;
  cursor: row-resize;
  background-color: #1b1b1b;
  transition: box-shadow 0.4s ease-in-out 0.2s;

  &:hover {
    box-shadow: inset 0 -2px 1px 0px rgba(128, 0, 128, 0.7);
  }
}
</style>
