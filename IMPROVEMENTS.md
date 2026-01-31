# 代码改进建议清单

## 📋 简介

本文档列出了所有可选的改进建议。这些改进不是必需的，但可以进一步提升代码质量、用户体验和可维护性。

---

## 🚀 高优先级建议

### 1. 添加 PNG 格式图标

**当前状态**: 使用 SVG 图标  
**问题**: 某些旧版 Chrome 可能不支持 SVG 作为扩展图标  
**建议**: 添加 16x16、48x48 和 128x128 像素的 PNG 图标

**实现步骤**:
1. 将现有 SVG 图标转换为 PNG（使用在线工具或图像编辑器）
2. 创建三种尺寸：icon-16.png、icon-48.png、icon-128.png
3. 更新 manifest.json:
```json
"icons": {
  "16": "icons/icon-16.png",
  "48": "icons/icon-48.png",
  "128": "icons/icon-128.png"
}
```

---

### 2. 优化 Cookie 过滤逻辑

**文件**: `popup.js` 第 6-17 行

**当前代码**:
```javascript
const COOKIE_BLACKLIST = {
    patterns: [
        /^Hm_lvt_/,  // 百度统计访问记录
    ],
    exactNames: [
        'GIFT_BLOCK_COOKIE',
        'bmg_af_switch',
        'bmg_src_def_domain'
    ]
};
```

**建议改进**:
```javascript
/**
 * Cookie 过滤黑名单配置
 * 用于过滤不必要显示的 Cookie（如追踪、统计等）
 */
const COOKIE_BLACKLIST = {
    // 正则模式匹配（用于前缀/后缀匹配）
    patterns: [
        /^Hm_lvt_/,      // 百度统计 - 访问时间
        /^Hm_lpvt_/,     // 百度统计 - 最后访问时间
        /^_ga/,          // Google Analytics
        /^_gid/,         // Google Analytics ID
        /^__utm/,        // Google Analytics 传统追踪
    ],
    // 精确匹配的 Cookie 名称
    exactNames: [
        'GIFT_BLOCK_COOKIE',
        'bmg_af_switch',
        'bmg_src_def_domain',
        'PVID',                    // 页面访问 ID
        'b_nut',                   // Bilibili 设备指纹
        'fingerprint',             // 设备指纹
    ]
};
```

**效果**: 更清晰的 Cookie 列表，减少干扰信息。

---

### 3. 添加批量操作中断功能

**文件**: `popup.js` 批量操作函数

**问题**: 批量操作开始后无法中断，用户体验不佳

**实现方案**:

```javascript
// 在文件顶部添加全局变量
let batchOperationCancelled = false;

/**
 * 批量拉黑用户
 */
async function handleBatchBlock() {
    if (parsedUids.length === 0) {
        showStatus('⚠️ 请先解析 UID 列表', 'warning');
        return;
    }

    const progressEl = document.getElementById('batchProgress');
    const batchBlockBtn = document.getElementById('batchBlockBtn');
    const batchUnblockBtn = document.getElementById('batchUnblockBtn');

    // 禁用操作按钮，启用取消按钮
    batchBlockBtn.disabled = true;
    batchUnblockBtn.disabled = true;
    batchOperationCancelled = false;
    
    // 创建取消按钮
    const cancelBtn = document.createElement('button');
    cancelBtn.id = 'cancelBatchBtn';
    cancelBtn.className = 'btn btn-warning';
    cancelBtn.textContent = '⏸️ 取消操作';
    cancelBtn.style.marginTop = '8px';
    cancelBtn.onclick = () => {
        batchOperationCancelled = true;
        cancelBtn.disabled = true;
        cancelBtn.textContent = '正在取消...';
    };
    progressEl.parentNode.insertBefore(cancelBtn, progressEl.nextSibling);

    let successCount = 0;
    let failCount = 0;

    for (let i = 0; i < parsedUids.length; i++) {
        // 检查是否取消
        if (batchOperationCancelled) {
            progressEl.innerHTML = `⚠️ 操作已取消: 已处理 ${i}/${parsedUids.length} (成功 ${successCount}, 失败 ${failCount})`;
            showStatus('⚠️ 批量操作已取消', 'warning', 5000);
            break;
        }

        const uid = parsedUids[i];
        const itemEl = document.querySelector(`.uid-item[data-uid="${uid}"] .uid-status`);

        // 更新进度
        const percentage = Math.round(((i + 1) / parsedUids.length) * 100);
        progressEl.textContent = `⏳ 正在拉黑 ${i + 1}/${parsedUids.length} (${percentage}%): ${uid}`;
        if (itemEl) itemEl.textContent = '⏳ 处理中...';

        try {
            const result = await blockUser(uid);

            if (result.success) {
                successCount++;
                if (itemEl) itemEl.innerHTML = '<span class="status-success">✓ 成功</span>';
            } else {
                failCount++;
                if (itemEl) {
                    const errorSpan = document.createElement('span');
                    errorSpan.className = 'status-error';
                    errorSpan.textContent = `✗ ${result.message}`;
                    itemEl.innerHTML = '';
                    itemEl.appendChild(errorSpan);
                }
            }
        } catch (error) {
            failCount++;
            if (itemEl) itemEl.innerHTML = `<span class="status-error">✗ 错误</span>`;
        }

        // 实时更新底部状态提示
        showStatus(`⏳ 批量拉黑进度: ${i + 1}/${parsedUids.length} (成功 ${successCount}, 失败 ${failCount})`, 'info', 0);

        // 添加延迟避免频率限制
        if (i < parsedUids.length - 1) {
            await new Promise(resolve => setTimeout(resolve, 500));
        }
    }

    // 完成或取消后清理
    if (cancelBtn.parentNode) {
        cancelBtn.remove();
    }
    
    if (!batchOperationCancelled) {
        progressEl.innerHTML = `✓ 批量拉黑完成: 成功 ${successCount} 个, 失败 ${failCount} 个`;
        showStatus(`✓ 批量拉黑完成: 成功 ${successCount} 个, 失败 ${failCount} 个`, 'success', 8000);
    }

    // 重新启用按钮
    batchBlockBtn.disabled = false;
    batchUnblockBtn.disabled = false;
}

// 同样的修改应用到 handleBatchUnblock()
```

**效果**: 用户可以中途取消长时间运行的批量操作。

---

## 💡 中优先级建议

### 4. 添加操作日志记录

**建议**: 在 localStorage 中记录所有拉黑/取消拉黑操作

**实现方案**:

```javascript
/**
 * 记录操作到本地存储
 * @param {string} type - 操作类型: 'block' 或 'unblock'
 * @param {string} uid - 用户 UID
 * @param {boolean} success - 是否成功
 * @param {string} message - 结果消息
 */
function logOperation(type, uid, success, message) {
    try {
        const logs = JSON.parse(localStorage.getItem('bilibiliBlockLogs') || '[]');
        
        logs.push({
            type,
            uid,
            success,
            message,
            timestamp: new Date().toISOString()
        });
        
        // 只保留最近 500 条记录
        if (logs.length > 500) {
            logs.splice(0, logs.length - 500);
        }
        
        localStorage.setItem('bilibiliBlockLogs', JSON.stringify(logs));
    } catch (error) {
        console.error('记录操作失败:', error);
    }
}

/**
 * 获取操作日志
 * @param {number} limit - 返回的最大记录数
 * @returns {Array} 操作日志数组
 */
function getOperationLogs(limit = 100) {
    try {
        const logs = JSON.parse(localStorage.getItem('bilibiliBlockLogs') || '[]');
        return logs.slice(-limit).reverse(); // 返回最新的 N 条，倒序
    } catch (error) {
        console.error('读取日志失败:', error);
        return [];
    }
}

/**
 * 清除操作日志
 */
function clearOperationLogs() {
    try {
        localStorage.removeItem('bilibiliBlockLogs');
        showStatus('✓ 日志已清除', 'success');
    } catch (error) {
        console.error('清除日志失败:', error);
        showStatus('✗ 清除日志失败', 'error');
    }
}

// 在 blockUser() 和 unblockUser() 调用后添加日志
async function handleBlockTest() {
    // ... 现有代码 ...
    try {
        const result = await blockUser(userId);
        
        // 记录操作
        logOperation('block', userId, result.success, result.message);
        
        // ... 其余代码 ...
    }
}
```

**UI 增强**: 在 popup.html 中添加一个"操作历史"标签页：

```html
<button class="tab-btn" data-tab="history">📜 操作历史</button>

<!-- 在 tab-content 中添加 -->
<div class="tab-panel" id="tab-history">
    <div class="panel-section">
        <h3>操作历史记录</h3>
        <div class="history-controls">
            <button id="refreshLogsBtn" class="btn btn-primary">🔄 刷新</button>
            <button id="clearLogsBtn" class="btn btn-danger">🗑️ 清除日志</button>
        </div>
        <div id="historyList" class="history-list"></div>
    </div>
</div>
```

**效果**: 用户可以查看历史操作记录，便于追踪和审计。

---

### 5. 添加导入/导出 UID 列表功能

**建议**: 允许用户保存和加载 UID 列表

**实现方案**:

```javascript
/**
 * 导出 UID 列表为文本文件
 */
function exportUidList() {
    if (parsedUids.length === 0) {
        showStatus('⚠️ 没有可导出的 UID', 'warning');
        return;
    }
    
    const content = parsedUids.join('\n');
    const blob = new Blob([content], { type: 'text/plain;charset=utf-8' });
    const url = URL.createObjectURL(blob);
    
    const link = document.createElement('a');
    link.href = url;
    link.download = `bilibili-uids-${Date.now()}.txt`;
    link.click();
    
    URL.revokeObjectURL(url);
    showStatus(`✓ 已导出 ${parsedUids.length} 个 UID`, 'success');
}

/**
 * 从文件导入 UID 列表
 */
function importUidList() {
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = '.txt,.csv';
    
    input.onchange = (e) => {
        const file = e.target.files[0];
        if (!file) return;
        
        const reader = new FileReader();
        reader.onload = (event) => {
            const content = event.target.result;
            const lines = content.split(/[\r\n]+/);
            
            // 提取有效的 UID（纯数字）
            const uids = lines
                .map(line => line.trim())
                .filter(line => /^\d+$/.test(line))
                .filter((uid, index, self) => self.indexOf(uid) === index); // 去重
            
            if (uids.length > 0) {
                parsedUids = uids;
                renderUidList(parsedUids);
                showStatus(`✓ 成功导入 ${uids.length} 个 UID`, 'success');
            } else {
                showStatus('⚠️ 文件中没有找到有效的 UID', 'warning');
            }
        };
        
        reader.onerror = () => {
            showStatus('✗ 文件读取失败', 'error');
        };
        
        reader.readAsText(file);
    };
    
    input.click();
}

// 在批量操作面板添加导入/导出按钮
// popup.html:
<div class="batch-controls">
    <button id="parseBtn" class="btn btn-primary">解析 UID</button>
    <button id="importUidsBtn" class="btn btn-secondary">📥 导入</button>
    <button id="exportUidsBtn" class="btn btn-secondary">💾 导出</button>
    <button id="batchBlockBtn" class="btn btn-danger" disabled>批量拉黑</button>
    <button id="batchUnblockBtn" class="btn btn-warning" disabled>批量取消拉黑</button>
</div>

// 在 DOMContentLoaded 中绑定事件
document.getElementById('importUidsBtn').addEventListener('click', importUidList);
document.getElementById('exportUidsBtn').addEventListener('click', exportUidList);
```

**效果**: 用户可以保存 UID 列表供以后使用，或在多个设备间共享。

---

### 6. 优化 Cookie 读取性能

**文件**: `popup.js` `getAllBilibiliCookies()` 函数

**当前代码**:
```javascript
async function getAllBilibiliCookies() {
    return new Promise((resolve) => {
        chrome.cookies.getAll({ domain: '.bilibili.com' }, (cookies) => {
            chrome.cookies.getAll({ domain: 'bilibili.com' }, (cookies2) => {
                const allCookies = [...cookies, ...cookies2];
                const uniqueCookies = Array.from(
                    new Map(allCookies.map(c => [c.name, c])).values()
                );
                const filteredCookies = filterCookies(uniqueCookies);
                resolve(filteredCookies);
            });
        });
    });
}
```

**优化建议**:
```javascript
/**
 * 获取所有 Bilibili Cookie（包括 HttpOnly）
 * 优化版本：使用 Promise.all 并行获取
 */
async function getAllBilibiliCookies() {
    try {
        // 并行获取两个域名的 Cookie
        const [cookies1, cookies2] = await Promise.all([
            new Promise((resolve) => {
                chrome.cookies.getAll({ domain: '.bilibili.com' }, resolve);
            }),
            new Promise((resolve) => {
                chrome.cookies.getAll({ domain: 'bilibili.com' }, resolve);
            })
        ]);
        
        // 使用 Map 去重（O(n) 复杂度）
        const cookieMap = new Map();
        [...cookies1, ...cookies2].forEach(cookie => {
            cookieMap.set(cookie.name, cookie);
        });
        
        // 应用过滤规则
        const uniqueCookies = Array.from(cookieMap.values());
        const filteredCookies = filterCookies(uniqueCookies);
        
        return filteredCookies;
    } catch (error) {
        console.error('获取 Cookie 失败:', error);
        return [];
    }
}
```

**效果**: 稍微提升性能，代码更现代化。

---

## 🎨 低优先级建议

### 7. 添加深色模式支持

**建议**: 检测系统主题并相应调整 UI

**实现方案**:

```css
/* popup.css 中添加 */
@media (prefers-color-scheme: dark) {
    body {
        background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
    }
    
    .container {
        background: #0f3460;
        color: #e9ecef;
    }
    
    .header {
        background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
    }
    
    .stats {
        background: #16213e;
        border-bottom: 1px solid #1a1a2e;
    }
    
    .tab-btn {
        color: #adb5bd;
    }
    
    .tab-btn.active {
        background: #0f3460;
        color: #667eea;
    }
    
    /* 其他深色模式样式... */
}
```

**效果**: 更好的用户体验，减少眼睛疲劳。

---

### 8. 添加 JSDoc 注释

**建议**: 为所有主要函数添加 JSDoc 风格的注释

**示例**:
```javascript
/**
 * 拉黑用户（通过 Content Script 执行）
 * 
 * 此函数会获取当前的 Bilibili 标签页，并通过消息传递机制
 * 调用 Content Script 中的拉黑功能。
 * 
 * @async
 * @param {number|string} userId - 用户 ID（Bilibili UID）
 * @returns {Promise<{success: boolean, message: string, data?: any}>} 
 *          操作结果对象
 * @throws {Error} 当无法连接到 Bilibili 页面时抛出错误
 * 
 * @example
 * try {
 *   const result = await blockUser('1042653845');
 *   if (result.success) {
 *     console.log('拉黑成功');
 *   } else {
 *     console.error('拉黑失败:', result.message);
 *   }
 * } catch (error) {
 *   console.error('发生错误:', error);
 * }
 */
async function blockUser(userId) {
    // ...
}
```

**效果**: 更好的代码文档，IDE 智能提示更准确。

---

### 9. 添加配置选项页面

**建议**: 创建一个选项页面让用户自定义设置

**实现内容**:
- 批量操作延迟时间（当前固定 500ms）
- Cookie 过滤规则
- 日志保留数量
- 界面主题选择

**实现步骤**:
1. 创建 `options.html` 和 `options.js`
2. 在 manifest.json 中添加:
```json
{
  "options_page": "options.html"
}
```
3. 使用 `chrome.storage.sync` 保存用户设置

---

### 10. 添加统计信息

**建议**: 显示拉黑统计信息

**实现内容**:
- 总拉黑人数
- 今日拉黑人数
- 成功率
- 最近拉黑的用户

**实现方案**: 结合操作日志功能，在界面上显示统计数据。

---

### 11. 添加快捷键支持

**建议**: 添加键盘快捷键

**实现方案**:
```json
// manifest.json
{
  "commands": {
    "_execute_action": {
      "suggested_key": {
        "default": "Ctrl+Shift+B",
        "mac": "Command+Shift+B"
      },
      "description": "打开 Bilibili 拉黑助手"
    }
  }
}
```

---

## 🧪 测试建议

### 12. 添加单元测试

**建议**: 使用 Jest 或类似框架添加测试

**测试内容**:
- `validateUserId()` 函数
- `extractUids()` 函数
- `filterCookies()` 函数
- `escapeHtml()` 函数

**示例**:
```javascript
// tests/popup.test.js
describe('validateUserId', () => {
    test('应该接受有效的 UID', () => {
        expect(validateUserId('1042653845')).toBe(true);
        expect(validateUserId('123')).toBe(true);
    });
    
    test('应该拒绝无效的 UID', () => {
        expect(validateUserId('')).toBe(false);
        expect(validateUserId('abc')).toBe(false);
        expect(validateUserId('12345abc')).toBe(false);
        expect(validateUserId('123456789012345678901')).toBe(false); // 太长
    });
});
```

---

## 📚 文档建议

### 13. 添加开发文档

**建议**: 创建 `DEVELOPMENT.md`

**内容包括**:
- 开发环境设置
- 构建和测试说明
- 代码结构说明
- 贡献指南
- API 文档

---

### 14. 添加用户使用指南

**建议**: 在 README.md 中添加更详细的使用说明

**内容包括**:
- 常见问题解答
- 故障排除指南
- 使用截图和 GIF 动图
- 视频教程链接（如果有）

---

## 📊 实施优先级总结

| 优先级 | 建议 | 估计时间 | 影响 |
|--------|------|----------|------|
| 高 | 1. PNG 图标 | 30 分钟 | 兼容性 |
| 高 | 2. Cookie 过滤优化 | 15 分钟 | 用户体验 |
| 高 | 3. 批量操作中断 | 1 小时 | 用户体验 |
| 中 | 4. 操作日志 | 2 小时 | 功能增强 |
| 中 | 5. 导入/导出 UID | 1.5 小时 | 功能增强 |
| 中 | 6. Cookie 读取优化 | 30 分钟 | 性能 |
| 低 | 7. 深色模式 | 2 小时 | 用户体验 |
| 低 | 8. JSDoc 注释 | 2 小时 | 可维护性 |
| 低 | 9. 配置选项页面 | 3 小时 | 功能增强 |
| 低 | 10. 统计信息 | 2 小时 | 功能增强 |
| 低 | 11. 快捷键支持 | 30 分钟 | 用户体验 |
| 低 | 12. 单元测试 | 4 小时 | 代码质量 |
| 低 | 13-14. 文档 | 2 小时 | 可维护性 |

---

**文档日期**: 2026-01-31  
**版本**: 1.0
