# Playwright 最佳实践

_版本：1.0 | 创建时间：2026-04-23 | 作者：Trump 🇺🇸_

---

## 📋 目录

1. [概述](#概述)
2. [核心优势](#核心优势)
3. [进程管理](#进程管理)
4. [浏览器操控](#浏览器操控)
5. [绕过检测](#绕过检测)
6. [多开方案](#多开方案)
7. [Cookie 管理](#cookie-管理)
8. [代码示例](#代码示例)
9. [性能对比](#性能对比)
10. [注意事项](#注意事项)

---

## 概述

Playwright 是微软开发的自动化浏览器测试框架，相比 Selenium 具有更快、更稳定、更简单的优势。

---

## 🚀 核心优势

| 特性 | Playwright | Selenium |
|------|-----------|----------|
| **开发商** | Microsoft | Selenium 社区 |
| **发布时间** | 2020 年 | 2004 年 |
| **速度** | ⚡ 快（原生 CDP） | 🐌 慢（WebDriver） |
| **稳定性** | ✅ 高（自动等待） | ⚠️ 中（手动等待） |
| **浏览器驱动** | ❌ 不需要 | ✅ 需要 |
| **自带浏览器** | ✅ 是 | ❌ 否 |
| **版本匹配** | ✅ 自动 | ❌ 手动 |
| **隐身能力** | ✅ 强 | ⚠️ 中 |

---

## 🔄 进程管理

### ✅ 不需要常驻进程

**Playwright 工作方式：**
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    # 启动浏览器
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    
    # 操作网页
    page.goto("https://example.com")
    page.click("button")
    
    # 关闭浏览器（自动释放资源）
    browser.close()
```

**特点：**
- ✅ 按需启动，用完关闭
- ✅ 资源占用低
- ✅ 生命周期短（几秒到几分钟）

---

## 🌐 浏览器操控

### ✅ 直接操控浏览器

**协议对比：**

| 浏览器 | Playwright 协议 | Selenium 协议 |
|--------|----------------|--------------|
| **Chromium** | CDP (Chrome DevTools Protocol) | WebDriver |
| **Firefox** | WebDriver BiDi | WebDriver |
| **WebKit** | WebKit 私有协议 | WebDriver |

**优势：**
- ✅ 直接控制浏览器，不经过 WebDriver
- ✅ 速度更快（少一层中转）
- ✅ 更稳定（自动等待元素）

---

## 🛡️ 绕过检测

### ✅ 天生隐身

**自动处理：**
| 检测点 | Playwright 处理 |
|--------|----------------|
| navigator.webdriver | ✅ 自动设为 undefined |
| Chrome 插件列表 | ✅ 模拟真实 Chrome |
| Canvas 指纹 | ✅ 支持自定义 |
| User-Agent | ✅ 可随意伪装 |

**高级伪装（注入脚本）：**
```python
page.add_init_script("""
    Object.defineProperty(navigator, 'webdriver', { get: () => undefined });
    Object.defineProperty(navigator, 'plugins', { get: () => [1, 2, 3, 4, 5] });
    Object.defineProperty(navigator, 'languages', { get: () => ['zh-CN', 'zh'] });
""")
```

**结论：** 对于 95% 的网站（包括小红书、推特），Playwright 的默认配置就足够隐身了。

---

## 🧊 多开方案

### ✅ 支持无限多开

**上下文隔离 (Context Isolation)：**
```python
# 启动一个浏览器
browser = p.chromium.launch(headless=True)

# 创建 5 个完全独立的隐身窗口
contexts = [browser.new_context() for _ in range(5)]

# 每个窗口独立操作
for i, ctx in enumerate(contexts):
    page = ctx.new_page()
    page.goto("https://www.xiaohongshu.com")
    # 每个窗口可以加载不同的 Cookie！
```

**特点：**
- ✅ 完全隔离 - 每个窗口独立的 Cookie、LocalStorage、History
- ✅ 互不干扰 - 窗口 A 登录账号 1，窗口 B 登录账号 2
- ✅ 资源节省 - 只启动一个 Chrome 进程

---

## 🍪 Cookie 管理

### ✅ 全状态保存

**保存状态（登录一次）：**
```python
# 登录成功后，保存所有状态
state = context.storage_state(path="my_account_state.json")
```

**加载状态（直接复用）：**
```python
# 下次启动，直接加载状态，无需重新登录
context = browser.new_context(storage_state="my_account_state.json")
page = context.new_page()
page.goto("https://creator.xiaohongshu.com") # 直接就是登录状态！
```

**操作单个 Cookie：**
```python
# 添加 Cookie
context.add_cookies([{"name": "session_id", "value": "xxx", "url": "https://example.com"}])

# 删除所有 Cookie
context.clear_cookies()
```

---

## 📝 代码示例

### 发布小红书笔记

```python
from playwright.sync_api import sync_playwright

def publish_xiaohongshu(title, content, images):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        context = browser.new_context(
            storage_state="xhs_login_state.json"  # 加载登录状态
        )
        page = context.new_page()
        
        # 1. 打开创作平台
        page.goto("https://creator.xiaohongshu.com/publish")
        
        # 2. 上传图片
        page.set_input_files("input[type=file]", images)
        
        # 3. 填写标题
        page.fill("input.title", title)
        
        # 4. 填写内容
        page.fill("textarea.content", content)
        
        # 5. 点击发布
        page.click("button.publish")
        
        browser.close()

# 调用
publish_xiaohongshu("标题", "内容", ["image.jpg"])
```

### 多账号并发发布

```python
def multi_account_publish(accounts, title, content):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        
        # 创建多个独立上下文
        tasks = []
        for account in accounts:
            context = browser.new_context(storage_state=account['state'])
            page = context.new_page()
            tasks.append((page, account, title, content))
        
        # 并发执行
        for page, account, title, content in tasks:
            page.goto("https://creator.xiaohongshu.com/publish")
            # ... 发布逻辑
        
        browser.close()
```

---

## 📊 性能对比

| 指标 | Playwright | Selenium |
|------|-----------|----------|
| **启动速度** | ⚡ 2-3 秒 | 🐌 5-8 秒 |
| **内存占用** | 💾 50-100MB | 💾 150-300MB |
| **操作速度** | ⚡ 快 | 🐌 慢 |
| **稳定性** | ✅ 99% | ⚠️ 85% |
| **错误率** | 📉 低 | 📈 高 |

---

## ⚠️ 注意事项

### 1. 环境要求
- Python 3.8+
- 需要安装 `playwright` 包
- 需要安装浏览器：`playwright install chromium`

### 2. 反爬策略
- 不要频繁请求（建议间隔 1-3 秒）
- 使用合理的 User-Agent
- 避免在深夜大量操作

### 3. 错误处理
```python
try:
    page.click("button.submit")
except Exception as e:
    print(f"操作失败：{e}")
    page.screenshot(path="error.png")  # 保存错误截图
```

### 4. 资源释放
- 始终使用 `with` 语句或 `try/finally`
- 确保浏览器正确关闭
- 避免内存泄漏

---

## 🎯 总结

**Playwright 的优势：**
1. ✅ 不需要常驻进程
2. ✅ 不需要 chromedriver
3. ✅ 自带浏览器，自动匹配版本
4. ✅ 绕过 95% 的反爬检测
5. ✅ 支持无限多开，Cookie 完全隔离
6. ✅ 状态保存，登录一次永久复用
7. ✅ 全控制能力（键盘、鼠标、网络、文件）

**适用场景：**
- 自媒体自动化发布
- 数据采集
- 自动化测试
- 多账号管理

---

_文档结束_
