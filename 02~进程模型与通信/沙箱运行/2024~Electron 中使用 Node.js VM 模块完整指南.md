# Electron 中使用 Node.js VM 模块完整指南

## 一、简介

Node.js VM 模块允许在隔离的上下文中运行代码，这在 Electron 应用中特别有用，比如实现插件系统、运行用户脚本等场景。本文将详细介绍如何在 Electron 中安全地使用 VM 模块。

## 二、基础配置

### 1. 项目初始化

```bash
# 创建项目
mkdir electron-vm-demo
cd electron-vm-demo
npm init -y

# 安装依赖
npm install electron --save
```

### 2. package.json 配置

```json
{
  "name": "electron-vm-demo",
  "version": "1.0.0",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "build": "electron-builder"
  },
  "dependencies": {
    "electron": "^25.0.0"
  },
  "devDependencies": {
    "electron-builder": "^24.6.0"
  }
}
```

## 三、沙箱实现

### 1. 基础沙箱类

```typescript
// src/sandbox/BaseSandbox.ts
import { vm } from "node";

export class BaseSandbox {
  protected context: vm.Context;

  constructor() {
    // 创建基础上下文
    this.context = vm.createContext({
      console: {
        log: (...args) => console.log("[Sandbox]", ...args),
        error: (...args) => console.error("[Sandbox]", ...args),
      },
      setTimeout,
      clearTimeout,
      Math,
      Date,
      JSON,
    });
  }

  async execute(code: string, timeout = 5000) {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error("Execution timeout"));
      }, timeout);

      try {
        const script = new vm.Script(code);
        const result = script.runInContext(this.context);
        clearTimeout(timer);
        resolve(result);
      } catch (error) {
        clearTimeout(timer);
        reject(error);
      }
    });
  }
}
```

### 2. 高级沙箱实现

```typescript
// src/sandbox/AdvancedSandbox.ts
import { vm } from "node";
import { app } from "electron";

interface SandboxOptions {
  timeout?: number;
  allowAsync?: boolean;
  memoryLimit?: number;
}

export class AdvancedSandbox {
  private context: vm.Context;
  private scripts: Map<string, vm.Script> = new Map();
  private options: SandboxOptions;

  constructor(options: SandboxOptions = {}) {
    this.options = options;
    this.initContext();
  }

  private initContext() {
    const sandbox = {
      // 基础 API
      console: {
        log: (...args) => console.log("[Sandbox]", ...args),
        error: (...args) => console.error("[Sandbox]", ...args),
        warn: (...args) => console.warn("[Sandbox]", ...args),
      },

      // 安全的定时器
      setTimeout: (fn: Function, delay: number) => {
        if (delay > this.options.timeout) {
          throw new Error("Timeout too long");
        }
        return setTimeout(fn, delay);
      },
      clearTimeout,

      // 工具函数
      JSON: {
        parse: JSON.parse,
        stringify: JSON.stringify,
      },

      // 自定义 API
      app: {
        getName: () => app.getName(),
        getVersion: () => app.getVersion(),
      },

      // 安全的存储 API
      storage: {
        get: this.safeStorageGet.bind(this),
        set: this.safeStorageSet.bind(this),
      },
    };

    this.context = vm.createContext(sandbox);
  }

  // 预编译脚本
  compileScript(code: string, filename = "script.js") {
    try {
      const script = new vm.Script(code, {
        filename,
        lineOffset: 0,
        columnOffset: 0,
        cachedData: undefined,
        produceCachedData: true,
      });

      this.scripts.set(filename, script);
      return script;
    } catch (error) {
      console.error("Script compilation error:", error);
      throw error;
    }
  }

  // 执行预编译脚本
  async runScript(filename: string) {
    const script = this.scripts.get(filename);
    if (!script) {
      throw new Error(`Script ${filename} not found`);
    }

    return this.executeInContext(script);
  }

  // 直接执行代码
  async execute(code: string) {
    try {
      const script = new vm.Script(code);
      return await this.executeInContext(script);
    } catch (error) {
      console.error("Code execution error:", error);
      throw error;
    }
  }

  private async executeInContext(script: vm.Script) {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error("Execution timeout"));
      }, this.options.timeout || 5000);

      try {
        const result = script.runInContext(this.context);
        clearTimeout(timer);
        resolve(result);
      } catch (error) {
        clearTimeout(timer);
        reject(error);
      }
    });
  }

  // 安全的存储访问
  private safeStorageGet(key: string) {
    // 实现访问控制
    if (this.isValidKey(key)) {
      return localStorage.getItem(key);
    }
    throw new Error("Invalid storage access");
  }

  private safeStorageSet(key: string, value: string) {
    if (this.isValidKey(key)) {
      return localStorage.setItem(key, value);
    }
    throw new Error("Invalid storage access");
  }

  private isValidKey(key: string): boolean {
    // 实现键名验证逻辑
    return /^[a-zA-Z0-9_-]+$/.test(key);
  }

  // 添加新的 API
  addToContext(name: string, value: any) {
    if (this.context) {
      Object.defineProperty(this.context, name, {
        value,
        configurable: false,
        writable: false,
      });
    }
  }
}
```

## 四、在主进程中使用

```typescript
// src/main.ts
import { app, BrowserWindow, ipcMain } from "electron";
import { AdvancedSandbox } from "./sandbox/AdvancedSandbox";

class App {
  private mainWindow: BrowserWindow | null = null;
  private sandbox: AdvancedSandbox;

  constructor() {
    this.sandbox = new AdvancedSandbox({
      timeout: 5000,
      allowAsync: true,
    });

    this.setupIPC();
  }

  private setupIPC() {
    ipcMain.handle("execute-code", async (event, code) => {
      try {
        return await this.sandbox.execute(code);
      } catch (error) {
        console.error("Code execution error:", error);
        throw error;
      }
    });
  }

  async init() {
    try {
      await this.createWindow();

      // 预编译一些常用脚本
      this.sandbox.compileScript(
        `
        function hello(name) {
          return 'Hello, ' + name;
        }
      `,
        "hello.js"
      );
    } catch (error) {
      console.error("Application initialization error:", error);
      app.quit();
    }
  }

  private async createWindow() {
    this.mainWindow = new BrowserWindow({
      width: 800,
      height: 600,
      webPreferences: {
        nodeIntegration: true,
        contextIsolation: true,
        preload: path.join(__dirname, "preload.js"),
      },
    });

    await this.mainWindow.loadFile("index.html");
  }
}

// 应用实例
const application = new App();

app.whenReady().then(() => {
  application.init();
});
```

## 五、渲染进程集成

### 1. Preload 脚本

```typescript
// src/preload.ts
import { contextBridge, ipcRenderer } from "electron";

contextBridge.exposeInMainWorld("sandbox", {
  execute: (code: string) => ipcRenderer.invoke("execute-code", code),
});
```

### 2. 渲染进程使用

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
  <head>
    <title>Electron VM Demo</title>
  </head>
  <body>
    <textarea id="code"></textarea>
    <button onclick="runCode()">Run</button>
    <div id="output"></div>

    <script>
      async function runCode() {
        const code = document.getElementById("code").value;
        try {
          const result = await window.sandbox.execute(code);
          document.getElementById("output").textContent = result;
        } catch (error) {
          document.getElementById("output").textContent =
            "Error: " + error.message;
        }
      }
    </script>
  </body>
</html>
```

## 六、安全性考虑

1. **限制访问范围**

```typescript
// 限制可访问的全局对象
const restrictedGlobals = {
  // 只允许访问安全的 API
  console: {
    log: console.log.bind(console),
  },
  Math: Object.freeze(Math),
  Date: Object.freeze(Date),
};
```

2. **资源限制**

```typescript
// 添加资源限制
const options = {
  timeout: 5000,
  memoryLimit: 50 * 1024 * 1024, // 50MB
  cpuLimit: 0.8, // 80% CPU 使用率
};
```

3. **错误处理**

```typescript
// 错误处理包装器
function wrapExecution(code: string) {
  return `
    try {
      ${code}
    } catch (error) {
      console.error('Script error:', error);
      throw error;
    }
  `;
}
```

## 七、最佳实践

1. **代码预编译**

```typescript
// 预编译常用代码
const commonScripts = {
  utils: `
    const utils = {
      formatDate: (date) => new Date(date).toLocaleDateString(),
      sum: (arr) => arr.reduce((a, b) => a + b, 0)
    };
  `,
  validators: `
    const validators = {
      isEmail: (email) => /^[^@]+@[^@]+\.[^@]+$/.test(email),
      isPhone: (phone) => /^\d{10}$/.test(phone)
    };
  `,
};

// 预编译所有脚本
Object.entries(commonScripts).forEach(([name, code]) => {
  sandbox.compileScript(code, `${name}.js`);
});
```

2. **性能优化**

```typescript
// 缓存编译结果
class ScriptCache {
  private cache: Map<string, vm.Script> = new Map();

  get(key: string): vm.Script | undefined {
    return this.cache.get(key);
  }

  set(key: string, script: vm.Script): void {
    this.cache.set(key, script);
  }

  clear(): void {
    this.cache.clear();
  }
}
```

3. **监控和日志**

```typescript
class SandboxMonitor {
  private metrics: {
    executionCount: number;
    errors: number;
    totalExecutionTime: number;
  } = {
    executionCount: 0,
    errors: 0,
    totalExecutionTime: 0,
  };

  recordExecution(duration: number) {
    this.metrics.executionCount++;
    this.metrics.totalExecutionTime += duration;
  }

  recordError() {
    this.metrics.errors++;
  }

  getMetrics() {
    return {
      ...this.metrics,
      averageExecutionTime:
        this.metrics.totalExecutionTime / this.metrics.executionCount,
    };
  }
}
```

## 八、常见问题解决

1. **内存泄漏**

```typescript
class MemoryManager {
  private maxMemory: number;
  private gcThreshold: number;

  constructor(maxMemory: number, gcThreshold: number) {
    this.maxMemory = maxMemory;
    this.gcThreshold = gcThreshold;
  }

  checkMemory() {
    const used = process.memoryUsage().heapUsed;
    if (used > this.maxMemory) {
      throw new Error("Memory limit exceeded");
    }
    if (used > this.gcThreshold) {
      global.gc();
    }
  }
}
```

2. **超时处理**

```typescript
class TimeoutManager {
  private timeouts: Map<string, NodeJS.Timeout> = new Map();

  setTimeout(id: string, fn: Function, delay: number) {
    const timeout = setTimeout(() => {
      fn();
      this.timeouts.delete(id);
    }, delay);

    this.timeouts.set(id, timeout);
  }

  clearTimeout(id: string) {
    const timeout = this.timeouts.get(id);
    if (timeout) {
      clearTimeout(timeout);
      this.timeouts.delete(id);
    }
  }

  clearAll() {
    this.timeouts.forEach((timeout) => clearTimeout(timeout));
    this.timeouts.clear();
  }
}
```

## 九、总结

在 Electron 中使用 VM 模块需要注意：

1. 安全性控制
2. 资源限制
3. 性能优化
4. 错误处理
5. 监控和日志

通过本文的实现方式，可以在 Electron 应用中安全地运行动态代码。

## 十、参考资源

- [Node.js VM 文档](https://nodejs.org/api/vm.html)
- [Electron 安全指南](https://www.electronjs.org/docs/tutorial/security)
- [VM2 项目](https://github.com/patriksimek/vm2)

希望这个完整的教程能帮助你在 Electron 项目中安全地使用 VM 模块。
