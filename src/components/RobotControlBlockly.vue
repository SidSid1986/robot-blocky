<template>
  <div class="control-panel">
    <div v-for="(joint, index) in joints" :key="index" class="slider-container">
      <label>{{ joint.label }}</label>
      <input
        style="width: 100px"
        type="range"
        v-model.number="jointValues[index]"
        :min="joint.min"
        :max="joint.max"
        :step="joint.step"
        @input="updateJoint(joint.name, jointValues[index], jointValues)"
      />
      <span>{{ Number(jointValues[index]).toFixed(2) }} rad</span>
    </div>

    <!-- 夹爪控制 -->

    <div class="button-area">
      <el-button @click="robotReset" type="warning">
        <Refresh style="width: 1em; height: 1em; margin-right: 8px" />
        机械臂复位
      </el-button>

      <el-button @click="startDemo" type="primary" style="margin-left: 10px">
        <VideoPlay style="width: 1em; height: 1em; margin-right: 8px" />
        开始演示
      </el-button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from "vue";
import { Refresh, VideoPlay } from "@element-plus/icons-vue"; // 引入刷新图标
import { ElButton } from "element-plus"; // 引入 Element Plus 按钮组件
import demoTrajectory from "./demo-trajectory.json";

const props = defineProps({
  jointArr: {
    type: Array,
    default: () => [],
  },
});

// 定义组件向父级传递的事件
const emit = defineEmits(["joint-change", "gripper-change", "reset-all"]);
// 当前正在插值的关节值
const currentJointValues = ref([0.0, 0.0, 1.57, 0.0, 1.57, 0.0]);

// 目标关节值（当前帧的目标）
const targetJointValues = ref([0.0, 0.0, 1.57, 0.0, 1.57, 0.0]);

// 当前轨迹帧索引
const currentFrameIndex = ref(0);

// 插值因子（越小越平滑，比如 0.02 ~ 0.05）
const lerpFactor = 0.03;

// 动画循环 ID（用于取消）
let animationFrameId = null;

// 定义所有可控制的关节信息

const joints = ref([
  {
    name: "joint1",
    label: "joint1",
    min: -3.04,
    max: 3.04,
    step: 0.01,
  },
  {
    name: "joint2",
    label: "joint2",
    min: -3.04,
    max: 3.04,
    step: 0.01,
  },
  {
    name: "joint3",
    label: "joint3",
    min: -3.04,
    max: 3.04,
    step: 0.01,
  },
  { name: "joint4", label: "joint4", min: -3.04, max: 3.04, step: 0.01 },
  { name: "joint5", label: "joint5", min: -3.04, max: 3.04, step: 0.01 },
  { name: "joint6", label: "joint6", min: -3.04, max: 3.04, step: 0.01 },
]);

// 当前各个关节的值，双向绑定到滑动条
const jointValues = ref(
  [
    0.0, // shoulder_joint
    0.0, // upperArm_joint
    0.0, // foreArm_joint
    0.0, // wrist1_joint
    0.0, // wrist2_joint
    0.0, // wrist3_joint
  ].map(Number)
);

// 所有关节的初始位置定义，用于复位时传递给父组件

const INITIAL_POSITIONS = {
  joint1: 0.0,
  joint2: 0.0,
  joint3: 0.0,
  joint4: 0.0,
  joint5: 0.0,
  joint6: 0.0,
  joint7: 0.0, // 夹爪关节
};

let isDemoRunning = ref(false); // 防止重复点击

/**
 * 机械臂复位功能：将所有关节重置为初始角度，夹爪闭合
 */
const robotReset = () => {
  console.log("机械臂复位");

  // 重置六个主要关节的值为初始值（单位：弧度）
  jointValues.value = [
    0.0, // shoulder_joint
    0.0, // upperArm_joint
    0.0, // foreArm_joint
    0.0, // wrist1_joint
    0.0, // wrist2_joint
    0.0, // wrist3_joint
  ];

  // 向父组件发送复位事件，并传递所有关节的初始值
  emit("reset-all", INITIAL_POSITIONS);
};

/**
 * 更新某个关节的角度
 */
// const updateJoint = (jointName, value, jointValues) => {
//   console.log(`更新关节 ${jointName} 的角度为 ${value} rad`);
//   console.log("所有轴的数据", jointValues);

//   emit("joint-change", {
//     jointName,
//     angle: parseFloat(value), // 确保传递的是数值类型
//     jointValues,
//   });
// };
const updateJoint = (jointName, value, jointValues) => {
  // 我们仍然传 jointName 和 value 用于日志等，但真正有用的是 jointValues
  // console.log(`更新关节 ${jointName} 的角度为 ${value} rad`);
  console.log("所有轴的数据", jointValues);

  //  将整个 jointValues 数组传给父组件，父组件会一次性更新所有关节
  emit("joint-change", {
    jointValues: jointValues.map(Number), // 确保是数字类型
  });
};

/**
 * 平滑插值执行函数
 */
const smoothDemoLoop = () => {
  if (!isDemoRunning.value) return;

  // 获取当前目标帧
  if (currentFrameIndex.value < demoTrajectory.length) {
    targetJointValues.value = demoTrajectory[currentFrameIndex.value];
  } else {
    // 所有帧执行完毕
    console.log("码垛操作完成（所有轨迹帧执行完毕）");
    isDemoRunning.value = false;
    if (animationFrameId) cancelAnimationFrame(animationFrameId);
    return;
  }

  // 对每个关节进行 lerp 插值
  const newJointValues = currentJointValues.value.map((current, i) => {
    const target = targetJointValues.value[i];
    return current + (target - current) * lerpFactor; // lerp 公式
  });

  // 更新机械臂关节（通过父组件）
  emit("joint-change", {
    jointValues: newJointValues,
  });

  // 更新当前值
  currentJointValues.value = newJointValues;

  // 检查是否足够接近目标，可设置一个阈值，比如 0.01
  const isClose = targetJointValues.value.every(
    (t, i) => Math.abs(t - newJointValues[i]) < 0.01
  );

  if (isClose) {
    // 接近目标，切换到下一帧
    currentFrameIndex.value++;
  }

  // 请求下一帧动画
  animationFrameId = requestAnimationFrame(smoothDemoLoop);
};

/**
 * 开始演示 - 逐步改变关节值到目标位置
 */
const startDemo = (jointArr) => {
  if (isDemoRunning.value) return;

  // 目标关节值demo
  // const targetPositions = [-1.45, -0.43, 1.45, -1.45, -1.3, -1.95];

  const targetPositions = jointArr;

  // 检查是否已经在目标位置
  const isAtTarget = jointValues.value.every(
    (current, index) => Math.abs(current - targetPositions[index]) < 0.01
  );

  if (isAtTarget) {
    console.log("已经在目标位置，无需移动");
    return;
  }

  isDemoRunning.value = true;
  currentFrameIndex.value = 0;

  // 使用插值动画逐步移动到目标位置
  smoothDemoLoopToTarget(targetPositions);
};

/**
 * 平滑插值到目标位置
 */
const smoothDemoLoopToTarget = (targetPositions) => {
  // 插值参数
  const lerpFactor = 0.05; // 调整这个值可以改变动画速度
  const stepSize = 0.1; // 步长

  const animate = () => {
    if (!isDemoRunning.value) return;

    let allReached = true;

    // 对每个关节进行插值
    const newJointValues = jointValues.value.map((current, index) => {
      const target = targetPositions[index];
      const difference = target - current;

      // 如果差距很小，直接设为目标值
      if (Math.abs(difference) < stepSize) {
        return target;
      }

      allReached = false;

      // 按步长递增/递减
      if (difference > 0) {
        return Math.min(current + stepSize, target);
      } else {
        return Math.max(current - stepSize, target);
      }
    });

    // 更新当前关节值
    jointValues.value = newJointValues;

    // 发送更新到父组件
    emit("joint-change", {
      jointValues: newJointValues.map(Number),
    });

    if (allReached) {
      // 到达目标位置
      console.log("✅ 已到达目标位置");
      isDemoRunning.value = false;

      // 显示最终的角度值（转换为度数）
      const degrees = targetPositions.map((rad) =>
        ((rad * 180) / Math.PI).toFixed(2)
      );
      console.log("目标位置（度）:", degrees);
      console.log("目标位置（弧度）:", targetPositions);
    } else {
      // 继续动画
      requestAnimationFrame(animate);
    }
  };

  // 开始动画
  animate();
};

watch(
  () => props.jointArr,
  (newVal) => {
    if (newVal.length > 0) {
      startDemo(newVal);
    }
  },
  { deep: true, immediate: true }
);

onMounted(() => {
  // console.log(demoTrajectory);
});
</script>

<style scoped>
.control-panel {
  background: rgba(245, 245, 245, 0.95);
  height: 100%;
  overflow-y: auto;
  position: relative;
  box-shadow: 2px 0 8px rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(5px);
}

.slider-container {
  margin-bottom: 15px;
  display: flex;
  flex-direction: row;
  justify-content: flex-start;
  align-items: center;
}

/* 滑动条标签 */
.slider-container label {
  display: block;

  font-weight: bold;
}

/* 滑动条样式 */
.slider-container input[type="range"] {
  width: 100%;
  margin: 5px 0;
}

/* 数值显示 */
.slider-container span {
  font-size: 0.9em;
  color: #555;
}

/* 按钮区域样式 */
.button-area {
  margin-top: 20px;
}

/* 重置按钮悬停效果  */
.reset-all-btn:hover {
  background-color: #c0392b;
}
</style>
