<template>
  <div class="h5-room-wrap">
    <div class="head">
      <div class="left">
        <div
          class="avatar"
          :style="{
            backgroundImage: `url(${anchorInfo?.avatar})`,
          }"
        ></div>
        <div class="username">
          {{ anchorInfo?.username }}
        </div>
      </div>
      <div class="right">
        <div
          class="btn"
          @click="router.push({ name: mobileRouterName.h5 })"
        >
          返回主页
        </div>
      </div>
    </div>
    <div
      v-loading="videoLoading"
      class="video-wrap"
      :style="{ height: videoWrapHeight + 'px' }"
    >
      <div
        class="cover"
        :style="{
          backgroundImage: `url(${appStore.liveRoomInfo?.cover_img})`,
        }"
      ></div>
      <div
        v-if="!roomLiving"
        class="no-live"
      >
        主播还没开播~
      </div>
      <div
        class="media-list"
        ref="remoteVideoRef"
        :class="{ item: appStore.allTrack.length > 1 }"
      ></div>
      <div
        v-if="showPlayBtn && roomLiving && appStore.liveRoomInfo"
        class="tip-btn"
        @click="startPull"
      >
        点击播放
      </div>
      <VideoControls
        v-else
        :resolution="videoHeight"
        @refresh="handleRefresh"
      ></VideoControls>
    </div>
    <div class="danmu-list">
      <div class="title">弹幕专区</div>
      <div
        ref="containerRef"
        class="list"
        :style="{ height: containerHeight + 'px' }"
      >
        <div
          v-for="(item, index) in damuList"
          :key="index"
          class="item"
        >
          <template v-if="item.msgType === DanmuMsgTypeEnum.danmu">
            <span class="name">
              {{ item.userInfo?.username || item.socket_id }}：
            </span>
            <span class="msg">{{ item.msg }}</span>
          </template>
          <template v-else-if="item.msgType === DanmuMsgTypeEnum.otherJoin">
            <span class="name system">系统通知：</span>
            <span class="msg">
              {{ item.userInfo?.username || item.socket_id }}进入直播！
            </span>
          </template>
          <template v-else-if="item.msgType === DanmuMsgTypeEnum.userLeaved">
            <span class="name system">系统通知：</span>
            <span class="msg">
              {{ item.userInfo?.username || item.socket_id }}离开直播！
            </span>
          </template>
        </div>
      </div>
    </div>
    <div
      ref="bottomRef"
      class="send-msg"
    >
      <input
        v-model="danmuStr"
        class="ipt"
        @keydown="keydownDanmu"
      />
      <n-button
        type="info"
        size="small"
        color="#ffd700"
        @click="sendDanmu"
      >
        发送
      </n-button>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { nextTick, onMounted, onUnmounted, ref, watch } from 'vue';
import { useRoute } from 'vue-router';

import { fetchFindLiveRoom } from '@/api/liveRoom';
import { usePull } from '@/hooks/use-pull';
import { DanmuMsgTypeEnum, LiveRoomTypeEnum } from '@/interface';
import router, { mobileRouterName } from '@/router';
import { useAppStore } from '@/store/app';
import { usePiniaCacheStore } from '@/store/cache';

const route = useRoute();
const cacheStore = usePiniaCacheStore();
const appStore = useAppStore();

const bottomRef = ref<HTMLDivElement>();
const containerRef = ref<HTMLDivElement>();
const showPlayBtn = ref(false);
const containerHeight = ref(0);
const videoWrapHeight = ref(0);
const remoteVideoRef = ref<HTMLDivElement>();

const {
  handlePlay,
  initPull,
  keydownDanmu,
  sendDanmu,
  closeRtc,
  closeWs,
  autoplayVal,
  videoLoading,
  damuList,
  danmuStr,
  roomLiving,
  anchorInfo,
  remoteVideo,
  videoHeight,
} = usePull();

watch(
  () => remoteVideo.value,
  (newVal) => {
    newVal.forEach((item) => {
      remoteVideoRef.value?.appendChild(item);
    });
  },
  {
    deep: true,
    immediate: true,
  }
);

watch(
  () => damuList.value.length,
  () => {
    setTimeout(() => {
      if (containerRef.value) {
        containerRef.value.scrollTop = containerRef.value.scrollHeight;
      }
    }, 0);
  }
);

function handleRefresh() {
  if (appStore.liveRoomInfo) {
    handlePlay(appStore.liveRoomInfo);
  }
}

async function getLiveRoomInfo() {
  try {
    videoLoading.value = true;
    const res = await fetchFindLiveRoom(route.params.roomId as string);
    if (res.code === 200) {
      appStore.setLiveRoomInfo(res.data);
      if (res.data.type === LiveRoomTypeEnum.user_wertc) {
        autoplayVal.value = true;
        showPlayBtn.value = false;
      } else {
        showPlayBtn.value = true;
      }
      initPull(autoplayVal.value);
    }
  } catch (error) {
    console.error(error);
  } finally {
    videoLoading.value = false;
  }
}

function startPull() {
  cacheStore.setMuted(false);
  showPlayBtn.value = false;
  handlePlay(appStore.liveRoomInfo!);
}
onUnmounted(() => {
  closeWs();
  closeRtc();
});

onMounted(() => {
  setTimeout(() => {
    scrollTo(0, 0);
  }, 100);
  videoWrapHeight.value =
    document.documentElement.clientWidth / appStore.videoRatio;
  nextTick(() => {
    if (containerRef.value && bottomRef.value) {
      const res =
        bottomRef.value.getBoundingClientRect().top -
        containerRef.value.getBoundingClientRect().top;
      containerHeight.value = res;
    }
  });
  getLiveRoomInfo();
});
</script>

<style lang="scss" scoped>
.h5-room-wrap {
  .head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    box-sizing: border-box;
    padding: 0 20px;
    width: 100%;
    height: 70px;
    background-color: black;
    color: white;
    .left {
      display: flex;
      align-items: center;
      .avatar {
        width: 40px;
        height: 40px;
        border-radius: 50%;

        @extend %containBg;
      }
      .username {
        margin-left: 10px;
      }
    }
    .right {
      .btn {
      }
    }
  }
  .video-wrap {
    position: relative;
    overflow: hidden;
    flex: 1;
    background-color: rgba($color: #000000, $alpha: 0.5);
    .cover {
      position: absolute;
      background-position: center center;
      background-size: cover;
      filter: blur(10px);

      inset: 0;
    }
    .no-live {
      position: absolute;
      top: 50%;
      left: 50%;
      z-index: 20;
      color: white;
      font-size: 28px;
      transform: translate(-50%, -50%);
    }
    .media-list {
      position: relative;
      overflow-y: scroll;
      :deep(video) {
        display: block;
        width: 100%;
        height: 100%;
      }
      :deep(canvas) {
        display: block;
        width: 100%;
        height: 100%;
      }
    }
    .tip-btn {
      position: absolute;
      top: 50%;
      left: 50%;
      z-index: 20;
      align-items: center;
      padding: 10px 20px;
      border: 2px solid rgba($color: papayawhip, $alpha: 0.5);
      border-radius: 6px;
      background-color: rgba(0, 0, 0, 0.3);
      color: $theme-color-gold;
      font-size: 14px;
      cursor: pointer;
      transform: translate(-50%, -50%);
      &:hover {
        background-color: rgba($color: papayawhip, $alpha: 0.5);
        color: white;
      }
    }
  }

  .danmu-list {
    box-sizing: border-box;
    padding: 0 15px;
    background-color: #0c1622;
    text-align: initial;
    .title {
      padding: 15px 0;
      color: #fff;
      font-size: 16px;
    }
    .list {
      overflow-y: scroll;
      height: 100vh;

      @extend %hideScrollbar;
    }
    .item {
      margin-bottom: 10px;
      font-size: 12px;
      .name {
        color: #ccc;
        &.system {
          color: red;
        }
      }
      .msg {
        color: #fff;
      }
    }
  }
  .send-msg {
    position: fixed;
    bottom: 0;
    left: 0;
    display: flex;
    align-items: center;
    justify-content: space-evenly;
    box-sizing: border-box;
    padding: 0 10px;
    width: 100%;
    height: 40px;
    background-color: #0c1622;
    .ipt {
      display: block;
      box-sizing: border-box;
      padding: 10px;
      width: 80%;
      height: 30px;
      outline: none;
      border: 1px solid hsla(0, 0%, 60%, 0.2);
      border-radius: 4px;
      background-color: #f1f2f3;
      font-size: 14px;
    }
  }
}
</style>
