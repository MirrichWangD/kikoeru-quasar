<template>
  <div>
    <div class="text-h5 text-weight-regular q-ma-md row">
      {{ pageTitle }}
      <div>
        <!-- 搜索信息展示 -->
        <q-chip class="q-ma-xs" v-for="(kw, index) in keywords" :key="kw">
          {{ kw }}
          <q-btn
            class="q-ml-sm search-tag-close-btn"
            padding="xs"
            round
            flat
            size="xs"
            icon="close"
            @click="onRemoveSearchKeyword(index)"
          />
        </q-chip>
      </div>
      <!-- 作品数量 -->
      <span v-show="totalCount">&nbsp;({{ totalCount }})&nbsp;</span>
    </div>

    <!-- 主界面 -->
    <div :class="`row justify-center ${showMode === 'list' ? 'list' : 'q-mx-md'}`">
      <q-infinite-scroll :offset="250" :disable="true" class="col">
        <div class="row justify-between q-mb-md q-mr-sm">
          <!-- 排序选择框 -->
          <q-select
            dense
            rounded
            outlined
            transition-show="scale"
            transition-hide="scale"
            v-model="sortOption"
            :options="sortOptions"
            label="排序"
            class="col-auto"
            :bg-color="$q.dark.isActive ? 'black' : 'white'"
          />

          <!-- 是否包含字幕 -->
          <q-toggle v-model="lyricStatus" label="是否包含字幕" @input="onLyricStatusChange"></q-toggle>

          <!-- 切换显示模式按钮 -->
          <q-btn-toggle
            dense
            spread
            rounded
            v-model="showMode"
            toggle-color="primary"
            text-color="primary"
            :options="showOptions"
            style="width: 85px;"
            class="col-auto"
            :class="$q.dark.isActive ? 'bg-black' : 'bg-white'"
          />
        </div>

        <div v-if="!stopLoad" class="row justify-center q-mb-md q-mr-sm">
          <!-- 加载动画 -->
          <q-spinner-dots color="primary" size="40px" />
        </div>

        <!-- 列表模式 -->
        <q-list v-if="showMode == 'list'" bordered separator class="shadow-2">
          <WorkListItem v-for="work in works" :key="work.id" :metadata="work" :showLabel="$q.screen.width > 700" />
        </q-list>

        <!-- 卡片模式 -->
        <div v-else class="row q-col-gutter-x-md q-col-gutter-y-lg">
          <div class="col-xs-12 col-sm-4 col-md-3" :class="'col-lg-2 col-xl-2'" v-for="work in works" :key="work.id">
            <WorkCard :metadata="work" :thumbnailMode="showMode == 'thumbnail'" class="fit" />
          </div>
        </div>
      </q-infinite-scroll>
    </div>

    <!-- 分页按钮 -->
    <div class="row justify-center q-py-lg">
      <a-pagination
        showQuickJumper
        :current="page"
        :total="totalCount"
        :pageSize="pageSize"
        @change="onPageChange"
        :class="{ dark: $q.dark.isActive }"
      />
    </div>
  </div>
</template>

<script>
import WorkCard from 'components/WorkCard';
import WorkListItem from 'components/WorkListItem';
import NotifyMixin from '../mixins/Notification.js';

export default {
  name: 'Works',

  mixins: [NotifyMixin],

  components: {
    WorkCard,
    WorkListItem
  },

  data() {
    return {
      showMode: 'card',
      stopLoad: false,
      works: [],
      pageTitle: '',
      // 分页参数
      page: 1,
      totalCount: 0,
      pageSize: 12,
      seed: 7, // random sort

      // 排序标签按钮
      sortOption: { label: '最新收录', order: 'add_time', sort: 'desc' },
      sortOptions: [
        { label: '最新收录', order: 'add_time', sort: 'desc' },
        { label: '发售日期倒序', order: 'release', sort: 'desc' },
        { label: '我的评价排序', order: 'rating', sort: 'desc' },
        { label: '发售日期顺序', order: 'release', sort: 'asc' },
        { label: '售出数量倒序', order: 'dl_count', sort: 'desc' },
        { label: '价格顺序', order: 'price', sort: 'asc' },
        { label: '价格倒序', order: 'price', sort: 'desc' },
        { label: '评价倒序', order: 'rate_average_2dp', sort: 'desc' },
        { label: '评论数量倒序', order: 'review_count', sort: 'desc' },
        { label: 'RJ号倒序', order: 'id', sort: 'desc' },
        { label: 'RJ号顺序', order: 'id', sort: 'asc' },
        { label: '全年龄顺序', order: 'nsfw', sort: 'asc' },
        { label: '随机排序', order: 'random', sort: 'desc' }
      ],
      // 是否包含字幕
      lyricStatus: false,

      // 显示模式按钮
      showOptions: [
        { icon: 'view_module', value: 'thumbnail' },
        { icon: 'view_column', value: 'card' },
        { icon: 'list', value: 'list' }
      ],

      // 搜索
      keywords: [],

      lastQuery: { page: 1, keyword: '' }
    };
  },

  created() {
    this.init();
    this.reset();
  },

  computed: {
    url() {
      const query = this.$route.query;
      if (query.keyword) {
        return `/api/search/`;
      } else {
        return '/api/works';
      }
    }
  },

  watch: {
    sortOption(newSortOption) {
      const localSortOption = this.$q.localStorage.getItem('sortOption') || { label: '最新收录' };
      if (newSortOption.label !== localSortOption.label) {
        console.log('switch sort to:', `${newSortOption.order} ${newSortOption.sort}`)
        this.$q.localStorage.set('sortOption', newSortOption);
        this.reset();
      }
    },

    showMode(newShowModeSetting) {
      this.$q.localStorage.set('showMode', newShowModeSetting);
    },

    keywords(newKeywords) {
      this.$q.sessionStorage.set('keyword', newKeywords);
    },

    '$route.query': {
      handler: function(query) {
        const page = parseInt(query.page) || 1;
        const keyword = query.keyword || '';
        if (this.$route.path === '/works' && (keyword !== this.lastQuery.keyword || page !== this.lastQuery.page)) {
          console.log('get page from query', query);
          if (keyword !== this.lastQuery.keyword) {
            this.lastQuery.keyword = keyword;
            this.keywords = keyword ? keyword.split(';') : [];
          }
          if (page !== this.lastQuery.page) {
            this.lastQuery.page = page;
          }
          this.reset();
        }
      }
    }
  },

  methods: {
    init() {
      if (this.$route.query.keyword) {
        this.keywords = this.$route.query.keyword.split(';');
        this.lastQuery.keyword = this.$route.query.keyword;
      } else {
        this.$q.sessionStorage.remove('keyword');
      }
      if (this.$route.query.page) {
        this.lastQuery.page = parseInt(this.$route.query.page);
      }
      if (this.$q.localStorage.has('showMode')) {
        this.showMode = this.$q.localStorage.getItem('showMode');
      }
      if (this.$q.localStorage.has('sortOption')) {
        this.sortOption = this.$q.localStorage.getItem('sortOption');
      }
      if (this.$q.localStorage.has('pageSize')) {
        this.pageSize = this.$q.localStorage.getItem('pageSize');
      }
    },

    requestWorksQueue() {
      console.log(`requestWorksQueue: ${this.seed}`);
      this.works = [];
      this.totalCount = 0;
      const params = {
        order: this.sortOption.order,
        sort: this.sortOption.sort,
        page: this.$route.query.page || 1,
        pageSize: this.pageSize,
        lyricStatus: this.lyricStatus,
        seed: this.seed
      };

      if (this.$route.query.keyword) {
        params.keywords = this.$route.query.keyword.split(';');
      }

      return this.$axios
        .get(this.url, { params })
        .then(response => {
          const works = response.data.works;
          this.works = works.concat();
          this.page = response.data.page;
          this.totalCount = response.data.totalCount;
          this.pageSize = response.data.pageSize;
        })
        .catch(error => {
          if (error.response) {
            // 请求已发出，但服务器响应的状态码不在 2xx 范围内
            if (error.response.status !== 401) {
              this.showErrNotif(error.response.data.error || `${error.response.status} ${error.response.statusText}`);
            }
          } else {
            this.showErrNotif(error.message || error);
          }
          this.stopLoad = true;
        });
    },

    refreshPageTitle() {
      this.totalCount = 0;
      if (this.$route.query.keyword) {
        this.pageTitle = `Search By`;
      } else {
        this.pageTitle = 'All works';
      }
    },

    reset() {
      this.seed = Math.floor(Math.random() * 100);
      this.refreshPageTitle();
      this.requestWorksQueue().then(() => {
        this.stopLoad = true;
      });
    },

    onLyricStatusChange(newLyricStatus) {
      console.log('switch lyric status:', newLyricStatus);
      this.lyricStatus = newLyricStatus;
      this.page = 1;
      const query = { ...this.$route.query };
      if (this.$route.query.page) {
        delete query.page;
        this.$router.push({ query: query });
        return;
      }
      this.reset();
    },

    onPageChange(page) {
      console.log(`change page to: ${page}`);
      this.$router.push({ query: { ...this.$route.query, page: page } });
      this.page = page;
    },

    // 搜索功能
    onRemoveSearchKeyword(index) {
      const keyword = this.$route.query.keyword.split(';');
      keyword.splice(index, 1);
      const query = keyword.length ? { keyword: keyword.join(';') } : {};
      this.$router.push({ name: 'works', query: query });
    }
  }
};
</script>

<style lang="scss" scoped>
.dark.ant-pagination {
  filter: invert(100%) hue-rotate(180deg);
}

.list {
  // 宽度 >= $breakpoint-sm-min
  @media (min-width: $breakpoint-sm-min) {
    padding: 0px 20px;
  }
}

.work-card {
  // 宽度 > $breakpoint-xl-min
  @media (min-width: $breakpoint-md-min) {
    width: 560px;
  }
}
</style>
