<!-- ./src/components/Chathead.vue -->
<template>
  <!-- 新增：外层椭圆容器（带阴影） -->
  <div class="seg-outer-container" :style="outerContainerStyle">
    <!-- 原有的分段选择器容器 -->
    <div class="segmented-container" :style="containerStyle">
      <button
        v-for="item, in options"
        :key="item.key"
        class="seg-btn"
        :class="{ active: activeKey === item.key }"
        @click="handleBtnClick(item.key)"
        :style="btnStyle"
      >
        <span v-if="item.icon" class="seg-icon">{{ item.icon }}</span>
        <span class="seg-text">{{ item.label }}</span>
      </button>
      <div
        class="seg-bg"
        :style="{
          left: bgLeft,
          width: bgWidth,
          transition: `left ${duration}ms ease`
        }"
      ></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, watch } from 'vue';

// 1. 定义选项的接口
interface OptionItem {
  key: string;
  label: string;
  icon?: string;
}

// 2. 显式定义 Props
// 使用这种写法能确保模板 100% 识别到 options
const props = defineProps({
  options: {
    type: Array as () => OptionItem[],
    required: true,
    default: () => []
  },
  modelValue: { type: String, default: '' },
  bgColor: { type: String, default: '#5a7bff' },
  textColor: { type: String, default: '#666' },
  activeTextColor: { type: String, default: '#fff' },
  containerBg: { type: String, default: '#f5f5f5' },
  duration: { type: Number, default: 300 },
  padding: { type: String, default: '8px 20px' }
});

const outerContainerStyle = {}; 

const emit = defineEmits(['update:modelValue', 'change']);

// 3. 计算激活的 key
const activeKey = computed({
  get() {
    return props.options.some(item => item.key === props.modelValue) 
      ? props.modelValue 
      : props.options[0]?.key || '';
  },
  set(val: string) {
    emit('update:modelValue', val);
    emit('change', val);
  }
});

// 4. 背景宽度
const bgWidth = computed(() => {
  const len = props.options?.length || 1;
  return `calc(${100 / len}% - 8px)`;
});

// 5. 左侧偏移量
const bgLeft = computed(() => {
  const len = props.options?.length || 1;
  const activeIndex = props.options.findIndex(item => item.key === activeKey.value);
  return `calc(${(activeIndex === -1 ? 0 : activeIndex) * (100 / len)}% + 4px)`;
});

const containerStyle = computed(() => ({
  backgroundColor: props.containerBg,
}));

const btnStyle = computed(() => ({
  color: props.textColor,
  padding: props.padding,
}));

const handleBtnClick = (key: string) => {
  activeKey.value = key;
};

watch(activeKey, (newVal) => {
  console.log('分段选择器选中项变更：', newVal);
});
</script>


<style scoped>
/* 新增：外层椭圆容器样式 */
.seg-outer-container {
  display: inline-block;
  border-radius: 999px; /* 外层椭圆圆角 */
  padding: 1px; /* 预留阴影空间 */
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08); /* 柔和阴影效果 */
  background: #fff; /* 外层背景色，与页面背景融合 */
}

/* 容器样式：椭圆外层容器 */
.segmented-container {
  position: relative;
  display: inline-flex;
  border-radius: 999px; /* 椭圆圆角，数值足够大就是椭圆 */
  padding: 4px; /* 容器内边距，给背景椭圆留空间 */
  overflow: hidden; /* 隐藏超出容器的内容 */
  box-sizing: border-box;
}

/* 按钮样式：基础未选中状态 */
.seg-btn {
  position: relative; /* 用于层级控制 */
  z-index: 1; /* 保证按钮在背景椭圆上方 */
  display: inline-flex;
  align-items: center;
  gap: 6px; /* 图标和文字的间距 */
  border: none; /* 去掉默认边框 */
  background: transparent; /* 透明背景 */
  font-size: 14px;
  cursor: pointer; /* 鼠标悬浮手型 */
  transition: color 0.3s ease; /* 文字颜色过渡动画 */
  white-space: nowrap; /* 防止文字换行 */
  box-sizing: border-box;
}

/* 按钮激活状态：文字颜色变化 */
.seg-btn.active {
  color: v-bind(activeTextColor) !important; /* 绑定选中文字颜色 */
  font-weight: 500; /* 选中时文字加粗（可选） */
}

/* 图标样式（可选） */
.seg-icon {
  font-size: 12px;
}

/* 文字样式（可选） */
.seg-text {
  line-height: 1;
}

/* 椭圆背景样式：动态位移的核心元素 */
.seg-bg {
  position: absolute;
  top: 4px; /* 和容器内边距对应 */
  height: calc(100% - 8px); /* 减去上下内边距，保证背景椭圆不超出容器 */
  background: v-bind(bgColor); /* 绑定背景颜色 */
  border-radius: 999px; /* 椭圆圆角 */
  box-sizing: border-box;
}
</style>