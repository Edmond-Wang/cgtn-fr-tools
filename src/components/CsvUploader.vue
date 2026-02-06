<template>
  <!-- CSV上传卡片组件 -->
  <el-card class="uploader-card">
    <!-- 卡片头部区域 -->
    <template #header>
      <div class="card-header">
        <span>📁 上传CSV文件</span>
      </div>
    </template>

    <!-- 拖拽上传区域 -->
    <div
      class="upload-area"
      @dragover.prevent="handleDragOver"
      @dragleave="handleDragLeave"
      @drop.prevent="handleDrop"
      :class="{ dragging: isDragging }"
      @click="triggerFileInput"
    >
      <div class="upload-content">
        <i class="el-icon-upload" style="font-size: 48px; color: #409eff"></i>
        <p class="upload-text">{{ uploadText }}</p>
        <p class="upload-hint">支持 .csv格式</p>
      </div>
    </div>

    <!-- 隐藏的文件输入框 -->
    <input
      ref="fileInput"
      type="file"
      accept=".csv"
      @change="handleFileSelect"
      style="display: none"
    />

    <!-- 操作按钮区域 -->
    <div v-if="currentFile && !isParsing" class="action-section">
      <div class="action-buttons">
        <el-button
          type="primary"
          @click="parseCSVFile"
          :loading="isParsing"
          icon="el-icon-document-checked"
        >
          {{ isParsing ? "解析中..." : "解析文件" }}
        </el-button>
        <el-button
          @click="clearFile"
          :disabled="isParsing"
          icon="el-icon-delete"
        >
          清除
        </el-button>
      </div>
    </div>

    <!-- 文件信息显示区域 -->
    <div v-if="currentFile" class="file-info">
      <el-alert :title="fileInfo" type="info" show-icon :closable="false" />
    </div>

    <!-- 解析状态显示区域 -->
    <div v-if="isParsing" class="parse-status">
      <el-alert :title="parseStatus" type="info" :closable="false" show-icon />
      <div class="progress-indicator">
        <i class="el-icon-loading" style="margin-right: 8px"></i>
        正在处理，请稍候...
      </div>
    </div>

    <!-- 解析错误提示区域 -->
    <div v-if="parseError" class="error-message">
      <el-alert
        :title="errorTitle"
        type="error"
        show-icon
        :description="errorDescription"
        @close="clearError"
      >
        <template v-if="showErrorDetails">
          <div class="error-details">
            <p><strong>错误详情：</strong></p>
            <pre class="error-detail-content">{{ parseErrorDetails }}</pre>
          </div>
          <div class="error-solution">
            <p><strong>解决方案：</strong></p>
            <ul>
              <li v-for="solution in errorSolutions" :key="solution">
                {{ solution }}
              </li>
            </ul>
          </div>
        </template>
      </el-alert>
    </div>

    <!-- 浏览器兼容性提示区域 -->
    <div v-if="showBrowserWarning" class="browser-warning">
      <el-alert
        title="浏览器兼容性提示"
        type="warning"
        show-icon
        :closable="true"
        @close="showBrowserWarning = false"
      >
        <div slot="title">
          <i class="el-icon-warning" style="margin-right: 8px"></i>
          检测到您正在使用可能存在兼容性问题的浏览器
        </div>
        <p>建议使用 Chrome、Firefox、Edge 等现代浏览器获得最佳体验</p>
      </el-alert>
    </div>
  </el-card>
</template>

<script>
// 导入PapaParse库用于CSV解析
import Papa from "papaparse";

export default {
  name: "CsvUploader",

  data() {
    return {
      // 文件相关
      currentFile: null,

      // 状态相关
      isDragging: false,
      isParsing: false,
      parseStatus: "等待解析...",
      parseError: "",
      parseErrorDetails: "",
      errorSolutions: [],

      // 解析配置
      parseConfig: {
        header: true,
        dynamicTyping: true,
        skipEmptyLines: true,
        worker: false,
        encoding: "UTF-8",
        delimitersToGuess: [",", ";", "\t", "|"],
        preview: 0,
        chunk: null,
        fastMode: false,
      },

      // 浏览器检测
      userAgent: navigator.userAgent,
      isCompatibleBrowser: true,
      showBrowserWarning: false,

      // 加载提示实例
      loadingInstance: null,
    };
  },

  computed: {
    // 上传区域显示的文本
    uploadText() {
      if (this.isDragging) return "释放文件";
      if (this.currentFile) return "已选择文件，点击重新选择或拖拽新文件";
      return "点击或拖拽文件到此处";
    },

    // 文件信息显示
    fileInfo() {
      if (!this.currentFile) return "";
      const sizeMB = (this.currentFile.size / 1024 / 1024).toFixed(2);
      return `已选择文件: ${this.currentFile.name} (${sizeMB} MB)`;
    },

    // 错误标题
    errorTitle() {
      if (this.parseError.includes("编码")) {
        return "文件编码错误";
      } else if (this.parseError.includes("格式")) {
        return "文件格式错误";
      } else if (this.parseError.includes("大小")) {
        return "文件过大";
      } else if (this.parseError.includes("读取")) {
        return "文件读取失败";
      }
      return "解析失败";
    },

    // 错误描述
    errorDescription() {
      if (this.parseError.includes("360")) {
        return "检测到360浏览器兼容性限制";
      }
      return this.parseError;
    },

    // 是否显示错误详情
    showErrorDetails() {
      return this.parseErrorDetails && this.parseErrorDetails.length > 0;
    },
  },

  mounted() {
    // 组件挂载时检测浏览器兼容性
    this.detectBrowser();
  },

  methods: {
    /**
     * 检测浏览器兼容性
     */
    detectBrowser() {
      const ua = this.userAgent.toLowerCase();

      // 检测IE浏览器
      if (ua.includes("msie") || ua.includes("trident")) {
        this.isCompatibleBrowser = false;
        this.showBrowserWarning = true;
        console.error("IE浏览器不支持，请使用现代浏览器");
      }
    },

    /**
     * 触发文件选择对话框
     */
    triggerFileInput() {
      this.$refs.fileInput.click();
    },

    /**
     * 处理文件选择事件
     */
    handleFileSelect(event) {
      const file = event.target.files[0];
      if (file) {
        this.processSelectedFile(file);
      }
      // 重置input，允许选择同一个文件
      event.target.value = "";
    },

    /**
     * 处理拖拽进入事件
     */
    handleDragOver() {
      this.isDragging = true;
    },

    /**
     * 处理拖拽离开事件
     */
    handleDragLeave() {
      this.isDragging = false;
    },

    /**
     * 处理文件拖放事件
     */
    handleDrop(event) {
      this.isDragging = false;
      const file = event.dataTransfer.files[0];
      if (file) {
        this.processSelectedFile(file);
      }
    },

    /**
     * 处理选择的文件
     */
    processSelectedFile(file) {
      // 验证文件
      if (!this.validateFile(file)) {
        return;
      }

      this.currentFile = file;
      this.clearError();
      this.parseStatus = '文件已准备就绪，点击"解析文件"开始解析';

      // 显示文件信息
      this.$message.success(`已选择文件: ${file.name}`);
    },

    /**
     * 验证文件格式和大小
     */
    validateFile(file) {
      // 检查文件类型
      const validExtensions = [".csv", ".txt"];
      const fileExtension = file.name
        .toLowerCase()
        .substring(file.name.lastIndexOf("."));

      if (!validExtensions.includes(fileExtension)) {
        this.showError("请选择CSV或文本文件 (.csv, .txt)", "文件类型错误");
        return false;
      }

      // 检查文件大小（10MB限制）
      const maxSize = 10 * 1024 * 1024;
      if (file.size > maxSize) {
        this.showError(
          `文件大小不能超过10MB，当前文件大小: ${(
            file.size /
            1024 /
            1024
          ).toFixed(2)}MB`,
          "文件过大"
        );
        return false;
      }

      // 检查空文件
      if (file.size === 0) {
        this.showError("文件为空，请选择有效的CSV文件", "空文件错误");
        return false;
      }

      return true;
    },

    /**
     * 显示错误信息
     */
    showError(message, type = "错误") {
      this.parseError = `${type}: ${message}`;
      this.parseErrorDetails = this.getErrorDetails();
      this.errorSolutions = this.getErrorSolutions(type);
      this.$message.error(message);
    },

    /**
     * 清除错误信息
     */
    clearError() {
      this.parseError = "";
      this.parseErrorDetails = "";
      this.errorSolutions = [];
    },

    /**
     * 获取错误详情
     */
    getErrorDetails() {
      return `错误时间: ${new Date().toLocaleString()}
              用户代理: ${this.userAgent}
              文件名称: ${this.currentFile ? this.currentFile.name : "未知"}
              文件大小: ${
        this.currentFile
          ? (this.currentFile.size / 1024).toFixed(2) + "KB"
          : "未知"
      }
      浏览器信息: ${this.getBrowserInfo()}`;
    },

    /**
     * 获取浏览器信息
     */
    getBrowserInfo() {
      const ua = this.userAgent;
      if (ua.includes("Chrome") && ua.includes("Edg")) {
        return "Microsoft Edge";
      } else if (ua.includes("Chrome")) {
        return "Google Chrome";
      } else if (ua.includes("Firefox")) {
        return "Mozilla Firefox";
      } else if (ua.includes("Safari") && !ua.includes("Chrome")) {
        return "Safari";
      } else if (ua.includes("360se") || ua.includes("qihu")) {
        return "360浏览器";
      } else if (ua.includes("msie") || ua.includes("trident")) {
        return "Internet Explorer";
      }
      return "未知浏览器";
    },

    /**
     * 获取错误解决方案
     */
    getErrorSolutions(errorType) {
      const solutions = [];

      switch (errorType) {
        case "文件类型错误":
          solutions.push("请确保文件扩展名为 .csv 或 .txt");
          solutions.push("检查文件是否损坏或格式不正确");
          break;

        case "文件过大":
          solutions.push("请选择小于10MB的文件");
          solutions.push("可以将大文件分割成多个小文件分批处理");
          break;

        case "空文件错误":
          solutions.push("请选择非空文件");
          solutions.push("检查文件内容是否被清空");
          break;

        case "编码错误":
          solutions.push("尝试将文件另存为UTF-8编码格式");
          solutions.push("使用文本编辑器检查文件编码");
          solutions.push("避免使用特殊字符或BOM头");
          break;

        case "解析错误":
          solutions.push("检查CSV文件格式是否正确");
          solutions.push("确保分隔符使用一致（建议使用逗号）");
          solutions.push("检查是否有未闭合的引号");
          solutions.push("尝试使用其他浏览器（如Chrome、Firefox）");
          break;

        default:
          solutions.push("尝试重新选择文件");
          solutions.push("检查文件是否损坏");
          solutions.push("尝试使用其他浏览器");
          solutions.push("联系技术支持提供错误详情");
      }

      return solutions;
    },

    /**
     * 解析CSV文件
     */
    async parseCSVFile() {
      if (!this.currentFile) {
        this.$message.warning("请先选择文件");
        return;
      }

      // 添加加载提示
      this.loadingInstance = this.$loading({
        lock: true,
        text: "正在解析CSV文件，请稍候...",
        spinner: "el-icon-loading",
        background: "rgba(0, 0, 0, 0.7)",
      });

      // 重置状态
      this.isParsing = true;
      this.clearError();
      this.parseStatus = "开始解析文件...";

      try {
        // 使用FileReader读取文件
        const csvText = await this.readFileWithFallback(this.currentFile);

        if (!csvText) {
          throw new Error("文件读取失败，可能为空或编码不正确");
        }

        this.parseStatus = "正在解析数据...";

        // 使用PapaParse解析CSV
        const results = await this.parseWithPapaParse(csvText);

        // 处理解析结果
        this.handleParseResults(results);
      } catch (error) {
        this.handleParseError(error);
      } finally {
        this.isParsing = false;
        if (this.loadingInstance) {
          this.loadingInstance.close();
        }
      }
    },

    /**
     * 读取文件
     */
    readFileWithFallback(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();

        reader.onload = (event) => {
          try {
            let content = event.target.result;

            // 处理BOM头
            if (content.charCodeAt(0) === 0xfeff) {
              content = content.slice(1);
            }

            resolve(content);
          } catch (error) {
            reject(new Error(`文件读取错误: ${error.message}`));
          }
        };

        reader.onerror = () => {
          reject(new Error("文件读取失败，请检查文件权限或格式"));
        };

        reader.onabort = () => {
          reject(new Error("文件读取被中止"));
        };

        // 设置超时
        const timeout = setTimeout(() => {
          reader.abort();
          reject(new Error("文件读取超时"));
        }, 30000);

        reader.onloadend = () => {
          clearTimeout(timeout);
        };

        // 尝试不同的读取方式
        try {
          reader.readAsText(file, "UTF-8");
        } catch (error) {
          try {
            reader.readAsText(file);
          } catch (fallbackError) {
            reject(new Error(`无法读取文件: ${fallbackError.message}`));
          }
        }
      });
    },

    /**
     * 使用PapaParse解析
     */
    parseWithPapaParse(csvText) {
      return new Promise((resolve, reject) => {
        try {
          Papa.parse(csvText, {
            ...this.parseConfig,
            complete: (results) => {
              resolve(results);
            },
            error: (error) => {
              reject(new Error(`CSV解析错误: ${error.message}`));
            },
          });
        } catch (error) {
          reject(new Error(`解析过程异常: ${error.message}`));
        }
      });
    },

    /**
     * 处理解析结果
     */
    handleParseResults(results) {
      if (!results || !results.data) {
        throw new Error("解析结果为空");
      }

      const rawData = results.data;

      if (rawData.length === 0) {
        throw new Error("CSV文件内容为空");
      }

      // 获取表头
      let headers = [];
      if (this.parseConfig.header && results.meta && results.meta.fields) {
        headers = results.meta.fields;
      } else if (rawData[0]) {
        const firstRow = rawData[0];
        if (firstRow && typeof firstRow === "object") {
          headers = Object.keys(firstRow).map(function (_, i) {
            return "列" + (i + 1);
          });
        }
      }

      // 过滤无效数据
      const validData = rawData.filter(function (row) {
        if (!row || Object.keys(row).length === 0) {
          return false;
        }

        return Object.values(row).some(function (value) {
          return (
            value != null &&
            value !== "" &&
            !(typeof value === "number" && isNaN(value))
          );
        });
      });

      if (validData.length === 0) {
        throw new Error("没有有效的数据行");
      }

      // 添加解析成功提示
      this.$message.success(`解析成功，${validData.length} 行数据`);

      // 发送解析完成事件
      this.$emit("data-loaded", {
        data: validData,
        headers: headers,
      });

      this.parseStatus = "解析完成！";

      // 自动重置状态
      setTimeout(() => {
        this.parseStatus = "";
      }, 2000);
    },

    /**
     * 处理解析错误
     */
    handleParseError(error) {
      let errorMessage = error.message || "未知错误";
      let errorType = "解析错误";

      this.parseError = `${errorType}: ${errorMessage}`;
      this.parseErrorDetails = this.getErrorDetails();
      this.errorSolutions = this.getErrorSolutions(errorType);

      console.error("CSV解析失败:", error);
      this.$message.error(`解析失败: ${errorMessage}`);
    },

    /**
     * 清除文件
     */
    clearFile() {
      this.currentFile = null;
      this.isParsing = false;
      this.parseStatus = "";
      this.clearError();
      this.$refs.fileInput.value = "";
    },
  },
};
</script>

<style scoped>
/* 上传卡片样式 */
.uploader-card {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 12px;
  border: 1px solid #ebeef5;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  margin-bottom: 20px;
}

.uploader-card:hover {
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.15);
  transform: translateY(-2px);
}

/* 卡片头部样式 */
.card-header {
  font-weight: 600;
  color: #303133;
  font-size: 1.2rem;
  display: flex;
  align-items: center;
  gap: 10px;
}

.card-header::before {
  content: "";
  display: inline-block;
  width: 4px;
  height: 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 2px;
}

/* 上传区域样式 */
.upload-area {
  border: 2px dashed #409eff;
  border-radius: 10px;
  padding: 50px 20px;
  margin: 20px 0;
  text-align: center;
  cursor: pointer;
  background: linear-gradient(
    135deg,
    rgba(64, 158, 255, 0.05) 0%,
    rgba(102, 126, 234, 0.05) 100%
  );
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
}

.upload-area::before {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(
    135deg,
    rgba(64, 158, 255, 0.1) 0%,
    rgba(102, 126, 234, 0.1) 100%
  );
  opacity: 0;
  transition: opacity 0.3s ease;
}

.upload-area:hover::before,
.upload-area.dragging::before {
  opacity: 1;
}

.upload-area:hover,
.upload-area.dragging {
  border-color: #1e9fff;
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(64, 158, 255, 0.2);
}

.upload-content {
  position: relative;
  z-index: 1;
}

.upload-text {
  font-size: 1.2rem;
  margin: 20px 0 10px;
  color: #409eff;
  font-weight: 500;
  transition: all 0.3s ease;
}

.upload-area:hover .upload-text {
  color: #1e9fff;
  transform: scale(1.05);
}

.upload-hint {
  font-size: 0.9rem;
  color: #909399;
  line-height: 1.5;
}

/* 操作按钮区域样式 */
.action-section {
  margin-top: 20px;
  padding: 20px;
  background: #f8f9fa;
  border-radius: 8px;
  border: 1px solid #e9ecef;
}

.action-buttons {
  display: flex;
  gap: 12px;
  justify-content: center;
}

.action-buttons .el-button {
  min-width: 120px;
  padding: 12px 20px;
  font-weight: 500;
  border-radius: 8px;
  transition: all 0.3s ease;
}

.action-buttons .el-button--primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border: none;
}

.action-buttons .el-button--primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 20px rgba(102, 126, 234, 0.4);
}

.action-buttons .el-button:not(.el-button--primary):hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

/* 文件信息样式 */
.file-info {
  margin-top: 20px;
  animation: slideIn 0.5s ease;
}

/* 解析状态样式 */
.parse-status {
  margin-top: 20px;
  animation: slideIn 0.5s ease;
}

.progress-indicator {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 10px;
  padding: 10px;
  background: #f0f9ff;
  border-radius: 6px;
  color: #409eff;
  font-size: 0.9rem;
  animation: pulse 2s infinite;
}

/* 错误信息样式 */
.error-message {
  margin-top: 20px;
  animation: slideIn 0.5s ease;
}

/* 浏览器警告样式 */
.browser-warning {
  margin-top: 20px;
  animation: slideIn 0.5s ease;
}

/* 动画效果 */
@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes pulse {
  0% {
    opacity: 0.6;
  }
  50% {
    opacity: 1;
  }
  100% {
    opacity: 0.6;
  }
}

/* 响应式样式 */
@media (max-width: 768px) {
  .upload-area {
    padding: 30px 15px;
  }

  .upload-text {
    font-size: 1.1rem;
  }

  .action-buttons {
    flex-direction: column;
  }

  .action-buttons .el-button {
    width: 100%;
  }
}
</style>
