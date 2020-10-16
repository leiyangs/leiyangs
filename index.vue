<!-- 代发月结账单 -->
<template>
  <div class="app-container">
    <div class="filter-container">
     <AdvancedSearch class="filter-item" ref="advancedSearch" @clearQuery="clearQuery" :isShowMore="false">
        <div slot="searchHeader"  >
          <span class="filter-item">
            <el-date-picker
              style="width: 280px;"
              v-model="listQuery.monthrange"
              type="monthrange"
              align="right"
              unlink-panels
              range-separator="-"
              start-placeholder="开始月份"
              end-placeholder="结束月份"
              :picker-options="pickerOptions"
              @keyup.enter.native="handleFilter">
            </el-date-picker>
          </span>
          <el-input placeholder='账户名称' v-model.trim='listQuery.org_code' clearable filterable style="width: 200px;" class="filter-item" @keyup.enter.native="handleFilter" />
          <el-select v-model.trim='listQuery.action_flag' class="filter-item " style="width: 120px;margin-right:10px;" @keyup.enter.native="handleFilter" filterable placeholder='状态'>
            <el-option v-for="item in action_flag" :key="item.value" :label="item.label" :value="item.value">
            </el-option>
          </el-select>
        </div>
        <div slot="searchButton" class="searchButton">
          <el-button icon="el-icon-search" class="filter-item" type="primary" @click="handleFilter"></el-button>
          <el-button v-waves class="filter-item" type="primary" @click="handleImport">
            <svg-icon class-name="excel" icon-class="excel" style="margin:0 10px 0 0" />导入
          </el-button>
          <el-button class="filter-item" type="primary">批量提交</el-button>
        </div>
        <div  slot="searchList" class="searchList"  >
          <el-form ref="searchForm" :model="listQuery" label-position="right" label-width="100px">
            <el-row>
            </el-row>
          </el-form>
        </div>
      </AdvancedSearch>
    </div>
    <el-table class="tableStyle2" v-loading="listLoading" :key="tableKey" ref="table" :data="list" fit stripe highlight-current-row @header-contextmenu="openHeaderMenu" @header-dragend="handleDragent" @row-contextmenu="openMenu" @row-click="openMenu"  style="width: 100%;" :height="height-50" header-cell-class-name="headerstyle2">
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
      </div>
    </Contextmenu>
    <HeaderContextmenu ref="headerMenu" :method="field_conf.function_method" @refresh="getList"></HeaderContextmenu>
    <pagination v-show="total>0" :total="total" :page.sync="listQuery.page_no" :limit.sync="listQuery.page_size" @pagination="getList" />
    <!-- 导入 -->
    <el-dialog title="账单导入" :visible.sync="dialogUploadVisible" :append-to-body="true">
      <el-form ref="uploadSend" :model="formdata" :rules="formRules" label-position="left" label-width="180px">
        <el-form-item label="下载模板">
          <a target="view_window" :href="exampleFile">
            <i class="el-icon-document"></i>下载模板</a>
        </el-form-item>
        <el-form-item label="上传文件">
          <UploadFile :files="formdata.file_path" id="logo" :acceptedFiles="'.xlsx,'" :maxFiles="1" :disabled='true' @changeFile="changeFile"></UploadFile>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogUploadVisible = false">取消</el-button>
        <el-button type="primary" @click="submitFile()">确认</el-button>
      </div>
    </el-dialog>
  </div>
</template>
<script>
import {
  GetSellerListByCondition
} from "@/api/distribution";
import waves from "@/directive/waves"; // Waves directive
import { parseTime } from "@/utils/index";
import Pagination from "@/components/Pagination"; // Secondary package based on el-pagination
import Contextmenu from "@/components/Contextmenu";
import HeaderContextmenu from "@/components/HeaderContextmenu";
import AdvancedSearch from "@/components/AdvancedSearch";
import UploadFile from "@/components/UploadFile";
export default {
  name: "UndertakesMonthlyStatementBill",
  components: { Pagination, AdvancedSearch, Contextmenu, HeaderContextmenu, UploadFile },
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
      listQuery: {
        page_size: 50,
        page_no: 1,
        order_by: "",
        direction: "",
      }, //搜索条件
      list: [],
      activeTableRow: {},
      field_conf: {},
      type: "create",
      action_flag: [
        { label: "全部状态", value: '' },
        { label: "停用", value: '0' },
        { label: "正常", value: '1' }
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
        exampleFile: '',
        formdata: {},
        formRules: {
          file_path: [{required: true, message: '请上传文件', trigger: 'change'}]
        },

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
        
      }else {

      }
    },
    // 获取列表
    getList() {
      this.listLoading = true;
      this.initListQuery();
      GetSellerListByCondition(this.listQuery).then(response => {
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
    getInfo() {},
    // 导入button
    handleImport() {
      this.dialogUploadVisible = true;
      this.$set(this.formdata, 'file_path', '');
    },
    changeFile(i) {
      this.formdata.file_path = i;
    },
    // 提交导入
    submitFile() {
      this.$refs["uploadSend"].validate(valid => {
        if(valid) {
          ImportBrandActivityManagement(this.formdata).then(response => {
            if (response.data.res_status_code == 0) {
              this.$message({
                type: "success",
                message: "导入成功"
              });
              this.getInfo();
              this.getList();
              this.dialogUploadVisible = false;
            } else {
              this.$message({
                type: "error",
                message: response.data.res_message
              });
            }
          });
        }else {
          return false
        }
      })
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

