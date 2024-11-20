# BrowserWindow vs BrowserView

## 一、基本概念

### BrowserWindow

```javascript
// 主进程
const { BrowserWindow } = require("electron");

// 创建窗口
const win = new BrowserWindow({
  width: 800,
  height: 600,
  webPreferences: {
    nodeIntegration: true,
  },
});

// 加载内容
win.loadURL("https://example.com");
```

- 是一个完整的窗口容器
- 包含标题栏、窗口控制按钮
- 可以包含多个 BrowserView
- 直接与操作系统窗口管理器交互

### BrowserView

```javascript
// 主进程
const { BrowserView, BrowserWindow } = require("electron");

const win = new BrowserWindow({ width: 800, height: 600 });

// 创建视图
const view = new BrowserView();
win.setBrowserView(view);

// 设置视图位置和大小
view.setBounds({ x: 0, y: 0, width: 800, height: 600 });

// 加载内容
view.webContents.loadURL("https://example.com");
```

- 是嵌入在 BrowserWindow 中的内容区域
- 没有自己的窗口装饰
- 可以独立加载和显示网页内容
- 多个 View 可以在同一个 Window 中切换

## 二、关系图解

```plaintext
+------------------------+
|     BrowserWindow      |
|  +------------------+ |
|  |   BrowserView 1  | |
|  |                  | |
|  +------------------+ |
|  +------------------+ |
|  |   BrowserView 2  | |
|  |                  | |
|  +------------------+ |
+------------------------+
```

## 三、主要区别

### 1. 功能范围

```javascript
// BrowserWindow - 完整窗口控制
const win = new BrowserWindow({
  frame: false, // 无边框
  transparent: true, // 透明
  titleBarStyle: "hidden",
});

// BrowserView - 仅内容区域
const view = new BrowserView({
  webPreferences: {
    nodeIntegration: true,
  },
});
```

### 2. 使用场景

```javascript
// 1. 多内容区域切换
const view1 = new BrowserView();
const view2 = new BrowserView();

win.setBrowserView(view1); // 显示视图1
win.setBrowserView(view2); // 切换到视图2

// 2. 嵌入式浏览器
view.setBounds({
  x: 0,
  y: 70, // 留出顶部空间
  width: 800,
  height: 530,
});
```

## 四、常见使用模式

### 1. 单窗口多视图

```javascript
class MultiViewWindow {
  constructor() {
    this.window = new BrowserWindow({
      width: 800,
      height: 600,
    });

    this.views = new Map();
  }

  addView(id, url) {
    const view = new BrowserView();
    view.webContents.loadURL(url);
    this.views.set(id, view);
  }

  switchView(id) {
    const view = this.views.get(id);
    if (view) {
      this.window.setBrowserView(view);
    }
  }
}
```

### 2. 嵌入式浏览器

```javascript
class EmbeddedBrowser {
  constructor() {
    this.window = new BrowserWindow({
      width: 800,
      height: 600,
    });

    // 创建工具栏区域
    this.window.loadFile("toolbar.html");

    // 创建浏览器视图
    this.browserView = new BrowserView();
    this.window.setBrowserView(this.browserView);

    // 设置视图位置（留出工具栏空间）
    this.browserView.setBounds({
      x: 0,
      y: 40,
      width: 800,
      height: 560,
    });
  }

  navigate(url) {
    this.browserView.webContents.loadURL(url);
  }
}
```

## 五、最佳实践

### 1. 何时使用 BrowserWindow

```javascript
// 适用场景：
// 1. 独立窗口应用
const mainWindow = new BrowserWindow({
  width: 800,
  height: 600,
});

// 2. 需要窗口装饰和控制
const customWindow = new BrowserWindow({
  frame: false,
  transparent: true,
});

// 3. 单一内容展示
mainWindow.loadFile("index.html");
```

### 2. 何时使用 BrowserView

```javascript
// 适用场景：
// 1. 多内容区域管理
const views = {
  main: new BrowserView(),
  settings: new BrowserView(),
  help: new BrowserView(),
};

// 2. 嵌入式内容
const contentView = new BrowserView();
win.setBrowserView(contentView);
contentView.setBounds(/* ... */);

// 3. 独立内容隔离
view.webContents.setWindowOpenHandler(/* ... */);
```

## 六、注意事项

1. **内存管理**

```javascript
// 及时清理不需要的视图
view.webContents.destroy();
```

2. **安全考虑**

```javascript
const view = new BrowserView({
  webPreferences: {
    nodeIntegration: false,
    contextIsolation: true,
  },
});
```

3. **性能优化**

```javascript
// 避免频繁切换视图
// 复用视图而不是重新创建
```

4. **布局管理**

```javascript
// 监听窗口大小变化
win.on("resize", () => {
  const bounds = win.getBounds();
  view.setBounds({
    x: 0,
    y: 0,
    width: bounds.width,
    height: bounds.height,
  });
});
```

选择使用 BrowserWindow 还是 BrowserView 主要取决于：

- 应用架构需求
- 内容展示方式
- 交互模式设计
- 性能和资源考虑
