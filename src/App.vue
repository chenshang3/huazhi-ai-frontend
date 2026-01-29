<script setup lang="ts">
import { ref, nextTick } from 'vue'
import HistorySidebar from './components/HistorySidebar.vue'
import CatalogPanel from './components/CatalogPanel.vue'
import MessageBubble from './components/MessageBubble.vue'
import Chathead from './components/Chathead.vue'
import ChatInput from './components/ChatInput.vue'
import { getMockData } from './api/mock.ts' 

// 1. 定义表格行数据的类型（因为 SQL 查询结果字段不固定，建议用索引签名）
interface TableRow {
  category: string;
  [key: string]: any; // 允许添加任何其他属性，如 sales, sales_qty, gmv 等
}

// 2. 定义单条消息的类型
interface Message {
  role: 'user' | 'assistant';
  content?: string;
  sql?: string;
  tableData?: TableRow[]; // 使用上面定义的灵活类型
  explanation?: string;
}

// 3. 在定义 ref 时传入这个类型
const messages = ref<Message[]>([
  {
    role: 'user',
    content: '帮我查询本月品类销量 Top3'
  },
  {
    role: 'assistant',
    sql: 'SELECT category, SUM(sales) FROM orders GROUP BY category LIMIT 3',
    tableData: [
      { category: '手机数码', sales: 14500 },
      { category: '家用电器', sales: 12000 },
      { category: '美妆个护', sales: 9800 }
    ],
    explanation: '已为您查询到本月销量排名前三的品类。'
  }
])

const selectedKey = ref('ai')
const chatScrollRef = ref<HTMLElement | null>(null)

// 【连接点 A】：存储从右侧面板传过来的“排除名单”
const excludedTables = ref<string[]>([])

// --- 2. 核心逻辑处理 ---

/**
 * 滚动到底部
 */
const scrollToBottom = async () => {
  await nextTick()
  if (chatScrollRef.value) {
    chatScrollRef.value.scrollTo({
      top: chatScrollRef.value.scrollHeight,
      behavior: 'smooth'
    })
  }
}
// --- 新增：接收历史对话切换事件 ---
const handleChatChange = (chat: ChatHistoryItem) => {
  currentChat.value = chat;
  // 同步消息列表
  messages.value = chat.messages;
  // 切换后滚动到底部
  nextTick(() => scrollToBottom());
};

// --- 改造发送逻辑：消息存入当前激活对话 ---
const handleSendText = async (data: { content: string; mode: string }) => {
  if (!data.content.trim()) return
 // 新增：如果是新对话（标题为“新对话”），自动用第一条用户消息作为标题
  if (currentChat.value.title === '新对话') {
    currentChat.value.title = data.content.length > 20 ? `${data.content.slice(0, 20)}...` : data.content;
    currentChat.value.time = '今天'; // 更新时间
  }
  // 1. 先把用户消息添加到当前对话
  const userMessage = { role: 'user', content: data.content };
  currentChat.value.messages.push(userMessage);
  messages.value = [...currentChat.value.messages]; // 触发响应式更新
  scrollToBottom()

  try {
    const res = await getMockData({ query: data.content, mode: data.mode })
    const serverData = res.data
    
    // 2. 添加AI消息到当前对话
    const aiMessage = {
      role: 'assistant',
      content: serverData.explanation || '查询已完成，结果如下：',
      sql: serverData.sql || '', 
      tableData: serverData.tableData || serverData.results || [],
      explanation: serverData.explanation || ''
    };
    currentChat.value.messages.push(aiMessage);
    messages.value = [...currentChat.value.messages]; // 触发响应式更新

  } catch (error) {
    console.error('接口请求失败，进入备用模拟模式:', error)
    const mockAiMessage = {
      role: 'assistant',
      content: `（模拟回复）关于“${data.content}”的查询结果如下：`,
      sql: "SELECT category, SUM(sales) FROM mock_table GROUP BY category;",
      tableData: [
        { category: '电子产品', sales: 1200, gmv: 500000 },
        { category: '日用百货', sales: 800, gmv: 20000 }
      ],
      explanation: "由于后端服务（localhost:8084）未启动，当前显示的是前端预设的模拟数据。"
    };
    currentChat.value.messages.push(mockAiMessage);
    messages.value = [...currentChat.value.messages]; // 触发响应式更新
  } finally {
    scrollToBottom()
  }
};
/**
 * 【连接点 B】：处理来自 CatalogPanel 的排除名单更新
 */
const handleUpdateExclude = (list: string[]) => {
  excludedTables.value = list
  console.log('App.vue 已同步最新的排除名单:', list)
}

/**
 * 【连接点 C】：处理来自 CatalogPanel 的“执行”操作
 */
const handleExecuteFeature = (featureName: string) => {
  // 当点击右侧“执行”时，直接调用 handleSendText 模拟发送
  handleSendText({
    content: `帮我执行：${featureName}`,
    mode: 'auto' // 默认使用自动模式
  })
}

// 模型切换逻辑
const handleSegChange = (aiModel: string) => {
  console.log('切换模型：', aiModel)
}

const handleModeChange = (mode: string) => {
  console.log('切换模式：', mode)
}
</script>

<template>
  <div class="app">
    <!-- 左侧：历史记录 -->
    <aside class="sidebar">
      <HistorySidebar @chat-change="handleChatChange" />
    </aside>
    
    <!-- 中间：主聊天区 -->
    <main class="main">
      <header class="chat-header">
        <Chathead
          v-model="selectedKey"
          :options="[
            { key: 'ai', label: '华智Ai', icon: '✨' },
            { key: 'ultra', label: '华智Ultra', icon: '⚡' }
          ]"
          @change="handleSegChange"
        />
      </header>

      <!-- 聊天内容滚动区 -->
      <div class="chat-scroll" ref="chatScrollRef" >
        <!-- 遍历渲染消息 -->
        <MessageBubble
          v-for="(item, index) in messages"
          :key="index"
          :data="item"
        />
        <div class="scroll-bottom-pad"></div>
      </div>
      
      <div class="chat-input">
        <ChatInput
          @mode-change="handleModeChange"
          @send-text="handleSendText"
          @scroll-to-bottom="scrollToBottom"
        />
      </div>
    </main>

    <!-- 右侧：目录/详情面板 -->
    <aside class="right">
      <!-- 关键点：监听子组件传出来的两个事件 -->
      <CatalogPanel 
        @update-exclude="handleUpdateExclude"
        @execute-feature="handleExecuteFeature"
      />
    </aside>
  </div>
</template>

<style lang="scss">
.app {
  height: 100vh;
  display: grid;
  grid-template-columns: 288px 1fr 360px;
  gap: 26px;
  padding: 22px 22px 18px;
  box-sizing: border-box;
  background: linear-gradient(180deg, #f7f9ff 0%, #f3f6ff 100%);
  overflow-y: auto;
}

.sidebar, .right {
  background: #ffffff;
  border-radius: 20px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.05);
  overflow: hidden;
}


.main {
  position: relative;
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden; // 关键：防止主容器溢出
  
  .chat-header {
    text-align: center;
    padding: 10px;
    flex-shrink: 0;
  }
 .chat-scroll {
  display: flex;
  flex-direction: column; 
  flex: 1; 
  padding: 16px;
  background: #f9fafb;
  overflow-y: auto;
  padding-bottom: 96px;
  }
  
  .chat-input {
      position: absolute; bottom: 0; left: 0; right: 0; padding: 20px; z-index: 20;
      background: linear-gradient(180deg, rgba(247,249,255,0) 0%, rgba(247,249,255,1) 50%);
  }
}
</style>