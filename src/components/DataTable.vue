<template>
  <!-- 数据表格组件主容器 -->
  <div class="data-table">
    <!-- 控制栏：包含日期选择器、标签搜索、复原按钮和表格信息 -->
    <div class="table-controls">
      <div class="controls-left">
        <!-- 日期区间选择器 -->
        <div class="date-range-container">
          <el-date-picker
            v-model="dateRange"
            type="daterange"
            range-separator="至"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
            format="yyyy-MM-dd"
            value-format="yyyy-MM-dd"
            @change="handleDateChange"
            :disabled="isDateRangeInvalid"
            class="custom-date-picker"
          ></el-date-picker>
        </div>
        <!-- 国家/落地媒体搜索区域 - 使用Element UI输入框+前置选择器 -->
        <el-input
          v-model="searchKeyword"
          placeholder="请输入搜索内容"
          class="category-search-input input-with-select"
          size="medium"
        >
          <el-select
            v-model="searchType"
            slot="prepend"
            size="medium"
            class="search-type-select"
          >
            <el-option label="国家" value="国家"></el-option>
            <el-option label="落地媒体" value="落地媒体"></el-option>
          </el-select>
        </el-input>
        <!-- 标签搜索区域 -->
        <div class="tag-search-container">
          <el-input
            v-model="tagExpression"
            placeholder="请输入标签，逻辑运算符两侧用空格隔开"
            class="tag-input"
            :class="{ 'expression-error': expressionError }"
          ></el-input>
          <el-tooltip placement="top">
            <div
              slot="content"
              style="white-space: pre-line; text-align: left; line-height: 1.5"
            >
              支持的运算符：AND(和)、OR(或)、NOT(非/不包括)<br />
              可使用括号改变优先级，运算符与括号中文和英文大小写均可<br />
              请注意运算符两侧需用空格隔开<br />
              例1：(标签1 AND 标签2) OR NOT 标签3<br />
              例2：（标签1 和 标签2） 或 不包括 标签3<br />
              未加括号的情况下三种运算符处于同一优先级，从左到右运算<br />
              在描述复杂逻辑时建议使用括号，例如：(标签1 AND 标签2) OR (标签3
              AND 标签4)
            </div>
            <el-button
              icon="el-icon-question"
              size="small"
              type="text"
              class="expression-help-btn"
            ></el-button>
          </el-tooltip>
          <div v-if="expressionError" class="expression-error-message">
            {{ expressionError }}
          </div>
        </div>
      </div>
      <div class="controls-right">
        <div class="action-buttons">
          <el-button
            type="primary"
            size="medium"
            @click="searchByDateRange"
            icon="el-icon-search"
          >
            检索
          </el-button>
          <el-button
            type="success"
            size="medium"
            @click="showStatistics"
            icon="el-icon-data-analysis"
          >
            统计
          </el-button>
          <el-button
            type="info"
            size="medium"
            @click="resetTable"
            icon="el-icon-refresh"
          >
            复原表格
          </el-button>
          <el-button
            type="primary"
            size="medium"
            plain
            @click="toggleFullscreen"
            icon="el-icon-full-screen"
          >
            展开全屏
          </el-button>
        </div>
      </div>
    </div>

    <!-- 表格容器：包含表格组件，支持全屏模式 -->
    <div
      class="table-container"
      :class="{ fullscreen: isFullscreen }"
      ref="tableContainer"
    >
      <!-- 表格组件：使用Element UI的el-table实现数据展示 -->
      <el-table
        ref="dataTable"
        :data="filteredData"
        border
        :striped="false"
        :height="tableHeight"
        empty-text="暂无数据"
        style="width: 100%"
        :max-height="isFullscreen ? 'calc(100vh - 100px)' : tableHeight"
        :row-class-name="tableRowClassName"
        :cell-class-name="tableCellClassName"
        :class="{ 'no-data': filteredData.length === 0 }"
        v-loading="loading"
      >
        <!-- 自定义空状态显示 -->
        <template #empty>
          <el-empty description="暂无数据"></el-empty>
        </template>
        <!-- 动态生成列：根据headers属性动态渲染表格列 -->
        <el-table-column
          v-for="header in headers"
          :key="header"
          :prop="header"
          :label="header"
          :min-width="getColumnWidth(header)"
        >
          <template #default="{ row, $index, column }">
            <div
              class="cell-container"
              @mouseenter="
                handleCellHover($index, column.property, row[column.property])
              "
            >
              <!-- 单元格内容容器：处理长内容显示和样式 -->
              <div
                class="cell-content"
                :class="{
                  'is-long': isLongContent(row[column.property]),
                  'fullscreen-expanded':
                    isFullscreen && isLongContent(row[column.property]),
                }"
              >
                <span class="cell-text">
                  {{
                    formatCellValue(row[column.property], row, column.property)
                  }}
                </span>
              </div>

              <!-- 单元格操作按钮：复制内容和查看详情 -->
              <div
                class="cell-actions"
                v-if="
                  row[column.property] != null && row[column.property] !== ''
                "
              >
                <el-tooltip
                  content="复制内容"
                  placement="top"
                  effect="light"
                  :disabled="isFullscreen"
                >
                  <el-button
                    type="text"
                    size="mini"
                    icon="el-icon-document-copy"
                    @click.stop="copyCellContent(row[column.property])"
                    class="cell-action-btn"
                  />
                </el-tooltip>
                <!-- 使用v-if控制查看详情按钮在非全屏模式下显示 -->
                <el-tooltip
                  content="查看详情"
                  placement="top"
                  effect="light"
                  v-if="!isFullscreen"
                >
                  <el-button
                    type="text"
                    size="mini"
                    icon="el-icon-view"
                    @click.stop="
                      showCellDetail(row[column.property], header, $index + 1)
                    "
                    class="cell-action-btn"
                  />
                </el-tooltip>
              </div>
            </div>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <!-- 统计结果对话框 -->
    <el-dialog
      title="数据统计"
      :visible.sync="statisticsDialogVisible"
      width="90%"
      max-width="1200px"
      center
      :close-on-click-modal="false"
      :modal-append-to-body="false"
      :append-to-body="true"
      custom-class="statistics-dialog"
      :modal="true"
      :lock-scroll="false"
      @open="handleStatisticsDialogOpen"
    >
      <div class="statistics-content">
        <el-table
          :data="statisticsData"
          border
          style="width: 100%"
          size="small"
          :show-header="true"
        >
          <el-table-column
            prop="articleCount"
            label="总篇数"
            width="120"
          ></el-table-column>
          <el-table-column
            prop="countryCount"
            label="国家数"
            width="120"
          ></el-table-column>
          <el-table-column
            prop="mediaCount"
            label="媒体数"
            width="120"
          ></el-table-column>
          <el-table-column
            prop="totalViews"
            label="总落地次数"
            width="150"
          ></el-table-column>
          <el-table-column prop="mediaDetails" label="媒体详情" min-width="180">
            <template #default="{ row }">
              <div class="cell-container">
                <div class="cell-content">
                  <span class="cell-text">
                    {{ row.mediaDetails }}
                  </span>
                </div>
                <div class="cell-actions" v-if="row.mediaDetails">
                  <el-tooltip content="复制内容" placement="top" effect="light">
                    <el-button
                      type="text"
                      size="mini"
                      icon="el-icon-document-copy"
                      @click.stop="copyCellContent(row.mediaDetails)"
                      class="cell-action-btn"
                    />
                  </el-tooltip>
                  <el-tooltip
                    content="查看详情"
                    placement="top"
                    effect="light"
                    v-if="!isFullscreen"
                  >
                    <el-button
                      type="text"
                      size="mini"
                      icon="el-icon-view"
                      @click.stop="
                        handleStatisticCellHover(row.mediaDetails, '媒体详情')
                      "
                      class="cell-action-btn"
                    />
                  </el-tooltip>
                </div>
              </div>
            </template>
          </el-table-column>
          <!-- 署名作者+文章名列 -->
          <el-table-column
            v-if="hasSignatureArticleTag"
            prop="signatureInfo"
            label="署名作者+文章名"
            min-width="200"
          >
            <template #default="{ row }">
              <div class="cell-container">
                <div class="cell-content">
                  <span class="cell-text">
                    {{ row.signatureInfo || "暂无数据" }}
                  </span>
                </div>
                <!-- 添加单元格操作按钮 -->
                <div class="cell-actions" v-if="row.signatureInfo">
                  <el-tooltip content="复制内容" placement="top" effect="light">
                    <el-button
                      type="text"
                      size="mini"
                      icon="el-icon-document-copy"
                      @click.stop="copyCellContent(row.signatureInfo)"
                      class="cell-action-btn"
                    />
                  </el-tooltip>
                  <el-tooltip
                    content="查看详情"
                    placement="top"
                    effect="light"
                    v-if="!isFullscreen"
                  >
                    <el-button
                      type="text"
                      size="mini"
                      icon="el-icon-view"
                      @click.stop="
                        handleStatisticCellHover(
                          row.signatureInfo,
                          '署名作者+文章名'
                        )
                      "
                      class="cell-action-btn"
                    />
                  </el-tooltip>
                </div>
              </div>
            </template>
          </el-table-column>
        </el-table>
      </div>
    </el-dialog>
    <!-- 单元格详情对话框：展示单元格详细内容 -->
    <el-dialog
      title="单元格详情"
      :visible.sync="detailDialogVisible"
      width="700px"
      center
      :close-on-click-modal="false"
      :modal-append-to-body="false"
      :append-to-body="true"
      custom-class="cell-detail-dialog"
      :class="{ 'fullscreen-dialog': isFullscreen }"
      :modal="true"
      :lock-scroll="false"
    >
      <div class="cell-detail-content">
        <!-- 信息卡片：显示单元格位置信息 -->
        <div class="info-card">
          <div class="info-header">
            <i class="el-icon-info"></i>
            <span class="info-title">信息</span>
          </div>
          <div class="info-content">
            <div class="info-row">
              <span class="info-value">
                <!-- 根据来源决定是否显示行和序号信息 -->
                <template v-if="!isFromStatistics">
                  <span class="info-item row-item">第{{ detailRow }}行</span>
                  <span class="info-separator">；</span>
                  <span class="info-item sequence-item"
                    >序号{{ detailSequenceValue }}</span
                  >
                  <span class="info-separator">；</span>
                </template>
                <span class="info-item column-item">“{{ detailColumn }}”</span>
              </span>
            </div>
          </div>
        </div>
        <!-- 内容卡片：显示单元格详细内容 -->
        <div class="content-card">
          <div class="content-header">
            <i class="el-icon-document"></i>
            <span class="content-title">内容</span>
          </div>
          <div class="content-body">
            <div class="content-text" ref="detailContent">
              {{ detailContent }}
            </div>
          </div>
        </div>
      </div>

      <template #footer>
        <el-button @click="detailDialogVisible = false">关闭</el-button>
        <el-button type="primary" @click="copyDetailContent"
          >复制内容</el-button
        >
      </template>
    </el-dialog>

    <!-- 右下角控制按钮组：包含滚动到顶部、底部和退出全屏按钮 -->
    <div class="control-buttons">
      <el-tooltip
        content="滚动到顶部"
        placement="left"
        effect="dark"
        :open-delay="300"
      >
        <el-button
          circle
          size="medium"
          @click="scrollToTop"
          class="control-button scroll-button"
        >
          <i class="el-icon-top"></i>
        </el-button>
      </el-tooltip>

      <el-tooltip
        content="滚动到底部"
        placement="left"
        effect="dark"
        :open-delay="300"
      >
        <el-button
          circle
          size="medium"
          @click="scrollToBottom"
          class="control-button scroll-button"
        >
          <i class="el-icon-bottom"></i>
        </el-button>
      </el-tooltip>

      <el-tooltip
        v-if="isFullscreen"
        content="退出全屏"
        placement="left"
        effect="dark"
        :open-delay="300"
      >
        <el-button
          circle
          size="medium"
          @click="toggleFullscreen"
          class="control-button exit-fullscreen-button"
        >
          <i class="el-icon-close"></i>
        </el-button>
      </el-tooltip>
    </div>
  </div>
</template>

<script>
export default {
  name: "DataTable", // 组件名称

  // 组件属性
  props: {
    data: {
      // 表格原始数据
      type: Array,
      default: function () {
        return [];
      },
    },
  },

  // 组件状态数据
  data() {
    return {
      headers: [
        "序号",
        "稿件标题",
        "标签",
        "单条总结/主题总结放最新一条",
        "日期",
        "国家",
        "落地媒体",
        "落地平台",
        "落地次数",
        "链接",
        "截图",
        "其他",
        "其他_1",
      ],
      filteredData: [], // 筛选后的数据
      allData: [], // 所有数据（用于搜索筛选）
      originalData: [], // 原始数据备份
      tableHeight: 600, // 表格高度
      isFullscreen: false, // 是否全屏显示
      dateRange: [], // 日期范围选择
      isDateRangeInvalid: false, // 日期范围是否无效,
      tagExpression: "", // 标签搜索表达式
      expressionError: "", // 表达式错误信息
      searchType: "", // 搜索类型（国家/落地媒体）
      searchKeyword: "", // 搜索关键词
      statisticsDialogVisible: false, // 统计对话框显示状态
      statisticsData: [], // 统计数据
      isFromStatistics: false, // 标识是否从统计面板打开详情
      detailDialogVisible: false, // 详情对话框显示状态
      detailContent: "", // 详情内容
      detailColumn: "", // 详情列名
      detailRow: 0, // 详情行号
      detailSequenceValue: "", // 详情序号值
      sequenceGroups: null, // 用于存储序号分组信息
      currentGroupId: 0, // 当前分组ID
      currentSequenceValue: null, // 当前分组的序号值
      hasSignatureArticleTag: false, // 标记是否包含"署名文章"标签
      loading: false, // 添加加载状态
    };
  },

  // 监听器：监听数据变化
  watch: {
    data: {
      immediate: true, // 立即执行
      handler: function (newData) {
        // 处理数据：填充序号列空白值
        const processedData = this.fillSequenceNumbers(newData);
        // 过滤空列
        const dataWithFilteredColumns = this.filterEmptyColumns(processedData);
        this.originalData = dataWithFilteredColumns;
        this.allData = dataWithFilteredColumns;
        this.filteredData = dataWithFilteredColumns;
        // 计算表格高度
        this.$nextTick(function () {
          this.calculateTableHeight();
        });
      },
    },
  },

  // 生命周期钩子：组件挂载后
  mounted() {
    // 添加小延迟确保父容器已完全渲染
    setTimeout(() => {
      this.calculateTableHeight();
    }, 100);
    window.addEventListener("resize", this.calculateTableHeight);
  },

  // 生命周期钩子：组件销毁前
  beforeDestroy() {
    window.removeEventListener("resize", this.calculateTableHeight); // 移除窗口大小变化监听
  },

  methods: {
    /**
     * 过滤空列（除表头外完全没有内容的列）
     * @param {Array} data - 原始数据
     * @returns {Array} 过滤后的数据
     */
    filterEmptyColumns(data) {
      if (!data || data.length === 0) return [];

      // 复制一份数据
      const newData = JSON.parse(JSON.stringify(data));
      const columnsToRemove = [];

      // 检查每一列是否为空
      this.headers.forEach((header) => {
        let columnHasContent = false;

        // 检查该列是否有内容
        for (let i = 0; i < newData.length; i++) {
          const value = newData[i][header];
          if (value !== null && value !== undefined && value !== "") {
            columnHasContent = true;
            break;
          }
        }

        // 如果列没有内容，标记为要删除
        if (!columnHasContent) {
          columnsToRemove.push(header);
        }
      });

      // 更新表头，移除空列
      this.headers = this.headers.filter(
        (header) => !columnsToRemove.includes(header)
      );

      // 从数据中移除空列
      newData.forEach((row) => {
        columnsToRemove.forEach((column) => {
          delete row[column];
        });
      });

      return newData;
    },

    /**
     * 处理日期范围变化
     */
    handleDateChange() {
      if (this.dateRange && this.dateRange.length === 2) {
        const startDate = new Date(this.dateRange[0]);
        const endDate = new Date(this.dateRange[1]);
        this.isDateRangeInvalid = endDate < startDate;

        if (this.isDateRangeInvalid) {
          this.$message.warning({
            message: "结束日期不能早于开始日期",
            center: true,
          });
        }
      }
    },

    /**
     * 解析日期字符串
     * @param {string} dateStr - 日期字符串，如"2025年1月2日"或"1月2日"
     * @returns {Date} 解析后的日期对象
     */
    parseDate(dateStr) {
      if (!dateStr) return null;

      // 检查是否包含年份
      const hasYear = /\d{4}年/.test(dateStr);

      // 提取年、月、日
      let year, month, day;

      if (hasYear) {
        // 带年份的格式：YYYY年MM月DD日
        const match = dateStr.match(/(\d{4})年(\d{1,2})月(\d{1,2})日/);
        if (!match) return null;

        year = parseInt(match[1]);
        month = parseInt(match[2]) - 1; // 月份从0开始
        day = parseInt(match[3]);
      } else {
        // 不带年份的格式：MM月DD日，默认使用当前年份
        const match = dateStr.match(/(\d{1,2})月(\d{1,2})日/);
        if (!match) return null;

        year = new Date().getFullYear();
        month = parseInt(match[1]) - 1; // 月份从0开始
        day = parseInt(match[2]);
      }

      return new Date(year, month, day);
    },

    /**
     * 解析标签字符串，支持中英文分号分隔
     * @param {string} tagStr - 标签字符串
     * @returns {Array} 标签数组
     */
    parseTags(tagStr) {
      if (!tagStr) return [];
      // 使用正则表达式分割中英文分号，并过滤空标签
      return tagStr
        .split(/[；;]/)
        .map((tag) => tag.trim())
        .filter((tag) => tag);
    },

    /**
     * 词法分析器：将表达式字符串转换为标记流
     * @param {string} expression - 搜索表达式
     * @returns {Array} 标记数组
     */
    tokenizeExpression(expression) {
      const tokens = [];
      let currentToken = "";
      let i = 0;
      const len = expression.length;

      // 支持的运算符映射
      const operatorMap = {
        AND: "AND",
        OR: "OR",
        NOT: "NOT",
        和: "AND",
        或: "OR",
        非: "NOT",
        不包括: "NOT",
      };

      // 运算符列表，用于判断完整单词匹配
      const operators = new Set([
        "AND",
        "OR",
        "NOT",
        "和",
        "或",
        "非",
        "不包括",
      ]);

      while (i < len) {
        const char = expression[i];

        // 跳过空格
        if (/\s/.test(char)) {
          if (currentToken) {
            // 先检查是否为完整运算符单词
            if (operators.has(currentToken)) {
              tokens.push({
                type: "OPERATOR",
                value: operatorMap[currentToken],
              });
            } else {
              tokens.push({ type: "TAG", value: currentToken });
            }
            currentToken = "";
          }
          i++;
          continue;
        }

        // 处理括号（支持中英文括号）
        if (char === "(" || char === ")" || char === "（" || char === "）") {
          if (currentToken) {
            if (operators.has(currentToken)) {
              tokens.push({
                type: "OPERATOR",
                value: operatorMap[currentToken],
              });
            } else {
              tokens.push({ type: "TAG", value: currentToken });
            }
            currentToken = "";
          }
          // 统一转换为英文括号以便后续处理
          const normalizedParen =
            char === "（" ? "(" : char === "）" ? ")" : char;
          tokens.push({ type: "PAREN", value: normalizedParen });
          i++;
          continue;
        }

        // 处理其他字符
        currentToken += char;
        i++;
      }

      // 处理最后一个token
      if (currentToken) {
        if (operators.has(currentToken)) {
          tokens.push({ type: "OPERATOR", value: operatorMap[currentToken] });
        } else {
          tokens.push({ type: "TAG", value: currentToken });
        }
      }

      // 在NOT运算符前自动添加AND（如果前面是标签或右括号）
      for (let i = 0; i < tokens.length; i++) {
        if (tokens[i].type === "OPERATOR" && tokens[i].value === "NOT") {
          if (
            i > 0 &&
            (tokens[i - 1].type === "TAG" || tokens[i - 1].value === ")")
          ) {
            tokens.splice(i, 0, { type: "OPERATOR", value: "AND" });
            i++; // 跳过新插入的AND
          }
        }
      }

      return tokens;
    },

    /**
     * 语法分析器：将标记流转换为抽象语法树(AST)
     * @param {Array} tokens - 标记数组
     * @returns {Object} 抽象语法树
     */
    parseExpression(tokens) {
      let index = 0;

      // 解析表达式
      function parseExpression() {
        let node = parseTerm();

        while (
          index < tokens.length &&
          tokens[index].type === "OPERATOR" &&
          tokens[index].value === "OR"
        ) {
          const operator = tokens[index++];
          const right = parseTerm();
          node = {
            type: "OPERATOR",
            value: operator.value,
            left: node,
            right: right,
          };
        }

        return node;
      }

      // 解析项（处理AND运算符）
      function parseTerm() {
        let node = parseFactor();

        // 只处理AND运算符，NOT在parseFactor中处理
        while (
          index < tokens.length &&
          tokens[index].type === "OPERATOR" &&
          tokens[index].value === "AND"
        ) {
          const operator = tokens[index++];
          const right = parseFactor();
          node = {
            type: "OPERATOR",
            value: operator.value,
            left: node,
            right: right,
          };
        }

        return node;
      }

      // 解析因子（处理NOT运算符和括号）
      function parseFactor() {
        if (index >= tokens.length) {
          throw new Error("表达式不完整");
        }

        const token = tokens[index];

        if (token.type === "OPERATOR" && token.value === "NOT") {
          index++;
          const operand = parseFactor();
          return { type: "OPERATOR", value: "NOT", right: operand };
        } else if (token.type === "PAREN" && token.value === "(") {
          index++; // 跳过左括号
          const expr = parseExpression();
          if (
            index >= tokens.length ||
            tokens[index].type !== "PAREN" ||
            tokens[index].value !== ")"
          ) {
            throw new Error("缺少右括号");
          }
          index++; // 跳过右括号
          return expr;
        } else if (token.type === "TAG") {
          index++;
          return { type: "TAG", value: token.value };
        } else {
          throw new Error(`意外的标记: ${token.value}`);
        }
      }

      const ast = parseExpression();

      // 检查是否有剩余标记
      if (index < tokens.length) {
        throw new Error(`意外的标记: ${tokens[index].value}`);
      }

      return ast;
    },
    /**
     * 将AST转换为逻辑表达式字符串
     * @param {Object} ast - 抽象语法树
     * @returns {string} 逻辑表达式字符串
     */
    astToLogicalString(ast) {
      if (!ast) return "";

      switch (ast.type) {
        case "TAG":
          return ast.value;

        case "OPERATOR":
          switch (ast.value) {
            case "AND":
              return `(${this.astToLogicalString(
                ast.left
              )} && ${this.astToLogicalString(ast.right)})`;
            case "OR":
              return `(${this.astToLogicalString(
                ast.left
              )} || ${this.astToLogicalString(ast.right)})`;
            case "NOT":
              return `!(${this.astToLogicalString(ast.right)})`;
            default:
              return "";
          }

        default:
          return "";
      }
    },

    /**
     * 评估表达式AST
     * @param {Object} ast - 抽象语法树
     * @param {Set} rowTags - 行标签集合
     * @returns {boolean} 评估结果
     */
    evaluateExpression(ast, rowTags) {
      if (!ast) return true; // 空表达式匹配所有

      switch (ast.type) {
        case "TAG":
          // 标签匹配区分大小写，转为小写后比较
          const tagValue = ast.value.toLowerCase();
          return Array.from(rowTags).some(
            (tag) => tag.toLowerCase() === tagValue
          );

        case "OPERATOR":
          switch (ast.value) {
            case "AND":
              return (
                this.evaluateExpression(ast.left, rowTags) &&
                this.evaluateExpression(ast.right, rowTags)
              );
            case "OR":
              return (
                this.evaluateExpression(ast.left, rowTags) ||
                this.evaluateExpression(ast.right, rowTags)
              );
            case "NOT":
              return !this.evaluateExpression(ast.right, rowTags);
            default:
              throw new Error(`未知的运算符: ${ast.value}`);
          }

        default:
          throw new Error(`未知的节点类型: ${ast.type}`);
      }
    },

    /**
     * 根据标签表达式筛选数据
     * @param {Array} data - 原始数据
     * @param {string} expression - 标签搜索表达式
     * @returns {Array} 筛选后的数据
     */
    filterByTags(data, expression) {
      const trimmedExpr = expression.trim();
      if (!trimmedExpr) return data;

      try {
        this.expressionError = "";
        const tokens = this.tokenizeExpression(trimmedExpr);
        const ast = this.parseExpression(tokens);

        // 检查是否包含"署名文章"标签
        this.hasSignatureArticleTag = trimmedExpr
          .toLowerCase()
          .includes("署名文章");

        // 生成并输出逻辑表达式到控制台
        const logicalString = this.astToLogicalString(ast);
        console.log("标签搜索逻辑:", logicalString);

        // 按序号组分组
        const sequenceGroups = new Map();
        data.forEach((row) => {
          const groupId = row._sequenceGroup;
          if (!groupId) return;

          if (!sequenceGroups.has(groupId)) {
            sequenceGroups.set(groupId, []);
          }
          sequenceGroups.get(groupId).push(row);
        });

        // 2. 筛选符合条件的序号组
        const matchedGroups = [];
        sequenceGroups.forEach((groupRows) => {
          // 使用组内第一行的标签作为整个组的标签
          const firstRow = groupRows[0];
          if (!firstRow["标签"]) return;

          const groupTags = new Set(this.parseTags(firstRow["标签"]));
          if (this.evaluateExpression(ast, groupTags)) {
            matchedGroups.push(...groupRows);
          }
        });

        return matchedGroups;
      } catch (error) {
        console.error("标签表达式解析错误:", error);
        this.expressionError = `表达式错误: ${error.message}`;

        // 尝试简单标签匹配作为回退方案
        try {
          const tag = trimmedExpr;
          console.log("使用简单匹配模式搜索:", tag);

          // 回退方案也使用按组匹配
          const sequenceGroups = new Map();
          data.forEach((row) => {
            const groupId = row._sequenceGroup;
            if (!groupId) return;

            if (!sequenceGroups.has(groupId)) {
              sequenceGroups.set(groupId, []);
            }
            sequenceGroups.get(groupId).push(row);
          });

          const matchedGroups = [];
          sequenceGroups.forEach((groupRows) => {
            const firstRow = groupRows[0];
            if (!firstRow["标签"]) return;

            const rowTags = this.parseTags(firstRow["标签"]);
            if (
              rowTags.some((rTag) =>
                rTag.toLowerCase().includes(tag.toLowerCase())
              )
            ) {
              matchedGroups.push(...groupRows);
            }
          });

          return matchedGroups;
        } catch (fallbackError) {
          this.expressionError = `表达式错误: ${error.message}`;
          console.error("标签筛选错误:", error);
          return [];
        }
      }
    },

    /**
     * 检查AST中是否包含"署名文章"标签
     * @param {Object} ast - 抽象语法树
     * @returns {boolean} 是否包含"署名文章"标签
     */
    checkSignatureArticleTag(ast) {
      if (!ast) return false;

      if (ast.type === "TAG" && ast.value === "署名文章") {
        return true;
      }

      if (ast.type === "OPERATOR") {
        if (ast.left && this.checkSignatureArticleTag(ast.left)) {
          return true;
        }
        if (ast.right && this.checkSignatureArticleTag(ast.right)) {
          return true;
        }
      }

      return false;
    },

    /**
     * 根据日期范围和标签搜索数据
     */
    searchByDateRange() {
      // 先应用日期筛选
      let filtered = [...this.allData];
      if (this.dateRange && this.dateRange.length === 2) {
        const startDate = new Date(this.dateRange[0]);
        const endDate = new Date(this.dateRange[1]);

        // 显式设置时间部分，确保包含整个日期
        startDate.setHours(0, 0, 0, 0); // 起始日期从当天0点开始
        endDate.setHours(23, 59, 59, 999); // 结束日期到当天23:59:59结束

        if (endDate < startDate) {
          this.$message.warning({
            message: "结束日期不能早于开始日期",
            center: true,
          });
          return;
        }

        // 筛选日期在范围内的数据
        filtered = filtered.filter((row) => {
          if (!row["日期"]) return false;

          const rowDate = this.parseDate(row["日期"]);
          if (!rowDate) return false;

          return rowDate >= startDate && rowDate <= endDate;
        });
      }
      // 应用国家/落地媒体搜索筛选
      if (this.searchType && this.searchKeyword) {
        const keyword = this.searchKeyword.toLowerCase();
        filtered = filtered.filter((row) => {
          if (!row[this.searchType]) return false;
          return String(row[this.searchType]).toLowerCase().includes(keyword);
        });
      }
      // 再应用标签筛选
      filtered = this.filterByTags(filtered, this.tagExpression);

      this.filteredData = filtered;

      // 触发筛选数据变化事件
      this.$emit("filter-change", this.filteredData);

      // 为每个分组的第一行添加序号
      this.addSequenceToGroupFirstRows();

      // 重置分组信息，确保筛选后重新计算分组颜色
      this.sequenceGroups = null;
      this.currentGroupId = 0;

      // 如果没有筛选结果，显示提示信息
      if (this.filteredData.length === 0) {
        this.$message.info({
          message: "没有找到符合条件的数据",
          center: true,
        });
      } else {
        // 检索成功提示
        this.$message.success({
          message: "检索成功",
          center: true,
        });
      }
    },

    /**
     * 为每个分组的第一行添加序号
     */
    addSequenceToGroupFirstRows() {
      if (!this.filteredData || this.filteredData.length === 0) return;

      const firstHeader = this.headers[0] || "序号";
      let lastGroup = null;

      this.filteredData.forEach((row) => {
        const currentGroup = row._sequenceGroup;

        // 如果是新分组的第一行且序号为空，则添加分组序号
        if (
          currentGroup &&
          currentGroup !== lastGroup &&
          (!row[firstHeader] || row[firstHeader] === "")
        ) {
          row[firstHeader] = currentGroup;
          row[`${firstHeader}_isFiltered`] = true; // 标记为筛选时添加的序号
        }

        lastGroup = currentGroup;
      });
    },

    /**
     * 重置表格到初始状态
     */
    resetTable() {
      // 保存当前状态用于比较
      const currentState = {
        dateRange: [...this.dateRange],
        tagExpression: this.tagExpression,
        searchType: this.searchType,
        searchKeyword: this.searchKeyword,
        filteredDataLength: this.filteredData.length,
      };

      // 强制重置所有筛选条件
      this.dateRange = [];
      this.isDateRangeInvalid = false;
      this.tagExpression = "";
      this.expressionError = "";
      this.searchType = "";
      this.searchKeyword = "";
      // 重置署名文章标签状态
      this.hasSignatureArticleTag = false;
      // 强制恢复原始数据
      this.filteredData = JSON.parse(JSON.stringify(this.originalData));
      this.$emit("filter-change", this.filteredData);
      // 移除筛选时添加的序号
      const firstHeader = this.headers[0] || "序号";
      this.filteredData.forEach((row) => {
        if (row[`${firstHeader}_isFiltered`]) {
          row[firstHeader] = "";
          delete row[`${firstHeader}_isFiltered`];
        }
      });

      // 重置分组信息，确保重置后重新计算分组颜色
      this.sequenceGroups = null;
      this.currentGroupId = 0;

      // 使用$nextTick确保DOM更新后再显示消息，确保消息总是显示
      this.$nextTick(() => {
        this.$message.success({
          message: "已复原表格",
          center: true,
          zIndex: 99999, // 确保消息在最上层显示
        });
      });
    },

    /**
     * 填充序号列的空白值
     * @param {Array} data - 原始数据
     * @returns {Array} 处理后的新数据
     */
    fillSequenceNumbers: function (data) {
      if (
        !data ||
        data.length === 0 ||
        !this.headers ||
        this.headers.length === 0
      ) {
        return data || [];
      }

      const processedData = JSON.parse(JSON.stringify(data));
      const firstHeader = this.headers[0] || "序号";

      // 增强的序号列判断逻辑
      let isSequenceColumn = false;
      let sequenceType = null; // 'number' | 'string' | null
      let sequenceStart = null;
      let sequenceStep = 1;

      // 分析前20行数据来判断是否为序号列
      const sampleRows = processedData.slice(0, 20);
      const values = sampleRows
        .map((row) => row[firstHeader])
        .filter((value) => value != null && value !== "");

      if (values.length > 0) {
        // 检查是否为数字序列
        const numbers = values.map((v) => Number(v)).filter((n) => !isNaN(n));
        if (numbers.length > 0) {
          isSequenceColumn = true;
          sequenceType = "number";

          // 分析序列步长
          if (numbers.length >= 2) {
            sequenceStep = numbers[1] - numbers[0];
            let stepConsistent = true;
            for (let i = 2; i < numbers.length; i++) {
              if (numbers[i] - numbers[i - 1] !== sequenceStep) {
                stepConsistent = false;
                break;
              }
            }
            if (stepConsistent) {
              sequenceStart = numbers[0];
            }
          }
        } else if (values.length > 0) {
          // 检查是否为字符串序号
          const hasSequenceKeywords = values.some((v) => {
            const str = String(v).toLowerCase();
            return (
              str.includes("序号") || str.includes("编号") || str.includes("no")
            );
          });

          if (hasSequenceKeywords) {
            isSequenceColumn = true;
            sequenceType = "string";
          }
        }
      }

      if (!isSequenceColumn) {
        return processedData;
      }

      // 智能填充序号并添加分组标识
      let lastSequenceValue = null;
      let lastNumberValue = null;
      let currentGroup = null; // 当前分组标识，用于记录当前行属于哪个序号组

      processedData.forEach((row, index) => {
        const currentValue = row[firstHeader];

        if (currentValue != null && currentValue !== "") {
          // 遇到新的有效序号值，开始新分组
          lastSequenceValue = currentValue;
          currentGroup = currentValue; // 设置当前分组为这个新序号值
          row[`${firstHeader}_isOriginal`] = true;

          // 如果是数字序号，记录数值用于后续填充
          if (sequenceType === "number") {
            lastNumberValue = Number(currentValue);
          }
        } else if (lastSequenceValue != null) {
          // 空白值，继承上一个有效序号的分组，但不填充序号值
          row[`${firstHeader}_isFilled`] = true;
        }

        // 为每一行添加分组标识，即使没有序号列也有分组
        row._sequenceGroup = currentGroup;
      });

      return processedData;
    },

    /**
     * 表格行样式类名
     * @param {Object} param0 - 行信息
     * @returns {string} 样式类名
     */
    tableRowClassName: function ({ row }) {
      // 获取当前行的分组标识
      const sequenceGroup = row._sequenceGroup;

      // 初始化分组映射
      if (!this.sequenceGroups) {
        this.sequenceGroups = new Map();
        this.currentGroupId = 0;
      }

      // 为新分组创建分组ID（仅当分组标识存在时）
      if (
        sequenceGroup !== undefined &&
        sequenceGroup !== null &&
        !this.sequenceGroups.has(sequenceGroup)
      ) {
        this.sequenceGroups.set(sequenceGroup, this.currentGroupId);
        this.currentGroupId++;
      }

      // 根据分组ID返回交替样式类名
      const groupId = this.sequenceGroups.get(sequenceGroup);
      return groupId % 2 === 0 ? "group-a-row" : "group-b-row";
    },

    /**
     * 计算表格高度
     */
    calculateTableHeight: function () {
      // 使用$nextTick确保DOM已完全渲染
      this.$nextTick(() => {
        if (this.isFullscreen) {
          this.tableHeight = window.innerHeight - 100;
        } else {
          // 调整高度计算，使用父容器高度
          const container = this.$parent.$el.querySelector(".data-container");
          if (container) {
            // 获取控制栏实际高度
            const controls = this.$el.querySelector(".table-controls");
            const controlsHeight = controls ? controls.offsetHeight : 0;

            // 从父容器获取可用高度，减去控制栏高度和内边距
            const containerHeight = container.clientHeight;
            // 调整偏移量计算方式，使用实际控制栏高度 + 小的边距
            this.tableHeight = containerHeight - controlsHeight - 20;
          } else {
            // 回退方案
            this.tableHeight = window.innerHeight - 300;
          }

          // 最小高度限制
          if (this.tableHeight < 200) {
            this.tableHeight = 200;
          }
        }

        // 强制表格重新布局
        this.forceTableUpdate();
      });
    },

    /**
     * 判断内容是否过长
     * @param {*} value - 单元格值
     * @returns {boolean} 是否过长
     */
    isLongContent: function (value) {
      if (value == null || value === "") return false;
      const str = String(value);
      return str.length > 30; // 超过30个字符视为长内容
    },

    /**
     * 处理单元格悬停事件
     * @param {number} rowIndex - 行索引
     * @param {string} column - 列名
     * @param {*} value - 单元格值
     */
    handleCellHover: function (rowIndex, column, value) {
      // 单元格悬停处理
    },

    /**
     * 处理统计面板单元格悬停事件
     * @param {*} content - 单元格内容
     * @param {string} column - 列名
     */
    handleStatisticCellHover: function (content, column) {
      // 使用现有的详情对话框显示统计面板单元格内容
      this.detailContent = content != null ? String(content) : "";
      this.detailColumn = column;
      this.detailRow = "统计面板";
      this.detailSequenceValue = "";
      this.isFromStatistics = true; // 设置为从统计面板打开
      this.detailDialogVisible = true;

      // 全屏模式层级
      this.$nextTick(() => {
        this.fixDialogZIndex();
      });
    },

    /**
     * 处理统计对话框打开事件，确保垂直居中
     */
    handleStatisticsDialogOpen() {
      this.$nextTick(() => {
        const dialog = document.querySelector(".statistics-dialog .el-dialog");
        if (dialog) {
          // 强制设置垂直居中
          dialog.style.position = "fixed";
          dialog.style.top = "50%";
          dialog.style.left = "50%";
          dialog.style.transform = "translate(-50%, -50%)";
          dialog.style.margin = "0";
        }
      });
    },

    /**
     * 复制单元格内容到剪贴板
     * @param {*} content - 要复制的内容
     */
    copyCellContent: function (content) {
      if (content == null) return;

      const text = String(content);
      this.copyToClipboard(text);
      this.$message.success({
        message: "已复制到剪贴板",
        center: true,
      });
    },

    /**
     * 显示单元格详情对话框
     * @param {*} content - 单元格内容
     * @param {string} column - 列名
     * @param {number} row - 行号
     */
    showCellDetail: function (content, column, row) {
      // 全屏模式下不显示详情对话框
      if (this.isFullscreen) return;
      this.detailContent = content != null ? String(content) : "";
      this.detailColumn = column;
      this.detailRow = row;
      this.isFromStatistics = false; // 设置为从正常表格打开

      // 获取序号值
      if (
        this.filteredData &&
        this.filteredData.length > 0 &&
        this.headers &&
        this.headers.length > 0
      ) {
        const firstHeader = this.headers[0];
        const rowData = this.filteredData[row - 1];
        if (rowData && firstHeader) {
          // 显示序号组信息
          this.detailSequenceValue =
            rowData._sequenceGroup || rowData[firstHeader] || "";
        } else {
          this.detailSequenceValue = "";
        }
      } else {
        this.detailSequenceValue = "";
      }

      this.detailDialogVisible = true;

      // 全屏模式层级
      this.$nextTick(() => {
        this.fixDialogZIndex();
      });
    },

    /**
     * 复制详情对话框内容
     */
    copyDetailContent: function () {
      this.copyToClipboard(this.detailContent);
      this.$message.success({
        message: "已复制到剪贴板",
        center: true,
      });
    },

    /**
     * 复制文本到剪贴板
     * @param {string} text - 要复制的文本
     */
    copyToClipboard: function (text) {
      const textarea = document.createElement("textarea");
      textarea.value = text;
      textarea.style.position = "fixed";
      textarea.style.opacity = "0";
      document.body.appendChild(textarea);
      textarea.select();

      try {
        document.execCommand("copy");
      } catch (err) {
        console.error("复制失败:", err);
        this.$message.error({
          message: "复制失败，请手动复制",
          center: true,
        });
      } finally {
        document.body.removeChild(textarea);
      }
    },

    /**
     * 表格单元格样式类名
     * @param {Object} param0 - 单元格信息
     * @returns {string} 样式类名
     */
    tableCellClassName: function ({ row, column, rowIndex, columnIndex }) {
      if (this.isLongContent(row[column.property])) {
        return "long-content-cell"; // 长内容单元格样式
      }
      return "";
    },

    /**
     * 强制表格重新布局
     */
    forceTableUpdate: function () {
      this.$refs.dataTable &&
        this.$refs.dataTable.doLayout &&
        this.$refs.dataTable.doLayout();
    },

    /**
     * 滚动到表格顶部
     */
    scrollToTop: function () {
      const tableRef = this.$refs.dataTable;
      if (tableRef && tableRef.$el) {
        const tableWrapper = tableRef.$el.querySelector(
          ".el-table__body-wrapper"
        );
        if (tableWrapper) {
          tableWrapper.scrollTop = 0;
        }
      }
    },

    /**
     * 滚动到表格底部
     */
    scrollToBottom: function () {
      const tableRef = this.$refs.dataTable;
      if (tableRef && tableRef.$el) {
        const tableWrapper = tableRef.$el.querySelector(
          ".el-table__body-wrapper"
        );
        if (tableWrapper) {
          tableWrapper.scrollTop = tableWrapper.scrollHeight;
        }
      }
    },

    /**
     * 切换全屏模式
     */
    toggleFullscreen: function () {
      this.isFullscreen = !this.isFullscreen;
      this.calculateTableHeight();

      // 控制body滚动
      if (this.isFullscreen) {
        document.body.style.overflow = "hidden";
      } else {
        document.body.style.overflow = "";
      }

      // 强制表格重新布局
      this.$nextTick(function () {
        this.forceTableUpdate();
      });
    },

    /**
     * 获取列宽度
     * @param {string} header - 列头
     * @returns {number} 列宽度
     */
    getColumnWidth: function (header) {
      if (!this.allData || this.allData.length === 0) return 250;

      // 取前100行数据计算最大内容长度
      const sampleValues = this.allData.slice(0, 100).map(function (row) {
        const value = row[header];
        if (value === null || value === undefined) return "";
        return String(value);
      });

      // 计算最大长度（考虑列头和内容）
      const maxLength = Math.max(
        header.length,
        ...sampleValues.map(function (str) {
          return str.length;
        })
      );

      // 计算最小宽度
      const minWidth = Math.max(250, Math.min(maxLength * 15, 800));
      return minWidth;
    },

    /**
     * 显示统计对话框并计算统计数据
     */
    showStatistics() {
      this.calculateStatistics();
      this.statisticsDialogVisible = true;
    },

    /**
     * 计算统计数据
     */
    calculateStatistics() {
      if (!this.filteredData || this.filteredData.length === 0) {
        this.statisticsData = [
          {
            articleCount: 0,
            countryCount: 0,
            mediaCount: 0,
            totalViews: 0,
            mediaDetails: "无数据",
            signatureInfo: this.hasSignatureArticleTag ? "" : undefined, // 署名信息字段
          },
        ];
        return;
      }

      // 国家数（去重）
      const countries = new Set();
      // 媒体数（去重）
      const medias = new Map();
      // 总落地次数
      let totalViews = 0;
      // 总篇数（不同序号的数量）
      const sequenceNumbers = new Set();
      const sequenceColumn = this.headers[0] || "序号"; // 获取序号列名称

      // 处理署名作者+文章名
      let signatureArticles = [];
      if (this.hasSignatureArticleTag) {
        // 按序号组分组
        const sequenceGroups = new Map();
        this.filteredData.forEach((row) => {
          const groupId = row._sequenceGroup;
          if (!groupId) return;

          if (!sequenceGroups.has(groupId)) {
            sequenceGroups.set(groupId, []);
          }
          sequenceGroups.get(groupId).push(row);
        });

        // 处理每个分组的第一个标题
        sequenceGroups.forEach((groupRows) => {
          const firstRow = groupRows[0];
          if (
            !firstRow["稿件标题"] ||
            !firstRow["稿件标题"].includes("署名文章")
          )
            return;

          const title = firstRow["稿件标题"];
          // 找到"署名文章"后的第一个"》"
          const 署名文章Index = title.indexOf("署名文章");
          if (署名文章Index === -1) return;

          // 从"署名文章"之后开始查找第一个"》"
          const startSearchIndex = 署名文章Index + "署名文章".length;
          const closingBracketIndex = title.indexOf("》", startSearchIndex);

          if (closingBracketIndex !== -1) {
            // 截取从开头到找到的"》"（包含"》"）
            const processedTitle = title.substring(0, closingBracketIndex + 1);
            signatureArticles.push(processedTitle);
          }
        });
      }

      this.filteredData.forEach((row) => {
        // 统计国家
        if (row["国家"] && row["国家"] !== "") {
          countries.add(row["国家"]);
        }

        // 统计媒体 - 标准化处理后去重
        if (row["落地媒体"] && row["落地媒体"] !== "") {
          // 标准化媒体名称：去除前后空格，转换为小写用于去重判断
          const normalizedMedia = row["落地媒体"].trim().toLowerCase();
          // 如果不存在，添加到Map中，保留原始名称
          if (!medias.has(normalizedMedia)) {
            medias.set(normalizedMedia, row["落地媒体"].trim());
          }
        }

        // 统计落地次数
        if (row["落地次数"] && !isNaN(row["落地次数"])) {
          totalViews += Number(row["落地次数"]);
        }
        // 统计总篇数（不同序号的数量）
        if (row[sequenceColumn] && row[sequenceColumn] !== "") {
          sequenceNumbers.add(row[sequenceColumn]);
        }
      });

      // 媒体详情（去重后用顿号分隔）
      const mediaDetails = Array.from(medias.values()).join("、");

      this.statisticsData = [
        {
          articleCount: sequenceNumbers.size,
          countryCount: countries.size,
          mediaCount: medias.size,
          totalViews: totalViews,
          mediaDetails: mediaDetails || "无数据",
          signatureInfo: this.hasSignatureArticleTag
            ? signatureArticles.join("\n")
            : undefined, // 署名信息字段
        },
      ];
    },

    /**
     * 复制统计结果
     */
    copyStatistics() {
      if (!this.statisticsData || this.statisticsData.length === 0) return;

      const stats = this.statisticsData[0];
      const text = `国家数: ${stats.countryCount}\n媒体数: ${stats.mediaCount}\n总落地次数: ${stats.totalViews}\n媒体详情: ${stats.mediaDetails}`;

      this.copyToClipboard(text);
      this.$message.success({
        message: "统计结果已复制到剪贴板",
        center: true,
      });
    },

    /**
     * 格式化单元格值
     * @param {*} value - 原始值
     * @param {Object} row - 当前行数据
     * @param {string} column - 当前列名
     * @returns {string} 格式化后的值
     */
    formatCellValue: function (value, row, column) {
      if (value == null) return "";

      // 获取当前列是否为序号列
      const firstHeader = this.headers[0] || "序号";

      // 序号列格式化 - 仅显示原始序号，分组行不显示
      if (column === firstHeader) {
        return row[`${firstHeader}_isOriginal`] ||
          row[`${firstHeader}_isFiltered`]
          ? value
          : "";
      }

      // 数字格式化
      if (typeof value === "number") {
        if (Number.isInteger(value)) {
          return value.toLocaleString(); // 整数格式化（带千分位）
        }
        return Number(value.toFixed(2)).toLocaleString(); // 小数格式化（保留两位小数）
      }

      // 布尔值格式化
      if (typeof value === "boolean") {
        return value ? "是" : "否";
      }

      // 其他类型转为字符串
      return String(value);
    },

    /**
     * 对话框层级
     */
    fixDialogZIndex: function () {
      if (!this.detailDialogVisible) return;

      this.$nextTick(() => {
        // 选择详情对话框的包装器和遮罩层
        const detailDialogWrapper = document.querySelector(
          ".cell-detail-dialog .el-dialog__wrapper"
        );
        const detailModal = document.querySelector(
          ".cell-detail-dialog + .v-modal"
        );

        if (detailDialogWrapper && detailModal) {
          // 设置极高的z-index确保在所有元素之上
          // 大幅提高全屏模式下的z-index值，确保超过表格容器的9999
          const baseZIndex = this.isFullscreen ? 100000 : 2000;
          detailDialogWrapper.style.zIndex = baseZIndex + 2;
          detailModal.style.zIndex = baseZIndex + 1;

          // 确保对话框本身也继承高z-index
          const dialogElement = detailDialogWrapper.querySelector(".el-dialog");
          if (dialogElement) {
            dialogElement.style.zIndex = baseZIndex + 2;
          }
        }
      });
    },
  },
};
</script>

<style scoped>
/* 整体布局：表格组件容器样式 */
.data-table {
  display: flex;
  flex-direction: column;
  height: 100%;
  position: relative;
  background: #f8fafc;
  padding: 10px 20px 20px;
  border-radius: 12px;
}

/* 控制栏：日期选择器、标签搜索、复原按钮和操作按钮区域样式 */
.table-controls {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 10px;
  padding: 10px 20px;
  border-radius: 10px;
  background: linear-gradient(135deg, #e6f7ff 0%, #bae7ff 100%);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  border: 1px solid #f0f0f0;
  overflow: hidden;
}

.controls-left {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  padding: 5px 0;
}

/* 日期选择器容器样式 */
.date-range-container {
  display: flex;
  align-items: center;
  margin-right: 60px;
}

/* 国家/落地媒体搜索容器样式 - 前置选择器输入框样式 */
:deep(.category-search-input) {
  width: 330px;
  height: 40px;
  margin-right: 60px;
}

/* 前置选择器样式 */
:deep(.input-with-select .el-select) {
  width: 120px;
  height: 100%;
  box-sizing: border-box;
  border: 1px solid #dcdfe6;
}

/* 输入框样式 */
:deep(.input-with-select .el-input__inner) {
  padding: 0 15px;
  height: 40px;
  line-height: 40px;
  box-sizing: border-box;
}

/* 获得焦点时的样式 */
:deep(.category-search-input:focus-within .el-input__inner),
:deep(.category-search-input:focus-within .el-select .el-select__input) {
  border-color: #409eff;
  box-shadow: 0 0 0 1px rgba(64, 158, 255, 0.2);
  /* 通过内边距补偿边框宽度变化 */
  padding: 0 14px;
}

/* 选择器下拉箭头颜色 */
:deep(.search-type-select .el-input__suffix-inner .el-icon) {
  color: #606266;
}
/* 标签搜索容器样式 */
.tag-search-container {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 360px;
  position: relative;
}

.tag-input {
  flex: 1;
}

.expression-help-btn {
  width: 20px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
}

.expression-error {
  border-color: #f56c6c !important;
}

.expression-error-message {
  position: absolute;
  top: 100%;
  left: 0;
  font-size: 12px;
  color: #f56c6c;
  margin-top: 4px;
  white-space: nowrap;
}

.controls-right {
  display: flex;
  align-items: center;
}

/* 操作按钮区域 */
.action-buttons {
  display: flex;
  align-items: center;
  /* 统计和检索按钮之间间距 */
  gap: 30px;
}

/* 日期选择器样式 */
:deep(.custom-date-picker.el-date-editor) {
  width: 330px !important;
  display: inline-flex !important; /* 确保内部元素正确排列 */
  align-items: center;
}

:deep(.el-date-editor .el-range-separator) {
  padding: 0 12px;
  font-size: 15px;
  color: #606266;
  font-weight: 500;
  white-space: nowrap; /* 确保"至"字不被截断 */
  min-width: 30px; /* 增加最小宽度，确保"至"字完全显示 */
  text-align: center; /* 明确设置文本居中 */
  flex-shrink: 0; /* 防止被压缩 */
}

:deep(.el-date-editor .el-range-input) {
  padding: 0 15px;
  font-size: 14px;
  width: 130px;
  height: 38px; /* 明确设置高度 */
}

.controls-right {
  display: flex;
  align-items: center;
  gap: 20px;
}

/* 表格容器：包含表格的容器样式 */
.table-container {
  flex: 1;
  min-height: 0;
  border: 1px solid #e8eaec;
  border-radius: 10px;
  overflow: hidden;
  background: white;
  box-shadow: 0 3px 12px rgba(0, 0, 0, 0.08);
  transition: all 0.3s ease;
  position: relative;
  z-index: 1;
  /* 确保表格容器填充可用空间 */
  display: flex;
  flex-direction: column;
}

/* 调整表格样式，使其填充容器 */
:deep(.el-table) {
  flex: 1;
  display: flex;
  flex-direction: column;
  height: 100% !important;
  font-size: 0.92rem;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    "Helvetica Neue", Arial, sans-serif;
}

/* 全屏模式：表格全屏显示样式 */
.table-container.fullscreen {
  position: fixed;
  top: 20px;
  left: 20px;
  right: 20px;
  bottom: 20px;
  z-index: 1000;
  border-radius: 12px;
  box-shadow: 0 10px 50px rgba(0, 0, 0, 0.2);
  border: none;
  background: white;
  padding: 0;
}

/* 表格样式 - 文本居中 */
:deep(.el-table) {
  font-size: 0.92rem;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    "Helvetica Neue", Arial, sans-serif;
}

/* 表头样式 */
:deep(.el-table th) {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  font-weight: 500;
  border-right: 1px solid rgba(255, 255, 255, 0.15);
  position: sticky;
  top: 0;
  z-index: 2;
  height: 52px;
  font-size: 0.95rem;
  min-width: 220px;
  text-align: center;
}

:deep(.el-table th .cell) {
  color: white;
  font-weight: 500;
  padding: 16px 12px;
  line-height: 1.4;
  white-space: normal;
  word-break: break-word;
  text-overflow: ellipsis;
  overflow: hidden;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  text-align: center;
  display: flex;
  justify-content: center;
  align-items: center;
}

/* 表格行样式 */
:deep(.el-table tr) {
  transition: all 0.2s ease;
}

:deep(.el-table tr:hover) {
  background-color: inherit !important; /* 继承原有背景色 */
  transform: translateY(-1px);
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.05);
}

/* 移除隔行变色样式 */
:deep(.el-table--striped .el-table__body tr.el-table__row--striped td) {
  background: transparent !important;
}

/* 单元格文本居中 */
:deep(.el-table td) {
  padding: 12px 8px;
  border-color: #f0f0f0;
  transition: background-color 0.2s;
  min-width: 220px;
  text-align: center; /* 单元格整体居中 */
  vertical-align: middle;
}

/* 单元格容器居中 */
:deep(.cell-container) {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100%;
  padding: 8px 10px;
  position: relative;
  min-height: 48px;
  cursor: default;
  text-align: center;
}

/* 单元格内容居中 */
:deep(.cell-content) {
  flex: 1;
  min-width: 0;
  padding: 0 10px; /* 左右内边距相等 */
  position: relative;
  line-height: 1.5;
  text-align: center; /* 内容在容器中居中 */
  display: flex;
  justify-content: center;
  align-items: center;
  max-width: 100%;
}

/* 单元格文本居中 */
:deep(.cell-text) {
  text-align: center; /* 确保文本居中 */
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  display: block;
  color: #333;
  margin: 0 auto; /* 在容器中居中 */
  max-width: 100%; /* 限制最大宽度 */
}

/* 普通模式下，长内容显示省略号 */
:deep(.cell-content.is-long:not(.fullscreen-expanded) .cell-text) {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  display: block;
  max-width: 100%;
  line-height: 1.5;
  color: #333;
  text-align: center; /* 确保文本居中 */
}

/* 普通模式短内容 */
:deep(.cell-content:not(.is-long) .cell-text) {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: #333;
  text-align: center;
  display: block;
  width: 100%;
}

/* 全屏模式下，长内容完全展开 */
:deep(.cell-content.fullscreen-expanded .cell-text) {
  white-space: normal !important;
  word-break: break-word !important;
  overflow-wrap: break-word !important;
  overflow: visible !important;
  max-height: none !important;
  display: block !important;
  line-height: 1.6;
  color: #333;
  padding: 2px 0;
  text-align: center;
}

/* 操作按钮样式 */
:deep(.cell-actions) {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  display: flex;
  gap: 6px;
  background: rgba(255, 255, 255, 0.95);
  padding: 4px 6px;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  border: 1px solid #f0f0f0;
  backdrop-filter: blur(4px);
  opacity: 0;
  transition: all 0.2s ease;
  z-index: 10;
}

:deep(.cell-container:hover .cell-actions) {
  opacity: 1;
  transform: translateY(-50%) scale(1.05);
}

:deep(.cell-action-btn) {
  padding: 4px;
  min-height: auto;
  width: 24px;
  height: 24px;
  font-size: 13px;
  color: #606266;
  transition: all 0.2s;
  opacity: 0.8;
}

:deep(.cell-action-btn:hover) {
  color: #409eff;
  background: #f0f9ff;
  border-radius: 4px;
  transform: scale(1.1);
  opacity: 1;
}

/* 详情对话框 - 全屏模式层级 */
:deep(.cell-detail-dialog) {
  z-index: 2000 !important;
}

:deep(.cell-detail-dialog.fullscreen-dialog) {
  z-index: 100000 !important;
}

:deep(.cell-detail-dialog .el-dialog) {
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.15);
  z-index: inherit !important;
}

/* 对话框头部样式 */
:deep(.cell-detail-dialog .el-dialog__header) {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 20px;
  margin: 0;
}

:deep(.cell-detail-dialog .el-dialog__title) {
  color: white;
  font-size: 1.2rem;
  font-weight: 600;
}

:deep(.cell-detail-dialog .el-dialog__headerbtn) {
  top: 20px;
  right: 20px;
}

:deep(.cell-detail-dialog .el-dialog__headerbtn .el-dialog__close) {
  color: white;
  font-size: 1.2rem;
}

:deep(.cell-detail-dialog .el-dialog__body) {
  padding: 0;
}

:deep(.cell-detail-dialog .el-dialog__footer) {
  padding: 20px;
  border-top: 1px solid #ebeef5;
  background: #f8f9fa;
}

/* 详情内容容器 */
.cell-detail-content {
  padding: 0;
  max-height: 60vh;
  overflow-y: auto;
}

/* 信息卡片 */
.info-card {
  background: white;
  border-radius: 0;
  margin: 0;
  border-bottom: 1px solid #ebeef5;
}

.info-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 20px 20px 15px 20px;
  background: linear-gradient(135deg, #f8f9fa 0%, #f1f3f5 100%);
  border-bottom: 1px solid #ebeef5;
}

.info-header i {
  color: #409eff;
  font-size: 1.1rem;
}

.info-title {
  color: #303133;
  font-size: 1.1rem;
  font-weight: 600;
}

.info-content {
  padding: 20px;
}

.info-row {
  display: flex;
  align-items: center;
  gap: 12px;
}

.info-value {
  color: #303133;
  font-weight: 500;
  font-size: 1rem;
  line-height: 1.6;
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 4px;
}

/* 信息项样式 */
.info-item {
  padding: 6px 10px;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.9rem;
  border: 1px solid;
  transition: all 0.2s ease;
}

.row-item {
  background: linear-gradient(135deg, #e6f7ff 0%, #bae7ff 100%);
  color: #1890ff;
  border-color: #91d5ff;
}

.sequence-item {
  background: linear-gradient(135deg, #f6ffed 0%, #d9f7be 100%);
  color: #52c41a;
  border-color: #b7eb8f;
}

.column-item {
  background: linear-gradient(135deg, #fff2e8 0%, #ffd591 100%);
  color: #fa8c16;
  border-color: #ffc069;
}

.info-separator {
  color: #d9d9d9;
  font-weight: 400;
  margin: 0 2px;
}

/* 内容卡片 */
.content-card {
  background: white;
  border-radius: 0;
  margin: 0;
}

/* 填充序号的样式 */
:deep(.filled-sequence) {
  color: #8c8c8c;
  font-style: italic;
  position: relative;
}

:deep(.filled-sequence):after {
  content: "";
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 1px;
  background: #8c8c8c;
}

.content-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 20px 20px 15px 20px;
  background: linear-gradient(135deg, #f8f9fa 0%, #f1f3f5 100%);
  border-bottom: 1px solid #ebeef5;
}

.content-header i {
  color: #67c23a;
  font-size: 1.1rem;
}

.content-title {
  color: #303133;
  font-size: 1.1rem;
  font-weight: 600;
}

.content-body {
  padding: 20px;
}

/* 内容文本区域 */
.content-text {
  padding: 16px;
  background: #f8f9fa;
  border-radius: 8px;
  border: 1px solid #e9ecef;
  white-space: pre-wrap;
  word-break: break-word;
  line-height: 1.6;
  max-height: 300px;
  overflow-y: auto;
  font-family: "SF Mono", "Consolas", "Monaco", "Courier New", monospace;
  font-size: 0.9rem;
  color: #333;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.05);
  min-height: 100px;
}

/* 控制按钮组：右下角功能按钮 */
.control-buttons {
  position: fixed;
  right: 25px;
  bottom: 25px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  z-index: 10000;
  filter: drop-shadow(0 4px 12px rgba(0, 0, 0, 0.15));
}

.control-button {
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  width: 50px;
  height: 50px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: white;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  padding: 0;
  font-size: 20px;
  line-height: 1;
  margin: 0;
  position: relative;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.control-button i {
  font-size: 20px;
  line-height: 1;
  transition: transform 0.2s;
}

.control-button:hover {
  transform: translateY(-4px) scale(1.05);
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.2);
}

.control-button:hover i {
  transform: scale(1.1);
}

/* 滚动按钮样式 */
.scroll-button {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.scroll-button:hover {
  background: linear-gradient(135deg, #5a6fd8 0%, #6a4a9a 100%);
  box-shadow: 0 8px 20px rgba(102, 126, 234, 0.5);
}

/* 退出全屏按钮样式 */
.exit-fullscreen-button {
  background: linear-gradient(135deg, #ff4d4f 0%, #ff7875 100%);
  color: white;
  border: none;
  box-shadow: 0 4px 12px rgba(255, 77, 79, 0.4);
}

.exit-fullscreen-button:hover {
  background: linear-gradient(135deg, #ff3336 0%, #ff6666 100%);
  box-shadow: 0 8px 20px rgba(255, 77, 79, 0.5);
}

/* 分组背景色样式 - 加深颜色提高区分度 */
:deep(.el-table tr.group-a-row) {
  background-color: rgba(204, 232, 255, 0.7) !important; /* 加深的淡蓝色背景 */
}

:deep(.el-table tr.group-b-row) {
  background-color: rgba(220, 255, 204, 0.7) !important; /* 加深的淡绿色背景 */
}

/* 悬停样式选择器，确保覆盖Element UI默认样式 */
:deep(.el-table__body tr:hover td) {
  background-color: inherit !important; /* 继承行的背景色 */
}

:deep(.el-table tr:hover) {
  transform: translateY(-1px);
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.05);
}

/* 移除隔行变色样式 */
:deep(.el-table--striped .el-table__body tr.el-table__row--striped td) {
  background: transparent !important;
}

/* 分组背景色样式 - 加深颜色提高区分度 */
:deep(.el-table tr.group-a-row) {
  background-color: rgba(204, 232, 255, 0.7) !important; /* 加深的淡蓝色背景 */
}

:deep(.el-table tr.group-b-row) {
  background-color: rgba(220, 255, 204, 0.7) !important; /* 加深的淡绿色背景 */
}

/* 统计对话框样式 */
:deep(.statistics-dialog .el-dialog) {
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.15);
}

:deep(.statistics-dialog .el-dialog__header) {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 20px;
  margin: 0;
}

:deep(.statistics-dialog .el-dialog__title) {
  color: white;
  font-size: 1.2rem;
  font-weight: 600;
}

:deep(.statistics-dialog .el-dialog__headerbtn .el-dialog__close) {
  color: white;
  font-size: 1.2rem;
}

.statistics-content {
  padding: 20px;
  overflow-x: auto; /* 允许横向滚动 */
}

:deep(.statistics-dialog .el-table) {
  min-width: 600px; /* 设置最小宽度 */
}

:deep(.statistics-dialog .el-table th) {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  font-weight: 500;
  text-align: center;
}

:deep(.statistics-dialog .el-table td) {
  text-align: center;
  vertical-align: middle;
  white-space: nowrap; /* 防止内容换行 */
}

:deep(.statistics-dialog .el-table .cell) {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 统计面板单元格样式 */
:deep(.statistic-cell) {
  cursor: pointer;
  position: relative;
}

:deep(.statistic-cell:hover) {
  color: #409eff;
  text-decoration: underline;
}
/* 统计面板媒体详情列自动省略号样式 */
:deep(.statistics-dialog .cell-text) {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  display: block;
  width: 100%;
}
/* 增强无数据状态样式的特异性 */
:deep(.el-table.no-data) {
  display: flex;
  flex-direction: column;
  width: 100% !important;
  overflow-x: hidden !important;
}

/* 无数据时表格列宽度均分 */
:deep(.el-table.no-data th, .el-table.no-data td) {
  min-width: auto !important;
  width: auto !important;
  flex: 1 !important;
}

/* 确保表格容器不横向滚动 */
:deep(.el-table.no-data .el-table__header-wrapper),
:deep(.el-table.no-data .el-table__body-wrapper) {
  width: 100% !important;
  overflow-x: hidden !important;
}

/* 确保空状态提示居中显示 */
:deep(.el-table.no-data .el-table__empty-block) {
  position: absolute !important;
  top: 50% !important;
  left: 50% !important;
  transform: translate(-50%, -50%) !important;
  margin: 0 !important;
  width: 100% !important;
}

/* 增强空状态文本样式 */
:deep(.el-table.no-data .el-empty__description) {
  font-size: 16px;
  color: #606266;
}
/* 提高所有Element UI消息提示的层级，确保在全屏表格上方显示 */
body.el-message {
  z-index: 2000 !important;
}
/* 响应式设计 */
@media (max-width: 768px) {
  .data-table {
    padding: 10px;
  }

  .table-controls {
    flex-direction: column;
    align-items: stretch;
    gap: 12px;
    padding: 12px;
  }

  .controls-left,
  .controls-right {
    width: 100%;
  }

  .tag-search-container {
    width: 100%;
  }

  .controls-left .el-input {
    width: 100% !important;
  }

  .controls-right {
    justify-content: space-between;
  }

  .control-buttons {
    right: 15px;
    bottom: 15px;
  }

  .control-button {
    width: 44px;
    height: 44px;
  }

  .control-button i {
    font-size: 18px;
  }
}
</style>
