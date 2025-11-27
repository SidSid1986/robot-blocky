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

        <div class="code-setting">
          <!-- 设置机械臂关节 -->
          <div class="joint-settings">
            <div class="setting-title">关节角度设置</div>
            <div class="joint-controls">
              <div
                class="joint-item"
                v-for="(config, index) in jointConfig"
                :key="config.key"
              >
                <label>{{ config.label }}:</label>

                <el-input-number
                  v-model="config.value"
                  :min="config.min"
                  :max="config.max"
                  :step="0.1"
                  @change="updateJointAngle(index, config.value)"
                  size="small"
                />
                <span class="unit">°</span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="robot-model-container">
        <RobotModelBlockly
          :highlight-block-id="currentExecutingBlock"
          :codeArr="codeArr"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, reactive, nextTick, watch } from "vue";
import RobotModelBlockly from "@/components/RobotModelBlockly.vue";

const num = ref(0);

// 机器人相关数据
const codeArr = ref([]);
const jointValues = reactive([0, 0, 0, 0, 0, 0]);

// 使用 reactive 而不是 ref，这样内部对象的属性变化也能被追踪
const jointConfig = reactive([
  { key: 0, label: "J1", min: -180, max: 180, value: -83.08 },
  { key: 1, label: "J2", min: -180, max: 180, value: -24.64 },
  { key: 2, label: "J3", min: -180, max: 180, value: 83.08 },
  { key: 3, label: "J4", min: -180, max: 180, value: -83.08 },
  { key: 4, label: "J5", min: -180, max: 180, value: -74.48 },
  { key: 5, label: "J6", min: -180, max: 180, value: -111.73 },
]);

const demoJointData = [
  [15.0, 25.0, 15.0, -35.0, -25.0, -15.0],
  [5.0, 20.0, 5.0, -40.0, -30.0, -25.0],
  [-5.0, 15.0, -5.0, -45.0, -35.0, -35.0],
  [-15.0, 10.0, -15.0, -50.0, -40.0, -45.0],
  [-25.0, 5.0, -25.0, -55.0, -45.0, -55.0],
  [-35.0, 0.0, -35.0, -60.0, -50.0, -65.0],
  [-45.0, -5.0, -45.0, -65.0, -55.0, -75.0],
  [-55.0, -10.0, -55.0, -70.0, -60.0, -85.0],
  [-65.0, -15.0, -65.0, -75.0, -65.0, -95.0],
  [-75.0, -20.0, -75.0, -80.0, -70.0, -105.0],
  [-83.08, -24.64, -83.08, -83.08, -74.48, -111.73],
];

// 单独的数组存放当前设置的关节角度值
const jontsArr = ref([-83.08, -24.64, 83.08, -83.08, -74.48, -111.73]);

// 监听 jointConfig 的变化，同步到 jontsArr
watch(
  jointConfig,
  (newConfig) => {
    newConfig.forEach((config, index) => {
      jontsArr.value[index] = config.value;
    });
    console.log("jontsArr updated:", jontsArr.value);
  },
  { deep: true }
);

// 工具箱项目配置
const generatePointTemplate = () => {
  return `移动到j1:${jontsArr.value[0]}°, j2:${jontsArr.value[1]}°, j3:${jontsArr.value[2]}°, j4:${jontsArr.value[3]}°, j5:${jontsArr.value[4]}°, j6:${jontsArr.value[5]}°`;
};

const toolboxItems = ref([
  { type: "start", label: "开始", icon: "▶️", template: "开始代码" },
  { type: "stop", label: "停止", icon: "⏹️", template: "停止代码" },
  {
    type: "point",
    label: "直线点",
    icon: "📍",
    getTemplate: generatePointTemplate,
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

// 更新关节角度的方法
const updateJointAngle = (index, value) => {
  console.log(`关节 J${index + 1} 角度更新为: ${value}°`);

  // 确保值在范围内
  const clampedValue = Math.max(-180, Math.min(180, value));

  // 更新 jointConfig
  jointConfig[index].value = clampedValue;

  // 更新 jointValues（用于传递给 RobotModelBlockly）
  jointValues[index] = clampedValue;

  // jontsArr 会通过 watch 自动更新
};

// 代码行数据
const codeLines = ref([]);
const selectedLine = ref(-1);
const nextLineId = ref(1);

// 执行状态相关
const isExecuting = ref(false);
const currentExecuteIndex = ref(-1);
const executionInterval = ref(null);

// 拖拽相关状态
const dragOverIndex = ref(-1);
const showInsertIndicator = ref(false);
const indicatorPosition = ref(0);
const dropPosition = ref(-1);
let draggedItem = null;

const currentExecutingBlock = ref("");

// 滚动容器引用
const scrollContainer = ref(null);

// 获取工具的模板
const getItemTemplate = (item) => {
  if (typeof item.getTemplate === "function") {
    return item.getTemplate();
  }
  return item.template;
};

// 解析代码行，提取关节角度数据
const parseLineToJoints = (lineText) => {
  if (!lineText) return null;

  // 匹配 "移动到j1:xxx°, j2:xxx°, ..." 的格式
  const pointMatch = lineText.match(
    /移动到j1:([-\d.]+)°, j2:([-\d.]+)°, j3:([-\d.]+)°, j4:([-\d.]+)°, j5:([-\d.]+)°, j6:([-\d.]+)°/
  );
  if (pointMatch) {
    return {
      type: "point",
      joints: [
        parseFloat(pointMatch[1]),
        parseFloat(pointMatch[2]),
        parseFloat(pointMatch[3]),
        parseFloat(pointMatch[4]),
        parseFloat(pointMatch[5]),
        parseFloat(pointMatch[6]),
      ],
    };
  }

  // 匹配速度设置
  const speedMatch = lineText.match(/设置速度:\s*([\d.]+)\s*mm\/s/);
  if (speedMatch) {
    return {
      type: "speed",
      value: parseFloat(speedMatch[1]),
    };
  }

  // 匹配延时
  const delayMatch = lineText.match(/延时:\s*([\d.]+)\s*ms/);
  if (delayMatch) {
    return {
      type: "delay",
      value: parseInt(delayMatch[1]),
    };
  }

  // 开始和停止指令
  if (lineText.includes("开始代码")) {
    return { type: "start" };
  }
  if (lineText.includes("停止代码")) {
    return { type: "stop" };
  }

  // 循环指令
  const loopMatch = lineText.match(/循环执行\s*(\d+)\s*次/);
  if (loopMatch) {
    return {
      type: "loop",
      count: parseInt(loopMatch[1]),
    };
  }

  return { type: "unknown", text: lineText };
};

// 执行单行代码
const executeSingleLine = (lineIndex) => {
  const line = codeLines.value[lineIndex];
  if (!line) return;

  const parsedData = parseLineToJoints(line.text, line);
  if (!parsedData) {
    console.warn(`无法解析第 ${lineIndex + 1} 行: ${line.text}`);
    return;
  }

  console.log(`执行第 ${lineIndex + 1} 行:`, line.text);
  console.log("解析后的数据:", parsedData);

  // 高亮当前执行的代码行
  currentExecutingBlock.value = `line-${line.id}`;
  selectedLine.value = lineIndex;

  // 如果是停止指令，直接返回特殊标记，让主函数停止执行
  if (parsedData.type === "stop") {
    console.log("停止执行程序");
    // 发送停止指令
    sendCommandToHardware("stop");
    return "STOP_EXECUTION"; // 返回特殊标记表示要停止执行
  }

  // 如果是关节移动指令，更新机械臂模型
  if (parsedData.type === "point" && parsedData.joints) {
    // 更新 jointValues 以驱动机械臂模型
    parsedData.joints.forEach((angle, index) => {
      jointValues[index] = angle;
    });

    // 这里可以发送数据到后端或硬件
    sendJointsToHardware(parsedData.joints);
  }

  // 如果是速度设置
  if (parsedData.type === "speed") {
    console.log(`设置速度为: ${parsedData.value} mm/s`);
    // 发送速度指令到硬件
    sendSpeedToHardware(parsedData.value);
  }

  // 如果是延时
  if (parsedData.type === "delay") {
    console.log(`延时: ${parsedData.value} ms`);
    // 延时期间保持当前状态
    return new Promise((resolve) => {
      setTimeout(resolve, parsedData.value);
    });
  }

  // 如果是开始指令
  if (parsedData.type === "start") {
    console.log("开始执行程序");
    // 发送开始指令
    sendCommandToHardware("start");
    return Promise.resolve(); // 开始指令立即完成
  }

  // 如果是循环指令，这里可以实现循环逻辑
  if (parsedData.type === "loop") {
    console.log(`循环执行 ${parsedData.count} 次`);
    // 简单的循环实现（实际可能需要更复杂的逻辑）
    return new Promise((resolve) => {
      setTimeout(resolve, 100); // 简化处理
    });
  }

  return Promise.resolve();
};

// 发送关节数据到硬件（模拟）
const sendJointsToHardware = (joints) => {
  console.log("发送到硬件的关节数据:", joints);
  codeArr.value = joints;
  // 这里替换为实际的 WebSocket 或 HTTP 请求
  // 例如: ws.send(JSON.stringify({ type: 'move', joints: joints }));
};

// 发送速度指令到硬件（模拟）
const sendSpeedToHardware = (speed) => {
  console.log("发送到硬件的速度数据:", speed);
  // 这里替换为实际的通信逻辑
};

// 发送通用指令到硬件（模拟）
const sendCommandToHardware = (command) => {
  console.log("发送到硬件的指令:", command);
  // 这里替换为实际的通信逻辑
};

// 执行代码 - 逐行执行
// 执行代码 - 逐行执行，从开始指令执行到停止指令，然后完全停止
const executeCode = async () => {
  // 先检查是否正在执行
  if (isExecuting.value) {
    console.log("正在执行中，请等待完成");
    return;
  }

  if (codeLines.value.length === 0) {
    console.log("没有可执行代码");
    return;
  }

  console.log("开始执行代码...");
  isExecuting.value = true;
  currentExecuteIndex.value = -1;

  try {
    // 查找开始和停止的位置
    const startIndex = codeLines.value.findIndex(
      (line) => line.type === "start"
    );
    const stopIndex = codeLines.value.findIndex((line) => line.type === "stop");

    if (startIndex === -1) {
      console.log('未找到开始指令，请在代码开头添加"开始代码"');
      return;
    }

    if (stopIndex === -1) {
      console.log('未找到停止指令，请在代码结尾添加"停止代码"');
      return;
    }

    if (startIndex >= stopIndex) {
      console.log("开始指令必须在停止指令之前");
      return;
    }

    console.log(`执行区间：第 ${startIndex + 1} 行到第 ${stopIndex + 1} 行`);

    // 从开始到停止执行代码（包含开始和停止指令）
    for (let i = startIndex; i <= stopIndex; i++) {
      currentExecuteIndex.value = i;

      // 滚动到当前执行的行
      scrollToLine(i);

      const result = executeSingleLine(i);

      // 检查是否遇到停止指令的特殊标记
      if (result === "STOP_EXECUTION") {
        console.log("遇到停止指令，立即停止执行");
        break;
      }

      // 处理异步操作（延时等）
      if (result instanceof Promise) {
        await result; // 等待延时等操作完成

        // 再次检查是否在延时后需要停止（防止在延时期间用户点击停止）
        // 这里可以根据需要添加额外的停止检查逻辑
      }

      // 行间间隔（最后一行不等待）
      if (i < stopIndex) {
        await new Promise((resolve) => setTimeout(resolve, 500)); // 500ms间隔
      }
    }

    console.log("执行已停止");

    // 执行完成后清除高亮
    setTimeout(() => {
      currentExecutingBlock.value = "";
      selectedLine.value = -1;
    }, 1000);
  } catch (error) {
    console.error("执行过程中出错:", error);
  } finally {
    // 确保在任何情况下都重置执行状态
    isExecuting.value = false;
    currentExecuteIndex.value = -1;
    console.log("执行状态已重置");
  }
};

// 滚动到指定行
const scrollToLine = (lineIndex) => {
  nextTick(() => {
    const lineElements = document.querySelectorAll(".code-line-row");
    if (lineElements[lineIndex]) {
      lineElements[lineIndex].scrollIntoView({
        behavior: "smooth",
        block: "center",
      });
    }
  });
};

// 停止执行
const stopExecution = () => {
  if (executionInterval.value) {
    clearInterval(executionInterval.value);
    executionInterval.value = null;
  }
  isExecuting.value = false;
  currentExecutingBlock.value = "";
  console.log("执行已停止");
};

// 添加工具箱项目到代码区
const addToolboxItemToCode = (item) => {
  const newLine = {
    id: nextLineId.value++,
    text: getItemTemplate(item),
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
    text: getItemTemplate(item),
    type: item.type,
    originalItem: item,
  };

  console.log(
    "inserting at index:",
    index,
    "current lines length:",
    codeLines.value.length
  );

  if (index < 0) {
    index = 0;
  } else if (index > codeLines.value.length) {
    index = codeLines.value.length;
  }

  codeLines.value.splice(index, 0, newLine);
  selectLine(index);
};

// 生成代码（保持原有功能）
const generateCode = () => {
  console.log(
    "生成的代码:",
    codeLines.value.map((line) => line.text)
  );
  codeArr.value = codeLines.value.map((line) => line.text);
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

// 清空工作区
const clearWorkspace = () => {
  codeLines.value = [];
  selectedLine.value = -1;
  nextLineId.value = 1;
  stopExecution();
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

// 处理从工具箱拖拽开始
const handleDragStart = (event, item) => {
  draggedItem = item;
  event.dataTransfer.effectAllowed = "copy";
  event.target.style.opacity = "0.5";
  dropPosition.value = -1;
  showInsertIndicator.value = false;
  dragOverIndex.value = -1;
};

// 处理拖拽结束
const handleDragEnd = (event) => {
  event.target.style.opacity = "1";
  draggedItem = null;
  setTimeout(() => {
    showInsertIndicator.value = false;
    dragOverIndex.value = -1;
    dropPosition.value = -1;
  }, 200);
};

// 处理拖拽经过代码行
const handleLineDragOver = async (event, index) => {
  event.preventDefault();
  event.stopPropagation();
  dragOverIndex.value = index;
  dropPosition.value = 1;

  await nextTick();
  const lineElement = event.currentTarget;
  const rect = lineElement.getBoundingClientRect();
  const containerRect = lineElement
    .closest(".scroll-container")
    .getBoundingClientRect();

  const mouseY = event.clientY - containerRect.top;
  const lineTop = rect.top - containerRect.top;
  const lineHeight = rect.height;
  const lineCenter = lineTop + lineHeight / 2;

  if (mouseY < lineCenter) {
    indicatorPosition.value = lineTop;
    dropPosition.value = 1;
  } else {
    indicatorPosition.value = lineTop + lineHeight;
    dropPosition.value = 3;
  }

  showInsertIndicator.value = true;
};

// 处理拖拽离开代码行
const handleDragLeave = (event) => {
  if (!event.currentTarget.contains(event.relatedTarget)) {
    dragOverIndex.value = -1;
    showInsertIndicator.value = false;
  }
};

// 处理容器拖拽经过
const handleContainerDragOver = (event) => {
  event.preventDefault();
  if (dragOverIndex.value === -1) {
    dropPosition.value = 0;
    showInsertIndicator.value = false;
  }
};

// 处理在代码行之间放置
const handleLineDrop = (event, index) => {
  event.preventDefault();
  event.stopPropagation();

  if (draggedItem && (dropPosition.value === 1 || dropPosition.value === 3)) {
    let insertIndex = index;

    if (dropPosition.value === 3) {
      insertIndex = index + 1;
    }

    insertToolboxItemAt(draggedItem, insertIndex);
  }

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

  showInsertIndicator.value = false;
  dragOverIndex.value = -1;
  dropPosition.value = -1;
};

onMounted(() => {
  addToolboxItemToCode(toolboxItems.value[0]);
  addToolboxItemToCode(toolboxItems.value[1]);

  //模拟codeLines测试，后续删除
  codeLines.value = [
    {
      id: nextLineId.value++,
      text: "开始代码",
      type: "start",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[0][0]}°, j2:${demoJointData[0][1]}°, j3:${demoJointData[0][2]}°, j4:${demoJointData[0][3]}°, j5:${demoJointData[0][4]}°, j6:${demoJointData[0][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[1][0]}°, j2:${demoJointData[1][1]}°, j3:${demoJointData[1][2]}°, j4:${demoJointData[1][3]}°, j5:${demoJointData[1][4]}°, j6:${demoJointData[1][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[2][0]}°, j2:${demoJointData[2][1]}°, j3:${demoJointData[2][2]}°, j4:${demoJointData[2][3]}°, j5:${demoJointData[2][4]}°, j6:${demoJointData[2][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[3][0]}°, j2:${demoJointData[3][1]}°, j3:${demoJointData[3][2]}°, j4:${demoJointData[3][3]}°, j5:${demoJointData[3][4]}°, j6:${demoJointData[3][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[4][0]}°, j2:${demoJointData[4][1]}°, j3:${demoJointData[4][2]}°, j4:${demoJointData[4][3]}°, j5:${demoJointData[4][4]}°, j6:${demoJointData[4][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[5][0]}°, j2:${demoJointData[5][1]}°, j3:${demoJointData[5][2]}°, j4:${demoJointData[5][3]}°, j5:${demoJointData[5][4]}°, j6:${demoJointData[5][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[6][0]}°, j2:${demoJointData[6][1]}°, j3:${demoJointData[6][2]}°, j4:${demoJointData[6][3]}°, j5:${demoJointData[6][4]}°, j6:${demoJointData[6][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[7][0]}°, j2:${demoJointData[7][1]}°, j3:${demoJointData[7][2]}°, j4:${demoJointData[7][3]}°, j5:${demoJointData[7][4]}°, j6:${demoJointData[7][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[8][0]}°, j2:${demoJointData[8][1]}°, j3:${demoJointData[8][2]}°, j4:${demoJointData[8][3]}°, j5:${demoJointData[8][4]}°, j6:${demoJointData[8][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[9][0]}°, j2:${demoJointData[9][1]}°, j3:${demoJointData[9][2]}°, j4:${demoJointData[9][3]}°, j5:${demoJointData[9][4]}°, j6:${demoJointData[9][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: `移动到j1:${demoJointData[10][0]}°, j2:${demoJointData[10][1]}°, j3:${demoJointData[10][2]}°, j4:${demoJointData[10][3]}°, j5:${demoJointData[10][4]}°, j6:${demoJointData[10][5]}°`,
      type: "point",
    },
    {
      id: nextLineId.value++,
      text: "停止代码",
      type: "stop",
    },
  ];

  // 设置选中的行和 nextLineId
  if (codeLines.value.length > 0) {
    selectedLine.value = 0;
    nextLineId.value = Math.max(...codeLines.value.map((line) => line.id)) + 1;
  }

  console.log("初始化完成，加载了", codeLines.value.length, "行代码");
});

onUnmounted(() => {
  stopExecution();
});
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

.code-setting {
  padding: 15px;
  border-top: 1px solid #dee2e6;
  background: #f8f9fa;
  height: calc(100% - 53vh - 3vh - 3vh); // 根据实际布局调整
  box-sizing: border-box;
}

.joint-settings {
  height: 100%;

  .setting-title {
    font-weight: bold;
    color: #2c3e50;
    margin-bottom: 15px;
    font-size: 14px;
    border-bottom: 1px solid #dee2e6;
    padding-bottom: 8px;
  }

  .joint-controls {
    display: flex;
    flex-direction: row;
    justify-content: flex-start;
    align-items: center;
    flex-wrap: wrap;
    gap: 10px;

    .joint-item {
      display: flex;
      flex-direction: row;
      justify-content: flex-start;
      align-items: center;
      gap: 8px;

      label {
        width: 10px;
        font-weight: bold;
        color: #495057;
        font-size: 13px;
      }

      input {
        width: 50px;
        padding: 6px 8px;
        border: 1px solid #ced4da;
        border-radius: 4px;
        font-size: 13px;
        text-align: center;

        &:focus {
          outline: none;
          border-color: #007bff;
          box-shadow: 0 0 0 2px rgba(0, 123, 255, 0.25);
        }
      }

      .unit {
        color: #6c757d;
        font-size: 12px;
        width: 15px;
      }
    }
  }
}
</style>
