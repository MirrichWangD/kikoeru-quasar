<template>
  <div class="q-pa-md q-gutter-md flex-center" style="max-width: 700px; margin: auto;">
    <q-toolbar>
      <q-toolbar-title>通用设置</q-toolbar-title>
    </q-toolbar>
    <q-card class="q-py-none">
      <q-list separator>
        <!-- 倒带跳跃秒数设置 -->
        <q-item>
          <q-item-section avatar>
            <q-icon size="md" name="library_music" />
          </q-item-section>
          <q-item-section main>
            <q-item-label>作品显示数量</q-item-label>
            <q-item-label class="text-caption text-grey">输入整数回车后生效</q-item-label>
          </q-item-section>
          <q-item-section side>
            <q-input input-class="text-right" type="number" v-model="pageSize" @keyup.enter="setPageSize" />
          </q-item-section>
        </q-item>
      </q-list>
    </q-card>

    <q-toolbar>
      <q-toolbar-title>播放设置</q-toolbar-title>
    </q-toolbar>
    <q-card class="q-py-none">
      <q-list separator>
        <!-- 倒带跳跃秒数设置 -->
        <q-item>
          <q-item-section avatar>
            <q-icon size="md" name="fast_rewind" />
          </q-item-section>
          <q-item-section main>
            <q-item-label>倒带跳跃秒数</q-item-label>
          </q-item-section>
          <q-item-section side>
            <q-select borderless v-model="rewindSeekTimeOption" :options="seekTimeOptions" @input="setRewindSeekTime" />
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
            <q-select
              borderless
              v-model="forwardSeekTimeOption"
              :options="seekTimeOptions"
              @input="setForwardSeekTime"
            />
          </q-item-section>
        </q-item>
        <q-item>
          <q-item-section avatar>
            <q-icon flat dense size="md" name="image_search" />
          </q-item-section>
          <q-item-section main>
            <q-item-label>是否启用媒体查看器</q-item-label>
            <q-item-label class="text-caption text-grey">开启可能占用性能，仅查看图片/视频</q-item-label>
          </q-item-section>
          <q-item-section side>
            <q-toggle borderless v-model="isViewer" @input="setIsViewer" />
          </q-item-section>
        </q-item>
      </q-list>
    </q-card>

    <span class="flex flex-center text-grey">&nbsp;</span>
  </div>
</template>

<script>
import NotifyMixin from '../mixins/Notification.js';
import { mapMutations } from 'vuex';

export default {
  name: 'Settings',

  mixins: [NotifyMixin],

  data() {
    return {
      config: {},
      // 页面数量设置
      pageSize: 12,
      // 倒带、前进秒数设置
      rewindSeekTimeOption: { label: '5 秒', value: 5 },
      forwardSeekTimeOption: { label: '30 秒', value: 30 },
      seekTimeOptions: [
        { label: '5 秒', value: 5 },
        { label: '10 秒', value: 10 },
        { label: '30 秒', value: 30 }
      ],
      // 媒体查看器是否启用
      isViewer: true
    };
  },

  mounted() {
    if (this.$q.localStorage.has('pageSize')) {
      this.pageSize = this.$q.localStorage.getItem('pageSize');
    }
    if (this.$q.localStorage.has('rewindSeekTime')) {
      const rewindSeekTime = this.$q.localStorage.getItem('rewindSeekTime');
      this.rewindSeekTimeOption = this.seekTimeOptions.find(item => item.value === rewindSeekTime);
      this.SET_REWIND_SEEK_TIME(rewindSeekTime);
    }
    if (this.$q.localStorage.has('forwardSeekTime')) {
      const forwardSeekTime = this.$q.localStorage.getItem('forwardSeekTime');
      this.forwardSeekTimeOption = this.seekTimeOptions.find(item => item.value === forwardSeekTime);
      this.SET_FORWARD_SEEK_TIME(forwardSeekTime);
    }
    if (this.$q.localStorage.has('isViewer')) {
      this.isViewer = this.$q.localStorage.getItem('isViewer');
    }
  },

  methods: {
    ...mapMutations('AudioPlayer', ['SET_REWIND_SEEK_TIME', 'SET_FORWARD_SEEK_TIME']),

    setPageSize() {
      console.log('set pageSize:', this.pageSize);
      this.$q.localStorage.set('pageSize', parseInt(this.pageSize));
      window.location.reload();
    },

    setRewindSeekTime(newRewindSeekTimeOption) {
      console.log('set rewindSeekTime:', newRewindSeekTimeOption.value);
      this.rewindSeekTimeOption = newRewindSeekTimeOption;
      this.SET_REWIND_SEEK_TIME(newRewindSeekTimeOption.value);
      this.$q.localStorage.set('rewindSeekTime', newRewindSeekTimeOption.value);
    },

    setForwardSeekTime(newForwardSeekTimeOption) {
      console.log('set forwardSeekTime:', newForwardSeekTimeOption.value);
      this.forwardSeekTimeOption = newForwardSeekTimeOption;
      this.SET_FORWARD_SEEK_TIME(newForwardSeekTimeOption.value);
      this.$q.localStorage.set('forwardSeekTime', newForwardSeekTimeOption.value);
    },

    setIsViewer(newIsViewer) {
      console.log('set Media Viewer:', newIsViewer);
      this.isViewer = newIsViewer;
      this.$q.localStorage.set('isViewer', this.isViewer);
    }
  }
};
</script>
<style lang="scss">
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
</style>
