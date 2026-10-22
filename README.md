# 从零做一个属于自己的 GitHub 客户端

一个**单文件 HTML** 的 GitHub 客户端：没有后端、没有构建工具、没有 npm 依赖。
双击打开就能用，也能丢到任何静态托管上。

本文讲清楚三件事：**它怎么跑起来**、**每个部分是怎么写的**、**你想改哪里该怎么改**。

---

## 一、先跑起来（5 分钟）

### 1. 拿一个令牌（Token）

打开 <https://github.com/settings/tokens> → **Generate new token (classic)**。

勾选这些权限就够了：

| 用途 | 勾选 |
|---|---|
| 看公开仓库 | `public_repo`（或不勾，只读也能看） |
| 看 / 改私有仓库 | `repo` |
| 读写 Gist | `gist` |
| 看通知 | `notifications` |
| 触发 Actions | `workflow` |

生成后**只显示一次**，立刻复制保存。

> ⚠️ **令牌等于你的账号密码。** 详见 [`docs/04-令牌安全.md`](docs/04-令牌安全.md)。
> 本项目所有文件都不含任何真实令牌，你也不需要把令牌写进代码里。

### 2. 打开页面

下载 `mini-client.html`，浏览器打开 → 粘贴令牌 → 开始用。

令牌只存在你**自己浏览器的 localStorage** 里，不会发到任何别的地方。

---

## 二、它为什么能"没有后端"

关键在于：**GitHub 的 REST API 允许跨域（CORS）**。

```
你的浏览器 ──fetch──> api.github.com
                  <──JSON──
```

浏览器直接跟 GitHub 对话，中间不需要你自己的服务器。
所以整个东西可以就是一个 `.html` 文件。

### 一次请求的完整写法

```js
async function gh(path, opts = {}) {
  const r = await fetch('https://api.github.com' + path, {
    ...opts,
    headers: {
      Authorization: 'Bearer ' + TOKEN,
      Accept: 'application/vnd.github+json',
      'X-GitHub-Api-Version': '2022-11-28',
      ...(opts.headers || {})
    }
  });
  if (!r.ok) throw new Error('HTTP ' + r.status);
  return r.json();
}

// 用法
const repos = await gh('/user/repos?per_page=100');
```

**就是这么简单。** 其余几千行都是在这一个函数上长出来的。

### 速率限制

| 登录状态 | 配额 |
|---|---|
| 未登录 | 60 次 / 小时 |
| 带令牌 | **5000 次 / 小时** |

所以一定要带令牌。查剩余额度：

```js
const r = await gh('/rate_limit');
console.log(r.rate.remaining, '/', r.rate.limit);
```

---

## 三、项目里有什么

| 文件 | 说明 |
|---|---|
| `mini-client.html` | **精简可跑模板**（约 300 行）。登录 + 列仓库 + 看文件 + 搜用户，核心逻辑全在里面 |
| `docs/01-原理.md` | 单文件架构、路由、状态管理、渲染套路 |
| `docs/02-自定义.md` | 改主题、加页面、加菜单、加功能 —— 照着改就行 |
| `docs/03-API速查.md` | 常用端点清单 + 分页 + GraphQL 对照 |
| `docs/04-令牌安全.md` | 令牌该放哪、不该放哪、泄露了怎么办 |

`mini-client.html` 是**给你改的**。它故意写得很直白、注释很密，
读完再去看完整版就不会觉得无从下手。

---

## 四、从模板到完整版

完整版（一千多行）比模板多了这些，**每一个都是独立模块，可以单独拆走**：

| 模块 | 核心做法 |
|---|---|
| 仓库浏览器 | 文件树递归 + 内容 base64 解码 |
| 文件编辑器 | `PUT /repos/{o}/{r}/contents/{path}` + `sha` 校验 |
| Issue / PR | 状态 SVG 图标、标签取真实颜色、审核状态异步补 |
| 通知 | 标记已读、按仓库过滤 |
| 搜索 | 拼 `q=` 查询串，支持 `language:go stars:>100` 这类语法 |
| 云终端 | `workflow_dispatch` 触发 Actions 当"远程命令执行"用 |
| 内嵌网页 | 自己抓 HTML → 清洗 → `srcdoc` 渲染，绕开 `X-Frame-Options` |
| 界面翻译 | 术语词典替换，只翻界面词、不翻代码标识符 |

---

## 五、常见问题

**Q：我能直接把它挂到网上给别人用吗？**
可以，但**页面里绝对不能内置你自己的令牌**。让别人填自己的令牌。
（内置令牌 = 任何人打开页面都能用你的账号删仓库。）

**Q：为什么我打开是白屏？**
`file://` 协议下部分浏览器会限制 `localStorage` 和 `fetch`。
用本地静态服务器：`python3 -m http.server 8000`，然后访问 `localhost:8000`。

**Q：GraphQL 和 REST 该用哪个？**
REST 简单、无需额外权限；GraphQL 一次拿齐嵌套数据（比如讨论区楼中楼）。
**两者共用同一个令牌**，GraphQL 走 `https://api.github.com/graphql`（POST）。

**Q：手机上能用吗？**
能。完整版针对触屏做了：单击进目录（不要双击，会被识别成缩放）、
加大点击热区、侧栏可收起。

---

## 六、License

MIT —— 随便改、随便商用，不用署名。
