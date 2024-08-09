<template>
  <div>
    <div class="text-h5 text-weight-regular q-ma-md">
      {{ pageTitle }}
      <span v-show="pagination.totalCount" style="margin-left: -6px">
        ({{ pagination.totalCount }})
      </span>
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
      pagination: {
        currentPage: 0,
        pageSize: 12,
        totalCount: 0
      },
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
      ]
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
      } else if (query.keyword) {
        return `/api/search/${query.keyword}`;
      } else {
        return "/api/works";
      }
    }
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
        seed: this.seed
      };

      return this.$axios
        .get(this.url, { params })
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
        let url = "",
          restrict = "";
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
            let pageTitle;

            switch (restrict) {
              case "tags":
                pageTitle = "Works tagged with ";
                break;
              case "vas":
                pageTitle = "Works voiced by ";
                break;
              case "circles":
                pageTitle = "Works by ";
                break;
            }
            pageTitle += name || "";

            this.pageTitle = pageTitle;
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
        this.pageTitle = `Search by ${this.$route.query.keyword}`;
      } else {
        this.pageTitle = "All works";
      }
    },

    reset() {
      this.stopLoad = true;
      this.refreshPageTitle();
      this.pagination = {
        currentPage: 0,
        pageSize: 12,
        totalCount: 0
      };
      this.requestWorksQueue().then(() => {
        this.stopLoad = false;
      });
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
