<template>
  <div class="blockly-demo">
    <div class="header">
      <h2>六轴机械臂可视化编程</h2>
      <div class="controls">
        <button @click="generateCode">生成代码</button>
        <button @click="executeCode">执行命令</button>
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
          <h3>生成的机械臂指令:</h3>
          <pre><code>{{ generatedCode }}</code></pre>
        </div>
        
        <div class="output-section">
          <h3>执行结果:</h3>
          <div class="output">{{ executionResult }}</div>
          <div class="joint-display">
            <h4>当前关节角度:</h4>
            <div class="joint-values-horizontal">
              <div v-for="(value, index) in jointValues" :key="index" class="joint-item">
                <span class="joint-label">J{{ index + 1 }}</span>
                <span class="joint-value">{{ value }}°</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, reactive } from 'vue'

// 正确导入 Blockly
import * as Blockly from 'blockly'
import { javascriptGenerator } from 'blockly/javascript'
import 'blockly/blocks'

// DOM 引用
const blocklyDiv = ref(null)

// 响应式数据
const generatedCode = ref('// 生成的机械臂指令将显示在这里')
const executionResult = ref('')
const jointValues = reactive([0, 0, 0, 0, 0, 0])
let workspace = null

// 自定义积木块定义
const createCustomBlocks = () => {
  // 开始模块 - 改为必须有后续连接
  Blockly.Blocks['robot_start'] = {
    init: function() {
      this.appendDummyInput()
        .appendField("🔰 开始程序")
      this.setNextStatement(true, null)
      this.setColour(230)
      this.setTooltip("机械臂程序开始点，必须连接其他模块")
      this.setHelpUrl("")
    }
  }

  // 设置关节角度模块 - 必须有前后连接
  Blockly.Blocks['set_joints'] = {
    init: function() {
      this.appendDummyInput()
        .appendField("🦾 设置关节角度: ")
        .appendField("J1:")
        .appendField(new Blockly.FieldNumber(0, -180, 180), "J1")
        .appendField("J2:")
        .appendField(new Blockly.FieldNumber(0, -180, 180), "J2")
        .appendField("J3:")
        .appendField(new Blockly.FieldNumber(0, -180, 180), "J3")
        .appendField("J4:")
        .appendField(new Blockly.FieldNumber(0, -180, 180), "J4")
        .appendField("J5:")
        .appendField(new Blockly.FieldNumber(0, -180, 180), "J5")
        .appendField("J6:")
        .appendField(new Blockly.FieldNumber(0, -180, 180), "J6")
      
      this.setPreviousStatement(true, null)
      this.setNextStatement(true, null)
      this.setColour(160)
      this.setTooltip("设置六个关节的角度值 (-180° 到 180°)")
      this.setHelpUrl("")
    }
  }

  // 打印关节角度模块 - 必须有前后连接
  Blockly.Blocks['print_joints'] = {
    init: function() {
      this.appendDummyInput()
        .appendField("📋 打印关节角度")
      this.setPreviousStatement(true, null)
      this.setNextStatement(true, null)
      this.setColour(120)
      this.setTooltip("打印当前设置的关节角度")
      this.setHelpUrl("")
    }
  }

  // 延时模块 - 必须有前后连接
  Blockly.Blocks['delay'] = {
    init: function() {
      this.appendDummyInput()
        .appendField("⏱️ 延时")
        .appendField(new Blockly.FieldNumber(2, 0.1, 60, 0.1), "DELAY_TIME")
        .appendField("秒")
      this.setPreviousStatement(true, null)
      this.setNextStatement(true, null)
      this.setColour(65)
      this.setTooltip("程序暂停指定的秒数，然后继续执行下一个指令")
      this.setHelpUrl("")
    }
  }
}

// 自定义代码生成器
const createCustomGenerators = () => {
  // 开始模块代码生成
  javascriptGenerator.forBlock['robot_start'] = function(block) {
    return 'await startProgram();\n'
  }

  // 设置关节角度代码生成
  javascriptGenerator.forBlock['set_joints'] = function(block) {
    const j1 = block.getFieldValue('J1')
    const j2 = block.getFieldValue('J2')
    const j3 = block.getFieldValue('J3')
    const j4 = block.getFieldValue('J4')
    const j5 = block.getFieldValue('J5')
    const j6 = block.getFieldValue('J6')
    
    return `await setJoints([${j1}, ${j2}, ${j3}, ${j4}, ${j5}, ${j6}]);\n`
  }

  // 打印关节角度代码生成
  javascriptGenerator.forBlock['print_joints'] = function(block) {
    return 'await printJoints();\n'
  }

  // 延时代码生成
  javascriptGenerator.forBlock['delay'] = function(block) {
    const delayTime = block.getFieldValue('DELAY_TIME')
    return `await delay(${delayTime});\n`
  }
}

// 工具箱配置
const toolbox = {
  kind: 'categoryToolbox',
  contents: [
    {
      kind: 'category',
      name: '🔰 程序控制',
      colour: '#4A148C',
      contents: [
        { kind: 'block', type: 'robot_start' },
        { kind: 'block', type: 'set_joints' },
        { kind: 'block', type: 'print_joints' },
        { kind: 'block', type: 'delay' }
      ]
    }
  ]
}

// 初始化 Blockly
const initBlockly = () => {
  if (!blocklyDiv.value) return

  // 创建自定义积木块
  createCustomBlocks()
  createCustomGenerators()

  workspace = Blockly.inject(blocklyDiv.value, {
    toolbox: toolbox,
    grid: {
      spacing: 20,
      length: 3,
      colour: '#ccc',
      snap: true
    },
    zoom: {
      controls: true,
      wheel: true,
      startScale: 1.0,
      maxScale: 3,
      minScale: 0.3,
      scaleSpeed: 1.2
    },
    move: {
      scrollbars: true,
      drag: true,
      wheel: true
    },
    // 禁用声音和远程媒体
    sounds: false,
    renderer: 'geras',
    theme: Blockly.Themes.Classic
  })

  // 监听工作区变化
  workspace.addChangeListener((event) => {
    if (!event.isUiEvent) {
      generateCode()
    }
  })
}

// 生成代码 - 只生成从开始模块连接的代码
const generateCode = () => {
  if (workspace) {
    try {
      // 找到所有开始模块
      const startBlocks = workspace.getTopBlocks(true).filter(block => block.type === 'robot_start')
      
      if (startBlocks.length === 0) {
        generatedCode.value = '// 请添加"开始程序"模块并连接其他模块'
        return
      }

      let allCode = ''
      
      // 为每个开始模块生成代码
      startBlocks.forEach(startBlock => {
        // 只生成从开始模块连接的代码链
        const code = javascriptGenerator.blockToCode(startBlock)
        if (code) {
          allCode += code + '\n'
        }
      })

      generatedCode.value = allCode || '// 请连接其他模块到"开始程序"模块'
      console.log('生成的代码:', generatedCode.value)
    } catch (error) {
      console.error('生成代码错误:', error)
      generatedCode.value = `// 生成代码时出错:\n// ${error.message}`
    }
  }
}

// 开始程序函数
const startProgram = async () => {
  return new Promise((resolve) => {
    const result = "🔰 开始程序执行"
    executionResult.value += result + '\n'
    console.log(result)
    resolve()
  })
}

// 设置关节角度函数
const setJoints = async (angles) => {
  return new Promise((resolve) => {
    if (angles && angles.length === 6) {
      angles.forEach((angle, index) => {
        jointValues[index] = parseInt(angle) || 0
      })
      const result = `🦾 设置关节角度: J1:${angles[0]}° J2:${angles[1]}° J3:${angles[2]}° J4:${angles[3]}° J5:${angles[4]}° J6:${angles[5]}°`
      executionResult.value += result + '\n'
      console.log(result)
    } else {
      const result = '❌ 错误: 需要6个关节角度值'
      executionResult.value += result + '\n'
      console.log(result)
    }
    resolve()
  })
}

// 打印关节角度函数
const printJoints = async () => {
  return new Promise((resolve) => {
    const result = `📋 当前关节角度: J1:${jointValues[0]}° J2:${jointValues[1]}° J3:${jointValues[2]}° J4:${jointValues[3]}° J5:${jointValues[4]}° J6:${jointValues[5]}°`
    executionResult.value += result + '\n'
    console.log(result)
    resolve()
  })
}

// 延时函数 - 修复：实际等待
const delay = async (seconds) => {
  return new Promise((resolve) => {
    const startTime = new Date().getTime()
    
    const updateTimer = () => {
      const currentTime = new Date().getTime()
      const elapsed = (currentTime - startTime) / 1000
      const remaining = (seconds - elapsed).toFixed(1)
      
      executionResult.value = executionResult.value.split('\n')
        .filter(line => !line.includes('⏰ 剩余'))
        .join('\n')
      
      if (remaining > 0) {
        executionResult.value += `\n⏰ 剩余 ${remaining} 秒...`
        setTimeout(updateTimer, 100)
      } else {
        executionResult.value = executionResult.value.split('\n')
          .filter(line => !line.includes('⏰ 剩余'))
          .join('\n')
        executionResult.value += `\n✅ 延时 ${seconds} 秒结束`
        console.log(`✅ 延时 ${seconds} 秒结束`)
        resolve()
      }
    }
    
    executionResult.value += `\n⏱️ 开始延时 ${seconds} 秒`
    console.log(`⏱️ 开始延时 ${seconds} 秒`)
    updateTimer()
  })
}

// 执行代码
const executeCode = async () => {
  try {
    // 清空之前的执行结果
    executionResult.value = "🔄 开始执行程序...\n"
    
    console.log('执行代码:', generatedCode.value)

    // 创建执行环境
    const executeEnv = {
      startProgram,
      setJoints,
      printJoints,
      delay,
      console: {
        log: (...args) => {
          executionResult.value += args.join(' ') + '\n'
        }
      }
    }

    // 执行生成的代码
    if (generatedCode.value && !generatedCode.value.startsWith('//')) {
      try {
        // 将代码包装成异步函数
        const asyncCode = `
          return (async function() {
            ${generatedCode.value}
          })()
        `
        
        console.log('准备执行代码:', asyncCode)
        
        const func = new Function(...Object.keys(executeEnv), asyncCode)
        await func(...Object.values(executeEnv))
        
        executionResult.value += '✅ 程序执行完成\n'
      } catch (e) {
        console.error('执行错误:', e)
        executionResult.value += `❌ 执行错误: ${e.message}\n`
      }
    } else {
      executionResult.value += '❌ 没有可执行的代码\n'
    }
  } catch (error) {
    console.error('执行错误:', error)
    executionResult.value += `❌ 执行错误: ${error.message}\n`
  }
}

// 清空工作区
const clearWorkspace = () => {
  if (workspace) {
    workspace.clear()
    generatedCode.value = '// 生成的机械臂指令将显示在这里'
    executionResult.value = ''
    // 重置关节角度
    jointValues.splice(0, jointValues.length, ...[0, 0, 0, 0, 0, 0])
  }
}

// 创建积木块并设置位置的辅助函数
const createBlock = (type, x, y) => {
  const block = workspace.newBlock(type)
  block.moveBy(x, y)
  block.initSvg()
  block.render()
  return block
}

// 加载示例 - 包含延迟的示例
const loadDemo = () => {
  if (!workspace) return

  clearWorkspace()

  try {
    // 创建开始模块
    const startBlock = createBlock('robot_start', 50, 50)

    // 创建关节设置模块
    const jointsBlock = createBlock('set_joints', 50, 120)
    jointsBlock.setFieldValue('90', 'J1')
    jointsBlock.setFieldValue('45', 'J2')
    jointsBlock.setFieldValue('-30', 'J3')
    jointsBlock.setFieldValue('15', 'J4')
    jointsBlock.setFieldValue('-10', 'J5')
    jointsBlock.setFieldValue('5', 'J6')

    // 创建延迟模块
    const delayBlock = createBlock('delay', 50, 220)
    delayBlock.setFieldValue('3', 'DELAY_TIME')

    // 创建打印模块
    const printBlock = createBlock('print_joints', 50, 320)

    console.log('加载示例完成')
    
    // 手动连接所有模块
    startBlock.nextConnection.connect(jointsBlock.previousConnection)
    jointsBlock.nextConnection.connect(delayBlock.previousConnection)
    delayBlock.nextConnection.connect(printBlock.previousConnection)
    
    generateCode()
  } catch (error) {
    console.error('加载示例时出错:', error)
    executionResult.value = `❌ 加载示例错误: ${error.message}`
  }
}

// 保存工作区
const saveWorkspace = () => {
  if (workspace) {
    try {
      const data = Blockly.serialization.workspaces.save(workspace)
      localStorage.setItem('blocklyWorkspace', JSON.stringify(data))
      alert('✅ 工作区已保存！')
    } catch (error) {
      console.error('保存工作区时出错:', error)
      alert('❌ 保存失败！')
    }
  }
}

// 加载工作区
const loadWorkspace = () => {
  if (workspace) {
    try {
      const data = localStorage.getItem('blocklyWorkspace')
      if (data) {
        Blockly.serialization.workspaces.load(JSON.parse(data), workspace)
        alert('✅ 工作区已加载！')
        generateCode()
      } else {
        alert('❌ 没有找到保存的工作区')
      }
    } catch (error) {
      console.error('加载工作区时出错:', error)
      alert('❌ 加载失败！')
    }
  }
}

// 生命周期
onMounted(() => {
  initBlockly()
  setTimeout(() => {
    loadDemo()
  }, 100)
})

onUnmounted(() => {
  if (workspace) {
    workspace.dispose()
  }
})
</script>

<style scoped>
/* 样式保持不变 */
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
  min-width: 350px;
  border-left: 2px solid #bdc3c7;
  background: #f8f9fa;
}

.code-section, .output-section {
  padding: 15px;
  border-bottom: 1px solid #dee2e6;
}

.code-section {
  flex: 2;
}

.output-section {
  flex: 1;
}

h3, h4 {
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
  font-family: 'Courier New', monospace;
  font-size: 12px;
  line-height: 1.4;
}

.output {
  background: white;
  border: 1px solid #dee2e6;
  padding: 15px;
  border-radius: 5px;
  min-height: 60px;
  white-space: pre-wrap;
  font-family: 'Courier New', monospace;
  font-size: 12px;
  margin-bottom: 10px;
  line-height: 1.4;
  max-height: 300px;
  overflow-y: auto;
}

.joint-display {
  margin-top: 10px;
}

.joint-values-horizontal {
  display: flex;
  justify-content: space-between;
  gap: 5px;
  margin-top: 8px;
  flex-wrap: wrap;
}

.joint-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 8px;
  background: white;
  border: 1px solid #dee2e6;
  border-radius: 6px;
  min-width: 50px;
}

.joint-label {
  font-weight: bold;
  color: #2c3e50;
  font-size: 12px;
  margin-bottom: 4px;
}

.joint-value {
  color: #e74c3c;
  font-weight: bold;
  font-size: 14px;
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
  
  .joint-values-horizontal {
    justify-content: center;
  }
}
</style>