# 03 · API 速查

所有请求都走 `https://api.github.com`。本项目封装成了 `gh(path, opts)`，
下面写的都是**传给 `gh()` 的路径部分**。

---

## 认证

```
Authorization: Bearer <你的令牌>
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2022-11-28
```

---

## 用户

| 用途 | 请求 |
|---|---|
| 当前登录用户 | `GET /user` |
| 指定用户 | `GET /users/{username}` |
| 用户的仓库 | `GET /users/{u}/repos?per_page=100&sort=updated` |
| 用户的 Star | `GET /users/{u}/starred?per_page=100` |
| 关注列表 | `GET /users/{u}/following` |
| 粉丝 | `GET /users/{u}/followers` |
| 贡献者 | `GET /repos/{o}/{r}/contributors` |

## 仓库

| 用途 | 请求 |
|---|---|
| 我的全部仓库 | `GET /user/repos?per_page=100&sort=updated` |
| 单个仓库 | `GET /repos/{o}/{r}` |
| 分支列表 | `GET /repos/{o}/{r}/branches` |
| 标签 | `GET /repos/{o}/{r}/tags` |
| 语言占比 | `GET /repos/{o}/{r}/languages` |
| 提交列表 | `GET /repos/{o}/{r}/commits?per_page=30` |
| 单个提交（含 diff） | `GET /repos/{o}/{r}/commits/{sha}` |
| 贡献者 | `GET /repos/{o}/{r}/contributors` |
| 发布版 | `GET /repos/{o}/{r}/releases` |
| Star / 取消 | `PUT` / `DELETE /user/starred/{o}/{r}` |
| 新建 | `POST /user/repos` body:`{name,description,private}` |
| 删除 | `DELETE /repos/{o}/{r}` |

## 文件

| 用途 | 请求 |
|---|---|
| 列目录 | `GET /repos/{o}/{r}/contents/{path}` |
| 读文件 | `GET /repos/{o}/{r}/contents/{path}` → `content` 是 base64 |
| 写文件 | `PUT /repos/{o}/{r}/contents/{path}` body:`{message,content,sha}` |
| 删文件 | `DELETE` 同上，body 也要带 `sha` |
| 递归整棵树 | `GET /repos/{o}/{r}/git/trees/{branch}?recursive=1` |

> **写文件必须带 `sha`**（先用 GET 拿到），否则 422。
> `sha` 的作用是"我改的是你给我的那个版本"，防止覆盖别人的改动。

## Issue

| 用途 | 请求 |
|---|---|
| 列表 | `GET /repos/{o}/{r}/issues?state=open&per_page=100` |
| 单个 | `GET /repos/{o}/{r}/issues/{n}` |
| 评论 | `GET /repos/{o}/{r}/issues/{n}/comments` |
| 发评论 | `POST` 同上，body:`{body}` |
| 新建 | `POST /repos/{o}/{r}/issues` body:`{title,body,labels}` |
| 改状态 | `PATCH /repos/{o}/{r}/issues/{n}` body:`{state:'closed'}` |
| 标签 | `GET /repos/{o}/{r}/labels` |
| 搜索 | `GET /search/issues?q=...` |

> Issue 列表**也会返回 PR**。用 `item.pull_request` 是否存在来区分。

## Pull Request

| 用途 | 请求 |
|---|---|
| 列表 | `GET /repos/{o}/{r}/pulls?state=open` |
| 单个 | `GET /repos/{o}/{r}/pulls/{n}` |
| 改动文件 | `GET /repos/{o}/{r}/pulls/{n}/files` |
| 审核记录 | `GET /repos/{o}/{r}/pulls/{n}/reviews` |
| 合并 | `PUT /repos/{o}/{r}/pulls/{n}/merge` |

**审核状态怎么判**：取 reviews 数组，按人去重（一人多次以最后一次为准），
然后优先级：`CHANGES_REQUESTED` > `APPROVED` > `COMMENTED`。

## 通知

| 用途 | 请求 |
|---|---|
| 列表 | `GET /notifications?per_page=50` |
| 全部已读 | `PUT /notifications` body:`{}` |
| 单条已读 | `PATCH /notifications/threads/{id}` |

## Actions

| 用途 | 请求 |
|---|---|
| workflow 列表 | `GET /repos/{o}/{r}/actions/workflows` |
| 运行记录 | `GET /repos/{o}/{r}/actions/runs?per_page=10` |
| 手动触发 | `POST /repos/{o}/{r}/actions/workflows/{id}/dispatches` body:`{ref,inputs}` |
| 运行日志 | `GET /repos/{o}/{r}/actions/runs/{id}/logs` |

> **触发成功返回 204，body 是空的。** 别当 JSON 解析，会报
> `Unexpected end of JSON input`。
> 触发后要轮询 `GET .../runs` 找到新建的那条（按 `created_at` 比对）。

## Gist

| 用途 | 请求 |
|---|---|
| 列表 | `GET /gists` |
| 新建 | `POST /gists` body:`{files:{'a.md':{content:'...'}},public:false}` |
| 改 | `PATCH /gists/{id}` |

---

## 搜索语法

```
GET /search/repositories?q=关键词
GET /search/issues?q=关键词
GET /search/users?q=关键词
GET /search/code?q=关键词          （需登录）
```

常用限定词：

| 写法 | 含义 |
|---|---|
| `language:go` | 指定语言 |
| `stars:>1000` | 星数大于 |
| `user:torvalds` | 某用户的 |
| `is:private` | 私有的 |
| `state:open` | 开启状态 |
| `label:bug` | 带某标签 |
| `created:>2024-01-01` | 创建时间 |
| `in:title,body` | 限定搜索范围 |

组合：`q=language:rust+stars:>5000+pushed:>2025-01-01`

---

## GraphQL

**端点**：`POST https://api.github.com/graphql`（只有这一个）

```js
function gql(query, variables){
  return fetch('https://api.github.com/graphql', {
    method: 'POST',
    headers: { Authorization: 'Bearer ' + TOKEN,
               'Content-Type': 'application/json' },
    body: JSON.stringify({ query, variables })
  }).then(r => r.json())
    .then(d => { if (d.errors) throw new Error(d.errors[0].message);
                 return d.data; });
}
```

**什么时候该用 GraphQL**：

- ✅ 拿讨论区（Discussions）—— **REST 不支持，只能用 GraphQL**
- ✅ 一次要拿很多嵌套数据（评论 + 楼中楼 + 作者 + 头像）
- ✅ 只要某几个字段，想省流量

**什么时候别用**：

- ❌ 简单列表（REST 更直观）
- ❌ 要写操作（大部分 mutation 权限要求更高）

**讨论区查询示例**：

```graphql
query($o:String!, $n:String!){
  repository(owner:$o, name:$n){
    discussions(first:20){
      nodes{
        number title
        author{ login avatarUrl }
        comments(first:10){
          totalCount
          nodes{
            body createdAt
            author{ login avatarUrl }
            replies(first:10){ nodes{ body author{login} } }
          }
        }
      }
    }
  }
}
```

**用 GraphQL Explorer 试查询**：<https://docs.github.com/zh/graphql/overview/explorer>

---

## 速率限制

| 状态 | 配额 |
|---|---|
| 未登录 | 60 / 小时（按 IP） |
| 已登录 | **5000 / 小时** |
| GraphQL | 5000 点 / 小时（复杂度计费） |
| 搜索 | 30 / 分钟 |

查剩余：

```js
gh('/rate_limit').then(r => {
  console.log('核心:', r.rate.remaining + '/' + r.rate.limit);
  console.log('搜索:', r.search.remaining);
  console.log('重置:', new Date(r.rate.reset * 1000));
});
```

**省额度的四个办法**：

1. `per_page=100` 一次拉满，少翻页
2. 用 `If-None-Match` 条件请求，没变化返回 304 不算额度
3. 缓存结果（内存 + localStorage），别重复拉
4. 次要信息异步补、延后补，不要卡首屏
