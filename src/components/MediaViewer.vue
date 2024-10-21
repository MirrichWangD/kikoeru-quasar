<template>
  <q-card class="q-pa-none q-ma-none">
    <!-- title -->
    <q-card-section class="row justify-between items-center">
      <div>
        <div class="text-h6 ellipsis">{{ currentFile.title }}</div>
        <div class="text-subtitle2">
          {{ currentIndex + 1 }}/{{ files.length }}
        </div>
      </div>
      <div>
        <q-btn
          icon="close"
          @click="$emit('close')"
          :style="{ backgroundColor: $q.dark.isActive ? '#121212' : '' }"
        />
      </div>
    </q-card-section>
    <!-- content -->
    <q-card-section
      class="row justify-center items-center"
      :style="{ backgroundColor: $q.dark.isActive ? '#121212' : '' }"
    >
      <!-- 中心大图/视频 -->
      <div ref="fullscreenContainer">
        <q-img
          v-if="currentFile.type === 'image'"
          :src="getMediaUrl(currentFile)"
          style="width: calc(100vw - 200pt); max-height: calc(100vh - 200pt);"
          contain
        />
        <video
          v-else
          :src="getMediaUrl(currentFile)"
          style="max-width: calc(100vw - 200pt); max-height: calc(100vh - 200pt);"
          controls
          autoplay
        />
      </div>
    </q-card-section>
    <!-- controls -->
    <q-card-actions class="justify-center">
      <q-btn
        icon="arrow_left"
        @click="clickPrevious"
        :style="{ backgroundColor: $q.dark.isActive ? '#121212' : '' }"
      >
        <q-tooltip>上一个 (快捷键: Ctrl + ←)</q-tooltip>
      </q-btn>
      <q-btn
        icon="download"
        @click="clickDownload"
        :style="{ backgroundColor: $q.dark.isActive ? '#121212' : '' }"
      >
        <q-tooltip>下载</q-tooltip>
      </q-btn>
      <q-btn
        icon="fullscreen"
        @click="clickFullscreen"
        :style="{ backgroundColor: $q.dark.isActive ? '#121212' : '' }"
      >
        <q-tooltip>全屏显示</q-tooltip>
      </q-btn>
      <q-btn
        icon="arrow_right"
        @click="clickNext"
        :style="{ backgroundColor: $q.dark.isActive ? '#121212' : '' }"
      >
        <q-tooltip>下一个 (快捷键: Ctrl + →)</q-tooltip></q-btn
      >
    </q-card-actions>
  </q-card>
</template>

<script>
import { AppFullscreen } from "quasar";
import { mapState } from "vuex";

export default {
  name: "MediaViewer",

  props: {
    files: {
      type: Array,
      required: true
    },
    index: {
      type: Number,
      required: true
    }
  },
  data() {
    return {
      currentIndex: this.index,
      showThumbnail: false
    };
  },
  computed: {
    ...mapState("AudioPlayer", ["playing"]),
    currentFile() {
      console.log(`currentFile called`);
      const file = this.files[this.currentIndex];
      const isVideoSwitchPause = this.$q.localStorage.has("isVideoSwitchPause")
        ? this.$q.localStorage.getItem("isVideoSwitchPause")
        : true;
      if (file.type === "audio" && this.playing && isVideoSwitchPause) {
        this.$store.commit("AudioPlayer/TOGGLE_PLAYING");
      }
      return file;
    }
  },
  mounted() {
    console.log("mounted", this.$store.currentPlayingFile);
    document.addEventListener("keydown", this.handleChangeKeydown);
  },
  methods: {
    getMediaUrl(file, isDownload = false) {
      const token = this.$q.localStorage.getItem("jwt-token") || "";
      // Fallback to old API for an old backend
      let url;
      if (isDownload === true) {
        url = file.mediaDownloadUrl
          ? `${file.mediaDownloadUrl}?token=${token}`
          : `/api/media/download/${file.hash}?token=${token}`;
      } else {
        url = file.mediaStreamUrl
          ? `${file.mediaStreamUrl}?token=${token}`
          : `/api/media/stream/${file.hash}?token=${token}`;
      }
      return url;
    },

    clickPrevious() {
      if (this.currentIndex > 0) {
        this.currentIndex--;
      }
    },
    clickNext() {
      if (this.currentIndex < this.files.length - 1) {
        this.currentIndex++;
      }
    },
    clickDownload() {
      const file = this.files[this.currentIndex];
      const link = document.createElement("a");
      link.href = this.getMediaUrl(file, true);
      link.setAttribute("download", file.title); // 可选: 提供下载时的默认文件名
      link.click();
    },

    clickFullscreen() {
      const container = this.$refs.fullscreenContainer;
      try {
        if (AppFullscreen.isActive) {
          AppFullscreen.exit();
        } else {
          AppFullscreen.request(container);
        }
      } catch (error) {
        console.error("Failed to enter fullscreen mode:", error);
      }
    },
    handleChangeKeydown(event) {
      if (event.ctrlKey && event.key === "ArrowLeft") {
        this.clickPrevious();
      } else if (event.ctrlKey && event.key === "ArrowRight") {
        this.clickNext();
      }
    }
  }
};
</script>
