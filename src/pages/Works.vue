<template>
  <div>
    <div class="text-h5 text-weight-regular q-ma-md row">
      {{ pageTitle }}
      <div v-if="isAdvanceSearch"><!--高级搜索模式的多关键字展示-->
        <q-chip class="q-ma-xs" v-for="meta, index in advanceSearchKeywords" :key="meta.t + meta.d">
          {{ meta.d }}
          <q-btn class="q-ml-sm search-tag-close-btn" padding="xs" round flat size="xs" icon="close"
            @click="removeAdvanceSearchKeyword(index)" />
        </q-chip>
      </div>
      <div v-else> <!--普通搜索模式的信息展示-->

        <!-- <span v-if="searchMetas.length !== 0">By</span> -->
        <q-chip class="q-ma-xs" v-for="meta in searchMetas" :key="meta">
          {{ meta }}
          <q-btn class="q-ml-sm search-tag-close-btn" padding="xs" round flat size="xs" icon="close" to="/works" />
        </q-chip>
      </div>
      
      <span v-show="pagination.totalCount">
        &nbsp;({{ pagination.totalCount }})&nbsp;
      </span>
    </div>

    <!-- 搜索框 -->
    <div v-if="isAdvanceSearch" class="q-pa-md q-full-width">
      <q-input outlined autofocus label="关键字搜索" :hint="advanceSearchBarHint" v-model="editKeyword"
        @keyup.enter="onAddAdvanceSearchKeyword">
        <template v-slot:append>
          <q-btn round dense flat icon="add" @click="onAddAdvanceSearchKeyword" />
        </template>
      </q-input>
    </div>

    <div :class="`row justify-center ${showMode === 'list' ? 'list' : 'q-mx-md'}`">
      <q-infinite-scroll :offset="250" :disable="!stopLoad" class="col">
        <div class="row justify-between q-mb-md q-mx-sm">
          <!-- 排序选择框 -->
          <q-select dense rounded outlined bg-color="white" transition-show="scale" transition-hide="scale"
            v-model="sortOption" :options="sortOptions" label="排序" class="col-auto" />

          <!-- 切换显示模式按钮 -->
          <q-btn-toggle dense spread rounded v-model="showMode" toggle-color="primary" color="white"
            text-color="primary" :options="[
              { icon: 'view_module', value: 'miniCard' },
              { icon: 'view_column', value: 'card' },
              { icon: 'list', value: 'list' }
            ]" style="width: 85px;" class="col-auto" />

        </div>

        <!-- 列表模式 -->
        <q-list v-if="showMode == 'list'" bordered separator class="shadow-2">
          <WorkListItem v-for="work in works" :key="work.id" :metadata="work" :showLabel="$q.screen.width > 700" />
        </q-list>

        <!-- 卡片模式 -->
        <div v-else class="row q-col-gutter-x-md q-col-gutter-y-lg">
          <div class="col-xs-12 col-sm-4 col-md-3" :class="'col-lg-2 col-xl-2'" v-for="work in works" :key="work.id">
            <WorkCard :metadata="work" :thumbnailMode="showMode == 'miniCard'" class="fit" />
          </div>
        </div>

        <!-- 加载三点动画 -->
        <template v-slot:loading>
          <div class="row justify-center q-my-md">
            <q-spinner-dots color="primary" size="40px" />
          </div>
        </template>
      </q-infinite-scroll>
    </div>

    <!-- 分页按钮 -->
    <div class="row justify-center q-py-lg">
      <Pagination showQuickJumper v-model="pagination.currentPage" :defaultPageSize="pagination.pageSize"
        :total="pagination.totalCount" @change="onPageChange" />
    </div>
  </div>
</template>

<script>
import WorkCard from "components/WorkCard";
import WorkListItem from "components/WorkListItem";
import NotifyMixin from "../mixins/Notification.js";
import { Pagination } from "ant-design-vue";

const AdvanceSearchCondType = {
  UNKNOWN: 0,
  FUZZY: 1, // 全文模糊搜索，包括标题，
  VA: 2,
  TAG: 3,
  CIRCLE: 4,
};

export default {
  name: "Works",

  mixins: [NotifyMixin],

  components: {
    WorkCard,
    WorkListItem,
    Pagination
  },

  data() {
    return {
      showMode: "card",
      stopLoad: false,
      works: [],
      pageTitle: "",
      // 分页参数
      pagination: { currentPage: 0, pageSize: 12, totalCount: 0 },
      seed: 7, // random sort

      // 排序标签按钮
      sortOption: { label: "最新收录", order: "add_time", sort: "desc" },
      sortOptions: [
        { label: "最新收录", order: "add_time", sort: "desc" },
        { label: "发售日期倒序", order: "release", sort: "desc" },
        { label: "我的评价排序", order: "rating", sort: "desc" },
        { label: "发售日期顺序", order: "release", sort: "asc" },
        { label: "售出数量倒序", order: "dl_count", sort: "desc" },
        { label: "价格顺序", order: "price", sort: "asc" },
        { label: "价格倒序", order: "price", sort: "desc" },
        { label: "评价倒序", order: "rate_average_2dp", sort: "desc" },
        { label: "评论数量倒序", order: "review_count", sort: "desc" },
        { label: "RJ号倒序", order: "id", sort: "desc" },
        { label: "RJ号顺序", order: "id", sort: "asc" },
        { label: "全年龄顺序", order: "nsfw", sort: "asc" },
        { label: "随机排序", order: "random", sort: "desc" }
      ],

      // 聚合高级搜索
      searchMetas: [],
      editKeyword: "",
      advanceSearchKeywords: [],
      isAdvanceSearch: false
    };
  },

  created() {
    this.refreshPageTitle();
    this.seed = Math.floor(Math.random() * 100);
  },

  mounted() {
    if (localStorage.sortOption) {
      try {
        this.sortOption = JSON.parse(localStorage.sortOption);
      } catch {
        localStorage.removeItem("sortOption");
      }
    }
    if (localStorage.advanceSearchKeywords) {
      this.advanceSearchKeywords = JSON.parse(localStorage.advanceSearchKeywords || "[]");
    }
    this.checkAdvanceSearchMode();
  },

  computed: {
    url() {
      const query = this.$route.query;
      if (query.circleId) {
        return `/api/circles/${this.$route.query.circleId}/works`;
      } else if (query.tagId) {
        return `/api/tags/${this.$route.query.tagId}/works`;
      } else if (query.vaId) {
        return `/api/vas/${this.$route.query.vaId}/works`;
      } else if (query.keyword || this.isAdvanceSearch) {
        return `/api/search/`;
      } else {
        return "/api/works";
      }
    },
    advanceSearchBarHint() {
      if (this.editKeyword === "") return "模糊关键字，可搜索作品名、声优名、标签名、社团名"
      else return "按回车或者右侧加号添加"
    },
  },

  // keep-alive hooks
  // <keep-alive /> is set in MainLayout
  activated() {
    this.stopLoad = false;
  },

  deactivated() {
    this.stopLoad = true;
  },

  watch: {
    url() {
      this.reset();
    },

    sortOption(newSortOptionSetting) {
      localStorage.sortOption = JSON.stringify(newSortOptionSetting);
      this.seed = Math.floor(Math.random() * 100);
      this.reset();
    },

    showMode(newShowModeSetting) {
      localStorage.showMode = newShowModeSetting;
    },

    advanceSearchKeywords(newValue) {
      localStorage.advanceSearchKeywords = JSON.stringify(newValue, null, 0);
      this.reset()
    },

    "$route.name": {
      handler: function () {
        this.checkAdvanceSearchMode()
      },
      deep: true,
      immediate: true
    },

    "$route.query.keyword"() {
      this.reset();
    }
  },

  methods: {
    onLoad(index, done) {
      this.requestWorksQueue().then(() => done());
    },

    onPageChange(page) {
      this.stopLoad = true;
      this.pagination.currentPage = page;
      this.works = [];
      this.requestWorksQueue().then(() => {
        this.stopLoad = false;
      });
    },

    requestWorksQueue() {
      const params = {
        order: this.sortOption.order,
        sort: this.sortOption.sort,
        page: this.pagination.currentPage || 1,
        seed: this.seed,
        isAdvance: this.isAdvanceSearch ? 1 : 0
      };
      if (this.isAdvanceSearch) {
        params.keyword = JSON.stringify(this.advanceSearchKeywords, null, 0);
      } else if (this.$route.query.keyword) {
        params.keyword = this.$route.query.keyword;
      }

      return this.$axios.get(this.url, { params })
        .then(response => {
          const works = response.data.works;
          this.works = works.concat();
          this.pagination = response.data.pagination;
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
      if (this.$route.query.circleId || this.$route.query.tagId || this.$route.query.vaId) {
        let url = "", restrict = "";
        if (this.$route.query.circleId) {
          restrict = "circles";
          url = `/api/${restrict}/${this.$route.query.circleId}`;
        } else if (this.$route.query.tagId) {
          restrict = "tags";
          url = `/api/${restrict}/${this.$route.query.tagId}`;
        } else {
          restrict = "vas";
          url = `/api/${restrict}/${this.$route.query.vaId}`;
        }

        this.$axios.get(url)
          .then(response => {
            const name = response.data.name;
            // pageTitle += name || "";
            this.searchMetas = [name];
            this.pageTitle = "Search works";
          }).catch(error => {
            if (error.response) {
              // 请求已发出，但服务器响应的状态码不在 2xx 范围内
              if (error.response.status !== 401) {
                this.showErrNotif(error.response.data.error || `${error.response.status} ${error.response.statusText}`);
              }
            } else {
              this.showErrNotif(error.message || error);
            }
          });
      } else if (this.$route.query.keyword) {
        this.pageTitle = `Search By`;
        this.searchMetas = [this.$route.query.keyword];
      } else if (this.isAdvanceSearch) {
        this.pageTitle = "Search By";
      } else {
        this.pageTitle = "All works";
        this.searchMetas = [];
      }
    },

    reset() {
      this.seed = Math.floor(Math.random() * 100);
      this.stopLoad = true;
      this.refreshPageTitle();
      this.pagination = { currentPage: 0, pageSize: 12, totalCount: 0 };
      this.requestWorksQueue().then(() => { this.stopLoad = false });
    },

    onWorkCardTouch(id) {
      this.touchedWorkId = id;
    },

    checkAdvanceSearchMode() {
      this.isAdvanceSearch = this.$route.name == "advance search";
    },

    onAddAdvanceSearchKeyword() {
      const keyword = this.editKeyword.trim()
      if (keyword === "") {
        this.showErrNotif("无法添加空白的关键字");
        return;
      }

      for (let kw of this.advanceSearchKeywords) {
        if (kw.t == AdvanceSearchCondType.FUZZY && kw.d == keyword) {
          this.showErrNotif("关键字重复，添加失败");
          return;
        }
      }

      this.advanceSearchKeywords.push({
        t: AdvanceSearchCondType.FUZZY,
        d: keyword,
      });
      this.editKeyword = "";
      this.reset();
    },

    removeAdvanceSearchKeyword(index) {
      this.advanceSearchKeywords.splice(index, 1);
    }
  }
};
</script>

<style lang="scss" scoped>
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
