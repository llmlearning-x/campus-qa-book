# 3.5 前端页面实现

## 用户管理页面

`src/views/admin/UserManagement.vue`：

```vue
<template>
  <div class="user-management">
    <div class="header">
      <h2>用户管理</h2>
      <el-input v-model="keyword" placeholder="搜索用户名" style="width:200px" 
        @keyup.enter="loadUsers" clearable @clear="loadUsers">
        <template #prefix><el-icon><Search /></el-icon></template>
      </el-input>
    </div>
    
    <el-table :data="users" v-loading="loading" stripe>
      <el-table-column prop="id" label="ID" width="80" />
      <el-table-column prop="username" label="用户名" />
      <el-table-column prop="email" label="邮箱" />
      <el-table-column prop="role" label="角色" width="100">
        <template #default="{ row }">
          <el-tag :type="row.role === 'admin' ? 'danger' : 'info'">
            {{ row.role === 'admin' ? '管理员' : '普通用户' }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="is_active" label="状态" width="100">
        <template #default="{ row }">
          <el-tag :type="row.is_active ? 'success' : 'danger'">
            {{ row.is_active ? '正常' : '禁用' }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column label="操作" width="160">
        <template #default="{ row }">
          <el-button size="small" @click="toggleStatus(row)">
            {{ row.is_active ? '禁用' : '启用' }}
          </el-button>
        </template>
      </el-table-column>
    </el-table>
    
    <el-pagination
      v-model:current-page="page"
      v-model:page-size="pageSize"
      :total="total"
      @change="loadUsers"
      layout="total, prev, pager, next"
      style="margin-top: 16px; justify-content: flex-end; display: flex"
    />
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import request from '@/api/request'

const users = ref([])
const total = ref(0)
const page = ref(1)
const pageSize = ref(10)
const keyword = ref('')
const loading = ref(false)

async function loadUsers() {
  loading.value = true
  try {
    const data = await request.get('/api/users/', {
      params: { page: page.value, page_size: pageSize.value, keyword: keyword.value }
    })
    users.value = data.data.items
    total.value = data.data.total
  } finally {
    loading.value = false
  }
}

async function toggleStatus(user) {
  const action = user.is_active ? '禁用' : '启用'
  await ElMessageBox.confirm(`确定要${action}用户「${user.username}」吗？`)
  await request.put(`/api/users/${user.id}`, { is_active: user.is_active ? 0 : 1 })
  ElMessage.success(`${action}成功`)
  loadUsers()
}

onMounted(loadUsers)
</script>
```

---

## Pinia 用户状态管理

`src/stores/user.js`：

```javascript
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import request from '@/api/request'

export const useUserStore = defineStore('user', () => {
  const token = ref(localStorage.getItem('token'))
  const userInfo = ref(null)

  const isLoggedIn = computed(() => !!token.value)
  const isAdmin = computed(() => userInfo.value?.role === 'admin')

  async function login(username, password) {
    const data = await request.post('/api/users/login', { username, password })
    token.value = data.data.access_token
    userInfo.value = data.data.user
    localStorage.setItem('token', token.value)
  }

  function logout() {
    token.value = null
    userInfo.value = null
    localStorage.removeItem('token')
  }

  async function fetchUserInfo() {
    if (!token.value) return
    try {
      const data = await request.get('/api/users/me')
      userInfo.value = data.data
    } catch {
      logout()
    }
  }

  return { token, userInfo, isLoggedIn, isAdmin, login, logout, fetchUserInfo }
})
```

---

## 3.6 路由守卫：权限控制

`src/router/index.js`：

```javascript
import { createRouter, createWebHistory } from 'vue-router'
import { useUserStore } from '@/stores/user'

const routes = [
  { path: '/login', component: () => import('@/views/Login.vue'), meta: { public: true } },
  { path: '/chat', component: () => import('@/views/Chat.vue') },
  { path: '/admin/knowledge', component: () => import('@/views/admin/Knowledge.vue'), meta: { admin: true } },
  { path: '/admin/users', component: () => import('@/views/admin/UserManagement.vue'), meta: { admin: true } },
  { path: '/', redirect: '/chat' }
]

const router = createRouter({
  history: createWebHistory(),
  routes
})

// 全局路由守卫
router.beforeEach(async (to) => {
  const userStore = useUserStore()
  
  // 公开页面直接放行
  if (to.meta.public) return true
  
  // 未登录跳转到登录页
  if (!userStore.isLoggedIn) return '/login'
  
  // 管理员页面权限检查
  if (to.meta.admin && !userStore.isAdmin) {
    ElMessage.error('没有权限访问此页面')
    return '/chat'
  }
  
  return true
})

export default router
```

---

## 第3章小结

🎉 Day 3 完成！

**今天你构建了整个系统的安全基础**：
- ✅ JWT 认证：无状态、安全、可扩展
- ✅ 用户 CRUD：注册、查询、修改、禁用
- ✅ 权限控制：普通用户 vs 管理员
- ✅ 前端路由守卫：未登录自动拦截
- ✅ Pinia 状态管理：全局用户信息

**明天（Day 4）** 我们进入 AI 的核心领域：文档处理、向量化、FAISS 向量库构建。

---

_第3章完_
