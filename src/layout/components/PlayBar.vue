<template>
  <!-- 展开全屏 -->
  <music-full ref="MusicFullRef" v-model:music-full="musicFullVisible" :background="background" />
  <!-- 底部播放栏 -->

  <div
    class="music-play-bar"
    :class="setAnimationClass('animate__bounceInUp') + ' ' + (musicFullVisible ? 'play-bar-opcity' : '')"
  >
    <div class="music-time custom-slider">
      <n-slider v-model:value="timeSlider" :step="1" :max="allTime" :min="0" :format-tooltip="formatTooltip"></n-slider>
    </div>
    <div class="play-bar-img-wrapper" @click="setMusicFull">
      <n-image :src="getImgUrl(playMusic?.picUrl, '300y300')" class="play-bar-img" lazy preview-disabled />
      <div class="hover-arrow">
        <div class="hover-content">
          <!-- <i class="ri-arrow-up-s-line text-3xl" :class="{ 'ri-arrow-down-s-line': musicFullVisible }"></i> -->
          <i class="text-3xl" :class="musicFullVisible ? 'ri-arrow-down-s-line' : 'ri-arrow-up-s-line'"></i>
          <span class="hover-text">{{ musicFullVisible ? '收起' : '展开' }}歌词</span>
        </div>
      </div>
    </div>
    <div class="music-content">
      <div class="music-content-title">
        <n-ellipsis class="text-ellipsis" line-clamp="1">
          {{ playMusic.name }}
        </n-ellipsis>
      </div>
      <div class="music-content-name">
        <n-ellipsis class="text-ellipsis" line-clamp="1">
          <span v-for="(artists, artistsindex) in playMusic.ar || playMusic.song.artists" :key="artistsindex"
            >{{ artists.name
            }}{{ artistsindex < (playMusic.ar || playMusic.song.artists).length - 1 ? ' / ' : '' }}</span
          >
        </n-ellipsis>
      </div>
    </div>
    <div class="music-buttons">
      <div class="music-buttons-prev" @click="handlePrev">
        <i class="iconfont icon-prev"></i>
      </div>
      <div class="music-buttons-play" @click="playMusicEvent">
        <i class="iconfont icon" :class="play ? 'icon-stop' : 'icon-play'"></i>
      </div>
      <div class="music-buttons-next" @click="handleNext">
        <i class="iconfont icon-next"></i>
      </div>
    </div>
    <div class="audio-button">
      <div class="audio-volume custom-slider">
        <div class="volume-icon" @click="mute">
          <i class="iconfont" :class="getVolumeIcon"></i>
        </div>
        <div class="volume-slider">
          <n-slider v-model:value="volumeSlider" :step="0.01" :tooltip="false" vertical></n-slider>
        </div>
      </div>
      <n-tooltip trigger="hover" :z-index="9999999">
        <template #trigger>
          <i class="iconfont" :class="playModeIcon" @click="togglePlayMode"></i>
        </template>
        {{ playModeText }}
      </n-tooltip>
      <n-tooltip trigger="hover" :z-index="9999999">
        <template #trigger>
          <i class="iconfont icon-likefill" :class="{ 'like-active': isFavorite }" @click="toggleFavorite"></i>
        </template>
        喜欢
      </n-tooltip>
      <n-tooltip v-if="isElectron" class="music-lyric" trigger="hover" :z-index="9999999">
        <template #trigger>
          <i
            class="iconfont ri-netease-cloud-music-line"
            :class="{ 'text-green-500': isLyricWindowOpen }"
            @click="openLyricWindow"
          ></i>
        </template>
        歌词
      </n-tooltip>
      <n-popover
        trigger="click"
        :z-index="99999999"
        content-class="music-play"
        raw
        :show-arrow="false"
        :delay="200"
        arrow-wrapper-style=" border-radius:1.5rem"
        @update-show="scrollToPlayList"
      >
        <template #trigger>
          <n-tooltip trigger="manual" :z-index="9999999">
            <template #trigger>
              <i class="iconfont icon-list"></i>
            </template>
            播放列表
          </n-tooltip>
        </template>
        <div class="music-play-list">
          <div class="music-play-list-back"></div>
          <n-virtual-list ref="palyListRef" :item-size="62" item-resizable :items="playList">
            <template #default="{ item }">
              <div class="music-play-list-content">
                <song-item :key="item.id" :item="item" mini></song-item>
              </div>
            </template>
          </n-virtual-list>
        </div>
      </n-popover>
    </div>
    <!-- 播放音乐 -->
  </div>
</template>

<script lang="ts" setup>
import { useThrottleFn } from '@vueuse/core';
import { useTemplateRef } from 'vue';
import { useStore } from 'vuex';

import SongItem from '@/components/common/SongItem.vue';
import { allTime, isElectron, isLyricWindowOpen, nowTime, openLyric, sound } from '@/hooks/MusicHook';
import type { SongResult } from '@/type/music';
import { getImgUrl, secondToMinute, setAnimationClass } from '@/utils';

import MusicFull from './MusicFull.vue';

const store = useStore();

// 播放的音乐信息
const playMusic = computed(() => store.state.playMusic as SongResult);
// 是否播放
const play = computed(() => store.state.play as boolean);
const playList = computed(() => store.state.playList as SongResult[]);

const background = ref('#000');

watch(
  () => store.state.playMusic,
  async () => {
    background.value = playMusic.value.backgroundColor as string;
  },
  { immediate: true, deep: true },
);

// 使用 useThrottleFn 创建节流版本的 seek 函数
const throttledSeek = useThrottleFn((value: number) => {
  if (!sound.value) return;
  sound.value.seek(value);
  nowTime.value = value;
}, 50); // 50ms 的节流延迟

// 修改 timeSlider 计算属性
const timeSlider = computed({
  get: () => nowTime.value,
  set: throttledSeek,
});

const formatTooltip = (value: number) => {
  return `${secondToMinute(value)} / ${secondToMinute(allTime.value)}`;
};

// 音量条
const audioVolume = ref(localStorage.getItem('volume') ? parseFloat(localStorage.getItem('volume') as string) : 1);
const getVolumeIcon = computed(() => {
  // 0 静音 ri-volume-mute-line 0.5 ri-volume-down-line 1 ri-volume-up-line
  if (audioVolume.value === 0) {
    return 'ri-volume-mute-line';
  }
  if (audioVolume.value <= 0.5) {
    return 'ri-volume-down-line';
  }
  return 'ri-volume-up-line';
});

const volumeSlider = computed({
  get: () => audioVolume.value * 100,
  set: (value) => {
    if (!sound.value) return;
    localStorage.setItem('volume', (value / 100).toString());
    sound.value.volume(value / 100);
    audioVolume.value = value / 100;
  },
});

// 静音
const mute = () => {
  if (volumeSlider.value === 0) {
    volumeSlider.value = 30;
  } else {
    volumeSlider.value = 0;
  }
};

// 播放模式
const playMode = computed(() => store.state.playMode);
const playModeIcon = computed(() => {
  return playMode.value === 0 ? 'ri-repeat-2-line' : 'ri-repeat-one-line';
});
const playModeText = computed(() => {
  return playMode.value === 0 ? '列表循环' : '单曲循环';
});

// 切换播放模式
const togglePlayMode = () => {
  store.commit('togglePlayMode');
};

function handleNext() {
  store.commit('nextPlay');
}

function handlePrev() {
  store.commit('prevPlay');
}

const MusicFullRef = ref<any>(null);

// 播放暂停按钮事件
const playMusicEvent = async () => {
  if (play.value) {
    if (sound.value) {
      sound.value.pause();
    }
    store.commit('setPlayMusic', false);
  } else {
    if (sound.value) {
      sound.value.play();
    }
    store.commit('setPlayMusic', true);
  }
};

const musicFullVisible = ref(false);

// 设置musicFull
const setMusicFull = () => {
  musicFullVisible.value = !musicFullVisible.value;
};

const palyListRef = useTemplateRef('palyListRef');

const scrollToPlayList = (val: boolean) => {
  if (!val) return;
  setTimeout(() => {
    palyListRef.value?.scrollTo({ top: store.state.playListIndex * 62 });
  }, 50);
};

const isFavorite = computed(() => {
  return store.state.favoriteList.includes(playMusic.value.id);
});

const toggleFavorite = async (e: Event) => {
  e.stopPropagation();
  if (isFavorite.value) {
    store.commit('removeFromFavorite', playMusic.value.id);
  } else {
    store.commit('addToFavorite', playMusic.value.id);
  }
};

const openLyricWindow = () => {
  openLyric();
};
</script>

<style lang="scss" scoped>
.text-ellipsis {
  width: 100%;
}

.music-play-bar {
  @apply h-20 w-full absolute bottom-0 left-0 flex items-center box-border px-6 py-2 pt-3;
  z-index: 9999;
  box-shadow: 0px 0px 10px 2px rgba(203, 203, 203, 0.034);
  background-color: #212121;
  animation-duration: 0.5s !important;
  .music-content {
    width: 160px;
    @apply ml-4;

    &-title {
      @apply text-base text-white;
    }

    &-name {
      @apply text-xs mt-1 text-gray-100;
    }
  }
}

.play-bar-opcity {
  @apply bg-transparent;
  box-shadow: 0 0 20px 5px #0000001d;
}

.play-bar-img {
  @apply w-14 h-14 rounded-2xl;
}

.music-buttons {
  @apply mx-6 flex-1 flex justify-center;

  .iconfont {
    @apply text-2xl hover:text-green-500 transition;
  }

  .icon {
    @apply text-3xl hover:text-white;
  }

  @apply flex items-center;

  > div {
    @apply cursor-pointer;
  }

  &-play {
    background-color: #ffffff20;
    @apply flex justify-center items-center w-20 h-12 rounded-full mx-4 hover:bg-[#ffffff40] transition;
  }
}

.audio-volume {
  @apply flex items-center relative;
  &:hover {
    .volume-slider {
      @apply opacity-100 visible;
    }
  }
  .volume-icon {
    @apply cursor-pointer;

    .iconfont {
      @apply text-2xl hover:text-green-500 transition;
    }
  }

  .volume-slider {
    @apply absolute opacity-0 invisible transition-all duration-300 bottom-[30px] left-1/2 -translate-x-1/2 h-[180px] px-2 py-4 bg-gray-800 bg-opacity-80 rounded-xl;
  }
}

.audio-button {
  @apply flex items-center mx-4;

  .iconfont {
    @apply text-2xl hover:text-green-500 transition cursor-pointer m-4;
  }
}

.music-play {
  &-list {
    height: 50vh;
    width: 300px;
    @apply relative rounded-3xl overflow-hidden py-2;
    &-back {
      backdrop-filter: blur(20px);
      @apply absolute top-0 left-0 w-full h-full bg-gray-800 bg-opacity-75;
    }
    &-content {
      @apply mx-2;
    }
  }
}

.mobile {
  .music-play-bar {
    @apply px-4;
    bottom: 70px;
  }
  .music-time {
    display: none;
  }
  .ri-netease-cloud-music-line {
    display: none;
  }
  .audio-volume {
    display: none;
  }
  .audio-button {
    @apply mx-0;
  }
  .music-buttons {
    @apply m-0;
    &-prev,
    &-next {
      display: none;
    }
    &-play {
      @apply m-0;
    }
  }
  .music-content {
    flex: 1;
  }
}

// 添加自定义 slider 样式
.custom-slider {
  :deep(.n-slider) {
    --n-rail-height: 4px;
    --n-rail-color: rgba(255, 255, 255, 0.2);
    --n-fill-color: var(--primary-color);
    --n-handle-size: 12px;
    --n-handle-color: var(--primary-color);

    &.n-slider--vertical {
      height: 100%;

      .n-slider-rail {
        width: 4px;
      }

      &:hover {
        .n-slider-rail {
          width: 6px;
        }

        .n-slider-handle {
          width: 14px;
          height: 14px;
        }
      }
    }

    .n-slider-rail {
      @apply overflow-hidden transition-all duration-200;
    }

    .n-slider-handle {
      @apply transition-all duration-200;
      opacity: 0;
    }

    &:hover .n-slider-handle {
      opacity: 1;
    }
  }
}

:root {
  --primary-color: #18a058;
}

.play-bar-img-wrapper {
  @apply relative cursor-pointer w-14 h-14;

  .hover-arrow {
    @apply absolute inset-0 flex items-center justify-center opacity-0 transition-opacity duration-300 rounded-2xl;
    background: rgba(0, 0, 0, 0.5);

    .hover-content {
      @apply flex flex-col items-center justify-center;

      i {
        @apply text-white mb-0.5;
      }

      .hover-text {
        @apply text-white text-xs scale-90;
      }
    }
  }

  &:hover {
    .hover-arrow {
      @apply opacity-100;
    }
  }
}

.tooltip-content {
  @apply text-sm py-1 px-2;
}

.play-bar-img {
  @apply w-14 h-14 rounded-2xl;
}

.like-active {
  @apply text-red-600;
}

.icon-loop,
.icon-single-loop {
  font-size: 1.5rem;
}

.music-time .n-slider {
  position: absolute;
  top: 0;
  left: 0;
  padding: 0;
  border-radius: 0;
}
</style>
