# Playwright 浏览器自动化方案

> 基于 Docker 容器环境的 Playwright 最佳实践，对比 Selenium 等传统方案。

---

## 一、技术方案对比

### 1. Playwright vs Selenium vs Puppeteer

| 特性 | Playwright | Selenium | Puppeteer |
|------|-----------|----------|-----------|
| **开发商** | Microsoft | Selenium 社区 | Google |
| **发布时间** | 2020 年 | 2004 年 | 2017 年 |
| **速度** | ⚡ 快（原生 CDP） | 🐌 慢（WebDriver） | ⚡ 快（CDP） |
| **稳定性** | ✅ 高（自动等待） | ⚠️ 中（手动等待） | ✅ 高 |
| **浏览器驱动** | ❌ 不需要 | ✅ 需要 | ❌ 不需要 |
| **自带浏览器** | ✅ 是 | ❌ 否 | ✅ 仅 Chromium |
| **版本匹配** | ✅ 自动 | ❌ 手动 | ✅ 自动 |
| **隐身能力** | ✅ 强 | ⚠️ 中 | ✅ 强 |
| **多浏览器支持** | ✅ Chromium/Firefox/WebKit | ✅ 所有浏览器 | ❌ 仅 Chromium |
| **启动速度** | ⚡ 2-3 秒 | 🐌 5-8 秒 | ⚡ 2-3 秒 |
| **内存占用** | 💾 50-100MB | 💾 150-300MB | 💾 50-100MB |

### 2. 协议对比

| 浏览器 | Playwright 协议 | Selenium 协议 |
|--------|----------------|--------------|
| **Chromium** | CDP (Chrome DevTools Protocol) | WebDriver |
| **Firefox** | WebDriver BiDi | WebDriver |
| **WebKit** | WebKit 私有协议 | WebDriver |

**CDP 协议优势**：
- ✅ 直接控制浏览器（无中间层）
- ✅ 更快（少一层中转）
- ✅ 更稳定（原生协议）

---

## 二、进程管理模式

### 1. ⚠️ 关键区别：不需要常驻进程

| 场景 | Selenium | Playwright |
|------|----------|-----------|
| **空闲时** | 需要 WebDriver 常驻 | ❌ 无进程（零资源占用） |
| **执行时** | WebDriver + Chrome | 临时 Chrome 进程 |
| **完成后** | WebDriver 继续运行 | ❌ 全部关闭释放 |

### 2. 进程生命周期

**Playwright（按需模式）**：
```
脚本开始 → 启动 Chrome → 执行操作 → 关闭 Chrome → 脚本结束
（几秒到几分钟）
```

**Selenium（常驻模式）**：
```
[WebDriver] ──→ [Chrome] ──→ 操作
    ↑
    常驻运行（持续占用资源）
```

### 3. 工作代码示例

```python
from playwright.sync_api import sync_playwright

# 按需启动，用完关闭
with sync_playwright() as p:
    # 1. 启动浏览器（临时进程）
    browser = p.chromium.launch(headless=True)
    
    # 2. 操作页面
    page = browser.new_page()
    page.goto("https://example.com")
    
    # 3. 关闭浏览器（自动释放）
    browser.close()
# ← with 结束，Playwright 进程也结束，资源完全释放
```

---

## 三、定位器策略

### 1. 定位器优先级（Playwright 推荐）

| 定位器类型 | 示例 | 推荐度 | 说明 |
|-----------|------|--------|------|
| **Role** | `page.get_by_role("button", name="提交")` | ⭐⭐⭐⭐⭐ | **最推荐**，最稳定 |
| **Text** | `page.get_by_text("登录")` | ⭐⭐⭐⭐⭐ | 文本匹配，语义清晰 |
| **Label** | `page.get_by_label("用户名")` | ⭐⭐⭐⭐ | 表单元素，语义清晰 |
| **TestId** | `page.get_by_test_id("submit-btn")` | ⭐⭐⭐⭐⭐ | 测试专用，最稳定 |
| **Placeholder** | `page.get_by_placeholder("请输入...")` | ⭐⭐⭐⭐ | 输入框，语义清晰 |
| **CSS** | `page.locator("button.submit")` | ⭐⭐⭐⭐ | 通用，稳定 |
| **XPath** | `page.locator("//button[@id='submit']")` | ⭐⭐ | **不推荐**，脆弱易失效 |

### 2. 为什么不推荐 XPath

| 问题 | XPath | Role/Text |
|------|-------|-----------|
| **UI 结构变化** | ❌ 失效 | ✅ 仍然有效 |
| **元素位置变化** | ❌ 失效 | ✅ 仍然有效 |
| **维护成本** | 📈 高 | 📉 低 |

### 3. 代码对比

**✅ 推荐（Role/Text）**：
```python
# Role 定位器（最稳定）
page.get_by_role("button", name="发布").click()
page.get_by_role("textbox", name="标题").fill("我的标题")

# Text 定位器（语义清晰）
page.get_by_text("登录").click()
```

**❌ 不推荐（XPath）**：
```python
# XPath（脆弱，UI 变化就失效）
page.locator("//button[@id='submit']").click()
page.locator("//div[@class='content']/textarea").fill("内容")
```

---

## 四、绕过反爬检测

### 1. Playwright 自动处理

| 检测点 | Playwright 处理 |
|--------|----------------|
| `navigator.webdriver` | ✅ 自动设为 `undefined` |
| Chrome 插件列表 | ✅ 模拟真实 Chrome |
| Canvas 指纹 | ✅ 支持自定义 |
| User-Agent | ✅ 可随意伪装 |

### 2. 高级伪装（注入脚本）

```python
page.add_init_script("""
    Object.defineProperty(navigator, 'webdriver', { get: () => undefined });
    Object.defineProperty(navigator, 'plugins', { get: () => [1, 2, 3, 4, 5] });
    Object.defineProperty(navigator, 'languages', { get: () => ['zh-CN', 'zh'] });
""")
```

### 3. 结论

对于 **95% 的网站**（包括小红书、B 站、推特），Playwright 默认配置就足够隐身。

---

## 五、多账号多开方案

### 1. Context Isolation（上下文隔离）

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

### 2. 特点

| 特性 | 说明 |
|------|------|
| **完全隔离** | 每个窗口独立的 Cookie、LocalStorage、History |
| **互不干扰** | 窗口 A 登录账号 1，窗口 B 登录账号 2 |
| **资源节省** | 只启动一个 Chrome 进程 |

---

## 六、Cookie 管理

### 1. 保存登录状态

```python
# 登录成功后，保存所有状态
state = context.storage_state(path="my_account_state.json")
```

### 2. 加载状态复用

```python
# 下次启动，直接加载状态，无需重新登录
context = browser.new_context(storage_state="my_account_state.json")
page = context.new_page()
page.goto("https://creator.xiaohongshu.com")  # 直接就是登录状态！
```

### 3. 单个 Cookie 操作

```python
# 添加 Cookie
context.add_cookies([{
    "name": "session_id", 
    "value": "xxx", 
    "url": "https://example.com"
}])

# 删除所有 Cookie
context.clear_cookies()
```

---

## 七、完整示例

### 1. 发布小红书笔记

```python
from playwright.sync_api import sync_playwright

def publish_xiaohongshu(title, content, images):
    with sync_playwright() as p:
        # 启动浏览器
        browser = p.chromium.launch(headless=True)
        
        # 加载登录状态（Cookie）
        context = browser.new_context(
            storage_state="xhs_login_state.json"
        )
        page = context.new_page()
        
        # 1. 打开创作平台
        page.goto("https://creator.xiaohongshu.com/publish/publish_note")
        
        # 2. 上传图片
        page.locator("input[type=file]").set_input_files(images)
        
        # 3. 等待上传完成
        page.wait_for_selector(".upload-success", timeout=30000)
        
        # 4. 填写标题
        page.locator("input[name='title']").fill(title)
        
        # 5. 填写内容
        page.locator("textarea.content").fill(content)
        
        # 6. 点击发布
        page.get_by_role("button", name="发布").click()
        
        # 7. 等待成功提示
        page.wait_for_text("发布成功", timeout=10000)
        
        browser.close()

# 调用
publish_xiaohongshu(
    title="美食分享",
    content="今天做了好吃的...",
    images=["food1.jpg", "food2.jpg"]
)
```

### 2. 多账号并发发布

```python
def multi_account_publish(accounts, title, content):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        
        # 创建多个独立上下文
        tasks = []
        for account in accounts:
            context = browser.new_context(
                storage_state=account['state']
            )
            page = context.new_page()
            tasks.append((page, account))
        
        # 并发执行
        for page, account in tasks:
            page.goto("https://creator.xiaohongshu.com/publish")
            # ... 发布逻辑
        
        browser.close()
```

---

## 八、容器环境配置

### 1. 安装 Playwright

```bash
# 安装 Python 包
pip install playwright

# 安装浏览器（自动下载）
playwright install chromium
```

### 2. 浏览器位置

```bash
# 内置浏览器路径
~/.cache/ms-playwright/chromium-1217/chrome-linux/chrome
```

### 3. 系统依赖（需要管理员安装）

```bash
# Debian/Ubuntu
apt-get install -y libnspr4 libnss3 libatk1.0-0 libatk-bridge2.0-0 \
    libcups2 libdrm2 libxkbcommon0 libxcomposite1 libxdamage1 \
    libxfixes3 libxrandr2 libgbm1 libasound2
```

---

## 九、最佳实践总结

### 1. 核心规则

| 规则 | 说明 |
|------|------|
| **优先 Role/Text** | 不要用 XPath |
| **使用 with 语句** | 确保资源释放 |
| **按需启动** | 不需要常驻进程 |
| **保存登录状态** | 登录一次永久复用 |
| **降低频率** | 防风控，间隔 1-3 秒 |

### 2. 适用场景

| 场景 | 推荐方案 |
|------|----------|
| **有官方 API 的平台** | API 直接调用（B 站、推特） |
| **无官方 API 的平台** | Playwright（小红书、西瓜视频） |
| **多账号管理** | Playwright Context Isolation |
| **自动化测试** | Playwright Test |

---

## 十、故障排查

### 1. 元素找不到

```python
# 使用 wait_for 等待
page.wait_for_selector("button.submit", timeout=10000)
page.click("button.submit")

# 或使用 expect 断言
from playwright.sync_api import expect
expect(page.locator("button.submit")).to_be_visible()
```

### 2. 反爬被拦截

```python
# 添加伪装脚本
page.add_init_script("""
    Object.defineProperty(navigator, 'webdriver', { get: () => undefined });
""")

# 设置合理的 User-Agent
context = browser.new_context(
    user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) ..."
)
```

### 3. 超时问题

```python
# 增加超时时间
page.goto("https://example.com", timeout=60000)

# 或使用 slow_mo 降低速度
browser = p.chromium.launch(headless=True, slow_mo=100)
```

---

## 十一、参考资料

| 资源 | 链接 |
|------|------|
| **官方文档** | https://playwright.dev/python/ |
| **API 参考** | https://playwright.dev/python/docs/api/class-playwright |
| **最佳实践** | https://playwright.dev/python/docs/best-practices |
| **反爬技巧** | https://playwright.dev/python/docs/emulation |

---

_文档版本: 2026-04-29_
_适用环境: Docker 容器 + OpenClaw + 自媒体自动化_