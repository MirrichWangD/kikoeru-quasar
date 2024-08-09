<template>
  <q-form @submit="onSubmit">
    <q-card class="q-ma-md">
      <q-toolbar>
        <q-toolbar-title>播放器设置 (#TODO未完善)</q-toolbar-title>
      </q-toolbar>

      <q-list>
        <q-item style="height: 70px;">
          <q-item-section>
            <q-item-label>后退按钮跳跃秒数</q-item-label>
            <q-item-label caption>播放时后退按钮跳跃秒数</q-item-label>
          </q-item-section>

          <q-item-section avatar>
            <div class="q-gutter-sm">
              <q-radio dense v-model="rewindSeekTime" val=5 label="5 秒" />
              <q-radio dense v-model="rewindSeekTime" val=10 label="10 秒" />
              <q-radio dense v-model="rewindSeekTime" val=30 label="30 秒" />
            </div>
          </q-item-section>
        </q-item>

        <q-item>
          <q-item-section>
            <q-item-label>前进按钮跳跃秒数</q-item-label>
            <q-item-label caption>播放时前进按钮跳跃秒数</q-item-label>
          </q-item-section>

          <q-item-section avatar>
            <div class="q-gutter-sm">
              <q-radio dense v-model="forwardSeekTime" val="5" label="5 秒" />
              <q-radio dense v-model="forwardSeekTime" val="10" label="10 秒" />
              <q-radio dense v-model="forwardSeekTime" val="30" label="30 秒" />
            </div>
          </q-item-section>
        </q-item>
      </q-list>
    </q-card>


    <div class="q-ma-lg row justify-end">
      <q-btn :loading="loading" label="保存" type="submit" color="primary" />
    </div>
  </q-form>
</template>

<script>
import NotifyMixin from '../mixins/Notification.js'

export default {
  name: 'Settings',

  mixins: [NotifyMixin],

  data() {
    return {
      config: {},
      loading: false,
      rewindSeekTime: '5',
      forwardSeekTime: '30'
    }
  },

  methods: {
    requestConfig() {
      this.$axios.get('/api/config/admin')
        .then((response) => {
          this.config = response.data.config;
          // Integer => String
          this.rewindSeekTime = this.config.rewindSeekTime.toString()
          this.forwardSeekTime = this.config.forwardSeekTime.toString()
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

    onSubmit() {
      // String => Integer
      this.config.rewindSeekTime = parseInt(this.rewindSeekTime)
      this.config.forwardSeekTime = parseInt(this.forwardSeekTime)

      this.loading = true
      this.$axios.put('/api/config/admin', {
        config: this.config
      })
        .then((response) => {
          this.loading = false
          this.showSuccNotif(response.data.message)
        })
        .catch((error) => {
          this.loading = false
          if (error.response) {
            // 请求已发出，但服务器响应的状态码不在 2xx 范围内
            this.showErrNotif(error.response.data.error || `${error.response.status} ${error.response.statusText}`)
          } else {
            this.showErrNotif(error.message || error)
          }
        })
    },
  },

  created() {
    this.requestConfig()
  }
}
</script>