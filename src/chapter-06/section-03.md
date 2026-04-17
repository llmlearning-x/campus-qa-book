# 6.3 项目封装与 UI 优化

## 前端构建优化

### 配置生产环境 API 地址

在 `vite.config.js` 中区分开发和生产环境：

```javascript
export default defineConfig(({ mode }) => ({
  // ...
  define: {
    __API_BASE__: JSON.stringify(
      mode === 'production' 
        ? ''           // 生产环境：同域，通过 Nginx 代理
        : 'http://localhost:8000'  // 开发环境：跨域
    )
  }
}))
```

### 常见 UI 优化点

**1. 全局加载状态**
```javascript
// src/main.js
import NProgress from 'nprogress'
import 'nprogress/nprogress.css'

router.beforeEach(() => NProgress.start())
router.afterEach(() => NProgress.done())
```

**2. 错误边界**
在 App.vue 中捕获未处理错误，显示友好提示而不是白屏。

**3. 骨架屏**（Element Plus 提供现成组件）
```vue
<el-skeleton :rows="5" animated v-if="loading" />
<el-table v-else :data="list" />
```

---

## 本节小结

- 生产构建和开发构建使用不同的 API 地址配置
- NProgress 进度条提升页面切换的流畅感
- 骨架屏比 loading 转圈有更好的视觉体验
