<template>
  <div class="home-view">
    <!-- 页面容器 -->
    <div class="home-container">
      <!-- 页头 -->
      <div class="home-header">
        <h1>📁 文件上传</h1>
        <p class="subtitle">请上传CSV文件进行分析和查看</p>
      </div>
      
      <!-- 主内容 -->
      <div class="home-content">
        <!-- 上传组件 -->
        <CsvUploader @data-loaded="handleDataLoaded" />
        
        <!-- 使用说明 -->
        <div class="usage-tips">
          <h3>📋 使用说明</h3>
          <ul>
            <li>拖拽或点击上传CSV文件</li>
            <li>解析成功后自动跳转到数据页</li>
            <li>数据在浏览器本地处理，不上传服务器</li>
          </ul>
        </div>
      </div>
      
      <!-- 页脚 -->
      <div class="home-footer">
        <p>CSV文件分析工具 | version 1.0.0</p>
      </div>
    </div>
  </div>
</template>

<script>
import CsvUploader from '../components/CsvUploader.vue'

export default {
  name: 'Home',
  components: {
    CsvUploader
  },
  methods: {
    handleDataLoaded(data) {
      // 保存数据到本地存储
      localStorage.setItem('csvData', JSON.stringify(data.data))
      localStorage.setItem('csvHeaders', JSON.stringify(data.headers))
      
      // 跳转到数据页
      this.$router.push('/data')
    }
  }
}
</script>

<style scoped>
.home-view {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 12px;
}

.home-container {
  width: 100%;
  max-width: 800px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 12px;
  box-shadow: 0 8px 40px rgba(0, 0, 0, 0.2);
  overflow: hidden;
}

.home-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 20px 20px;
  text-align: center;
}

.home-header h1 {
  font-size: 2rem;
  margin: 0 0 10px 0;
  font-weight: 300;
}

.subtitle {
  font-size: 0.9rem;
  opacity: 0.9;
  margin: 0;
}

.home-content {
  padding: 15px;
}

.usage-tips {
  margin-top: 20px;
  padding: 15px;
  background: #f8f9fa;
  border-radius: 8px;
  border-left: 4px solid #667eea;
}

.usage-tips h3 {
  color: #303133;
  margin: 0 0 15px 0;
  font-weight: 500;
}

.usage-tips ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.usage-tips li {
  color: #606266;
  padding: 8px 0;
  font-size: 0.9rem;
  position: relative;
  padding-left: 20px;
}

.usage-tips li:before {
  content: '✓';
  color: #67C23A;
  position: absolute;
  left: 0;
  font-weight: bold;
}

.home-footer {
  background: #f8f9fa;
  padding: 15px 20px;
  text-align: center;
  color: #606266;
  font-size: 0.85rem;
  border-top: 1px solid #ebeef5;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .home-view {
    padding: 10px;
  }
  
  .home-container {
    border-radius: 8px;
  }
  
  .home-header h1 {
    font-size: 1.5rem;
  }
  
  .home-content {
    padding: 15px;
  }
}
</style>
