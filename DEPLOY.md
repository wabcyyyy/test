# 阿里云 OSS 静态页面部署指南

本文档说明如何将静态网站部署到阿里云 OSS。

## 前置准备

### 1. 创建阿里云 OSS Bucket

1. 登录 [阿里云 OSS 控制台](https://oss.console.aliyun.com/)
2. 创建一个新的 Bucket:
   - **Bucket 名称**: 自定义(例如: `my-website-bucket`)
   - **地域**: 根据你的目标用户选择
     - 如果主要用户在大陆: 选择 `华东1(杭州)`、`华北2(北京)` 等
     - 如果想要避免备案: 选择 `香港` 或海外节点
   - **存储类型**: 标准存储
   - **读写权限**: 公共读(重要!用于网站访问)

### 2. 配置 Bucket 为静态网站

1. 在 Bucket 列表中,点击你的 Bucket 名称
2. 进入 `数据管理` > `静态页面`
3. 点击 `设置`,按如下配置:
   - **默认首页**: `index.html`
   - **默认 404 页**: `404.html`
   - 点击 `保存`

### 3. 获取访问密钥

1. 访问 [RAM 访问控制](https://ram.console.aliyun.com/manage/ak)
2. 创建 AccessKey(如果没有):
   - 建议创建 RAM 子用户并授予 OSS 权限,而不是使用主账号
3. 记录以下信息:
   - `AccessKey ID`
   - `AccessKey Secret`

### 4. 确定 OSS Endpoint

根据你的 Bucket 所在区域,endpoint 格式为:
- 华东1(杭州): `oss-cn-hangzhou.aliyuncs.com`
- 华北2(北京): `oss-cn-beijing.aliyuncs.com`
- 华东2(上海): `oss-cn-shanghai.aliyuncs.com`
- 香港: `oss-cn-hongkong.aliyuncs.com`

更多区域请参考: [OSS Region 和 Endpoint](https://help.aliyun.com/document_detail/31837.html)

## GitHub Actions 配置

### 配置 Secrets

在你的 GitHub 仓库中配置以下 Secrets:

1. 进入仓库的 `Settings` > `Secrets and variables` > `Actions`
2. 点击 `New repository secret`,添加以下 4 个密钥:

| 密钥名称 | 值 | 说明 |
|---------|---|------|
| `ACCESS_KEY_ID` | 你的 AccessKey ID | 阿里云访问密钥 ID |
| `ACCESS_KEY_SECRET` | 你的 AccessKey Secret | 阿里云访问密钥 Secret |
| `OSS_BUCKET` | 你的 Bucket 名称 | 例如: `my-website-bucket` |
| `OSS_ENDPOINT` | OSS 访问域名 | 例如: `oss-cn-hangzhou.aliyuncs.com` |

### 工作流说明

部署工作流文件位于: `.github/workflows/deploy-oss.yml`

**触发条件**:
- 推送到 `main` 或 `master` 分支时
- 创建 Pull Request 到 `main` 或 `master` 分支时

**主要功能**:
1. 检出代码
2. 增量上传文件到 OSS
3. 自动配置静态页面设置

### 自定义配置

如果需要修改部署配置,编辑 `.github/workflows/deploy-oss.yml`:

```yaml
# 要部署的文件夹
folder: .           # 当前目录,如果要部署特定文件夹,改为: folder: dist

# 404 页面
notFoundPage: 404.html

# 增量上传(默认开启)
incremental: true   # 设为 false 则会清空 OSS 后重新上传

# 排除某些文件
exclude: |
  README.md
  .git/
  .github/

# 缓存控制
htmlCacheControl: no-cache           # HTML 文件不缓存
imageCacheControl: max-age=864000    # 图片缓存 10 天
otherCacheControl: max-age=2592000   # 其他文件缓存 30 天
```

## 部署流程

### 自动部署

1. 将代码提交并推送到 GitHub:
   ```bash
   git add .
   git commit -m "更新网站内容"
   git push origin main
   ```

2. GitHub Actions 会自动触发部署

3. 查看部署状态:
   - 进入仓库的 `Actions` 标签页
   - 查看最新的 workflow 运行状态

### 手动触发

如果需要手动触发部署,可以在工作流文件中添加 `workflow_dispatch`:

```yaml
on:
  workflow_dispatch:  # 手动触发
  push:
    branches:
      - main
```

然后在 GitHub 网站的 `Actions` 标签页手动运行工作流。

## 访问网站

部署完成后,可以通过以下方式访问:

1. **OSS 默认域名**:
   - 格式: `https://<bucket-name>.<endpoint>/`
   - 例如: `https://my-website-bucket.oss-cn-hangzhou.aliyuncs.com/`

2. **自定义域名**(推荐):
   - 在 OSS 控制台绑定自定义域名
   - 配置 CNAME 解析
   - 申请并配置 SSL 证书(支持 HTTPS)

## 配置 CDN 加速(可选)

如果需要使用 CDN 加速:

1. 在阿里云控制台开通 CDN
2. 添加域名并配置源站为 OSS Bucket
3. 开启 CDN 缓存自动刷新:
   - 刷新方式: 目录刷新
   - 刷新条件: PutObject, DeleteObject

**注意**: 使用 CDN 时,不要在 workflow 中使用自定义域名作为 endpoint。

## 常见问题

### 1. 部署后网站无法访问?

检查:
- Bucket 读写权限是否设置为"公共读"
- 静态页面配置是否正确
- 防火墙或网络策略是否阻止访问

### 2. GitHub Actions 报错?

检查:
- Secrets 是否配置正确
- Bucket 名称和 endpoint 是否匹配
- AccessKey 是否有足够权限

### 3. 如何删除 OSS 中的文件?

有三种方式:
1. 在 OSS 控制台手动删除
2. 设置 `incremental: false`,会自动删除多余文件
3. 使用 OSS API 或 SDK 删除

### 4. 更新后 CDN 未刷新?

在 OSS 控制台开启 CDN 缓存自动刷新功能,设置触发操作为 `PutObject` 和 `DeleteObject`。

## 安全建议

1. **使用 RAM 子用户**: 不要使用主账号的 AccessKey
2. **最小权限原则**: 只授予 OSS 必要的权限
3. **定期轮换密钥**: 定期更换 AccessKey
4. **加密敏感数据**: 不要在代码中硬编码密钥

## 相关链接

- [阿里云 OSS 文档](https://help.aliyun.com/product/31815.html)
- [aliyun-oss-website-action](https://github.com/marketplace/actions/aliyun-oss-website-action)
- [GitHub Actions 文档](https://docs.github.com/en/actions)