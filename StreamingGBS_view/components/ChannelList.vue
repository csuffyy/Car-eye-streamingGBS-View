<template>
    <div>
        <div class="box box-primary">
          <div class="box-header">
            <h4 class="text-primary text-center">通道列表({{devicecode||devicename}})</h4>
          </div>
          <div class="box-body">
            <form class="form-inline" autocomplete="off" spellcheck="false">
              <div class="form-group form-group-sm">
                <button type="button" class="btn btn-sm btn-primary" @click.prevent="$router.go(-1)">
                  <i class="fa fa-chevron-left"></i> 返回
                </button>
              </div>
              <div class="input-group input-group-sm" v-if="hasAnyRole(buttons, userInfo, '756091030016753664')">
                <button type="button" class="btn btn-sm btn-primary" @click.prevent="addChannel()">
                  <i class="fa fa-plus"></i> 添加通道
                </button>
              </div>
              <span class="hidden-xs">&nbsp;&nbsp;</span>
              <div class="form-group form-group-sm">
                <label>搜索</label>
                <input type="text" class="form-control" placeholder="关键字" v-model.trim="q" @keydown.enter.prevent ref="q">
              </div>
              <span class="hidden-xs">&nbsp;&nbsp;</span>
              <div class="form-group form-group-sm">
                <label>在线状态</label>
                <select class="form-control" v-model.trim="online">
                  <option value="">全部</option>
                  <option value="1">在线</option>
                  <option value="0">离线</option>
                </select>
              </div>
            </form>
            <br>
            <el-table :data="channels" stripe :default-sort="{prop: 'id', order: 'desc'}" @sort-change="sortChange" v-loading="loading" element-loading-text="加载中...">
              <el-table-column prop="channelcode" label="通道编码" min-width="140" show-overflow-tooltip sortable="custom"></el-table-column>
              <el-table-column label="操作" min-width="120" v-if="isMobile()" class-name="opt-group">
                <template slot-scope="props">
                    <div class="btn-group btn-group-xs">
                        <button type="button" class="btn btn-warning" @click.prevent="editChannel(props.row)" v-if="hasAnyRole(buttons, userInfo, '756091457781235712')">
                          <i class="fa fa-edit"></i> 编辑
                        </button>
                        <button type="button" class="btn btn-danger" @click.prevent="removeChannel(props.row)" v-if="!props.row.status && hasAnyRole(buttons, userInfo, '756091557672779776')">
                          <i class="fa fa-remove"></i> 删除
                        </button>
                    </div>
                </template>
              </el-table-column>
              <el-table-column prop="channelname" label="通道名称" min-width="140" show-overflow-tooltip></el-table-column>
              <el-table-column prop="status" label="在线状态" min-width="100">
                <template slot-scope="props">
                  <span class="text-success" v-if="props.row.status === 1">在线</span>
                  <span class="text-gray" v-else>离线</span>
                </template>
              </el-table-column>
              <el-table-column prop="ptzEnable" label="是否支持云台" min-width="120">
                <template slot-scope="props">
                  <span class="text-success" v-if="props.row.ptzEnable === 1">支持</span>
                  <span class="text-gray" v-else>不支持</span>
                </template>
              </el-table-column>
              <el-table-column prop="talkEnbale" label="是否支持对讲" min-width="120">
                <template slot-scope="props">
                  <span class="text-success" v-if="props.row.talkEnbale === 1">支持</span>
                  <span class="text-gray" v-else>不支持</span>
                </template>
              </el-table-column>
              <el-table-column label="操作" min-width="120" fixed="right" v-if="!isMobile()" class-name="opt-group">
                <template slot-scope="props">
                    <div class="btn-group btn-group-xs">
                        <button type="button" class="btn btn-warning" @click.prevent="editChannel(props.row)" v-if="hasAnyRole(buttons, userInfo, '756091457781235712')">
                          <i class="fa fa-edit"></i> 编辑
                        </button>
                        <button type="button" class="btn btn-danger" @click.prevent="removeChannel(props.row)" v-if="!props.row.status && hasAnyRole(buttons, userInfo, '756091557672779776')">
                          <i class="fa fa-remove"></i> 删除
                        </button>
                    </div>
                </template>
              </el-table-column>
            </el-table>
          </div>
          <div class="box-footer" v-if="total > 0">
            <el-pagination layout="total,prev,pager,next" :pager-count="5" class="pull-right" :total="total" :page-size.sync="pageSize" :current-page.sync="currentPage"></el-pagination>
          </div>
        </div>

        <ChannelAddDlg ref="channelAddDlg" @submit="getChannels"></ChannelAddDlg>
        <ChannelEditDlg ref="channelEditDlg" @submit="getChannels"></ChannelEditDlg>
    </div>
</template>

<script>
import _ from "lodash";
import { mapState } from "vuex";
import ChannelAddDlg from 'components/ChannelAddDlg.vue'
import ChannelEditDlg from 'components/ChannelEditDlg.vue'
export default {
  props: {
    deviceid: {
      type: String,
      default: ""
    }
  },
  data() {
    return {
      q: "",
      online: "",
      total: 0,
      pageSize: 10,
      currentPage: 1,
      sort: "id",
      order: "desc",
      loading: false,
      timer: 0,
      channels: [],
      recorder: null,
      devicename: '',
      devicecode: ''
    };
  },
  computed: {
    ...mapState(['userInfo', 'buttons']),
  },
  watch: {
    q: function(newVal, oldVal) {
      this.doDelaySearch();
    },
    online: function(newVal, oldVal) {
      this.doSearch();
    },
    currentPage: function(newVal, oldVal) {
      this.doSearch(newVal);
    }
  },
  components: {
    ChannelAddDlg,ChannelEditDlg
  },
  mounted() {
    // this.$refs["q"].focus();
    this.getDeviceInfo();
    this.getChannels();
    this.timer = setInterval(() => {
        this.getChannels();
    }, 3000);
  },
  beforeDestroy() {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = 0;
    }
  },
  methods: {
    keyDown(e) {
      if(e.keyCode == 27) {
        this.$el.querySelector('.fa-chevron-left').click();
      }
    },
    isMobile() {
      return videojs.browser.IS_IOS || videojs.browser.IS_ANDROID;
    },
    doSearch(page = 1) {
      var query = {};
      if (this.q) query["q"] = this.q;
      if (this.online) query["online"] = this.online;
      this.$router.replace({
        path: `/devices/channels/${this.deviceid}/${page}`,
        query: query
      });
    },
    doDelaySearch: _.debounce(function() {
      this.doSearch();
    }, 500),
    getChannels() {
      $.get(this.$store.state.baseUrl + "/deviceChannelInfo/list", {
        deviceid: this.deviceid,
        q: this.q,
        page: this.currentPage,
        limit: this.pageSize,
        online: this.online,
        sort: this.sort,
        order: this.order
      })
      .then(ret => {
        this.total = ret.count;
        this.channels = ret.data;
      })
      .always(() => {
      });
    },
    sortChange(data) {
      this.sort = data.prop;
      this.order = data.order == "ascending" ? "asc" : "desc";
      this.getChannels();
    },
    formatName(row, col, cell) {
      if (cell) return cell;
      return "-";
    },
    getDeviceInfo(){
      $.get(this.$store.state.baseUrl + "/deviceInfo/details", {
        id: this.deviceid
      })
      .then(ret => {
        this.devicename = ret.data.devicename;
        this.devicecode = ret.data.devicecode;
      })
      .always(() => {
      });
    },
    removeChannel(row) {
      this.$confirm(`确认删除 ${row.channelname || row.channelcode}`, "提示").then(() => {
        $.post(this.$store.state.baseUrl + "/deviceChannelInfo/delete", {
          id: row.id
        }).always(() => {
          this.getChannels();
        });
      }).catch(() => {});
    },
    editChannel(row) {
      this.$refs["channelEditDlg"].show({
        id: row.id,
        deviceid: row.deviceid,
        channelcode: row.channelcode,
        channelname: row.channelname,
        ptzEnable: row.ptzEnable,
        talkEnbale: row.talkEnbale
      });
    },
    addChannel(){
      this.$refs["channelAddDlg"].show({
        deviceid: this.deviceid,
      });
    }
  },
  beforeRouteEnter(to, from, next) {
    next(vm => {
      vm.q = to.query.q || "";
      vm.online = to.query.online || "";
      vm.currentPage = parseInt(to.params.page) || 1;
    });
  },
  beforeRouteUpdate(to, from, next) {
    next();
    this.$nextTick(() => {
      this.q = to.query.q || "";
      this.online = to.query.online || "";
      this.currentPage = parseInt(to.params.page) || 1;
      this.channels = [];
      this.getChannels();
    });
  }
};
</script>

<style lang="less">
.opt-group .cell {
  overflow: visible;
}
</style>



