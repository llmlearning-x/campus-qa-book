# 3.6 权限控制

_本节内容已并入 [3.5 前端页面实现](section-05.md) 中的「路由守卫」部分。_

请参阅 3.5 节中关于 Vue Router 路由守卫的完整实现：

- `meta: { public: true }` 标记公开页面
- `meta: { admin: true }` 标记管理员专属页面
- `router.beforeEach` 全局守卫拦截未授权访问
