# 配置一套支持 Web 和小程序双预览的构建工具链

配置一套支持 **Web** 和 **小程序** 双预览的构建工具链，核心在于**环境变量的隔离**、**差异化插件配置**以及**并行开发流**。

目前行业内最成熟的方案是基于 **Vite + Taro 3.x**（或 uni-app）。由于你的 JD 强调了 **Vite** 和 **高性能**，我将以 **Taro + Vite** 为例，为你拆解一套资深工程师级别的构建配置方案。

---

## 1. 核心构建架构：环境变量驱动

跨端配置的本质是“条件编译”。我们需要在启动命令中注入不同的 `TARO_ENV`。

### 目录结构逻辑

建议采用 **Monorepo** 或 **特定后缀** 的方式管理差异：

* `component.tsx`（共有逻辑）
* `component.h5.tsx`（Web 端特有渲染）
* `component.weapp.tsx`（微信小程序特有渲染）

---

## 2. Vite 核心配置 (vite.config.ts)

你需要根据环境变量动态加载不同的插件和别名。

```typescript
import { defineConfig } from 'vite';
import taro from '@tarojs/vite-platform-taro'; // Taro 官方 Vite 插件

export default defineConfig(({ mode }) => {
  const isWeb = process.env.TARO_ENV === 'h5';
  const isWeapp = process.env.TARO_ENV === 'weapp';

  return {
    // 1. 平台差异化插件
    plugins: [
      taro(), 
      // 如果是 Web 端，可以加入 PWA 或分析插件
      isWeb && myWebSpecificPlugin(),
    ].filter(Boolean),

    // 2. 路径别名：抹平跨端引用差异
    resolve: {
      alias: [
        { find: '@', replacement: '/src' },
        // 强制将某些库指向端特定的实现
        { 
          find: '@/utils/storage', 
          replacement: isWeb ? '/src/utils/storage.h5' : '/src/utils/storage.mini' 
        }
      ]
    },

    // 3. 注入全局变量（代码内判断）
    define: {
      'process.env.TARO_ENV': JSON.stringify(process.env.TARO_ENV),
      '__DEV__': mode === 'development'
    },

    // 4. 样式处理：PX 转响应式单位
    css: {
      postcss: {
        plugins: [
          // 小程序用 rpx，Web 用 rem/vw
          require('postcss-pxtransform')({
            platform: process.env.TARO_ENV,
            designWidth: 750,
          })
        ]
      }
    }
  };
});

```

---

## 3. 实现双预览工作流 (Workflow)

为了实现“双预览”，你不能在同一个进程里跑两个端。我们需要利用 `concurrently` 来管理并行任务。

### package.json 脚本配置

```json
{
  "scripts": {
    "dev:h5": "taro build --type h5 --watch",
    "dev:weapp": "taro build --type weapp --watch",
    "dev:all": "concurrently \"npm run dev:h5\" \"npm run dev:weapp\""
  },
  "devDependencies": {
    "concurrently": "^8.0.0"
  }
}

```

---

## 4. 关键技术点：如何处理“非标准”差异？

在面试中，如果你能提到以下深度配置，会非常加分：

### A. 静态资源处理

小程序对本地图片大小极其敏感。在 Vite 配置中，可以针对小程序端开启 **图片自动压缩** 或 **自动上传 CDN 插件**，而 Web 端则保持标准的 Base64 内联策略。

### B. 依赖预构建 (OptimizeDeps)

Vite 开发环境下，小程序端的依赖包（如 `@tarojs/components`）需要进行特定的 CommonJS 转 ESM 处理。你需要在 `optimizeDeps` 中将其加入 `include`，防止初次启动时出现白屏或依赖丢失。

### C. 模拟环境 (Mocking)

* **Web 端：** 直接使用 `vite-plugin-mock`。
* **小程序端：** 由于没有真正的浏览器对象，需要拦截 `taro.request`。你可以编写一个简单的 Vite 插件，在构建小程序端代码时，动态注入一个 Mock 拦截层。

---

## 5. AI 工具 (Cursor) 的提效点

既然 JD 提到了 **Cursor**，你在配置这套工具链时可以这样运用 AI：

1. **自动补齐类型：** “根据 Vite 插件定义，为我的自定义跨端插件生成完整的 TypeScript Interface。”
2. **正则转换：** “写一个 PostCSS 插件逻辑，将所有类名带 `-web` 后缀的选择器在编译微信小程序时自动剔除。”
3. **CI/CD 脚本生成：** “编写一个 GitHub Action，当代码推送到 master 时，同时构建 Web 版静态资源到 Sentry 监控，并上传小程序代码草稿到微信开发者后台。”

---

### 总结

一套合格的资深级工具链需要具备：**环境感知、样式自动转换、依赖精简、并行调试**。
