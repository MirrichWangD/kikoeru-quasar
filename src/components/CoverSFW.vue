<template>
  <router-link :to="`/work/RJ${workid}`">
    <q-img
      :src="coverUrl"
      :ratio="4 / 3"
      :img-class="imgClass"
      style="max-width: 560px;"
      transition="fade"
      @mouseover="toggleBlurFlag()"
      @mouseout="toggleBlurFlag()"
    >
      <div class="absolute-top-left transparent" style="padding: 0;">
        <q-chip dense square color="brown" text-color="white" class="q-ma-sm">
          {{ `RJ${workid}` }}
        </q-chip>
      </div>

      <div :v-if="release" class="absolute-bottom-right" style="padding: 5px;">
        {{ release }}
      </div>
    </q-img>
  </router-link>
</template>

<script>
export default {
  name: 'CoverSFW',

  props: {
    workid: {
      type: String,
      required: true
    },

    nsfw: {
      type: Boolean,
      default: true
    },

    release: {
      required: true
    }
  },

  data() {
    return {
      blurFlag: true
    };
  },

  computed: {
    coverUrl() {
      return this.workid ? `/api/cover/${this.workid}` : '';
    },

    imgClass() {
      if (this.$q.platform.is.mobile) {
        // 在移动设备上图片直接显示
        return '';
      } else {
        if (!this.nsfw) {
          // 在PC上SFW的图片直接显示
          return '';
        } else {
          // 在PC上NSFW的图片鼠标悬停显示
          return this.blurFlag ? 'blur-image' : '';
        }
      }
    }
  },

  methods: {
    toggleBlurFlag() {
      this.blurFlag = !this.blurFlag;
    }
  }
};
</script>

<style lang="scss">
.blur-image {
  filter: blur(10px);
}
</style>
