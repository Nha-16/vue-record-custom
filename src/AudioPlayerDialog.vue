<script setup lang="ts">
import { ref, computed, watch, nextTick } from 'vue';
import Dialog from 'primevue/dialog';
import { customerHistoryStore } from '../customer-history.store';
import playAudio from '@/app/assets/svg/play.svg';
import pauseAudio from '@/app/assets/svg/puase.svg';
import playBack from '@/app/assets/svg/playBack.svg';
import playRight from '@/app/assets/svg/playRight.svg';

const store = customerHistoryStore();

const audioPlayerRef = ref<HTMLAudioElement | null>(null);
const isPlayAudio = ref(false);
const seekValue = ref(0);
const totalDuration = ref(0);

// Slider fill percent
const fillPercent = computed(() => {
  const max = totalDuration.value || 1;
  return `${(seekValue.value / max) * 100}%;`;
});

// --- Controls ---
const togglePlayAudio = async () => {
  const audio = audioPlayerRef.value;
  if (!audio) return;

  if (audio.paused) {
    try {
      await audio.play();
      isPlayAudio.value = true;
    } catch (err) {
      // ignore AbortError
    }
  } else {
    audio.pause();
    isPlayAudio.value = false;
  }
};

const skipBack = () => {
  const audio = audioPlayerRef.value;
  if (audio) {
    audio.currentTime = Math.max(audio.currentTime - 10, 0); // 10 seconds back
  }
};

const skipForward = () => {
  const audio = audioPlayerRef.value;
  if (audio) {
    audio.currentTime = Math.min(audio.currentTime + 10, totalDuration.value); // 10 seconds forward
  }
};

const onSeekInput = (event: Event) => {
  const audio = audioPlayerRef.value;
  if (audio) {
    const target = event.target as HTMLInputElement;
    audio.currentTime = parseFloat(target.value);
  }
};

// Format time mm:ss
const formatTime = (seconds: number) => {
  if (isNaN(seconds) || seconds < 0) return '0:00';
  const m = Math.floor(seconds / 60);
  const s = Math.floor(seconds % 60);
  return `${m}:${String(s).padStart(2, '0')}`;
};

// --- Watch for new audio URL ---
watch(
  () => store.recordUrl,
  async (newUrl) => {
    const audio = audioPlayerRef.value;
    if (!audio || !newUrl) return;

    audio.src = newUrl;
    audio.load();

    isPlayAudio.value = false;
    seekValue.value = 0;
    totalDuration.value = 0;

    // Use a promise to ensure metadata is loaded before proceeding
    await new Promise<void>((resolve) => {
      audio.onloadedmetadata = () => {
        totalDuration.value = audio.duration;
        resolve();
      };
    });

    audio.ontimeupdate = () => {
      seekValue.value = audio.currentTime;
    };

    audio.onended = () => {
      isPlayAudio.value = false;
      seekValue.value = 0;
    };
  },
  { immediate: true }
);

// --- Stop audio when dialog closes ---
watch(
  () => store.isOpen,
  (open) => {
    const audio = audioPlayerRef.value;
    if (!open && audio) {
      audio.pause();
      audio.currentTime = 0;
      isPlayAudio.value = false;
    }
  }
);
</script>

<template>
  <Dialog
    v-model:visible="store.isOpen"
    :closable="true"
    modal
    :draggable="false"
    :style="{ width: '420px' }"
  >
    <div class="flex justify-center mt-2">
      <img src="/src/app/assets/svg/audio.svg" alt="" />
    </div>
    <strong>
      <p class="mt-4 text-[24px] text-2xl text-center">Audio Record</p>
    </strong>
    <audio ref="audioPlayerRef" preload="metadata"></audio>

    <div style="display: flex; justify-content: space-between">
      <span>{{ formatTime(seekValue) }}</span>
      <span>{{ formatTime(totalDuration) }}</span>
    </div>

    <div
      id="audio-player-container"
      class="mt-1"
      style="background-color: rgba(243, 245, 249, 1); border-radius: 12px"
    >
      <div style="padding: 8px 12px">
        <input
          type="range"
          class="seek-slider"
          :max="totalDuration"
          step="0.01"
          :value="seekValue"
          @input="onSeekInput"
          :style="{ '--fill': fillPercent }"
        />
      </div>
    </div>

    <div class="flex justify-center mt-4">
      <div class="mr-3" @click="skipBack">
        <img
          style="width: 47px; height: 47px; cursor: pointer"
          :src="playBack"
        />
      </div>
      <div class="mr-3" @click="togglePlayAudio">
        <img
          style="width: 47px; height: 47px; cursor: pointer"
          :src="isPlayAudio ? pauseAudio : playAudio"
        />
      </div>
      <div @click="skipForward">
        <img
          style="width: 47px; height: 47px; cursor: pointer"
          :src="playRight"
        />
      </div>
    </div>
  </Dialog>
</template>

<style scoped>
/* Your CSS remains the same */
.seek-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 12px;
  border-radius: 12px;
  outline: none;
  background: #e7e7e780;
  background-image: linear-gradient(#035f9e, #035f9e);
  background-repeat: no-repeat;
  background-size: var(--fill, 0%) 100%;
}
.seek-slider::-webkit-slider-runnable-track {
  height: 12px;
  border-radius: 12px;
  background: transparent;
}
.seek-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 12px;
  height: 12px;
  background: #fff;
  border-radius: 6px;
  cursor: pointer;
  box-shadow: 0 0 0 6px #035f9e, 0 0 0 8px white;
}
.seek-slider::-moz-range-track {
  height: 12px;
  border-radius: 12px;
  background: #eef2f7;
}
.seek-slider::-moz-range-progress {
  height: 12px;
  border-radius: 12px;
  background: #2f80ed;
}
</style>
