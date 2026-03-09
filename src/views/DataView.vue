<template>
  <div class="data-view">
    <!-- 简单页头 -->
    <div class="data-header">
      <h2>📊 数据展示</h2>
      <!-- 表格数据统计：显示当前筛选后的数据行数和列数 -->
      <span class="row-count" v-if="hasData && !isLoading">
        <!-- 只在非加载状态显示统计 -->
        <i class="el-icon-s-data" style="margin-right: 6px"></i>
        共<span class="highlight-number">{{ filteredDataLength }}</span> 行数据
      </span>
      <!-- 返回按钮样式 -->
      <el-button type="text" @click="goToUpload" class="back-btn">
        <i class="el-icon-arrow-left" style="margin-right: 6px"></i>
        返回上传
      </el-button>
    </div>

    <!-- 数据表格容器 -->
    <div class="data-container" :class="{ 'is-loading': isLoading }">
      <!-- +++ 新增：现代化加载遮罩层 +++ -->
      <div v-if="isLoading" class="loading-mask">
        <div class="loading-content">
          <!-- 可选用Element UI的加载图标，或自定义SVG -->
          <div class="loading-spinner">
            <i class="el-icon-loading"></i>
            <!-- 或者使用纯CSS动画 -->
            <!-- <div class="spinner"></div> -->
          </div>
          <p class="loading-text">正在准备数据，请稍候...</p>
        </div>
      </div>

      <!-- 数据表格 -->
      <DataTable
        :data="csvData"
        v-if="hasData"
        @filter-change="handleFilterChange"
        @data-ready="handleDataReady"
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
      isLoading: true, // +++ 新增：加载状态控制变量 +++
    };
  },
  computed: {
    hasData() {
      return this.csvData && this.csvData.length > 0;
    },
  },
  created() {
    this.loadData();
    // +++ 新增：安全机制，防止data-ready事件意外未触发，最多显示8秒 +++
    this.loadingTimer = setTimeout(() => {
      if (this.isLoading) {
        console.warn("加载超时，强制关闭遮罩");
        this.isLoading = false;
      }
    }, 8000);
  },
  beforeDestroy() {
    // +++ 新增：清理定时器 +++
    if (this.loadingTimer) clearTimeout(this.loadingTimer);
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
          this.isLoading = false; // 没有数据也关闭加载
        }
      } catch (error) {
        console.error("加载数据失败:", error);
        this.$message.error("加载数据失败");
        this.isLoading = false; // 出错也关闭加载
      }
    },
    handleFilterChange(filteredData) {
      this.filteredDataLength = filteredData.length;
    },
    // +++ 新增：处理子组件数据就绪事件 +++
    handleDataReady() {
      console.log("DataTable 数据已就绪，关闭遮罩");
      this.isLoading = false;
      if (this.loadingTimer) clearTimeout(this.loadingTimer); // 关闭安全定时器
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
  justify-content: space-between;
  align-items: center;
  background: rgba(255, 255, 255, 0.9);
  padding: 10px 20px;
  border-radius: 8px;
  margin-bottom: 5px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  z-index: 1; /* 确保在遮罩层上方 */
  position: relative;
}

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

.highlight-number {
  font-weight: bold;
  font-size: 1.1em;
  color: #409eff;
  padding: 0 4px;
}

.back-btn {
  color: #303133;
  font-size: 1.1rem;
  font-weight: 500;
  padding: 6px 12px;
  display: flex;
  align-items: center;
}

.back-btn:hover {
  color: #409eff;
  background: #f8f9fa;
  border-radius: 6px;
  border: 1px solid #ebeef5;
}

/* +++ 新增：数据容器添加相对定位，作为遮罩层的参考 +++ */
.data-container {
  flex: 1;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 8px;
  padding: 10px 10px 1px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  min-height: 400px;
  margin-top: 0;
  display: flex;
  flex-direction: column;
  position: relative; /* 为绝对定位的遮罩层提供参考 */
  overflow: hidden; /* 防止遮罩层溢出时出现滚动条 */
}

/* +++ 新增：现代化加载遮罩层样式 +++ */
.loading-mask {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255, 255, 255, 0.92); /* 半透明白色背景，更柔和 */
  backdrop-filter: blur(4px); /* 毛玻璃效果，现代感更强 */
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
  border-radius: 8px;
  animation: fadeIn 0.3s ease-out;
}

.loading-content {
  text-align: center;
  padding: 40px;
  border-radius: 16px;
  /* 可选：给内容区域加一点非常 subtle 的阴影和背景 */
  /* background: rgba(255, 255, 255, 0.6); */
  /* box-shadow: 0 8px 32px rgba(0, 0, 0, 0.08); */
}

.loading-spinner {
  margin-bottom: 20px;
}

.loading-spinner .el-icon-loading {
  font-size: 54px;
  color: #667eea; /* 使用你主题的主色调 */
  animation: spin 1.5s linear infinite;
}

/* 纯CSS旋转动画备用 */
@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loading-text {
  font-size: 1.1rem;
  color: #606266;
  font-weight: 500;
  margin-top: 10px;
  letter-spacing: 0.5px;
}

/* 遮罩层淡入动画 */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
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
  .loading-content {
    padding: 30px 20px;
  }
  .loading-spinner .el-icon-loading {
    font-size: 44px;
  }
  .loading-text {
    font-size: 1rem;
  }
}
</style>