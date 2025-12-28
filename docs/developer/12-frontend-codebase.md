# 4.2 前端代码结构说明

## 1. 目录结构

```
frontend/
├── index.html              # 🌐 入口 HTML
├── vite.config.ts          # ⚙️ Vite 配置
├── tsconfig.json           # 📘 TypeScript 配置
├── tsconfig.node.json      # 📘 Node TypeScript 配置
├── package.json            # 📦 依赖配置
├── pnpm-lock.yaml          # 🔒 依赖锁定
├── .env.development        # 🔧 开发环境变量
├── .env.production         # 🔧 生产环境变量
│
├── public/                 # 📁 静态资源
│   ├── swagger-ui.css
│   └── swagger-ui-bundle.js
│
└── src/
    ├── main.ts             # 🚀 应用入口
    ├── App.vue             # 📱 根组件
    ├── vite-env.d.ts       # 📘 类型声明
    │
    ├── api/                # 🔌 API 请求
    ├── assets/             # 🎨 静态资源
    ├── components/         # 🧩 公共组件
    ├── entity/             # 📋 类型定义
    ├── i18n/               # 🌐 国际化
    ├── router/             # 🛣️ 路由配置
    ├── stores/             # 📦 状态管理
    ├── utils/              # 🔧 工具函数
    └── views/              # 📄 页面视图
```

---

## 2. 核心目录详解

### 2.1 src/api/ - API 请求模块

```
api/
├── assistant.ts        # 助手接口
├── audit.ts            # 审计日志
├── auth.ts             # 认证相关
├── chat.ts             # 智能问答 ⭐核心
├── dashboard.ts        # 仪表盘
├── datasource.ts       # 数据源
├── embedded.ts         # 嵌入式接口
├── login.ts            # 登录接口
├── permissions.ts      # 权限
├── prompt.ts           # 提示词
├── recommendedApi.ts   # 推荐问题
├── setting.ts          # 系统设置
├── system.ts           # 系统管理
├── training.ts         # 训练数据
├── user.ts             # 用户管理
└── workspace.ts        # 工作空间
```

### 2.2 src/views/ - 页面视图

```
views/
├── WelcomeView.vue     # 欢迎页
├── chat/               # 智能问答页面 ⭐核心
│   ├── index.vue       # 主页面
│   ├── ChatList.vue    # 对话列表
│   ├── ChatWindow.vue  # 对话窗口
│   └── components/     # 子组件
├── dashboard/          # 仪表盘
│   ├── index.vue
│   └── components/
├── ds/                 # 数据源管理
│   ├── index.vue       # 数据源列表
│   ├── TableList.vue   # 表管理
│   └── FieldList.vue   # 字段管理
├── embedded/           # 嵌入模式
├── login/              # 登录相关
│   ├── index.vue       # 登录页
│   └── ResetPwd.vue    # 密码重置
├── system/             # 系统管理
│   ├── user/           # 用户管理
│   ├── workspace/      # 工作空间
│   ├── aimodel/        # AI 模型
│   ├── assistant/      # 助手配置
│   ├── terminology/    # 术语库
│   └── training/       # 训练数据
├── work/               # 工作空间切换
└── error/              # 错误页面
```

### 2.3 src/components/ - 公共组件

```
components/
├── chart/              # 图表组件
│   ├── ChartRenderer.vue
│   └── TableChart.vue
├── common/             # 通用组件
│   ├── ConfirmDialog.vue
│   ├── LoadingSpinner.vue
│   └── EmptyState.vue
├── editor/             # 编辑器组件
│   ├── SqlEditor.vue
│   └── MarkdownEditor.vue
└── layout/             # 布局组件
    ├── Header.vue
    ├── Sidebar.vue
    └── Footer.vue
```

### 2.4 src/stores/ - 状态管理

```
stores/
├── index.ts            # Store 入口
├── user.ts             # 用户状态
├── chat.ts             # 聊天状态
├── workspace.ts        # 工作空间
├── setting.ts          # 系统设置
├── embedded.ts         # 嵌入模式
└── theme.ts            # 主题设置
```

### 2.5 src/utils/ - 工具函数

```
utils/
├── request.ts          # Axios 封装 ⭐核心
├── auth.ts             # Token 管理
├── storage.ts          # 本地存储
├── format.ts           # 格式化工具
├── validate.ts         # 验证工具
├── sse.ts              # SSE 处理
└── chart.ts            # 图表工具
```

---

## 3. 关键文件说明

### 3.1 应用入口

| 文件 | 说明 |
| :--- | :--- |
| `main.ts` | 创建 Vue 应用、注册插件 |
| `App.vue` | 根组件，包含路由出口 |
| `router/index.ts` | 路由配置与守卫 |

### 3.2 API 请求

| 文件 | 说明 |
| :--- | :--- |
| `utils/request.ts` | Axios 实例、拦截器配置 |
| `api/chat.ts` | 问答相关 API（最核心） |
| `api/datasource.ts` | 数据源 CRUD API |

### 3.3 状态管理

| Store | 说明 |
| :--- | :--- |
| `user.ts` | 用户信息、登录状态 |
| `workspace.ts` | 当前工作空间 |
| `chat.ts` | 当前对话状态 |

### 3.4 核心页面

| 页面 | 路径 | 说明 |
| :--- | :--- | :--- |
| 登录 | `views/login/` | 用户登录、密码重置 |
| 问答 | `views/chat/` | 智能问答主界面 |
| 数据源 | `views/ds/` | 数据源与表管理 |
| 系统 | `views/system/` | 用户、AI 模型等管理 |

---

## 4. 命名规范

### 4.1 文件命名

| 类型 | 规则 | 示例 |
| :--- | :--- | :--- |
| Vue 组件 | PascalCase | `ChatWindow.vue` |
| TypeScript | camelCase | `request.ts` |
| 样式文件 | kebab-case | `chat-list.less` |
| 测试文件 | `.spec.ts` | `chat.spec.ts` |

### 4.2 组件命名

| 类型 | 规则 | 示例 |
| :--- | :--- | :--- |
| 页面组件 | 功能名称 | `ChatWindow.vue` |
| 公共组件 | 通用描述 | `ConfirmDialog.vue` |
| 布局组件 | 位置描述 | `Sidebar.vue` |

### 4.3 变量命名

| 类型 | 规则 | 示例 |
| :--- | :--- | :--- |
| 响应式数据 | camelCase | `chatList` |
| 常量 | UPPER_SNAKE | `API_BASE_URL` |
| 类型定义 | PascalCase | `ChatRecord` |
| Props | camelCase | `modelValue` |

---

## 5. 组件通信模式

### 5.1 Props + Emit（父子通信）

```vue
<!-- 子组件 -->
<script setup lang="ts">
defineProps<{ title: string }>()
const emit = defineEmits<{ (e: 'update', value: string): void }>()
</script>

<!-- 父组件 -->
<ChildComponent :title="title" @update="handleUpdate" />
```

### 5.2 Pinia（跨组件通信）

```typescript
// stores/user.ts
export const useUserStore = defineStore('user', () => {
  const userInfo = ref<UserInfo | null>(null)
  
  async function fetchUserInfo() {
    userInfo.value = await getUserInfo()
  }
  
  return { userInfo, fetchUserInfo }
})

// 使用
const userStore = useUserStore()
userStore.fetchUserInfo()
```

### 5.3 Provide/Inject（深层传递）

```typescript
// 父组件
provide('theme', ref('dark'))

// 深层子组件
const theme = inject<Ref<string>>('theme')
```

---

## 6. 样式管理

### 6.1 全局样式

| 文件 | 说明 |
| :--- | :--- |
| `src/style.less` | 全局样式、变量定义 |
| `src/assets/styles/` | 样式资源 |

### 6.2 组件样式

```vue
<style scoped lang="less">
.chat-window {
  // 组件私有样式
}
</style>
```

### 6.3 主题变量

```less
// 主题色
@primary-color: #409EFF;
@success-color: #67C23A;

// 使用
.button {
  background: @primary-color;
}
```

---

## 7. 路由配置

### 7.1 路由结构

```typescript
// router/index.ts
const routes = [
  { path: '/login', component: LoginView },
  { 
    path: '/', 
    component: Layout,
    children: [
      { path: 'chat', component: ChatView },
      { path: 'ds', component: DatasourceView },
      { path: 'system', component: SystemView }
    ]
  },
  { path: '/embedded/:token', component: EmbeddedView }
]
```

### 7.2 路由守卫

```typescript
router.beforeEach((to, from, next) => {
  const token = getToken()
  
  if (to.meta.requiresAuth && !token) {
    next('/login')
  } else {
    next()
  }
})
```

---

## 8. 扩展指南

### 8.1 添加新页面

1. 创建视图文件：`views/new-page/index.vue`
2. 添加路由配置
3. 添加菜单入口（如需要）

### 8.2 添加新 API

1. 在 `api/` 下创建或编辑文件
2. 定义接口函数
3. 在组件中调用

### 8.3 添加新组件

1. 在 `components/` 下创建组件
2. 导出并在需要的地方导入使用
