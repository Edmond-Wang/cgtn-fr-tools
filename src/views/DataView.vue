<template>
  <div class="data-view">
    <!-- 简单页头 -->
    <div class="data-header">
      <h2>📊 数据展示</h2>
      <!-- 表格数据统计：显示当前筛选后的数据行数和列数 -->
      <span class="row-count" v-if="hasData">
        <i class="el-icon-s-data" style="margin-right: 6px"></i>
        共<span class="highlight-number">{{ filteredDataLength }}</span> 行数据
      </span>
      <!-- 返回按钮样式 -->
      <el-button type="text" @click="goToUpload" class="back-btn">
        <i class="el-icon-arrow-left" style="margin-right: 6px"></i>
        返回上传
      </el-button>
    </div>

    <!-- 数据表格 -->
    <div class="data-container">
      <DataTable
        :data="csvData"
        v-if="hasData"
        @filter-change="handleFilterChange"
      />
    </div>
  </div>
</template>

<script>
import DataTable from "../components/DataTable.vue";

export default {
  name: "DataView",
  components: {
    DataTable,
  },
  data() {
    return {
      csvData: [],
      csvHeaders: [],
      filteredDataLength: 0,
    };
  },
  computed: {
    hasData() {
      return this.csvData && this.csvData.length > 0;
    },
  },
  created() {
    this.loadData();
  },
  methods: {
    loadData() {
      try {
        const savedData = localStorage.getItem("csvData");
        const savedHeaders = localStorage.getItem("csvHeaders");

        if (savedData && savedHeaders) {
          this.csvData = JSON.parse(savedData);
          this.csvHeaders = JSON.parse(savedHeaders);
          this.filteredDataLength = this.csvData.length;
        } else {
          this.$message.warning("没有找到数据，请先上传文件");
        }
      } catch (error) {
        console.error("加载数据失败:", error);
        this.$message.error("加载数据失败");
      }
    },
    handleFilterChange(filteredData) {
      this.filteredDataLength = filteredData.length;
    },
    goToUpload() {
      this.$router.push("/");
    },
  },
};
</script>

<style scoped>
.data-view {
  height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 10px;
  display: flex;
  flex-direction: column;
  box-sizing: border-box;
}

.data-header {
  display: flex;
  justify-content: space-between; /* 三元素分散对齐 */
  align-items: center;
  background: rgba(255, 255, 255, 0.9);
  padding: 10px 20px;
  border-radius: 8px;
  margin-bottom: 5px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

/* 统计信息样式 */
/* 行计数：显示当前数据量信息 */
.row-count {
  color: #606266;
  font-size: 0.95rem;
  font-weight: 500;
  padding: 6px 12px;
  background: #f8f9fa;
  border-radius: 6px;
  border: 1px solid #ebeef5;
  display: flex;
  align-items: center;
}
/* 突出显示数字 */
.highlight-number {
  font-weight: bold;
  font-size: 1.1em;
  color: #409eff;
  padding: 0 4px;
}

.row-count i {
  color: #409eff;
  font-size: 16px;
}

.row-count i {
  color: #409eff;
  font-size: 16px;
}

.data-header h2 {
  color: #303133;
  font-weight: 500;
  margin: 0;
}

.back-btn {
  color: #303133;
  font-size: 1.1rem;
  font-weight: 500;
  padding: 6px 12px;
  display: flex;
  align-items: center;
}

.back-btn i {
  color: #409eff;
  font-size: 16px;
}

.back-btn:hover {
  color: #409eff;
  background: #f8f9fa;
  border-radius: 6px;
  border: 1px solid #ebeef5;
}
.data-container {
  flex: 1;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 8px;
  padding: 10px 10px 1px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  /* 移除固定max-height限制，改为最小高度确保内容能撑开 */
  min-height: 400px;
  margin-top: 0;
  /* 添加flex布局确保子元素能填充空间 */
  display: flex;
  flex-direction: column;
}

.no-data {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 60vh;
}

.data-footer {
  background: rgba(255, 255, 255, 0.9);
  padding: 10px 20px;
  border-radius: 8px;
  margin-top: 20px;
  text-align: center;
  color: #606266;
  font-size: 0.9rem;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

/* 响应式设计 */
@media (max-width: 768px) {
  .data-view {
    padding: 5px;
  }

  .data-header {
    flex-direction: column;
    gap: 12px;
    text-align: center;
  }

  .data-container {
    padding: 10px;
  }
}
</style>
