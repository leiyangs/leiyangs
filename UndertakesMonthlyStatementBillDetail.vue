<!-- 代发月结账单-详情 -->
<template>
  <div class="app-container">
    UndertakesMonthlyStatementBillDetail
    <div class="filter-container">
     <AdvancedSearch class="filter-item" ref="advancedSearch" @clearQuery="clearQuery" :isShowMore="false">
        <div slot="searchHeader"  >
          <span class="filter-item">
            <el-date-picker
              style="width: 250px;"
              v-model="listQuery.monthrange"
              type="monthrange"
              unlink-panels
              range-separator="-"
              start-placeholder="开始月份"
              end-placeholder="结束月份"
              :picker-options="pickerOptions"
              @keyup.enter.native="handleFilter">
            </el-date-picker>
          </span>
          <el-input placeholder='账户名称' v-model.trim='listQuery.account_name' clearable filterable style="width: 200px;" class="filter-item" @keyup.enter.native="handleFilter" />
          <el-select v-model.trim='listQuery.status' class="filter-item " style="width: 120px;margin-right:10px;" @keyup.enter.native="handleFilter" filterable placeholder='状态'>
            <el-option v-for="item in statusList" :key="item.value" :label="item.label" :value="item.value">
            </el-option>
          </el-select>
        </div>
        <div slot="searchButton" class="searchButton">
          <el-button icon="el-icon-search" class="filter-item" type="primary" @click="handleFilter"></el-button>
        </div>
        <div  slot="searchList" class="searchList"  >
          <el-form ref="searchForm" :model="listQuery" label-position="right" label-width="100px">
            <el-row>
            </el-row>
          </el-form>
        </div>
      </AdvancedSearch>
    </div>
    <el-table class="tableStyle2" v-loading="listLoading" :key="tableKey" ref="table" :data="list" fit stripe highlight-current-row @header-contextmenu="openHeaderMenu" @header-dragend="handleDragent" @row-contextmenu="openMenu" @row-click="openMenu"  style="width: 100%;" :height="height-50" header-cell-class-name="headerstyle2" @selection-change="handleSelectionChange">
      <template v-for="(item,index) in field_conf.field_list">
        <el-table-column v-if="item.field_code=='action_flag'" :label='item.field_name' :key="index" :sortable="true" :prop='item.field_code' :align="item.text_align" :min-width="item.field_width?item.field_width:''">
          <template slot-scope="scope">
            <tag :scene="'danger'" size="mini" v-if="scope.row.action_flag==0">停用</tag>
            <tag :scene="'success'" size="mini" v-if="scope.row.action_flag==1">正常</tag>
          </template>
        </el-table-column>
        <el-table-column v-else :label='item.field_name' :key="index" :sortable="true" :prop='item.field_code' :align="item.text_align" :min-width="item.field_width?item.field_width:''">
          <template slot-scope="scope">
            <span>{{ scope.row[item.field_code]}}</span>
          </template>
        </el-table-column>
      </template>
    </el-table>
    <Contextmenu ref="menu">
      <div slot="menuList">
        <el-dropdown-item size="mini" icon="el-icon-edit-outline" v-if="activeTableRow.status==0" @click.native="handleSubmitStatus(activeTableRow)">
          <span>提交</span>
        </el-dropdown-item>
        <el-dropdown-item size="mini" icon="el-icon-tickets">
          <Log title='操作日志' showType="text" type="pretransfer_warning_goods_tbl" :id="activeTableRow.id" />
        </el-dropdown-item>
      </div>
    </Contextmenu>
    <HeaderContextmenu ref="headerMenu" :method="field_conf.function_method" @refresh="getList"></HeaderContextmenu>
    <pagination v-show="total>0" :total="total" :page.sync="listQuery.page_no" :limit.sync="listQuery.page_size" @pagination="getList" />
  </div>
</template>
<script>
import {
  GetAgencyBillMonthList, ImportAgencyBillMonthDeatil, GetAccountListByCondition, CommitAgencyBillMonth
} from "@/api/distribution";
import waves from "@/directive/waves"; // Waves directive
import { parseTime } from "@/utils/index";
import Pagination from "@/components/Pagination"; // Secondary package based on el-pagination
import Contextmenu from "@/components/Contextmenu";
import HeaderContextmenu from "@/components/HeaderContextmenu";
import AdvancedSearch from "@/components/AdvancedSearch";
import UploadFile from "@/components/UploadFile";
import Log from "@/components/Log";
const newDate = parseTime(Date.now(), '{y}-{m}')
export default {
  name: "UndertakesMonthlyStatementBillDetail",
  components: { Pagination, AdvancedSearch, Contextmenu, HeaderContextmenu, UploadFile, Log },
  directives: { waves },
  props: {
    height: {
      type: Number
    }
  },
  data() {
    return {
      tableKey: 0,
      total: 0,
      listLoading: true, //加载loading
      dialogUploadVisible: false,
      btnLoading: false,
      listQuery: {
        page_size: 50,
        page_no: 1,
        order_by: "",
        direction: "",
        monthrange: [newDate, newDate]
      }, //搜索条件
      list: [],
      activeTableRow: {},
      field_conf: {},
      type: "create",
      statusList: [
        { label: "全部状态", value: '-1' },
        { label: "已提交", value: '1' },
        { label: "未提交", value: '0' }
      ],
      pickerOptions: {
          shortcuts: [{
            text: '本月',
            onClick(picker) {
              picker.$emit('pick', [new Date(), new Date()]);
            }
          }, {
            text: '今年至今',
            onClick(picker) {
              const end = new Date();
              const start = new Date(new Date().getFullYear(), 0);
              picker.$emit('pick', [start, end]);
            }
          }, {
            text: '最近六个月',
            onClick(picker) {
              const end = new Date();
              const start = new Date();
              start.setMonth(start.getMonth() - 6);
              picker.$emit('pick', [start, end]);
            }
          }]
        },
        exampleFile: 'http://cdn.upyun.mallimg.shjinjia.com.cn/erpapi/Uploads/20200924/通用导入模板_5be9833a-ae88-4f86-8b1b-744b3316b7bd.xlsx',
        formdata: { },
        formRules: { },
        accountNoList: [],
        behalf_list: [],
    };
  },
  mounted() {
    this.$refs.table.bodyWrapper.addEventListener("scroll", () => {
      this.$refs.menu.closeMenu();
    });
  },
  created() {
    // 获取信息
    this.getInfo();
    this.getList();
  },
  methods: {
    initListQuery() {
      if(this.listQuery.monthrange) {
        this.listQuery.start_time = this.listQuery.monthrange[0];
        this.listQuery.end_time = this.listQuery.monthrange[1];
      }else {
        this.listQuery.start_time = '';
        this.listQuery.end_time = '';
      }
    },
    // 获取列表
    getList() {
      this.listLoading = true;
      this.initListQuery();
      GetAgencyBillMonthList(this.listQuery).then(response => {
        this.listLoading = false;
        if (response.data.res_status_code == 0) {
          this.list = response.data.res_content.data_list;
          this.field_conf = response.data.res_content.field_conf;
          this.total = response.data.res_content.total_count;
          if (this.$refs.table) {
            this.$refs.table.bodyWrapper.scrollTop = 0;
          }
        } else {
          this.$message({
            type: "error",
            message: response.data.res_message,duration:"5000"
          });
        }
      });
    },
    getInfo() {
      GetAccountListByCondition().then(res => {
        if (res.data.res_status_code == 0) {
          this.accountNoList = res.data.res_content.data_list;
        }else {
          this.$message({
            type: "error",
            message: res.data.res_message,duration:"5000"
          });
        }
      })
    },
    handleSelectionChange(val) {
      this.behalf_list = val;
    },
    // 提交
    handleSubmitStatus(row) {
      this.$confirm(`确定提交吗?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      }).then(() => {
        CommitAgencyBillMonth({behalf_list: [row]}).then(response => {
          if (response.data.res_status_code == 0) {
            this.$message({
              type: "success",
              message: "保存成功"
            });
            this.getList();
          } else {
            this.$message({
              type: "error",
              message: response.data.res_message
            });
          }
        })
      })
      .catch(() => {
        this.$message({
          type: "info",
          message: "已取消操作"
        });
      });
    },
    handleFilter() {
      this.listQuery.page_no = 1;
      this.getList();
    },
    clearQuery() {
      this.$refs["searchForm"].resetFields();
      this.handleFilter();
    },
    openMenu(row, column, event) {
      this.$refs.menu.openMenu(row, column, event);
      this.activeTableRow = row;
    },
    openHeaderMenu(column, event) {
      this.$refs.headerMenu.openMenu(column, event);
    },
    handleDragent(newWidth, oldWidth, column, event) {
      this.$refs.headerMenu.handleDragent(newWidth, oldWidth, column, event);
    },
    parseTime
  }
};
</script>

