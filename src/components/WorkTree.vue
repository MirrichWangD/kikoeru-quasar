<template>
  <div class="q-ma-md">
    <q-breadcrumbs gutter="xs" v-if="path.length">
      <q-breadcrumbs-el>
        <q-btn no-caps flat dense size="md" icon="folder" style="height: 30px;" @click="path = []">ROOT</q-btn>
      </q-breadcrumbs-el>

      <q-breadcrumbs-el v-for="(folderName, index) in path" :key="index" class="cursor-pointer">
        <q-btn no-caps flat dense size="md" icon="folder" style="height: 30px;" @click="onClickBreadcrumb(index)">{{
          folderName }}</q-btn>
      </q-breadcrumbs-el>
    </q-breadcrumbs>

    <q-card>
      <q-list separator>
        <q-item clickable v-ripple v-for="(item, index) in fatherFolder" :key="index" :active="item.type === 'audio' && currentPlayingFile.hash === item.hash
          " active-class="text-white bg-teal" @click="onClickItem(item)" class="non-selectable">
          <q-item-section avatar>
            <q-icon size="34px" v-if="item.type === 'folder'" color="amber" name="folder" />
            <q-icon size="34px" v-else-if="item.type === 'text'" color="info" name="description" />
            <q-icon size="34px" v-else-if="item.type === 'image'" color="orange" name="photo" />
            <q-icon size="34px" v-else-if="item.type === 'other'" color="info" name="description" />
            <q-btn v-else round dense color="primary" :icon="playIcon(item.hash)"
              @click="onClickPlayButton(item.hash)" />
          </q-item-section>

          <q-item-section>
            <q-item-label lines="2">{{ item.title }}</q-item-label>
            <q-item-label v-if="item.children" caption lines="1">{{
              `${item.children.length} 项目`
            }}</q-item-label>

            <!-- 音频文件时长 -->
            <q-item-label v-if="item.type === 'audio' && typeof item.duration === 'number'" caption lines="1">
              <!-- <q-icon size="0.8rem" name="schedule" class="q-mr-xs" /> -->
              {{ formatSeconds(item.duration) }}
            </q-item-label>
          </q-item-section>

          <!-- 上下文菜单 -->
          <q-menu v-if="
            item.type === 'audio' ||
            item.type === 'text' ||
            item.type === 'image' ||
            item.type === 'other'
          " touch-position context-menu auto-close transition-show="jump-down" transition-hide="jump-up">
            <q-list separator>
              <q-item clickable @click="addToQueue(item)" v-if="item.type === 'audio'">
                <q-item-section>添加到播放列表</q-item-section>
              </q-item>

              <q-item clickable @click="playNext(item)" v-if="item.type === 'audio'">
                <q-item-section>下一曲播放</q-item-section>
              </q-item>

              <q-item clickable @click="download(item)">
                <q-item-section>下载文件</q-item-section>
              </q-item>
            </q-list>
          </q-menu>
        </q-item>
      </q-list>
    </q-card>

    <!-- 预览弹窗 -->
    <q-dialog v-model="showViewer" full-width>
      <!-- 视频播放 -->
      <q-card color="primary" class="q-pa-md" v-if="showVideo">
        <div class="pull-handler" @click="hideViewer" />
        <q-card-section>
          <!-- <div class="row items-center view-container" v-if="showVideo"> -->
          <video ref="video" class="video" controls autoplay />
          <!-- </div> -->
        </q-card-section>

      </q-card>
      <!-- 图片查看 -->
      <q-card color="primary" v-if="!showVideo">
        <q-card-section>
          <div class="row items-center no-wrap">
            <div class="col">
              <div class="text-h6 ellipsis">{{ preview_img_name }}</div>
              <div class="text-subtitle2">{{ preview_img_idx + 1 }}/{{ preview_img_list.length }}</div>
            </div>
            <div class="col-auto">
              <q-btn outline @click="openFile(preview_img_list[preview_img_idx])">全屏显示</q-btn>
            </div>
          </div>
        </q-card-section>
        <q-card-section class="q-pt-none">
          <q-img class="image" :src="preview_img_url" contain />
        </q-card-section>


        <q-card-actions align="around">
          <q-btn flat label="上一个" color="primary" @click="changePreviewImg(false)" />
          <q-btn flat label="关闭" color="negative" v-close-popup />
          <q-btn flat label="下一个" color="primary" @click="changePreviewImg(true)" />
        </q-card-actions>

      </q-card>
    </q-dialog>
    <span class="flex flex-center text-grey">&nbsp;</span>
  </div>
</template>

<script>
// import axios from "axios";
import { mapState, mapGetters } from "vuex";

export default {
  name: "WorkTree",

  data() {
    return {
      showViewer: false,
      showVideo: false,
      preview_img: false,
      preview_img_idx: 0,
      preview_img_list: [],
      preview_img_hash: "",
      path: []
    };
  },

  props: {
    tree: {
      type: Array,
      required: true
    }
  },

  watch: {
    tree() {
      this.initPath();
    }
  },

  computed: {
    fatherFolder() {
      let fatherFolder = this.tree.concat();
      this.path.forEach(folderName => {
        fatherFolder = fatherFolder.find(
          item => item.type === "folder" && item.title === folderName
        ).children;
      });
      return fatherFolder;
    },

    queue() {
      const queue = [];
      this.fatherFolder.forEach(item => {
        if (item.type === "audio") {
          queue.push(item);
        }
      });

      return queue;
    },

    preview_img_url() {
      const item = this.preview_img_list[this.preview_img_idx];
      return item ? this.originalMediaSrc(item) : "";
    },

    preview_img_name() {
      const item = this.preview_img_list[this.preview_img_idx];
      return item ? item.title : "";
    },

    ...mapState("AudioPlayer", ["playing"]),

    ...mapGetters("AudioPlayer", ["currentPlayingFile"])
  },

  methods: {
    // 播放图标
    playIcon(hash) {
      return this.playing && this.currentPlayingFile.hash === hash
        ? "pause"
        : "play_arrow";
    },

    initPath() {
      const initialPath = [];
      let fatherFolder = this.tree.concat();
      while (fatherFolder.length === 1) {
        if (fatherFolder[0].type === "audio") {
          break;
        }
        initialPath.push(fatherFolder[0].title);
        fatherFolder = fatherFolder[0].children;
      }
      this.path = initialPath;
    },

    onClickBreadcrumb(index) {
      this.path = this.path.slice(0, index + 1);
    },

    onClickItem(item) {
      const ext = this.getFileExt(item.title);
      if (item.type === "folder") {
        this.path.push(item.title);
      } else if (item.type === "text") {
        this.openFile(item);
      } else if (item.type === "image" || ext === "mp4") {
        this.openViewer(item);
      } else if (item.type === "other") {
        this.download(item);
      } else if (this.currentPlayingFile.hash !== item.hash) {
        this.$store.commit("AudioPlayer/SET_QUEUE", {
          queue: this.queue.concat(),
          index: this.queue.findIndex(file => file.hash === item.hash),
          resetPlaying: true
        });
      }
    },

    onClickPlayButton(hash) {
      const nextFileIndex = this.queue.findIndex(file => file.hash === hash);
      const nextFile = this.queue[nextFileIndex];
      const ext = this.getFileExt(nextFile.title);
      if (ext !== "mp4") {
        if (this.currentPlayingFile.hash === hash) {
          this.$store.commit("AudioPlayer/TOGGLE_PLAYING");
        } else {
          this.$store.commit("AudioPlayer/SET_QUEUE", {
            queue: this.queue.concat(),
            index: this.queue.findIndex(file => file.hash === hash),
            resetPlaying: true
          });
        }
      } else {
        this.$store.commit("AudioPlayer/EMPTY_QUEUE");
        this.openViewer(nextFile);
      }
    },

    addToQueue(file) {
      this.$store.commit("AudioPlayer/ADD_TO_QUEUE", file);
    },

    playNext(file) {
      this.$store.commit("AudioPlayer/PLAY_NEXT", file);
    },

    download(file) {
      const token = this.$q.localStorage.getItem("jwt-token") || "";
      // Fallback to old API for an old backend
      const url = file.mediaDownloadUrl
        ? `${file.mediaDownloadUrl}?token=${token}`
        : `/api/media/download/${file.hash}?token=${token}`;
      const link = document.createElement("a");
      link.href = url;
      link.target = "_blank";
      link.click();
    },

    openFile(file) {
      const url = this.originalMediaSrc(file);
      const link = document.createElement("a");
      link.href = url;
      link.target = "_blank";
      link.click();
    },

    openViewer(file) {
      const url = this.originalMediaSrc(file);
      this.showViewer = true;
      this.showVideo = (file.type === "image") ? false : true;
      if (file.type === "image") {
        this.openPreviewImg(file);
      } else {
        this.$nextTick(() => {
          this.$refs.video.src = url
        })
      }
    },

    originalMediaSrc(file) {
      const token = this.$q.localStorage.getItem('jwt-token') || '';
      // Fallback to old API for an old backend 
      const url = file.mediaStreamUrl ? `${file.mediaStreamUrl}?token=${token}` : `/api/media/stream/${file.hash}?token=${token}`;
      return url
    },

    openPreviewImg(item) {
      const preview_img_list = this.fatherFolder.filter(item => item.type === 'image')
      let preview_img_idx = -1;
      preview_img_list.forEach((i, idx) => {
        if (i.hash === item.hash) {
          preview_img_idx = idx;
        }
      });
      this.preview_img = true;
      this.preview_img_list = preview_img_list;
      this.preview_img_idx = preview_img_idx;
    },

    changePreviewImg(next) {
      if (this.preview_img_list.length <= 1) return;
      const length = this.preview_img_list.length;
      this.preview_img_idx = (length + this.preview_img_idx + (next ? 1 : -1)) % length;
    },


    hideViewer() {
      this.showViewer = false;
    },

    getFileExt(val) {
      const arr = val.split(".");
      return arr[arr.length - 1];
    },

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
    }
  }
};
</script>
<style>
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

.view-container {
  width: 100%;
  height: 100%;
  justify-content: center;
  align-items: center;
}

.video {
  width: 100%;
  max-height: calc(100vh - 100pt);
  object-fit: contain;
  background-color: black;
}

.image {
  width: 100%;
  height: calc(100vh - 200pt);
}
</style>
