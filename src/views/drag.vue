<template>
  <div class="blockly-demo">
    <div class="header">
      <h2>Demo</h2>
      <div class="controls">
        <button @click="generateCode">生成代码</button>
        <button @click="executeCode">执行命令</button>
        <button @click="clearWorkspace">清空工作区</button>
        <button @click="loadDemo">加载示例</button>
        <button @click="saveWorkspace">保存</button>
        <button @click="loadWorkspace">加载</button>
        <button @click="wsTest">模拟ws</button>
      </div>
    </div>
    <div class="container">
      <!-- Blockly容器 -->
      <div class="blockly-container">
        <div class="blockly-top">
          <div class="tool-title">工具箱</div>
          <div class="toolbox-items">
            <div
              v-for="item in toolboxItems"
              :key="item.type"
              class="toolbox-item"
              draggable="true"
              @dragstart="handleDragStart($event, item)"
              @dragend="handleDragEnd"
            >
              <div class="item-icon">{{ item.icon }}</div>
              <div class="item-label">{{ item.label }}</div>
            </div>
          </div>
        </div>

        <div class="print-container">
          <div class="print-title">输出</div>
          <div class="print-content"></div>
        </div>
      </div>

      <!-- 右侧代码面板 -->
      <div class="code-panel">
        <div class="code-header">
          <h3>代码编辑区</h3>
          <button @click="addNewLine" class="add-line-btn">+ 添加空行</button>
        </div>

        <div class="code-content">
          <!-- 表头 -->
          <!-- <div class="code-header-row">
            <div class="line-numbers-header">#</div>
            <div class="code-lines-header">代码</div>
          </div> -->

          <!-- 可滚动的代码区域 -->
          <div class="scroll-container" ref="scrollContainer">
            <div class="code-lines-wrapper">
              <div
                v-for="(line, index) in codeLines"
                :key="line.id"
                class="code-line-row"
                :class="{
                  selected: selectedLine === index,
                  dragging: dragOverIndex === index,
                }"
                @click="selectLine(index)"
                @dragover="handleLineDragOver($event, index)"
                @dragleave="handleDragLeave"
                @drop="handleLineDrop($event, index)"
              >
                <div
                  class="line-number"
                  :class="{ active: selectedLine === index }"
                  @click="selectLine(index)"
                >
                  {{ index + 1 }}
                </div>
                <div class="code-line">
                  <div class="line-content">
                    {{ line.text || "空行" }}
                  </div>
                  <div class="line-actions">
                    <button @click.stop="deleteLine(index)" class="delete-btn">
                      ×
                    </button>
                  </div>
                </div>
              </div>

              <!-- 拖拽插入指示器 -->
              <div
                v-if="showInsertIndicator"
                class="insert-indicator"
                :style="{ top: indicatorPosition + 'px' }"
              ></div>
            </div>
          </div>
        </div>

        <div class="code-setting"></div>
      </div>

      <div class="robot-model-container">
        <RobotModelBlockly
          :joint-angles="jointValues"
          :highlight-block-id="currentExecutingBlock"
          :codeArr="codeArr"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, reactive, nextTick } from "vue";
import RobotModelBlockly from "@/components/RobotModelBlockly.vue";

// 工具箱项目配置
const toolboxItems = ref([
  { type: "start", label: "开始", icon: "▶️", template: "开始代码" },
  { type: "stop", label: "停止", icon: "⏹️", template: "停止代码" },
  {
    type: "point",
    label: "直线点",
    icon: "📍",
    template: "移动到点位 (X: __, Y: __, Z: __)",
  },
  {
    type: "speed",
    label: "设置速度",
    icon: "⚡",
    template: "设置速度: __ mm/s",
  },
  { type: "delay", label: "延时", icon: "⏰", template: "延时: __ ms" },
  { type: "loop", label: "循环", icon: "🔄", template: "循环执行 __ 次" },
]);

// 代码行数据
const codeLines = ref([]);
const selectedLine = ref(-1);
const nextLineId = ref(1);

// 拖拽相关状态
const dragOverIndex = ref(-1);
const showInsertIndicator = ref(false);
const indicatorPosition = ref(0);
const dropPosition = ref(-1); // 0: 容器末尾, 1: 行之间, 2: 具体位置信息
let draggedItem = null;

// 机器人相关数据
const codeArr = ref([]);
const jointValues = reactive([0, 0, 0, 0, 0, 0]);
const currentExecutingBlock = ref("");

// 滚动容器引用
const scrollContainer = ref(null);

onMounted(() => {
  // 初始化一些示例代码
  addToolboxItemToCode(toolboxItems.value[0]); // 开始
  addToolboxItemToCode(toolboxItems.value[1]); // 停止
});

onUnmounted(() => {});

// 处理从工具箱拖拽开始
const handleDragStart = (event, item) => {
  draggedItem = item;
  event.dataTransfer.effectAllowed = "copy";
  event.target.style.opacity = "0.5";
  // 重置拖拽状态
  dropPosition.value = -1;
  showInsertIndicator.value = false;
  dragOverIndex.value = -1;
};

// 处理拖拽结束
const handleDragEnd = (event) => {
  event.target.style.opacity = "1";
  draggedItem = null;
  // 延迟清理，避免影响插入指示器显示
  setTimeout(() => {
    showInsertIndicator.value = false;
    dragOverIndex.value = -1;
    dropPosition.value = -1;
  }, 200);
};

// 处理拖拽经过代码行
const handleLineDragOver = async (event, index) => {
  event.preventDefault();
  event.stopPropagation(); // 阻止冒泡到容器
  dragOverIndex.value = index;
  dropPosition.value = 1; // 标记为行之间拖拽

  await nextTick();
  const lineElement = event.currentTarget;
  const rect = lineElement.getBoundingClientRect();
  const containerRect = lineElement
    .closest(".scroll-container")
    .getBoundingClientRect();

  //  基于鼠标相对于行的位置来判断插入位置
  const mouseY = event.clientY - containerRect.top;
  const lineTop = rect.top - containerRect.top;
  const lineHeight = rect.height;
  const lineCenter = lineTop + lineHeight / 2;

  //  鼠标在上半部分插入到当前行之前，在下半部分插入到当前行之后
  if (mouseY < lineCenter) {
    // 插入到当前行之前（当前行索引位置）
    indicatorPosition.value = lineTop;
    dropPosition.value = 1; // 在当前行之前插入
  } else {
    // 插入到当前行之后（当前行索引+1位置）
    indicatorPosition.value = lineTop + lineHeight;
    dropPosition.value = 3; // 在当前行之后插入（新的标识）
  }

  showInsertIndicator.value = true;
};

// 处理拖拽离开代码行
const handleDragLeave = (event) => {
  // 检查是否真的离开了当前元素
  if (!event.currentTarget.contains(event.relatedTarget)) {
    dragOverIndex.value = -1;
    showInsertIndicator.value = false;
  }
};

// 处理容器拖拽经过（用于末尾添加）
const handleContainerDragOver = (event) => {
  event.preventDefault();
  // 如果没有在任何行上悬停，则准备在末尾添加
  if (dragOverIndex.value === -1) {
    dropPosition.value = 0;
    showInsertIndicator.value = false;
  }
};

// 处理在代码行之间放置
const handleLineDrop = (event, index) => {
  event.preventDefault();
  event.stopPropagation(); // 阻止冒泡到容器

  if (draggedItem && (dropPosition.value === 1 || dropPosition.value === 3)) {
    let insertIndex = index;

    // 根据dropPosition决定插入位置
    if (dropPosition.value === 3) {
      // 在当前行之后插入，所以索引+1
      insertIndex = index + 1;
    }
    // dropPosition.value === 1 时就在当前索引位置插入

    insertToolboxItemAt(draggedItem, insertIndex);
  }

  // 清理状态
  showInsertIndicator.value = false;
  dragOverIndex.value = -1;
  dropPosition.value = -1;
};

// 处理在容器末尾放置
const handleContainerDrop = (event) => {
  event.preventDefault();

  if (draggedItem && dropPosition.value === 0) {
    addToolboxItemToCode(draggedItem);
  }

  // 清理状态
  showInsertIndicator.value = false;
  dragOverIndex.value = -1;
  dropPosition.value = -1;
};

// 添加工具箱项目到代码区
const addToolboxItemToCode = (item) => {
  const newLine = {
    id: nextLineId.value++,
    text: item.template,
    type: item.type,
    originalItem: item,
  };
  codeLines.value.push(newLine);
  selectLine(codeLines.value.length - 1);
};

// 在指定位置插入工具箱项目
const insertToolboxItemAt = (item, index) => {
  const newLine = {
    id: nextLineId.value++,
    text: item.template,
    type: item.type,
    originalItem: item,
  };

  console.log(
    "inserting at index:",
    index,
    "current lines length:",
    codeLines.value.length
  );

  // 边界检查和处理
  if (index < 0) {
    index = 0;
  } else if (index > codeLines.value.length) {
    index = codeLines.value.length;
  }

  codeLines.value.splice(index, 0, newLine);
  selectLine(index);
};

// 选择行
const selectLine = (index) => {
  selectedLine.value = index;
};

// 删除行
const deleteLine = (index) => {
  if (confirm("确定要删除这一行吗？")) {
    codeLines.value.splice(index, 1);
    if (selectedLine.value >= codeLines.value.length) {
      selectedLine.value = codeLines.value.length - 1;
    }
    if (selectedLine.value < 0 && codeLines.value.length > 0) {
      selectedLine.value = 0;
    }
  }
};

// 添加新空行
const addNewLine = () => {
  const newLine = {
    id: nextLineId.value++,
    text: "",
    type: "empty",
  };
  codeLines.value.push(newLine);
  selectLine(codeLines.value.length - 1);
};

// 生成代码
const generateCode = () => {
  console.log(
    "生成的代码:",
    codeLines.value.map((line) => line.text)
  );
  codeArr.value = codeLines.value.map((line) => line.text);
};

// 执行代码
const executeCode = () => {
  console.log("执行代码:", codeLines.value);
  // 这里可以添加实际的代码执行逻辑
};

// 清空工作区
const clearWorkspace = () => {
  codeLines.value = [];
  selectedLine.value = -1;
  nextLineId.value = 1;
};

// 加载示例
const loadDemo = () => {
  clearWorkspace();
  addToolboxItemToCode(toolboxItems.value[0]); // 开始
  addToolboxItemToCode(toolboxItems.value[2]); // 直线点
  addToolboxItemToCode(toolboxItems.value[4]); // 延时
  addToolboxItemToCode(toolboxItems.value[1]); // 停止
};

// 保存工作区
const saveWorkspace = () => {
  const workspaceData = JSON.stringify(codeLines.value);
  localStorage.setItem("blockly-workspace", workspaceData);
  alert("工作区已保存");
};

// 加载工作区
const loadWorkspace = () => {
  const savedData = localStorage.getItem("blockly-workspace");
  if (savedData) {
    try {
      const parsedData = JSON.parse(savedData);
      codeLines.value = parsedData;
      if (codeLines.value.length > 0) {
        nextLineId.value =
          Math.max(...codeLines.value.map((line) => line.id)) + 1;
        selectedLine.value = 0;
      }
      alert("工作区已加载");
    } catch (error) {
      alert("加载失败：数据格式错误");
    }
  } else {
    alert("没有找到保存的工作区");
  }
};

// 模拟WebSocket测试
const wsTest = () => {
  console.log("模拟WebSocket测试");
};
</script>

<style scoped lang="scss">
.blockly-demo {
  height: 100vh;
  display: flex;
  flex-direction: column;
  font-family: Arial, sans-serif;
}

.header {
  padding: 15px 20px;
  background: #2c3e50;
  color: white;
  border-bottom: 2px solid #34495e;
  box-sizing: border-box;
  height: 10vh;
}

.header h2 {
  margin: 0 0 10px 0;
  font-size: 1.5em;
}

.controls {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.controls button {
  padding: 8px 16px;
  background: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.3s;
}

.controls button:hover {
  background: #2980b9;
}

.container {
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  justify-content: flex-start;
  height: 90vh;
  overflow: hidden;
}

/* 工具箱样式 */
.toolbox-panel {
  width: 120px;
  height: 100%;
  background: #f8f9fa;
  border-right: 1px solid #dee2e6;
  padding: 15px;
  box-sizing: border-box;
  overflow-y: auto;
}

.toolbox-panel h3 {
  margin: 0 0 15px 0;
  color: #2c3e50;
  font-size: 14px;
  text-align: center;
}

.toolbox-items {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  max-height: 50vh;
  overflow: auto;
}

.toolbox-item {
  margin-top: 5px;
  width: 60px;
  height: 60px;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 10px;
  background: white;
  border: 1px solid #dee2e6;
  border-radius: 6px;
  cursor: grab;
  transition: all 0.3s;
  user-select: none;
}

.toolbox-item:hover {
  background: #e3f2fd;
  border-color: #2196f3;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.toolbox-item:active {
  cursor: grabbing;
}

.item-icon {
  font-size: 20px;
  margin-bottom: 5px;
}

.item-label {
  font-size: 12px;
  color: #2c3e50;
  text-align: center;
}

.print-container {
  height: 37vh;
  width: 100%;

  .print-title {
    height: 3vh;
    line-height: 3vh;
  }

  .print-content {
    height: 34vh;
    border-top: 1px solid #bdc3c7;
    box-sizing: border-box;
    padding: 10px;
    overflow-y: auto;
    // background-color: #f9f9f9;
    background-color: #111111;
    font-size: 12px;
    color: #2c3e50;
  }
}

/* Blockly容器 */
.blockly-container {
  width: 25vw;
  height: 100%;
  min-width: 0;
  border: 1px solid #bdc3c7;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;

  .blockly-top {
    .tool-title {
      height: 3vh;
      line-height: 3vh;
      // background-color: red;
    }
  }
}

/* 代码面板样式 */
.code-panel {
  width: 45vw;
  height: 100%;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  min-width: 400px;
  border-left: 2px solid #bdc3c7;
  background: #ffffff;
}

.code-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px;
  border-bottom: 1px solid #dee2e6;
  background: #f8f9fa;
  height: 3vh;
  box-sizing: border-box;
  // border: 1px solid red;
}

.code-header h3 {
  margin: 0;
  color: #2c3e50;
  font-size: 1.1em;
}

.add-line-btn {
  padding: 6px 12px;
  background: #28a745;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
}

.add-line-btn:hover {
  background: #218838;
}

.code-content {
  overflow: visible;
  height: 50vh;
  display: flex;
  flex-direction: column;
  border: 1px solid #dee2e6;
  border-top: none;
  background: white;
}

/* 表头样式 */
.code-header-row {
  display: flex;
  background: #f8f9fa;
  border-bottom: 1px solid #dee2e6;
  height: 35px;
  flex-shrink: 0;

  .line-numbers-header {
    width: 50px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 12px;
    color: #6c757d;
    border-right: 1px solid #dee2e6;
    background: #f1f3f4;
    font-weight: bold;
  }

  .code-lines-header {
    flex: 1;
    display: flex;
    align-items: center;
    padding: 0 15px;
    font-size: 12px;
    color: #6c757d;
    font-weight: bold;
  }
}

/* 滚动容器 */
.scroll-container {
  flex: 1;
  overflow: auto;
  position: relative;
}

.code-lines-wrapper {
  position: relative;
  min-height: 100%;
}

/* 代码行行容器 */
.code-line-row {
  display: flex;
  min-height: 40px;
  border-bottom: 1px solid #f1f3f4;
  cursor: pointer;
  transition: background-color 0.2s;

  &:hover {
    background: #f8f9fa;
  }

  &.selected {
    background: #e3f2fd;
  }

  &.dragging {
    background: #fff3cd;
  }
}

/* 行号样式 */
.line-number {
  width: 50px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  color: #6c757d;
  cursor: pointer;
  border-right: 1px solid #dee2e6;
  background: #f8f9fa;
  flex-shrink: 0;
  user-select: none;

  &.active {
    background: #007bff;
    color: white;
  }

  &:hover {
    background: #e9ecef;
  }
}

/* 代码行样式 */
.code-line {
  flex: 1;
  display: flex;
  align-items: center;
  padding: 0 15px;
  position: relative;
  border-left: 3px solid transparent;

  .selected & {
    border-left: 3px solid #2196f3;
  }
}

.line-content {
  flex: 1;
  font-family: "Courier New", monospace;
  font-size: 13px;
  color: #2c3e50;
}

.line-actions {
  opacity: 0;
  transition: opacity 0.2s;

  .code-line-row:hover & {
    opacity: 1;
  }
}

.delete-btn {
  width: 24px;
  height: 24px;
  border: none;
  background: #dc3545;
  color: white;
  border-radius: 50%;
  cursor: pointer;
  font-size: 14px;
  display: flex;
  align-items: center;
  justify-content: center;

  &:hover {
    background: #c82333;
  }
}

/* 插入指示器 */
.insert-indicator {
  position: absolute;
  left: 0;
  right: 0;
  height: 2px;
  background: #007bff;
  z-index: 10;
  box-shadow: 0 0 4px #007bff;
}

/* 机械臂 */
.robot-model-container {
  width: 30vw;
  height: 100%;
  min-width: 300px;
}

/* 响应式设计 */
@media (max-width: 1200px) {
  .container {
    flex-direction: column;
    height: auto;
  }

  .toolbox-panel,
  .blockly-container,
  .code-panel {
    width: 100%;
    height: 300px;
  }

  .robot-model-container {
    width: 100%;
    height: 400px;
  }
}
</style>
