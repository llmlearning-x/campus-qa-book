# 4.5 后台管理：知识库管理页面

## 文档上传与状态展示

`src/views/admin/Knowledge.vue`：

```vue
<template>
  <div class="knowledge-page">
    <div class="header">
      <h2>知识库管理</h2>
      <el-upload
        :action="uploadUrl"
        :headers="uploadHeaders"
        :on-success="onUploadSuccess"
        :on-error="onUploadError"
        :before-upload="beforeUpload"
        :show-file-list="false"
        accept=".pdf,.txt,.docx"
      >
        <el-button type="primary" icon="Plus">上传文档</el-button>
      </el-upload>
    </div>

    <!-- 文档列表 -->
    <el-table :data="documents" v-loading="loading" stripe style="margin-top:16px">
      <el-table-column prop="filename" label="文件名" show-overflow-tooltip />
      <el-table-column label="大小" width="100">
        <template #default="{ row }">
          {{ formatSize(row.file_size) }}
        </template>
      </el-table-column>
      <el-table-column label="状态" width="120">
        <template #default="{ row }">
          <el-tag :type="statusType(row.status)">{{ statusLabel(row.status) }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="chunk_count" label="片段数" width="100" />
      <el-table-column label="上传时间" width="180">
        <template #default="{ row }">{{ row.created_at }}</template>
      </el-table-column>
      <el-table-column label="操作" width="120">
        <template #default="{ row }">
          <el-button size="small" type="danger" @click="deleteDoc(row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <el-pagination
      v-model:current-page="page"
      :total="total"
      @change="loadDocuments"
      layout="total, prev, pager, next"
      style="margin-top:16px"
    />
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import request from '@/api/request'

const documents = ref([])
const total = ref(0)
const page = ref(1)
const loading = ref(false)

const uploadUrl = 'http://localhost:8000/api/knowledge/documents'
const uploadHeaders = computed(() => ({
  Authorization: `Bearer ${localStorage.getItem('token')}`
}))

async function loadDocuments() {
  loading.value = true
  try {
    const res = await request.get('/api/knowledge/documents', { params: { page: page.value } })
    documents.value = res.data.items
    total.value = res.data.total
  } finally {
    loading.value = false
  }
}

function beforeUpload(file) {
  const isValid = ['application/pdf', 'text/plain',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
  ].includes(file.type)
  if (!isValid) {
    ElMessage.error('只支持 PDF、TXT、DOCX 格式')
    return false
  }
  if (file.size > 50 * 1024 * 1024) {
    ElMessage.error('文件大小不能超过 50MB')
    return false
  }
  return true
}

function onUploadSuccess() {
  ElMessage.success('上传成功，正在后台处理...')
  loadDocuments()
}

function onUploadError() {
  ElMessage.error('上传失败')
}

async function deleteDoc(doc) {
  await ElMessageBox.confirm(`确定删除「${doc.filename}」？此操作不可恢复`, '警告', { type: 'warning' })
  await request.delete(`/api/knowledge/documents/${doc.id}`)
  ElMessage.success('删除成功')
  loadDocuments()
}

// 工具函数
function formatSize(bytes) {
  if (!bytes) return '-'
  if (bytes < 1024) return bytes + 'B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + 'KB'
  return (bytes / 1024 / 1024).toFixed(1) + 'MB'
}

function statusType(s) {
  return { pending: 'info', processing: 'warning', done: 'success', failed: 'danger' }[s] || 'info'
}

function statusLabel(s) {
  return { pending: '等待处理', processing: '处理中', done: '已完成', failed: '处理失败' }[s] || s
}

onMounted(loadDocuments)
</script>
```

## 4.6 前端文档上传（已整合在上方）

上面的 `Knowledge.vue` 已包含完整的文档上传与管理功能。

---

## 第4章小结

🎉 Day 4 完成！

**你实现了 RAG 系统中最核心的「记忆建立」过程**：

- ✅ 文档上传接口（支持 PDF/TXT/DOCX）
- ✅ 文本切分 Pipeline（LangChain + RecursiveCharacterTextSplitter）
- ✅ Embedding 服务（OpenAI API）
- ✅ FAISS 向量库（持久化存储 + 语义检索）
- ✅ 知识库管理前端页面

**明天（Day 5）** 是整本书最令人期待的一章：我们将把所有组件串联起来，实现完整的 RAG 问答流程，用户第一次能真正问出「食堂几点开门？」并得到准确回答！

---

_第4章完_
