# 5.7 前端：聊天界面

## 仿 ChatGPT 气泡布局 + 流式打字机效果

`src/views/Chat.vue`：

```vue
<template>
  <div class="chat-container">
    <!-- 消息列表 -->
    <div class="messages" ref="messagesRef">
      <div v-if="messages.length === 0" class="empty-hint">
        <el-icon size="48"><ChatDotRound /></el-icon>
        <p>你好！我是校园问答助手，有什么可以帮到你？</p>
      </div>
      
      <div
        v-for="msg in messages"
        :key="msg.id"
        :class="['message', msg.role]"
      >
        <div class="avatar">
          <span v-if="msg.role === 'user'">👤</span>
          <span v-else>🤖</span>
        </div>
        <div class="bubble">
          <div class="content" v-html="renderMarkdown(msg.content)"></div>
          <div v-if="msg.role === 'assistant' && msg.streaming" class="cursor">▌</div>
        </div>
      </div>
    </div>

    <!-- 输入框 -->
    <div class="input-area">
      <el-input
        v-model="inputText"
        type="textarea"
        :rows="3"
        placeholder="输入问题，按 Enter 发送（Shift+Enter 换行）"
        :disabled="isStreaming"
        @keydown.enter.exact.prevent="sendMessage"
        resize="none"
      />
      <el-button
        type="primary"
        :loading="isStreaming"
        @click="sendMessage"
        :disabled="!inputText.trim()"
      >
        {{ isStreaming ? '回答中...' : '发送' }}
      </el-button>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import { marked } from 'marked'
import DOMPurify from 'dompurify'

const messages = ref([])
const inputText = ref('')
const isStreaming = ref(false)
const messagesRef = ref()

function renderMarkdown(text) {
  return DOMPurify.sanitize(marked(text || ''))
}

function scrollToBottom() {
  nextTick(() => {
    if (messagesRef.value) {
      messagesRef.value.scrollTop = messagesRef.value.scrollHeight
    }
  })
}

async function sendMessage() {
  const question = inputText.value.trim()
  if (!question || isStreaming.value) return
  
  inputText.value = ''
  
  // 添加用户消息
  messages.value.push({ id: Date.now(), role: 'user', content: question })
  scrollToBottom()
  
  // 添加 AI 消息占位
  const aiMsg = { id: Date.now() + 1, role: 'assistant', content: '', streaming: true }
  messages.value.push(aiMsg)
  isStreaming.value = true
  
  try {
    // 使用 fetch + ReadableStream 接收 SSE
    const token = localStorage.getItem('token')
    const response = await fetch('http://localhost:8000/api/chat', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ question })
    })
    
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    
    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    
    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      
      const text = decoder.decode(value)
      const lines = text.split('\n')
      
      for (const line of lines) {
        if (!line.startsWith('data: ')) continue
        const data = JSON.parse(line.slice(6))
        
        if (data.type === 'text') {
          aiMsg.content += data.content
          scrollToBottom()
        } else if (data.type === 'done') {
          aiMsg.streaming = false
        } else if (data.type === 'error') {
          aiMsg.content = '❌ 发生错误：' + data.content
          aiMsg.streaming = false
        }
      }
    }
  } catch (e) {
    aiMsg.content = '❌ 网络错误，请稍后重试'
    aiMsg.streaming = false
  } finally {
    isStreaming.value = false
  }
}
</script>

<style scoped>
.chat-container {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 60px);
  max-width: 860px;
  margin: 0 auto;
  padding: 16px;
}

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 16px 0;
}

.message {
  display: flex;
  gap: 12px;
  margin-bottom: 20px;
}

.message.user {
  flex-direction: row-reverse;
}

.bubble {
  max-width: 72%;
  padding: 12px 16px;
  border-radius: 12px;
  line-height: 1.6;
}

.message.user .bubble {
  background: #409eff;
  color: white;
  border-radius: 12px 4px 12px 12px;
}

.message.assistant .bubble {
  background: #f5f5f5;
  border-radius: 4px 12px 12px 12px;
}

.cursor {
  display: inline;
  animation: blink 1s infinite;
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0; }
}

.input-area {
  display: flex;
  gap: 8px;
  padding-top: 16px;
  border-top: 1px solid #eee;
}
</style>
```

---

## 安装依赖

```bash
npm install marked dompurify
```

---

## 第5章小结

🎉 Day 5 完成！系统真正「活」了！

**今天你实现了整个 RAG 问答链路**：
- ✅ RAG 服务：检索 + 构建 Prompt + LLM 流式生成
- ✅ 防幻觉 Prompt：明确约束 AI 只基于资料回答
- ✅ SSE 流式接口：`/api/chat`
- ✅ 问答记录保存
- ✅ 仿 ChatGPT 聊天界面（气泡布局 + 打字机效果）

**现在你可以**：上传一份学校文件，然后用自然语言提问，系统会根据文件内容给出准确回答。

**明天（Day 6）** 是最后一天：联调测试、UI 优化、打包部署、项目总结。

---

_第5章完_
