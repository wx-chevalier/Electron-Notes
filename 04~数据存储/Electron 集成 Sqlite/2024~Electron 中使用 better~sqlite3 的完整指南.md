# Electron 中使用 better-sqlite3 的完整指南

## 一、简介

better-sqlite3 是一个高性能的 SQLite3 数据库驱动,在 Electron 应用中使用它需要特别注意跨平台兼容性问题。本文将详细介绍如何在 Electron 项目中正确集成和使用 better-sqlite3。

## 二、项目配置

### 1. 基础依赖安装

```bash
# 创建新项目
mkdir electron-sqlite-demo
cd electron-sqlite-demo
npm init -y

# 安装依赖
npm install electron better-sqlite3 --save
npm install electron-builder electron-rebuild --save-dev
```

### 2. package.json 配置

```json
{
  "name": "electron-sqlite-demo",
  "version": "1.0.0",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "postinstall": "electron-builder install-app-deps",
    "rebuild": "electron-rebuild -f -w better-sqlite3",
    "build": "electron-builder build",
    "build:win": "electron-builder build --win",
    "build:mac": "electron-builder build --mac",
    "build:linux": "electron-builder build --linux"
  },
  "dependencies": {
    "better-sqlite3": "^8.5.0",
    "electron": "^25.0.0"
  },
  "devDependencies": {
    "electron-builder": "^24.6.0",
    "electron-rebuild": "^3.2.9"
  },
  "build": {
    "appId": "com.example.app",
    "productName": "ElectronSQLiteDemo",
    "directories": {
      "output": "dist"
    },
    "files": ["**/*"],
    "extraResources": [
      {
        "from": "node_modules/better-sqlite3/build/Release/",
        "to": "better-sqlite3",
        "filter": ["*.node"]
      }
    ],
    "win": {
      "target": ["nsis"]
    },
    "mac": {
      "target": ["dmg"]
    },
    "linux": {
      "target": ["AppImage"]
    }
  }
}
```

## 三、数据库管理类实现

### 1. 数据库管理器

```typescript
// src/database/Database.ts
import { app } from "electron";
import path from "path";

interface DatabaseConfig {
  filename: string;
  memory?: boolean;
  readonly?: boolean;
  fileMustExist?: boolean;
}

export class Database {
  private db: any;
  private sqlite3: any;
  private readonly config: DatabaseConfig;

  constructor(config: DatabaseConfig) {
    this.config = config;
  }

  async connect() {
    try {
      await this.loadSqlite();
      const dbPath = this.getDatabasePath();

      this.db = new this.sqlite3(dbPath, {
        readonly: this.config.readonly || false,
        fileMustExist: this.config.fileMustExist || false,
        memory: this.config.memory || false,
      });

      this.db.pragma("foreign_keys = ON");
      console.log("Database connected successfully");
    } catch (error) {
      console.error("Database connection error:", error);
      throw error;
    }
  }

  private async loadSqlite() {
    const isDev = process.env.NODE_ENV === "development";

    try {
      if (isDev) {
        this.sqlite3 = require("better-sqlite3");
      } else {
        const binaryPath = this.getSqliteBinaryPath();
        this.sqlite3 = require(binaryPath);
      }
    } catch (error) {
      console.error("Failed to load SQLite binary:", error);
      throw error;
    }
  }

  private getSqliteBinaryPath(): string {
    const resourcePath = process.resourcesPath;
    const platform = process.platform;
    const arch = process.arch;
    const filename = this.getSqliteFileName(platform, arch);

    return path.join(resourcePath, "better-sqlite3", filename);
  }

  private getSqliteFileName(platform: string, arch: string): string {
    switch (platform) {
      case "win32":
        return "better_sqlite3.node";
      case "darwin":
        return "better_sqlite3.node";
      case "linux":
        return "better_sqlite3.node";
      default:
        throw new Error(`Unsupported platform: ${platform}`);
    }
  }

  private getDatabasePath(): string {
    if (this.config.memory) {
      return ":memory:";
    }

    const isDev = process.env.NODE_ENV === "development";

    if (isDev) {
      return path.join(__dirname, "..", "..", this.config.filename);
    } else {
      return path.join(app.getPath("userData"), this.config.filename);
    }
  }

  // 数据库操作方法
  prepare(sql: string) {
    return this.db.prepare(sql);
  }

  transaction(fn: Function) {
    return this.db.transaction(fn);
  }

  close() {
    if (this.db) {
      this.db.close();
    }
  }
}
```

### 2. 数据库迁移管理

```typescript
// src/database/Migration.ts
export class Migration {
  private db: Database;

  constructor(db: Database) {
    this.db = db;
  }

  async migrate() {
    this.db
      .prepare(
        `
      CREATE TABLE IF NOT EXISTS migrations (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        applied_at DATETIME DEFAULT CURRENT_TIMESTAMP
      )
    `
      )
      .run();

    const applied = this.db
      .prepare("SELECT name FROM migrations")
      .all()
      .map((row) => row.name);

    for (const migration of this.getMigrations()) {
      if (!applied.includes(migration.name)) {
        try {
          this.db.transaction(() => {
            migration.up(this.db);
            this.db
              .prepare("INSERT INTO migrations (name) VALUES (?)")
              .run(migration.name);
          })();
        } catch (error) {
          console.error(`Migration ${migration.name} failed:`, error);
          throw error;
        }
      }
    }
  }

  private getMigrations() {
    return [
      {
        name: "001-initial",
        up: (db: Database) => {
          db.prepare(
            `
            CREATE TABLE IF NOT EXISTS users (
              id INTEGER PRIMARY KEY AUTOINCREMENT,
              name TEXT NOT NULL,
              email TEXT UNIQUE NOT NULL,
              created_at DATETIME DEFAULT CURRENT_TIMESTAMP
            )
          `
          ).run();
        },
      },
    ];
  }
}
```

## 四、应用主进程实现

```typescript
// src/main.ts
import { app, BrowserWindow } from "electron";
import { Database } from "./database/Database";
import { Migration } from "./database/Migration";

class App {
  private db: Database;
  private mainWindow: BrowserWindow | null = null;

  constructor() {
    this.db = new Database({
      filename: "app.db",
    });
  }

  async init() {
    try {
      await this.db.connect();

      // 运行数据库迁移
      const migration = new Migration(this.db);
      await migration.migrate();

      // 创建主窗口
      await this.createWindow();

      console.log("Application initialized successfully");
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
        contextIsolation: false,
      },
    });

    await this.mainWindow.loadFile("index.html");
  }

  // 示例：用户相关操作
  addUser(name: string, email: string) {
    const stmt = this.db.prepare(
      "INSERT INTO users (name, email) VALUES (?, ?)"
    );
    return stmt.run(name, email);
  }

  getUser(id: number) {
    const stmt = this.db.prepare("SELECT * FROM users WHERE id = ?");
    return stmt.get(id);
  }

  getAllUsers() {
    const stmt = this.db.prepare("SELECT * FROM users");
    return stmt.all();
  }

  cleanup() {
    this.db.close();
  }
}

// 应用实例
const application = new App();

// 应用生命周期管理
app.on("ready", () => {
  application.init();
});

app.on("window-all-closed", () => {
  application.cleanup();
  if (process.platform !== "darwin") {
    app.quit();
  }
});

app.on("activate", () => {
  if (BrowserWindow.getAllWindows().length === 0) {
    application.init();
  }
});
```

## 五、构建和发布

### 1. 开发环境构建

```bash
# 安装依赖
npm install

# 重建 native 模块
npm run rebuild

# 启动应用
npm start
```

### 2. 生产环境构建

```bash
# Windows
npm run build:win

# macOS
npm run build:mac

# Linux
npm run build:linux
```

### 3. 自动化构建脚本

```yaml
# .github/workflows/build.yml
name: Build and Release

on:
  push:
    tags:
      - "v*"

jobs:
  build:
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os: [windows-latest, macos-latest, ubuntu-latest]

    steps:
      - uses: actions/checkout@v2

      - name: Setup Node.js
        uses: actions/setup-node@v2
        with:
          node-version: "16"

      - name: Install dependencies
        run: npm install

      - name: Rebuild native modules
        run: npm run rebuild

      - name: Build application
        run: npm run build

      - name: Upload artifacts
        uses: actions/upload-artifact@v2
        with:
          name: release-${{ matrix.os }}
          path: dist/*
```

## 六、注意事项

1. **跨平台兼容性**

   - 确保为每个目标平台编译 native 模块
   - 正确处理文件路径分隔符
   - 考虑不同平台的文件权限问题

2. **性能优化**

   - 使用预编译语句
   - 合理使用事务
   - 避免频繁打开关闭数据库连接

3. **安全性考虑**

   - 使用参数化查询防止 SQL 注入
   - 限制数据库文件访问权限
   - 加密敏感数据

4. **错误处理**

   - 实现完善的错误捕获机制
   - 提供用户友好的错误提示
   - 记录详细的错误日志

5. **数据备份**
   - 实现定期备份机制
   - 提供数据导入导出功能
   - 处理数据迁移和升级

## 七、常见问题解决

1. **模块加载失败**

```typescript
try {
  require("better-sqlite3");
} catch (error) {
  console.error("SQLite module loading failed:", error);
  // 提供友好的错误提示
  dialog.showErrorBox(
    "Database Error",
    "Failed to load database module. Please reinstall the application."
  );
}
```

2. **数据库文件访问错误**

```typescript
const ensureDatabaseDirectory = () => {
  const dbDir = path.dirname(dbPath);
  if (!fs.existsSync(dbDir)) {
    fs.mkdirSync(dbDir, { recursive: true });
  }
};
```

3. **跨平台路径问题**

```typescript
const dbPath = path
  .join(app.getPath("userData"), "database", "app.db")
  .replace(/\\/g, "/");
```

## 八、总结

在 Electron 应用中使用 better-sqlite3 需要特别注意：

1. 正确配置构建环境
2. 处理好跨平台兼容性
3. 实现完善的错误处理
4. 注意性能优化
5. 确保数据安全

通过本文的配置和实现方式，可以在 Electron 应用中稳定可靠地使用 SQLite 数据库。

## 九、参考资源

- [better-sqlite3 文档](https://github.com/JoshuaWise/better-sqlite3)
- [Electron 文档](https://www.electronjs.org/docs)
- [electron-builder 文档](https://www.electron.build/)

希望这个完整的教程能帮助你在 Electron 项目中成功集成和使用 better-sqlite3。
