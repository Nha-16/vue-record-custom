<template>
  <div class="card">
    <DataTable
      :value="customers"
      paginator
      :rows="5"
      :rowsPerPageOptions="[5, 10, 20, 50]"
      tableStyle="min-width: 50rem"
    >
      <Column field="name" header="Name" style="width: 25%"></Column>
      <Column header="Voice Record" style="width: 20%">
        <template #body="{ data }">
          <span style="cursor: pointer" @click="openDialog(data)">
            Open
            <i class="pi pi-volume-up"></i>
          </span>
        </template>
      </Column>
    </DataTable>
  </div>

  <Dialog
    v-model:visible="dialogVisible"
    :closable="true"
    modal
    :draggable="false"
    :style="{ width: '420px' }"
  >
    <div class="flex justify-center mt-2">
      <i class="pi pi-microphone" style="font-size: 3rem"></i>
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
        <i
          class="pi pi-step-backward-alt"
          style="font-size: 2rem; cursor: pointer"
        ></i>
      </div>
      <div class="mr-3" @click="togglePlayAudio">
        <i
          :class="{ 'pi pi-play': !isPlayAudio, 'pi pi-pause': isPlayAudio }"
          style="font-size: 2rem; cursor: pointer"
        ></i>
      </div>
      <div @click="skipForward">
        <i
          class="pi pi-step-forward-alt"
          style="font-size: 2rem; cursor: pointer"
        ></i>
      </div>
    </div>
  </Dialog>
</template>
<script setup lang="ts">
import { ref, computed, watch, nextTick, onUnmounted } from 'vue';
import DataTable from 'primevue/datatable';
import Column from 'primevue/column';
import Dialog from 'primevue/dialog';
import 'primeicons/primeicons.css';

// Local state for the dialog and audio player
const dialogVisible = ref(false);
const audioPlayerRef = ref<HTMLAudioElement | null>(null);
const isPlayAudio = ref(false);
const seekValue = ref(0);
const totalDuration = ref(0);
const selectedCustomerVoiceUrl = ref<string | null>(null);

// Static customer data
const customers = ref([
  {
    id: 1000,
    name: 'James Butt',
    country: { name: 'Algeria', code: 'dz' },
    company: 'Benton, John B Jr',
    representative: { name: 'Ioni Bowcher' },
    voiceUrl: 'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3',
  },
  {
    id: 1001,
    name: 'Josephine Darakjy',
    country: { name: 'Egypt', code: 'eg' },
    company: 'Chanay, Jeffrey A Esq',
    representative: { name: 'Amy Elsner' },
    voiceUrl: 'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-2.mp3',
  },
  {
    id: 1002,
    name: 'Art Venere',
    country: { name: 'United States', code: 'us' },
    company: 'Chemel, James L Cpa',
    representative: { name: 'Asiya Javayant' },
    voiceUrl: null,
  },
]);

// Methods to control the audio player
const openDialog = (customerData: any) => {
  // Check if a voice URL exists before opening the dialog
  if (customerData.voiceUrl) {
    selectedCustomerVoiceUrl.value = customerData.voiceUrl;
    dialogVisible.value = true;

    // Use nextTick to ensure the audio element is mounted before setting its source
    nextTick(() => {
      const audio = audioPlayerRef.value;
      if (audio) {
        audio.src = customerData.voiceUrl;
        audio.load();

        isPlayAudio.value = false;
        seekValue.value = 0;
        totalDuration.value = 0;

        // Set up event listeners
        audio.onloadedmetadata = () => {
          totalDuration.value = audio.duration;
        };
        audio.ontimeupdate = () => {
          seekValue.value = audio.currentTime;
        };
        audio.onended = () => {
          isPlayAudio.value = false;
          seekValue.value = 0;
        };
      }
    });
  } else {
    // Optionally, handle cases with no voice URL
    console.log('No voice record available for this customer.');
  }
};

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
    audio.currentTime = Math.max(audio.currentTime - 10, 0);
  }
};

const skipForward = () => {
  const audio = audioPlayerRef.value;
  if (audio) {
    audio.currentTime = Math.min(audio.currentTime + 10, totalDuration.value);
  }
};

const onSeekInput = (event: Event) => {
  const audio = audioPlayerRef.value;
  if (audio) {
    const target = event.target as HTMLInputElement;
    audio.currentTime = parseFloat(target.value);
  }
};

const formatTime = (seconds: number) => {
  if (isNaN(seconds) || seconds < 0) return '0:00';
  const m = Math.floor(seconds / 60);
  const s = Math.floor(seconds % 60);
  return `${m}:${String(s).padStart(2, '0')}`;
};

// Computed property for slider fill percent
const fillPercent = computed(() => {
  const max = totalDuration.value || 1;
  return `${(seekValue.value / max) * 100}%;`;
});

// Watch dialog visibility to stop audio when closed
watch(dialogVisible, (open) => {
  const audio = audioPlayerRef.value;
  if (!open && audio) {
    audio.pause();
    audio.currentTime = 0;
    isPlayAudio.value = false;
    // Remove listeners to prevent memory leaks and unexpected behavior
    audio.onloadedmetadata = null;
    audio.ontimeupdate = null;
    audio.onended = null;
  }
});

onUnmounted(() => {
  const audio = audioPlayerRef.value;
  if (audio) {
    audio.onloadedmetadata = null;
    audio.ontimeupdate = null;
    audio.onended = null;
  }
});
</script>
<style scoped>
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
