<template>
  <div>
    <!-- 播放器 -->
    <q-card class="fixed-bottom-right audio-player q-pa-sm text-black" @mousewheel.prevent @touchmove.prevent
      color="primary" :class="{ showStyle: showAudioPlayer, hideStyle: !showAudioPlayer }"
      :style="{ '--cover-url': `url(${coverUrl})` }">
      <!-- 顶部小横条 -->
      <div class="pull-handler" @click="toggleHide" v-touch-swipe.mouse.down="toggleHide"></div>

      <!-- 音声封面 -->
      <div class="row items-center albumart q-my-lg q-pa-sm relative-position non-selectable">
        <q-img cover class="rounded-borders shadow-2" transition="fade" :src="coverUrl" :ratio="4 / 3"
          @dblclick.prevent="openWorkDetail()" />
      </div>

      <!-- Place holder for iOS -->
      <!-- <div style="height: 5px" v-if="$q.platform.is.ios" /> -->

      <!-- 标题和当前音轨名字 -->
      <div class="text-center q-mb-sm column">
        <div class="ellipsis-2-lines text-bold q-pb-xs" style="width: 100%;">
          {{ currentPlayingFile.title }}
        </div>
        <Scrollable :stop="hide">
          <span class="text-caption" style="filter: opacity(0.54);">
            {{ currentPlayingFile.workTitle }}</span>
        </Scrollable>
      </div>

      <!-- Place holder for iOS -->
      <div style="height: 10px" v-if="$q.platform.is.ios" />

      <!-- 进度条控件 -->
      <div class="row items-center q-mx-lg">
        <AudioElement />
        <div class="col-12 flex justify-between">
          <div style="display: inline;">{{ formatSeconds(currentTime) }}</div>
          <div style="display: inline;">{{ formatSeconds(duration) }}</div>
        </div>
      </div>

      <!-- Place holder for iOS -->
      <div style="height: 10px" v-if="$q.platform.is.ios" />

      <!-- 播放按钮 -->
      <div class="row flex-center">
        <!-- 上一曲目 -->
        <q-btn flat dense size="lg" :icon="swapSeekButton ? rewindIcon : 'skip_previous'"
          @click="swapSeekButton ? rewind(true) : previousTrack()" style="width: 55px" class="col-auto" />
        <!-- 后退x秒 -->
        <q-btn flat dense size="lg" :icon="!swapSeekButton ? rewindIcon : 'skip_previous'" @click="rewind(true)"
          style="width: 55px;" class="col-auto" />
        <!-- 暂停/播放 -->
        <q-btn flat dense size="30px" :icon="playingIcon" @click="togglePlaying()" style="width: 60px"
          class="col-auto" />
        <!-- 前进x秒 -->
        <q-btn flat dense size="lg" :icon="!swapSeekButton ? forwardIcon : 'skip_next'" @click="forward(true)"
          style="width: 55px" class="col-auto" />
        <!-- 下一曲目 -->
        <q-btn flat dense size="lg" :icon="swapSeekButton ? forwardIcon : 'skip_next'"
          @click="swapSeekButton ? forward(true) : nextTrack()" style="width: 55px" class="col-auto" />
      </div>

      <!-- 音量控件 -->
      <!-- HTML5 volume in iOS is read-only -->
      <div class="row items-center q-mx-lg q-pt-sm">
        <q-icon name="volume_down" size="sm" class="col-auto" />
        <a-slider v-model="volume" :disabled="$q.platform.is.ios" :min="0" :max="1" :step="0.01" class="col q-mx-md" />
        <q-icon name="volume_up" size="sm" class="col-auto" />
      </div>

      <!-- 设置菜单 -->
      <div class="row self-center">
        <!-- 当前播放列表 -->
        <q-btn flat dense size="md" padding="none sm" icon="queue_music"
          @click="showCurrentPlayList = !showCurrentPlayList" class="q-ma-sm">
          <q-tooltip>打开当前播放列表</q-tooltip>
        </q-btn>
        <!-- 播放模式 -->
        <q-btn flat dense size="md" padding="none sm" :icon="playModeIcon" @click="changePlayMode()" class="q-ma-sm">
          <q-tooltip>播放模式: {{ this.playModeName() }}</q-tooltip>
        </q-btn>
        <!-- 打开作品详情 -->
        <q-btn flat dense size="md" padding="none sm" icon="link" @click="openWorkDetail()" class="q-ma-sm">
          <q-tooltip>打开作品详情</q-tooltip>
        </q-btn>
        <!-- 播放器设置 -->
        <q-btn flat dense size="md" padding="none sm" icon="tune" @click="openPlayerSettings()" class="q-ma-sm">
          <q-tooltip>打开播放器设置</q-tooltip>
        </q-btn>

      </div>
    </q-card>

    <!-- 当前播放列表 -->
    <q-dialog v-model="showCurrentPlayList">
      <q-card class="current-play-list">
        <!-- 操作当前播放列表的控制按钮 -->
        <div class="row" style="padding: 5px; height: 45px;">
          <q-btn dense round size="md" icon="edit" color="primary" @click="editCurrentPlayList = !editCurrentPlayList"
            style="height: 35px; width: 35px;" class="col-auto" />
          <q-btn dense round size="md" icon="save" color="teal" style="height: 35px; width: 35px;"
            class="col-auto q-mx-sm" />
          <q-space />
          <q-btn dense round size="md" icon="delete_forever" color="red" @click="emptyQueue()"
            style="height: 35px; width: 35px;" class="col-auto" />
        </div>

        <q-separator />

        <!-- 音频文件列表 -->
        <q-list style="max-height: 450px" class="scroll">
          <draggable handle=".handle" v-model="queueCopy" @change="val => onMoved(val.moved)">
            <q-item clickable v-ripple v-for="(track, index) in queueCopy" :key="index" :active="queueIndex === index"
              active-class="text-white bg-teal" class="non-selectable" style="height: 48px; padding: 0px 10px;"
              @click="onClickTrack(index)">
              <q-item-section side v-show="editCurrentPlayList">
                <q-icon name="clear" :color="queueIndex === index ? 'white' : 'red'" @click="removeFromQueue(index)" />
              </q-item-section>

              <q-item-section avatar>
                <q-img transition="fade" :src="samCoverUrl(track.hash)" style="height: 38px; width: 38px"
                  class="rounded-borders" />
              </q-item-section>

              <q-item-section>
                <q-item-label lines="1">{{ track.title }}</q-item-label>
                <q-item-label caption lines="1">{{
                  track.workTitle
                }}</q-item-label>
              </q-item-section>

              <q-item-section side class="handle" v-show="editCurrentPlayList">
                <q-icon name="reorder" :color="queueIndex === index ? 'white' : 'dark'" />
              </q-item-section>
            </q-item>
          </draggable>
        </q-list>
      </q-card>
    </q-dialog>

    <!-- 播放器设置 -->
    <q-dialog v-model="showPlayerSettings">
      <q-card class="player-settings q-py-sm justify-center">
        <q-toolbar>
          <q-toolbar-title>播放器设置</q-toolbar-title>
        </q-toolbar>
        <q-list bordered separator>
          <!-- 倒带跳跃秒数设置 -->
          <q-item>
            <q-item-section avatar>
              <q-icon size="md" name="fast_rewind" />
            </q-item-section>
            <q-item-section main>
              <q-item-label>倒带跳跃秒数</q-item-label>
            </q-item-section>
            <q-item-section side>
              <q-select borderless v-model="rewindSeekTimeModel" :options="seekTimeOptions"
                @input="setRewindSeekTime" />
            </q-item-section>
          </q-item>
          <!-- 前进跳跃秒数设置 -->
          <q-item>
            <q-item-section avatar>
              <q-icon flat dense size="md" name="fast_forward" />
            </q-item-section>
            <q-item-section main>
              <q-item-label>前进跳跃秒数</q-item-label>
            </q-item-section>
            <q-item-section side>
              <q-select borderless v-model="forwardSeekTimeModel" :options="seekTimeOptions"
                @input="setForwardSeekTime" />
            </q-item-section>
          </q-item>
          <!-- 交换进度与切换按钮 -->
          <q-item>
            <q-item-section avatar>
              <q-icon flat dense size="md" name="swap_horiz" />
            </q-item-section>
            <q-item-section main>
              <q-item-label>交换进度与切换按钮</q-item-label>
            </q-item-section>
            <q-item-section side>
              <q-toggle v-model="swapSeekButton" @change="swapSeekButton = !swapSeekButton" />
            </q-item-section>
          </q-item>
          <q-item>
            <q-item-section>
              <q-btn label="关闭" @click="hidePlayerSettings" />
            </q-item-section>
          </q-item>
        </q-list>
      </q-card>
    </q-dialog>
  </div>
</template>

<script>
import draggable from "vuedraggable";
import AudioElement from "components/AudioElement";
import Scrollable from "components/Scrollable";
import { mapState, mapGetters, mapMutations } from "vuex";
import NotifyMixin from '../mixins/Notification.js'

export default {
  name: "AudioPlayer",

  mixins: [NotifyMixin],

  components: {
    draggable,
    AudioElement,
    Scrollable,
  },

  data() {
    return {
      showCurrentPlayList: false,
      editCurrentPlayList: false,
      queueCopy: [],
      hideSeekButton: false,
      swapSeekButton: false,
      showPlayerSettings: false,
      rewindSeekTimeModel: { label: "5 秒", value: 5 },
      forwardSeekTimeModel: { label: "30 秒", value: 30 },
      seekTimeOptions: [
        { label: "5 秒", value: 5 },
        { label: "10 秒", value: 10 },
        { label: "30 秒", value: 30 }
      ]
    };
  },

  mounted() {
    if (this.$q.localStorage.has("hideSeekButton")) {
      this.hideSeekButton = this.$q.localStorage.getItem("hideSeekButton");
    }
    if (this.$q.localStorage.has("swapSeekButton")) {
      this.swapSeekButton = this.$q.localStorage.getItem("swapSeekButton");
    }
  },

  watch: {
    queue(val) {
      this.queueCopy = val.concat();
      // 在删除最后一个 track 时关闭当前播放列表
      if (this.queueCopy.length === 0) {
        this.showCurrentPlayList = false;
      }
    },

    showCurrentPlayList(flag) {
      // 关闭当前播放列表后，重置 editCurrentPlayList 状态为 false
      if (flag === false) {
        this.editCurrentPlayList = false;
      }
    },

    hideSeekButton(option) {
      this.$q.localStorage.set("hideSeekButton", option);
    },

    swapSeekButton(option) {
      this.$q.localStorage.set("swapSeekButton", option);
    }
  },

  computed: {
    showAudioPlayer() {
      return this.currentPlayingFile.hash && !this.hide;
    },

    coverUrl() {
      // 从 LocalStorage 中读取 token
      const token = this.$q.localStorage.getItem("jwt-token") || "";
      const hash = this.currentPlayingFile.hash;
      return hash ? `/api/cover/${hash.split("/")[0]}?token=${token}` : "";
    },

    workDetailUrl() {
      const hash = this.currentPlayingFile.hash;
      return hash ? `/work/RJ${hash.split("/")[0]}` : "";
    },

    volume: {
      get() {
        return this.$store.state.AudioPlayer.volume;
      },
      set(val) {
        this.SET_VOLUME(val);
      }
    },

    queue: {
      get() {
        return this.$store.state.AudioPlayer.queue;
      },
      set() { }
    },

    playModeIcon() {
      switch (this.playMode.name) {
        case "all repeat":
          return "repeat";
        case "repeat once":
          return "repeat_one";
        case "shuffle":
          return "shuffle";
        default:
          return "playlist_play";
      }
    },

    playingIcon() {
      return this.playing ? "pause" : "play_arrow";
    },

    rewindIcon() {
      switch (this.rewindSeekTime) {
        case 5:
          return "replay_5";
        case 10:
          return "replay_10";
        case 30:
          return "replay_30";
        default:
          return "replay_5";
      }
    },

    forwardIcon() {
      switch (this.forwardSeekTime) {
        case 5:
          return "forward_5";
        case 10:
          return "forward_10";
        case 30:
          return "forward_30";
        default:
          return "forward_5";
      }
    },

    ...mapState("AudioPlayer", [
      "playing",
      "hide",
      "currentTime",
      "duration",
      "queueIndex",
      "playMode",
      "rewindSeekTime",
      "forwardSeekTime"
    ]),

    ...mapGetters("AudioPlayer", ["currentPlayingFile"])
  },

  methods: {
    ...mapMutations("AudioPlayer", {
      toggleHide: "TOGGLE_HIDE",
      togglePlaying: "TOGGLE_PLAYING",
      nextTrack: "NEXT_TRACK",
      previousTrack: "PREVIOUS_TRACK",
      changePlayMode: "CHANGE_PLAY_MODE",
      setVolume: "SET_VOLUME",
      rewind: "SET_REWIND_SEEK_MODE",
      forward: "SET_FORWARD_SEEK_MODE",
    }),
    ...mapMutations("AudioPlayer", [
      "SET_TRACK",
      "SET_QUEUE",
      "REMOVE_FROM_QUEUE",
      "EMPTY_QUEUE",
      "SET_VOLUME",
      "SET_REWIND_SEEK_TIME",
      "SET_FORWARD_SEEK_TIME"
    ]),

    formatSeconds(seconds) {
      let h =
        Math.floor(seconds / 3600) < 10
          ? "0" + Math.floor(seconds / 3600)
          : Math.floor(seconds / 3600);

      let m =
        Math.floor((seconds / 60) % 60) < 10
          ? "0" + Math.floor((seconds / 60) % 60)
          : Math.floor((seconds / 60) % 60);

      let s =
        Math.floor(seconds % 60) < 10
          ? "0" + Math.floor(seconds % 60)
          : Math.floor(seconds % 60);

      return h === "00" ? m + ":" + s : h + ":" + m + ":" + s;
    },

    samCoverUrl(hash) {
      // 从 LocalStorage 中读取 token
      const token = this.$q.localStorage.getItem("jwt-token") || "";
      return hash
        ? `/api/cover/${hash.split("/")[0]}?type=sam&token=${token}`
        : "";
    },

    onClickTrack(index) {
      if (!this.editCurrentPlayList) {
        this.SET_TRACK(index);
        this.showCurrentPlayList = false;
      }
    },

    onMoved(moved) {
      let index = null;
      if (moved.oldIndex === this.queueIndex) {
        index = moved.newIndex;
      } else if (
        moved.oldIndex < this.queueIndex &&
        moved.newIndex >= this.queueIndex
      ) {
        index = this.queueIndex - 1;
      } else if (
        moved.oldIndex > this.queueIndex &&
        moved.newIndex <= this.queueIndex
      ) {
        index = this.queueIndex + 1;
      } else {
        index = this.queueIndex;
      }

      this.SET_QUEUE({
        queue: this.queueCopy.concat(),
        index: index,
        resetPlaying: false
      });
    },

    removeFromQueue(index) {
      this.REMOVE_FROM_QUEUE(index);
    },

    emptyQueue() {
      this.EMPTY_QUEUE();
    },

    playModeName() {
      switch(this.playMode.name) {
        case "order":
          return "顺序播放";
        case "all repeat":
          return "列表循环";
        case "repeat once":
          return "单曲循环";
        case "shuffle":
          return "随机播放";
      }
    },

    openWorkDetail() {
      if (this.workDetailUrl && this.$route.path !== this.workDetailUrl) {
        this.$router.push(this.workDetailUrl);
      }
      if (this.$q.screen.lt.sm) {
        this.toggleHide();
      }
    },

    openPlayerSettings() {
      this.showPlayerSettings = true;
    },

    hidePlayerSettings() {
      this.showPlayerSettings = false;
    },

    setRewindSeekTime(newRewindSeekTime) {
      this.rewindSeekTimeModel = newRewindSeekTime;
      this.SET_REWIND_SEEK_TIME(newRewindSeekTime.value);
      this.saveConfigSeekTime(newRewindSeekTime.value, 0);
    },

    setForwardSeekTime(newForwardSeekTime) {
      this.newForwardSeekTime = newForwardSeekTime;
      this.SET_FORWARD_SEEK_TIME(newForwardSeekTime.value);
      this.saveConfigSeekTime(0, newForwardSeekTime.value);
    },

    requestConfigSeekTime() {
      this.$axios.get('/api/config/admin')
        .then((response) => {
          this.config = response.data.config;

          this.rewindSeekTimeModel = { label: `${this.config.rewindSeekTime} 秒`, value: this.config.rewindSeekTime };
          this.forwardSeekTimeModel = { label: `${this.config.forwardSeekTime} 秒`, value: this.config.forwardSeekTime };
        })
        .catch((error) => {
          if (error.response) {
            // 请求已发出，但服务器响应的状态码不在 2xx 范围内
            if (error.response.status !== 401) {
              this.showErrNotif(error.response.data.error || `${error.response.status} ${error.response.statusText}`)
            }
          } else {
            this.showErrNotif(error.message || error)
          }
        })
    },

    saveConfigSeekTime(rewindSeekTime = 0, forwardSeekTime = 0) {
      this.config.rewindSeekTime = rewindSeekTime !== 0 ? rewindSeekTime : this.config.rewindSeekTime;
      this.config.forwardSeekTime = forwardSeekTime !== 0 ? forwardSeekTime : this.config.forwardSeekTime;

      this.loading = true
      this.$axios.put("/api/config/admin", {
        config: this.config
      }).then((response) => {
        this.loading = false;
        console.log(`change audio player seek time: ${response.data.message}`)
      }).catch((error) => {
        this.loading = false
        if (error.response) {
          // 请求已发出，但服务器响应的状态码不在 2xx 范围内
          this.showErrNotif(error.response.data.error || `${error.response.status} ${error.response.statusText}`)
        } else {
          this.showErrNotif(error.message || error)
        }
      })
    }

  },
  created() {
    this.requestConfigSeekTime();
  }
};
</script>

<style lang="scss" scoped>
.audio-player {

  // 宽度 > $breakpoint-sm-min
  @media (min-width: $breakpoint-sm-min) {
    width: 330px;
    margin: 0px 10px 10px 0px;
    // border-radius: 8px;
  }

  // 宽度 < $breakpoint-xs-max (599px)
  @media (max-width: $breakpoint-xs-max) {
    width: 100%;
    height: 100%;
    border-radius: 0;
  }

  transition: 0.6s;
  overflow: hidden;

  display: flex;
  flex-direction: column;
}

.player-settings {

  // 宽度 > $breakpoint-sm-min
  @media (min-width: $breakpoint-sm-min) {
    width: 330px;
    margin: 0px 10px 10px 0px;
    // border-radius: 8px;
  }

  // 宽度 < $breakpoint-xs-max (599px)
  @media (max-width: $breakpoint-xs-max) {
    width: 100%;
    height: 100%;
    border-radius: 0;
  }

  transition: 0.6s;
  overflow: hidden;

  display: flex;
  flex-direction: column;
}

.audio-player::before {
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  content: "";
  background-image: var(--cover-url);
  background-position: 50% 50%;
  background-size: contain;
  background-repeat: repeat;
  filter: blur(40px) opacity(0.2); // blur bigger than 80 will cause safari wrong render result
  transform: translate3d(0, 0, 0);
}

.hideStyle {
  transform: translateY(200%);
}

.showStyle {
  transform: translateY(0);
}

.albumart {

  // 宽度 < $breakpoint-xs-max (599px)
  @media (max-width: $breakpoint-xs-max) {
    width: 100%;
  }

  flex-grow: 1;
}

.current-play-list {
  max-height: 500px;

  // 宽度 > $breakpoint-xs-max
  @media (min-width: $breakpoint-xs-max) {
    width: 450px;
  }

  // 宽度 < $breakpoint-xs-max (599px)
  @media (max-width: $breakpoint-xs-max) {
    min-width: 280px;
  }
}

.pull-handler {
  height: 6px;
  width: 100px;
  background: rgba(150, 122, 116, 0.5);
  position: absolute;
  border-radius: 4px !important;
  overflow: hidden;
  left: 50%;
  top: 12px;
  transform: translateX(-50%);
  transition: 0.3s;
  cursor: pointer;
}

.pull-handler:hover {
  background: rgba(150, 122, 116, 0.8);
}

.pull-handler:before {
  content: "";
  position: absolute;
  left: -50px;
  right: -50px;
  top: -10px;
  bottom: -10px;
  cursor: pointer;
}
</style>
<style>
.ant-slider .ant-slider-rail {
  background-color: rgba(193, 201, 209, 0.66) !important;
}

.ant-slider .ant-slider-track {
  background-color: #57b5fa !important;
}

.ant-slider-handle {
  border: 1px solid rgba(166, 160, 165, 0.9) !important;
}


/* 这里覆盖 a-slider 的悬停样式 */
.ant-slider:hover .ant-slider-rail {
  background-color: rgba(193, 201, 209, 0.66) !important;
}

.ant-slider-handle:focus {
  box-shadow: none !important;
}
</style>