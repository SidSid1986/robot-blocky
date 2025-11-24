<template>
  <div class="blockly-demo">
    <div class="header">
      <h2>Vue3 + Blockly 可视化编程</h2>
      <div class="controls">
        <button @click="generateCode">生成代码</button>
        <button @click="clearWorkspace">清空工作区</button>
        <button @click="loadDemo">加载示例</button>
        <button @click="saveWorkspace">保存</button>
        <button @click="loadWorkspace">加载</button>
      </div>
    </div>

    <div class="container">
      <div ref="blocklyDiv" class="blockly-container"></div>

      <div class="code-panel">
        <div class="code-section">
          <h3>生成的 JavaScript 代码:</h3>
          <pre><code>{{ generatedCode }}</code></pre>
        </div>

        <div class="output-section">
          <h3>执行结果:</h3>
          <div class="output">{{ executionResult }}</div>
          <button @click="runCode" class="run-btn">运行代码</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";

// 正确导入 Blockly
import * as Blockly from "blockly";
import { javascriptGenerator } from "blockly/javascript";
import "blockly/blocks";

// DOM 引用
const blocklyDiv = ref(null);

// 响应式数据
const generatedCode = ref("// 生成的代码将显示在这里");
const executionResult = ref("");
let workspace = null;

// 工具箱配置
const toolbox = {
  kind: "categoryToolbox",
  contents: [
    {
      kind: "category",
      name: "逻辑",
      colour: "#5C81A6",
      contents: [
        { kind: "block", type: "controls_if" },
        { kind: "block", type: "logic_compare" },
        { kind: "block", type: "logic_operation" },
        { kind: "block", type: "logic_negate" },
        { kind: "block", type: "logic_boolean" },
      ],
    },
    {
      kind: "category",
      name: "循环",
      colour: "#5CA65C",
      contents: [
        { kind: "block", type: "controls_repeat_ext" },
        { kind: "block", type: "controls_whileUntil" },
        { kind: "block", type: "controls_for" },
        { kind: "block", type: "controls_flow_statements" },
      ],
    },
    {
      kind: "category",
      name: "数学",
      colour: "#5C68A6",
      contents: [
        { kind: "block", type: "math_number" },
        { kind: "block", type: "math_arithmetic" },
        { kind: "block", type: "math_single" },
        { kind: "block", type: "math_round" },
        { kind: "block", type: "math_modulo" },
      ],
    },
    {
      kind: "category",
      name: "文本",
      colour: "#5CA68D",
      contents: [
        { kind: "block", type: "text" },
        { kind: "block", type: "text_join" },
        { kind: "block", type: "text_append" },
        { kind: "block", type: "text_length" },
        { kind: "block", type: "text_isEmpty" },
      ],
    },
    {
      kind: "category",
      name: "变量",
      colour: "#A65C81",
      custom: "VARIABLE",
    },
    {
      kind: "category",
      name: "函数",
      colour: "#9A5CA6",
      custom: "PROCEDURE",
    },
  ],
};

// 初始化 Blockly
const initBlockly = () => {
  if (!blocklyDiv.value) return;

  workspace = Blockly.inject(blocklyDiv.value, {
    toolbox: toolbox,
    grid: {
      spacing: 20,
      length: 3,
      colour: "#ccc",
      snap: true,
    },
    zoom: {
      controls: true,
      wheel: true,
      startScale: 1.0,
      maxScale: 3,
      minScale: 0.3,
      scaleSpeed: 1.2,
    },
    move: {
      scrollbars: true,
      drag: true,
      wheel: true,
    },
  });

  // 监听工作区变化
  workspace.addChangeListener((event) => {
    if (!event.isUiEvent) {
      generateCode();
    }
  });
};

// 生成代码
const generateCode = () => {
  if (workspace) {
    try {
      const code = javascriptGenerator.workspaceToCode(workspace);
      generatedCode.value = code || "// 没有可执行的积木块";
    } catch (error) {
      generatedCode.value = `// 生成代码时出错:\n// ${error.message}`;
    }
  }
};

// 运行代码
const runCode = () => {
  try {
    // 创建一个安全的执行环境
    const output = [];
    const originalLog = console.log;
    console.log = (...args) => {
      output.push(args.join(" "));
      originalLog.apply(console, args);
    };

    // 执行生成的代码
    if (generatedCode.value && !generatedCode.value.startsWith("//")) {
      // 使用 Function 构造函数更安全
      try {
        const func = new Function(generatedCode.value);
        func();
      } catch (e) {
        output.push(`执行错误: ${e.message}`);
      }
    }

    console.log = originalLog;
    executionResult.value = output.join("\n") || "代码执行完成（无输出）";
  } catch (error) {
    executionResult.value = `执行错误: ${error.message}`;
  }
};

// 清空工作区
const clearWorkspace = () => {
  if (workspace) {
    workspace.clear();
    generatedCode.value = "// 生成的代码将显示在这里";
    executionResult.value = "";
  }
};

// 创建积木块并设置位置的辅助函数
const createBlock = (type, x, y) => {
  const block = workspace.newBlock(type);
  // 正确设置位置的方式
  block.moveBy(x, y);
  block.initSvg();
  block.render();
  return block;
};

// 加载示例
const loadDemo = () => {
  if (!workspace) return;

  clearWorkspace();

  try {
    // 创建示例：简单的打印语句
    const printBlock = createBlock("text_print", 50, 50);
    const textBlock = createBlock("text", 0, 0);
    textBlock.setFieldValue("Hello, Blockly!", "TEXT");
    printBlock.getInput("TEXT").connection.connect(textBlock.outputConnection);

    // 创建数学运算示例
    const mathBlock = createBlock("math_arithmetic", 50, 150);
    mathBlock.setFieldValue("MULTIPLY", "OP");

    const numberBlock1 = createBlock("math_number", 0, 0);
    numberBlock1.setFieldValue("5", "NUM");

    const numberBlock2 = createBlock("math_number", 0, 0);
    numberBlock2.setFieldValue("3", "NUM");

    mathBlock.getInput("A").connection.connect(numberBlock1.outputConnection);
    mathBlock.getInput("B").connection.connect(numberBlock2.outputConnection);

    // 创建第二个打印语句来显示数学结果
    const printBlock2 = createBlock("text_print", 50, 250);
    const textJoinBlock = createBlock("text_join", 0, 0);
    const textBlock2 = createBlock("text", 0, 0);
    textBlock2.setFieldValue("5 * 3 = ", "TEXT");

    textJoinBlock
      .getInput("ADD0")
      .connection.connect(textBlock2.outputConnection);
    textJoinBlock
      .getInput("ADD1")
      .connection.connect(mathBlock.outputConnection);
    printBlock2
      .getInput("TEXT")
      .connection.connect(textJoinBlock.outputConnection);

    // 添加一个简单的条件判断示例
    const ifBlock = createBlock("controls_if", 50, 350);

    const compareBlock = createBlock("logic_compare", 0, 0);
    compareBlock.setFieldValue("EQ", "OP");

    const num1Block = createBlock("math_number", 0, 0);
    num1Block.setFieldValue("10", "NUM");

    const num2Block = createBlock("math_number", 0, 0);
    num2Block.setFieldValue("10", "NUM");

    compareBlock.getInput("A").connection.connect(num1Block.outputConnection);
    compareBlock.getInput("B").connection.connect(num2Block.outputConnection);

    ifBlock.getInput("IF0").connection.connect(compareBlock.outputConnection);

    const printTrueBlock = createBlock("text_print", 0, 0);
    const textTrueBlock = createBlock("text", 0, 0);
    textTrueBlock.setFieldValue("条件成立！", "TEXT");

    printTrueBlock
      .getInput("TEXT")
      .connection.connect(textTrueBlock.outputConnection);
    ifBlock
      .getInput("DO0")
      .connection.connect(printTrueBlock.previousConnection);

    generateCode();
  } catch (error) {
    console.error("加载示例时出错:", error);
    executionResult.value = `加载示例错误: ${error.message}`;
  }
};

// 保存工作区
const saveWorkspace = () => {
  if (workspace) {
    try {
      const data = Blockly.serialization.workspaces.save(workspace);
      localStorage.setItem("blocklyWorkspace", JSON.stringify(data));
      alert("工作区已保存！");
    } catch (error) {
      console.error("保存工作区时出错:", error);
      alert("保存失败！");
    }
  }
};

// 加载工作区
const loadWorkspace = () => {
  if (workspace) {
    try {
      const data = localStorage.getItem("blocklyWorkspace");
      if (data) {
        Blockly.serialization.workspaces.load(JSON.parse(data), workspace);
        alert("工作区已加载！");
        generateCode();
      } else {
        alert("没有找到保存的工作区");
      }
    } catch (error) {
      console.error("加载工作区时出错:", error);
      alert("加载失败！");
    }
  }
};

// 生命周期
onMounted(() => {
  initBlockly();
  // 延迟加载示例，确保 Blockly 完全初始化
  setTimeout(() => {
    loadDemo();
  }, 100);
});

onUnmounted(() => {
  if (workspace) {
    workspace.dispose();
  }
});
</script>

<style scoped>
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
  flex: 1;
  height: calc(100vh - 120px);
}

.blockly-container {
  flex: 2;
  min-width: 0;
}

.code-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 300px;
  border-left: 2px solid #bdc3c7;
  background: #f8f9fa;
}

.code-section,
.output-section {
  padding: 15px;
  border-bottom: 1px solid #dee2e6;
}

.code-section {
  flex: 2;
}

.output-section {
  flex: 1;
}

h3 {
  margin: 0 0 10px 0;
  color: #2c3e50;
  font-size: 1.1em;
}

pre {
  background: #2d2d2d;
  color: #f8f8f2;
  padding: 15px;
  border-radius: 5px;
  overflow-x: auto;
  height: calc(100% - 40px);
  margin: 0;
  font-family: "Courier New", monospace;
  font-size: 12px;
  line-height: 1.4;
}

.output {
  background: white;
  border: 1px solid #dee2e6;
  padding: 15px;
  border-radius: 5px;
  min-height: 80px;
  white-space: pre-wrap;
  font-family: "Courier New", monospace;
  font-size: 12px;
  margin-bottom: 10px;
  line-height: 1.4;
}

.run-btn {
  padding: 8px 16px;
  background: #27ae60;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.3s;
}

.run-btn:hover {
  background: #219a52;
}

@media (max-width: 768px) {
  .container {
    flex-direction: column;
  }

  .code-panel {
    border-left: none;
    border-top: 2px solid #bdc3c7;
    min-height: 300px;
  }
}
</style>
