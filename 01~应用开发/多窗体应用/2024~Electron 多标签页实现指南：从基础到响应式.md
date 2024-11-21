# Electron 多标签页实现指南：从基础到响应式

在开发 Electron 应用时，多标签页功能是一个常见需求。本文将详细介绍如何实现一个功能完整、性能良好的多标签页系统，并着重讲解如何处理响应式布局。

## 一、实现方案对比

在开始具体实现之前，让我们先比较几种常见的实现方案：

### 1. webview 方案

```tsx
<webview
  src={tab.url}
  style={{ display: activeTabId === tab.id ? "block" : "none" }}
/>
```

优点：

- 实现简单直接
- 标签页内容完全隔离

缺点：

- webview 标签已被 Electron 团队不推荐使用
- 性能开销大
- 内存占用高

### 2. 纯 BrowserWindow 方案

```typescript
const win = new BrowserWindow({ show: false });
win.loadURL(url);
```

优点：

- 完全的进程隔离
- 稳定性好

缺点：

- 资源消耗较大
- 不适合同时打开大量标签页

### 3. BrowserView + 截图预览方案（推荐）

这是一个平衡性能和用户体验的方案，也是本文将要详细介绍的方案。

# 基础实现

## 一、核心数据结构

首先定义我们需要的接口和类型：

```typescript:src/main/types.ts
interface TabConfig {
  id: string;
  url: string;
  title: string;
  favicon?: string;
  isLoading?: boolean;
}

interface TabState {
  view: BrowserView;
  config: TabConfig;
  status: 'active' | 'suspended' | 'hibernated';
  lastAccessed: number;
  savedState?: any;
}
```

## 二、TabManager 完整实现

```typescript:src/main/TabManager.ts
import { BrowserView, BrowserWindow, ipcMain, Menu } from 'electron';
import { EventEmitter } from 'events';

export class TabManager extends EventEmitter {
  private tabs: Map<string, TabState> = new Map();
  private mainWindow: BrowserWindow;
  private activeView: BrowserView | null = null;

  constructor(mainWindow: BrowserWindow) {
    super();
    this.mainWindow = mainWindow;
    this.setupIPC();
    this.setupWindowEvents();
  }

  private setupIPC() {
    // 标签页基础操作
    ipcMain.handle('tab:create', async (_, url: string) => {
      return this.createTab(url);
    });

    ipcMain.handle('tab:close', async (_, tabId: string) => {
      return this.closeTab(tabId);
    });

    ipcMain.handle('tab:switch', async (_, tabId: string) => {
      return this.switchTab(tabId);
    });

    // 标签页状态查询
    ipcMain.handle('tab:getAll', () => this.getAllTabs());
    ipcMain.handle('tab:getActive', () => this.getActiveTab());

    // 导航操作
    ipcMain.handle('tab:reload', async (_, tabId: string) => {
      const tab = this.tabs.get(tabId);
      tab?.view.webContents.reload();
    });

    ipcMain.handle('tab:goBack', async (_, tabId: string) => {
      const tab = this.tabs.get(tabId);
      if (tab?.view.webContents.canGoBack()) {
        tab.view.webContents.goBack();
      }
    });

    ipcMain.handle('tab:goForward', async (_, tabId: string) => {
      const tab = this.tabs.get(tabId);
      if (tab?.view.webContents.canGoForward()) {
        tab.view.webContents.goForward();
      }
    });
  }

  private setupWindowEvents() {
    // 处理窗口大小变化
    this.mainWindow.on('resize', () => {
      this.handleResize();
    });

    // 处理窗口最大化/还原
    this.mainWindow.on('maximize', () => {
      this.handleResize();
      this.mainWindow.webContents.send('window:maximized', true);
    });

    this.mainWindow.on('unmaximize', () => {
      this.handleResize();
      this.mainWindow.webContents.send('window:maximized', false);
    });

    // 处理窗口聚焦
    this.mainWindow.on('focus', () => {
      if (this.activeView) {
        this.activeView.webContents.focus();
      }
    });
  }

  private setupViewEvents(view: BrowserView, tabId: string) {
    const webContents = view.webContents;

    // 标题更新
    webContents.on('page-title-updated', (event, title) => {
      this.updateTabConfig(tabId, { title });
    });

    // 图标更新
    webContents.on('page-favicon-updated', (event, favicons) => {
      if (favicons.length > 0) {
        this.updateTabConfig(tabId, { favicon: favicons[0] });
      }
    });

    // 加载状态
    webContents.on('did-start-loading', () => {
      this.updateTabConfig(tabId, { isLoading: true });
    });

    webContents.on('did-stop-loading', () => {
      this.updateTabConfig(tabId, { isLoading: false });
    });

    // 导航事件
    webContents.on('did-navigate', (event, url) => {
      this.updateTabConfig(tabId, { url });
    });

    webContents.on('did-navigate-in-page', (event, url) => {
      this.updateTabConfig(tabId, { url });
    });

    // 新窗口处理
    webContents.setWindowOpenHandler(({ url }) => {
      this.createTab(url);
      return { action: 'deny' };
    });

    // 错误处理
    webContents.on('render-process-gone', async (event, details) => {
      console.error('Tab crashed:', tabId, details);
      await this.handleTabCrash(tabId, details);
    });

    webContents.on('unresponsive', () => {
      this.emit('tabUnresponsive', tabId);
    });

    webContents.on('responsive', () => {
      this.emit('tabResponsive', tabId);
    });

    // 上下文菜单
    webContents.on('context-menu', (event, params) => {
      this.createContextMenu(tabId, params);
    });
  }

  private async createTab(url: string) {
    const tabId = Date.now().toString();

    const view = new BrowserView({
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
        sandbox: true
      }
    });

    const config: TabConfig = {
      id: tabId,
      url,
      title: 'New Tab',
      isLoading: true
    };

    this.tabs.set(tabId, {
      view,
      config,
      status: 'active',
      lastAccessed: Date.now()
    });

    this.setupViewEvents(view, tabId);
    await this.switchTab(tabId);
    view.webContents.loadURL(url);

    return tabId;
  }

  private async closeTab(tabId: string) {
    const tab = this.tabs.get(tabId);
    if (!tab) return;

    if (tab.view === this.activeView) {
      this.mainWindow.removeBrowserView(tab.view);
      this.activeView = null;

      // 切换到下一个标签页
      const tabIds = Array.from(this.tabs.keys());
      const currentIndex = tabIds.indexOf(tabId);
      const nextTabId = tabIds[currentIndex + 1] || tabIds[currentIndex - 1];

      if (nextTabId) {
        await this.switchTab(nextTabId);
      }
    }

    tab.view.webContents.destroy();
    this.tabs.delete(tabId);

    this.emit('tabsChanged');
  }

  private async switchTab(tabId: string) {
    const tab = this.tabs.get(tabId);
    if (!tab) return;

    if (this.activeView) {
      this.mainWindow.removeBrowserView(this.activeView);
    }

    this.mainWindow.addBrowserView(tab.view);
    tab.view.setBounds(this.getContentBounds());
    this.activeView = tab.view;
    tab.lastAccessed = Date.now();

    this.emit('activeTabChanged', tabId);
  }

  private handleResize = debounce(() => {
    if (this.activeView) {
      this.activeView.setBounds(this.getContentBounds());
    }
    const bounds = this.mainWindow.getBounds();
    this.mainWindow.webContents.send('window:resized', bounds);
  }, 100);

  private getContentBounds() {
    const bounds = this.mainWindow.getBounds();
    const isWindows = process.platform === 'win32';

    return {
      x: 0,
      y: 40, // 标签栏高度
      width: bounds.width,
      height: bounds.height - 40 - (isWindows ? 0 : 28) // macOS 需要考虑交通灯区域
    };
  }

  private async handleTabCrash(tabId: string, details: Electron.RenderProcessGoneDetails) {
    const tab = this.tabs.get(tabId);
    if (!tab) return;

    try {
      // 创建错误页面
      const errorView = new BrowserView({
        webPreferences: {
          nodeIntegration: false,
          contextIsolation: true
        }
      });

      // 加载错误页面
      await errorView.webContents.loadFile('src/renderer/error.html');

      // 更新标签页状态
      this.tabs.set(tabId, {
        ...tab,
        view: errorView,
        status: 'crashed'
      });

      // 如果是当前活动标签页，显示错误页面
      if (this.activeView === tab.view) {
        this.mainWindow.removeBrowserView(tab.view);
        this.mainWindow.addBrowserView(errorView);
        errorView.setBounds(this.getContentBounds());
        this.activeView = errorView;
      }

      this.emit('tabCrashed', tabId, details);
    } catch (error) {
      console.error('Failed to handle tab crash:', error);
    }
  }

  private createContextMenu(tabId: string, params: Electron.ContextMenuParams) {
    const menu = Menu.buildFromTemplate([
      {
        label: '返回',
        enabled: params.isEditable ? false : true,
        click: () => {
          const tab = this.tabs.get(tabId);
          if (tab?.view.webContents.canGoBack()) {
            tab.view.webContents.goBack();
          }
        }
      },
      {
        label: '前进',
        enabled: params.isEditable ? false : true,
        click: () => {
          const tab = this.tabs.get(tabId);
          if (tab?.view.webContents.canGoForward()) {
            tab.view.webContents.goForward();
          }
        }
      },
      { type: 'separator' },
      {
        label: '刷新',
        accelerator: 'CmdOrCtrl+R',
        click: () => {
          const tab = this.tabs.get(tabId);
          tab?.view.webContents.reload();
        }
      }
    ]);

    menu.popup();
  }

  private updateTabConfig(tabId: string, updates: Partial<TabConfig>) {
    const tab = this.tabs.get(tabId);
    if (!tab) return;

    const updatedConfig = {
      ...tab.config,
      ...updates
    };

    this.tabs.set(tabId, {
      ...tab,
      config: updatedConfig
    });

    // 通知渲染进程更新
    this.mainWindow.webContents.send('tab:updated', updatedConfig);
  }

  // 辅助方法
  private getAllTabs(): TabConfig[] {
    return Array.from(this.tabs.values()).map(tab => tab.config);
  }

  private getActiveTab(): TabConfig | null {
    if (!this.activeView) return null;
    const activeTab = Array.from(this.tabs.values())
      .find(tab => tab.view === this.activeView);
    return activeTab?.config || null;
  }
}

// 工具函数
function debounce(fn: Function, delay: number) {
  let timer: NodeJS.Timeout | null = null;
  return function (...args: any[]) {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

## 前端组件实现

### 1. TabBar 组件

```typescript:src/renderer/components/TabBar.tsx
import React, { useState, useEffect, useRef } from 'react';
import { ipcRenderer } from 'electron';
import { TabConfig } from '../../main/types';
import { TabItem } from './TabItem';
import { PlusIcon } from '@heroicons/react/24/outline';

export const TabBar: React.FC = () => {
  const [tabs, setTabs] = useState<TabConfig[]>([]);
  const [activeTabId, setActiveTabId] = useState<string | null>(null);
  const [isMaximized, setIsMaximized] = useState(false);
  const tabsContainerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const initTabs = async () => {
      const allTabs = await ipcRenderer.invoke('tab:getAll');
      const activeTab = await ipcRenderer.invoke('tab:getActive');
      setTabs(allTabs);
      setActiveTabId(activeTab?.id || null);
    };

    // 初始化标签页
    initTabs();

    // 设置事件监听
    const listeners = {
      'tab:updated': (_: any, tab: TabConfig) => {
        setTabs(prev => prev.map(t => t.id === tab.id ? tab : t));
      },
      'tab:created': (_: any, tab: TabConfig) => {
        setTabs(prev => [...prev, tab]);
        setActiveTabId(tab.id);
      },
      'tab:closed': (_: any, tabId: string) => {
        setTabs(prev => prev.filter(t => t.id !== tabId));
      },
      'tab:activated': (_: any, tabId: string) => {
        setActiveTabId(tabId);
      },
      'window:maximized': (_: any, maximized: boolean) => {
        setIsMaximized(maximized);
      }
    };

    // 注册所有监听器
    Object.entries(listeners).forEach(([event, handler]) => {
      ipcRenderer.on(event, handler);
    });

    // 清理函数
    return () => {
      Object.entries(listeners).forEach(([event]) => {
        ipcRenderer.removeAllListeners(event);
      });
    };
  }, []);

  const createTab = async () => {
    await ipcRenderer.invoke('tab:create', 'about:blank');
  };

  const closeTab = async (tabId: string, event?: React.MouseEvent) => {
    event?.stopPropagation();
    await ipcRenderer.invoke('tab:close', tabId);
  };

  const switchTab = async (tabId: string) => {
    await ipcRenderer.invoke('tab:switch', tabId);
  };

  return (
    <div
      className={`
        h-10 flex items-center bg-gray-100
        ${isMaximized ? 'pl-[70px]' : 'pl-0'}
        border-b border-gray-200
        transition-all duration-200
        drag // 自定义类，用于窗口拖拽
      `}
      data-platform={process.platform}
    >
      <div
        ref={tabsContainerRef}
        className="
          flex-1 flex overflow-x-auto overflow-y-hidden
          no-drag // 自定义类，禁用拖拽
          scrollbar-hide // 需要添加相应的 tailwind 插件
        "
      >
        {tabs.map(tab => (
          <TabItem
            key={tab.id}
            tab={tab}
            isActive={activeTabId === tab.id}
            onClose={closeTab}
            onClick={() => switchTab(tab.id)}
          />
        ))}
      </div>
      <button
        onClick={createTab}
        className="
          w-8 h-8 mx-2
          flex items-center justify-center
          rounded-md
          text-gray-600
          hover:bg-gray-200
          active:bg-gray-300
          transition-colors duration-150
          focus:outline-none focus:ring-2 focus:ring-blue-500
          no-drag
        "
        title="新建标签页"
      >
        <PlusIcon className="w-5 h-5" />
      </button>
    </div>
  );
};
```

### 2. TabItem 组件

```typescript:src/renderer/components/TabItem.tsx
import React from 'react';
import { TabConfig } from '../../main/types';
import { XMarkIcon } from '@heroicons/react/24/outline';
import { LoadingSpinner } from './LoadingSpinner';
import { GlobeAltIcon } from '@heroicons/react/24/outline';

interface TabItemProps {
  tab: TabConfig;
  isActive: boolean;
  onClose: (tabId: string, event?: React.MouseEvent) => void;
  onClick: () => void;
}

export const TabItem: React.FC<TabItemProps> = ({
  tab,
  isActive,
  onClose,
  onClick
}) => {
  return (
    <div
      className={`
        group
        min-w-[180px] max-w-[240px] h-full
        flex items-center
        px-3 py-1
        border-r border-gray-200
        cursor-default
        select-none
        transition-colors duration-150
        ${isActive
          ? 'bg-white'
          : 'bg-gray-50 hover:bg-gray-100'
        }
      `}
      onClick={onClick}
    >
      <div className="
        flex items-center gap-2
        w-full overflow-hidden
      ">
        {/* 图标/加载状态 */}
        <div className="flex-shrink-0 w-4 h-4">
          {tab.isLoading ? (
            <LoadingSpinner className="text-blue-500" />
          ) : tab.favicon ? (
            <img
              src={tab.favicon}
              alt=""
              className="w-full h-full object-contain"
            />
          ) : (
            <GlobeAltIcon className="w-full h-full text-gray-400" />
          )}
        </div>

        {/* 标题 */}
        <span className="
          flex-1
          truncate text-sm text-gray-700
          group-hover:text-gray-900
        " title={tab.title}>
          {tab.title}
        </span>

        {/* 关闭按钮 */}
        <button
          onClick={(e) => onClose(tab.id, e)}
          className="
            flex-shrink-0
            w-5 h-5
            flex items-center justify-center
            rounded-full
            text-gray-400
            opacity-0 group-hover:opacity-100
            hover:bg-gray-200
            hover:text-gray-700
            transition-all duration-150
            focus:outline-none focus:ring-2 focus:ring-blue-500
          "
          title="关闭标签页"
        >
          <XMarkIcon className="w-4 h-4" />
        </button>
      </div>
    </div>
  );
};
```

### 3. LoadingSpinner 组件

```typescript:src/renderer/components/LoadingSpinner.tsx
import React from 'react';

interface LoadingSpinnerProps {
  className?: string;
}

export const LoadingSpinner: React.FC<LoadingSpinnerProps> = ({
  className = ''
}) => {
  return (
    <div
      className={`
        animate-spin rounded-full
        border-2 border-gray-200
        border-t-blue-500
        ${className}
      `}
    />
  );
};
```

### 4. Tailwind 配置

```javascript:tailwind.config.js
module.exports = {
  content: [
    "./src/**/*.{js,jsx,ts,tsx}",
  ],
  theme: {
    extend: {
      animation: {
        'spin': 'spin 1s linear infinite',
      },
    },
  },
  plugins: [
    // 添加隐藏滚动条的插件
    require('tailwind-scrollbar-hide'),
    // 添加自定义拖拽类
    plugin(function({ addUtilities }) {
      const newUtilities = {
        '.drag': {
          '-webkit-app-region': 'drag',
        },
        '.no-drag': {
          '-webkit-app-region': 'no-drag',
        },
      }
      addUtilities(newUtilities)
    }),
  ],
}
```

### 5. 全局样式

```css:src/renderer/styles/global.css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* 基础样式重置 */
@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
    font-feature-settings: "rlig" 1, "calt" 1;
  }
}

/* 自定义工具类 */
@layer utilities {
  .scrollbar-hide::-webkit-scrollbar {
    display: none;
  }
}
```

这个实现提供了：

1. **现代化的 UI 设计**

   - 使用 Tailwind CSS 的原子类
   - 响应式设计
   - 平滑过渡动画

2. **良好的用户体验**

   - 标签页悬停效果
   - 加载状态动画
   - 优雅的关闭按钮

3. **可访问性**

   - 键盘焦点样式
   - 标题提示
   - 适当的颜色对比度

4. **平台适配**

   - macOS 窗口控制适配
   - 自定义窗口拖拽区域

5. **性能优化**
   - 条件渲染
   - 过渡动画优化
   - 滚动性能优化

# 进阶优化

## 三、性能优化实现

在基础功能的基础上，我们来添加一些性能优化的实现：

### 1. 虚拟化标签页管理

```typescript:src/main/TabManager.ts
export class TabManager extends EventEmitter {
  // 添加标签页状态管理
  private async manageTabStates() {
    const memoryLimit = 1024 * 1024 * 1024; // 1GB
    const processMemory = await process.getProcessMemoryInfo();

    if (processMemory.private > memoryLimit) {
      await this.suspendInactiveTabs();
    }
  }

  private async suspendInactiveTabs() {
    const now = Date.now();
    const activeTabId = this.getActiveTabId();

    for (const [tabId, tab] of this.tabs.entries()) {
      // 跳过活动标签页
      if (tabId === activeTabId) continue;

      // 如果标签页超过5分钟未访问，则挂起
      if (tab.status === 'active' && now - tab.lastAccessed > 5 * 60 * 1000) {
        await this.suspendTab(tabId);
      }
    }
  }

  private async suspendTab(tabId: string) {
    const tab = this.tabs.get(tabId);
    if (!tab || tab.status !== 'active') return;

    try {
      // 保存标签页状态
      const state = await this.captureTabState(tab.view);

      // 销毁 webContents 以释放内存
      tab.view.webContents.destroy();

      // 更新标签页状态
      this.tabs.set(tabId, {
        ...tab,
        status: 'suspended',
        savedState: state
      });

      this.emit('tabSuspended', tabId);
    } catch (error) {
      console.error('Failed to suspend tab:', error);
    }
  }

  private async resumeTab(tabId: string) {
    const tab = this.tabs.get(tabId);
    if (!tab || tab.status !== 'suspended') return;

    try {
      // 创建新的 BrowserView
      const view = new BrowserView({
        webPreferences: {
          nodeIntegration: false,
          contextIsolation: true,
          sandbox: true
        }
      });

      // 恢复标签页状态
      await this.restoreTabState(view, tab.savedState);

      // 更新标签页状态
      this.tabs.set(tabId, {
        ...tab,
        view,
        status: 'active',
        lastAccessed: Date.now(),
        savedState: undefined
      });

      this.emit('tabResumed', tabId);
    } catch (error) {
      console.error('Failed to resume tab:', error);
    }
  }

  // 标签页状态捕获和恢复
  private async captureTabState(view: BrowserView) {
    const webContents = view.webContents;
    return {
      url: webContents.getURL(),
      title: webContents.getTitle(),
      scrollPosition: await webContents.executeJavaScript(`
        JSON.stringify({
          x: window.scrollX,
          y: window.scrollY
        });
      `),
      formData: await this.captureFormData(webContents)
    };
  }

  private async restoreTabState(view: BrowserView, state: any) {
    await view.webContents.loadURL(state.url);
    await view.webContents.executeJavaScript(`
      window.scrollTo(${state.scrollPosition.x}, ${state.scrollPosition.y});
    `);
    if (state.formData) {
      await this.restoreFormData(view.webContents, state.formData);
    }
  }
}
```

### 2. 预加载优化

```typescript:src/main/TabManager.ts
export class TabManager {
  // 添加预加载管理
  private async preloadAdjacentTabs(activeTabId: string) {
    const adjacentTabs = this.getAdjacentTabs(activeTabId);

    for (const tabId of adjacentTabs) {
      const tab = this.tabs.get(tabId);
      if (tab?.status === 'suspended') {
        // 使用 warmup 而不是完全加载
        await this.warmupTab(tabId);
      }
    }
  }

  private getAdjacentTabs(tabId: string): string[] {
    const tabIds = Array.from(this.tabs.keys());
    const currentIndex = tabIds.indexOf(tabId);

    return [
      tabIds[currentIndex - 1],
      tabIds[currentIndex + 1]
    ].filter(Boolean);
  }

  private async warmupTab(tabId: string) {
    const tab = this.tabs.get(tabId);
    if (!tab) return;

    try {
      // 创建一个轻量级的 BrowserView
      const view = new BrowserView({
        webPreferences: {
          nodeIntegration: false,
          contextIsolation: true,
          sandbox: true
        }
      });

      // 只预加载 HTML 和基础资源
      await view.webContents.loadURL(tab.config.url, {
        httpReferrer: { url: tab.config.url, policy: 'no-referrer' },
        extraHeaders: 'Purpose: prefetch'
      });

      // 更新标签页状态
      this.tabs.set(tabId, {
        ...tab,
        status: 'warmed-up',
        view
      });
    } catch (error) {
      console.error('Failed to warmup tab:', error);
    }
  }
}
```

### 3. 资源管理优化

```typescript:src/main/ResourceManager.ts
export class ResourceManager {
  private maxMemoryUsage = 1024 * 1024 * 1024; // 1GB
  private maxTabs = 20;

  constructor(private tabManager: TabManager) {
    this.setupMemoryMonitoring();
  }

  private setupMemoryMonitoring() {
    setInterval(async () => {
      await this.checkMemoryUsage();
    }, 30000); // 每30秒检查一次
  }

  private async checkMemoryUsage() {
    const processMemory = await process.getProcessMemoryInfo();

    if (processMemory.private > this.maxMemoryUsage) {
      await this.optimizeMemoryUsage();
    }
  }

  private async optimizeMemoryUsage() {
    const tabs = Array.from(this.tabManager.tabs.entries())
      .sort((a, b) => b[1].lastAccessed - a[1].lastAccessed);

    // 保持活动标签页数量在限制之内
    if (tabs.length > this.maxTabs) {
      const tabsToSuspend = tabs.slice(this.maxTabs);
      for (const [tabId] of tabsToSuspend) {
        await this.tabManager.suspendTab(tabId);
      }
    }

    // 清理缓存
    await this.clearUnusedCache();
  }

  private async clearUnusedCache() {
    const session = this.tabManager.mainWindow.webContents.session;
    await session.clearCache();
    await session.clearCodeCaches({});
  }

  // 添加资源使用报告
  async generateResourceReport() {
    const report = {
      memory: await process.getProcessMemoryInfo(),
      tabs: Array.from(this.tabManager.tabs.entries()).map(([id, tab]) => ({
        id,
        status: tab.status,
        memory: tab.view?.webContents.getProcessMemoryInfo(),
        lastAccessed: tab.lastAccessed
      }))
    };

    return report;
  }
}
```

### 4. 网络优化

```typescript:src/main/NetworkOptimizer.ts
export class NetworkOptimizer {
  private resourceCache = new Map<string, Buffer>();
  private preloadQueue = new Set<string>();

  constructor(private tabManager: TabManager) {
    this.setupNetworkInterceptor();
  }

  private setupNetworkInterceptor() {
    const session = this.tabManager.mainWindow.webContents.session;

    session.webRequest.onBeforeRequest(
      { urls: ['<all_urls>'] },
      (details, callback) => {
        // 检查缓存
        if (this.resourceCache.has(details.url)) {
          callback({
            redirectURL: `data:${this.resourceCache.get(details.url)?.toString('base64')}`
          });
          return;
        }

        callback({});
      }
    );

    // 缓存资源
    session.webRequest.onCompleted({ urls: ['<all_urls>'] }, async (details) => {
      if (this.shouldCacheResource(details.url)) {
        try {
          const response = await fetch(details.url);
          const buffer = await response.arrayBuffer();
          this.resourceCache.set(details.url, Buffer.from(buffer));
        } catch (error) {
          console.error('Failed to cache resource:', error);
        }
      }
    });
  }

  private shouldCacheResource(url: string): boolean {
    // 根据URL和资源类型决定是否缓存
    const extension = new URL(url).pathname.split('.').pop()?.toLowerCase();
    const cacheable = ['js', 'css', 'png', 'jpg', 'jpeg', 'gif', 'svg'];
    return cacheable.includes(extension || '');
  }

  // 预加载资源
  async preloadResources(url: string) {
    if (this.preloadQueue.has(url)) return;
    this.preloadQueue.add(url);

    try {
      const resources = await this.analyzePageResources(url);
      await Promise.all(
        resources.map(async resource => {
          if (!this.resourceCache.has(resource)) {
            const response = await fetch(resource);
            const buffer = await response.arrayBuffer();
            this.resourceCache.set(resource, Buffer.from(buffer));
          }
        })
      );
    } finally {
      this.preloadQueue.delete(url);
    }
  }

  private async analyzePageResources(url: string): Promise<string[]> {
    try {
      const response = await fetch(url);
      const html = await response.text();

      // 解析HTML中的资源链接
      const resources = new Set<string>();
      const matches = html.matchAll(
        /<(script|link|img)[^>]+(src|href)=["']([^"']+)["']/g
      );

      for (const match of matches) {
        const resourceUrl = new URL(match[3], url).href;
        resources.add(resourceUrl);
      }

      return Array.from(resources);
    } catch (error) {
      console.error('Failed to analyze page resources:', error);
      return [];
    }
  }
}
```

这些优化实现主要关注：

1. **内存管理**

   - 标签页状态管理（活动/挂起/休眠）
   - 自动释放不活跃标签页
   - 内存使用监控

2. **预加载优化**

   - 相邻标签页预加载
   - 资源预热
   - 智能预加载策略

3. **资源管理**

   - 内存使用限制
   - 缓存清理
   - 资源使用报告

4. **网络优化**
   - 资源缓存
   - 请求拦截
   - 智能预加载

好的，让我介绍一些用户体验相关的优化实现。

## 四、用户体验优化

### 1. 标签页拖拽排序

```typescript:src/renderer/components/TabBar.tsx
import React, { useState, useEffect, useRef } from 'react';
import { TabConfig } from '../../main/types';
import { useDragSort } from '../hooks/useDragSort';

export const TabBar: React.FC = () => {
  // ... 之前的状态定义 ...

  // 拖拽排序逻辑
  const { handleDragStart, handleDragOver, handleDrop } = useDragSort({
    items: tabs,
    onSort: async (newTabs) => {
      await ipcRenderer.invoke('tab:reorder', newTabs.map(tab => tab.id));
      setTabs(newTabs);
    }
  });

  return (
    <div className="... drag">
      <div
        ref={tabsContainerRef}
        className="... no-drag"
        onDragOver={handleDragOver}
      >
        {tabs.map(tab => (
          <TabItem
            key={tab.id}
            tab={tab}
            isActive={activeTabId === tab.id}
            onClose={closeTab}
            onClick={() => switchTab(tab.id)}
            onDragStart={(e) => handleDragStart(e, tab)}
            onDrop={(e) => handleDrop(e, tab)}
          />
        ))}
      </div>
      {/* ... */}
    </div>
  );
};
```

```typescript:src/renderer/hooks/useDragSort.ts
import { useState } from 'react';

interface UseDragSortProps<T> {
  items: T[];
  onSort: (newItems: T[]) => void;
}

export function useDragSort<T extends { id: string }>({
  items,
  onSort
}: UseDragSortProps<T>) {
  const [draggedItem, setDraggedItem] = useState<T | null>(null);

  const handleDragStart = (e: React.DragEvent, item: T) => {
    setDraggedItem(item);
    e.dataTransfer.effectAllowed = 'move';
    e.dataTransfer.setData('text/plain', item.id);

    // 设置拖拽预览图
    const preview = document.createElement('div');
    preview.className = 'bg-white shadow-lg rounded-md p-2';
    preview.textContent = (item as any).title || item.id;
    document.body.appendChild(preview);
    e.dataTransfer.setDragImage(preview, 0, 0);
    setTimeout(() => preview.remove(), 0);
  };

  const handleDragOver = (e: React.DragEvent) => {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'move';
  };

  const handleDrop = (e: React.DragEvent, target: T) => {
    e.preventDefault();
    if (!draggedItem || draggedItem.id === target.id) return;

    const newItems = [...items];
    const draggedIndex = items.findIndex(item => item.id === draggedItem.id);
    const targetIndex = items.findIndex(item => item.id === target.id);

    newItems.splice(draggedIndex, 1);
    newItems.splice(targetIndex, 0, draggedItem);

    onSort(newItems);
    setDraggedItem(null);
  };

  return {
    handleDragStart,
    handleDragOver,
    handleDrop
  };
}
```

### 2. 标签页预览

```typescript:src/renderer/components/TabPreview.tsx
import React, { useState, useEffect } from 'react';
import { ipcRenderer } from 'electron';

interface TabPreviewProps {
  tabId: string;
  className?: string;
}

export const TabPreview: React.FC<TabPreviewProps> = ({
  tabId,
  className = ''
}) => {
  const [previewUrl, setPreviewUrl] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let mounted = true;

    const loadPreview = async () => {
      try {
        setLoading(true);
        const preview = await ipcRenderer.invoke('tab:getPreview', tabId);
        if (mounted) {
          setPreviewUrl(preview);
          setLoading(false);
        }
      } catch (error) {
        console.error('Failed to load preview:', error);
        if (mounted) setLoading(false);
      }
    };

    loadPreview();
    return () => { mounted = false; };
  }, [tabId]);

  if (loading) {
    return (
      <div className={`
        w-[320px] h-[180px]
        bg-gray-100 rounded-lg
        flex items-center justify-center
        ${className}
      `}>
        <LoadingSpinner className="w-8 h-8" />
      </div>
    );
  }

  return previewUrl ? (
    <img
      src={previewUrl}
      alt="Tab Preview"
      className={`
        w-[320px] h-[180px]
        object-cover rounded-lg
        shadow-lg
        ${className}
      `}
    />
  ) : null;
};
```

### 3. 标签页悬停预览

```typescript:src/renderer/components/TabItem.tsx
import React, { useState, useRef } from 'react';
import { useDebounce } from '../hooks/useDebounce';
import { TabPreview } from './TabPreview';

export const TabItem: React.FC<TabItemProps> = ({
  tab,
  isActive,
  onClose,
  onClick,
  onDragStart,
  onDrop
}) => {
  const [showPreview, setShowPreview] = useState(false);
  const previewTimeoutRef = useRef<NodeJS.Timeout>();

  const handleMouseEnter = useDebounce(() => {
    if (!isActive) {
      previewTimeoutRef.current = setTimeout(() => {
        setShowPreview(true);
      }, 500); // 延迟显示预览
    }
  }, 100);

  const handleMouseLeave = () => {
    if (previewTimeoutRef.current) {
      clearTimeout(previewTimeoutRef.current);
    }
    setShowPreview(false);
  };

  return (
    <div
      className="... relative"
      onClick={onClick}
      onMouseEnter={handleMouseEnter}
      onMouseLeave={handleMouseLeave}
      draggable
      onDragStart={onDragStart}
      onDrop={onDrop}
    >
      {/* ... 原有的标签页内容 ... */}

      {/* 预览弹窗 */}
      {showPreview && (
        <div className="
          absolute top-full left-0 mt-2
          z-50 animate-fade-in
          pointer-events-none
        ">
          <TabPreview
            tabId={tab.id}
            className="shadow-xl"
          />
        </div>
      )}
    </div>
  );
};
```

### 4. 手势支持

```typescript:src/renderer/hooks/useTabGestures.ts
import { useRef, useCallback } from 'react';

interface UseTabGesturesProps {
  onSwipeLeft?: () => void;
  onSwipeRight?: () => void;
  onSwipeUp?: () => void;
  threshold?: number;
}

export function useTabGestures({
  onSwipeLeft,
  onSwipeRight,
  onSwipeUp,
  threshold = 50
}: UseTabGesturesProps) {
  const touchStartRef = useRef<{ x: number; y: number } | null>(null);

  const handleTouchStart = useCallback((e: React.TouchEvent) => {
    touchStartRef.current = {
      x: e.touches[0].clientX,
      y: e.touches[0].clientY
    };
  }, []);

  const handleTouchMove = useCallback((e: React.TouchEvent) => {
    if (!touchStartRef.current) return;

    const deltaX = e.touches[0].clientX - touchStartRef.current.x;
    const deltaY = e.touches[0].clientY - touchStartRef.current.y;

    // 判断是水平还是垂直滑动
    if (Math.abs(deltaX) > Math.abs(deltaY)) {
      if (Math.abs(deltaX) > threshold) {
        if (deltaX > 0 && onSwipeRight) {
          onSwipeRight();
          touchStartRef.current = null;
        } else if (deltaX < 0 && onSwipeLeft) {
          onSwipeLeft();
          touchStartRef.current = null;
        }
      }
    } else if (Math.abs(deltaY) > threshold && deltaY < 0 && onSwipeUp) {
      onSwipeUp();
      touchStartRef.current = null;
    }
  }, [onSwipeLeft, onSwipeRight, onSwipeUp, threshold]);

  const handleTouchEnd = useCallback(() => {
    touchStartRef.current = null;
  }, []);

  return {
    handleTouchStart,
    handleTouchMove,
    handleTouchEnd
  };
}
```

### 5. 标签页动画

```typescript:src/renderer/components/TabBar.tsx
import { AnimatePresence, motion } from 'framer-motion';

export const TabBar: React.FC = () => {
  // ... 其他代码 ...

  return (
    <div className="...">
      <div className="...">
        <AnimatePresence>
          {tabs.map(tab => (
            <motion.div
              key={tab.id}
              initial={{ opacity: 0, x: -20 }}
              animate={{ opacity: 1, x: 0 }}
              exit={{ opacity: 0, x: 20 }}
              transition={{ duration: 0.2 }}
            >
              <TabItem
                tab={tab}
                isActive={activeTabId === tab.id}
                onClose={closeTab}
                onClick={() => switchTab(tab.id)}
                onDragStart={(e) => handleDragStart(e, tab)}
                onDrop={(e) => handleDrop(e, tab)}
              />
            </motion.div>
          ))}
        </AnimatePresence>
      </div>
    </div>
  );
};
```

### 6. 更新 Tailwind 配置以支持动画

```javascript:tailwind.config.js
module.exports = {
  // ... 其他配置 ...
  theme: {
    extend: {
      animation: {
        'spin': 'spin 1s linear infinite',
        'fade-in': 'fadeIn 0.2s ease-in-out',
        'slide-up': 'slideUp 0.2s ease-out',
        'slide-down': 'slideDown 0.2s ease-out',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideUp: {
          '0%': { transform: 'translateY(10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
        slideDown: {
          '0%': { transform: 'translateY(-10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
      },
    },
  },
  plugins: [
    // ... 之前的插件 ...
  ],
}
```

这些优化实现提供了：

1. **拖拽排序**

   - 平滑的拖拽体验
   - 视觉反馈
   - 动画效果

2. **标签页预览**

   - 悬停预览
   - 加载状态
   - 平滑过渡

3. **触摸支持**

   - 手势导航
   - 滑动切换
   - 自定义阈值

4. **动画效果**

   - 标签页切换动画
   - 预览弹出动画
   - 加载动画

5. **性能优化**
   - 延迟加载预览
   - 防抖处理
   - 动画性能优化
