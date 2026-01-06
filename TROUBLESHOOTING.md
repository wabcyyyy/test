# 部署问题排查指南

如果提交代码后 OSS 没有更新,请按照以下步骤逐一排查:

## 🔍 第一步:检查 GitHub Actions 运行状态

### 访问 Actions 页面
1. 打开: https://github.com/wabcyyyy/test/actions
2. 查看最新的 workflow 运行记录

### 可能的情况

#### ❌ 情况 A: 没有任何运行记录
**原因:** Workflow 没有被触发

**解决方案:**
- 确认代码已推送到 `main` 或 `master` 分支
- 检查分支名称是否正确(可以通过 `git branch` 查看)
- 确认 `.github/workflows/deploy-oss.yml` 文件存在且格式正确

```bash
# 检查分支
git branch

# 如果不在 main 分支,切换并推送
git checkout main
git push origin main
```

#### ❌ 情况 B: Workflow 运行失败(红色❌)
**原因:** 配置错误或权限问题

**解决方案:**
1. 点击失败的 workflow,查看详细错误日志
2. 常见错误信息及解决方法:

| 错误信息 | 原因 | 解决方法 |
|---------|------|---------|
| `Required secret not found` | GitHub Secrets 未配置 | 配置 Secrets (见第二步) |
| `InvalidAccessKeyId` | AccessKey ID 错误 | 检查 `ACCESS_KEY_ID` 是否正确 |
| `SignatureDoesNotMatch` | AccessKey Secret 错误 | 检查 `ACCESS_KEY_SECRET` 是否正确 |
| `NoSuchBucket` | Bucket 不存在 | 检查 `OSS_BUCKET` 名称和区域是否匹配 |
| `AccessDenied` | 权限不足 | 检查 RAM 用户是否有 OSS 操作权限 |

#### ✅ 情况 C: Workflow 运行成功,但 OSS 未更新
**原因:** OSS 配置问题或缓存问题

**解决方案:**
1. 检查 OSS Bucket 的访问域名
2. 确认浏览器缓存(强制刷新: Ctrl+F5)
3. 检查 OSS 静态页面配置是否正确

## 🔐 第二步:验证 GitHub Secrets 配置

### 访问 Secrets 配置页面
1. 打开: https://github.com/wabcyyyy/test/settings/secrets/actions
2. 确认以下 4 个 Secrets 已添加:

| Secret 名称 | 示例值 | 说明 |
|------------|--------|------|
| `ACCESS_KEY_ID` | `LTAI5tXXXXXX` | 阿里云 AccessKey ID |
| `ACCESS_KEY_SECRET` | `XXXXXXXXXX` | 阿里云 AccessKey Secret |
| `OSS_BUCKET` | `quanttide-test1` | Bucket 名称 |
| `OSS_ENDPOINT` | `oss-cn-hangzhou.aliyuncs.com` | OSS 访问域名 |

### 如果 Secrets 未配置

#### 添加 Secrets 步骤:

1. **创建 RAM 用户 AccessKey** (如果还没有)
   - 访问: https://ram.console.aliyun.com/manage/ak
   - 创建 AccessKey 并记录

2. **授予 OSS 权限**
   - 进入 RAM 用户管理
   - 添加权限策略: `AliyunOSSFullAccess`

3. **在 GitHub 添加 Secrets**
   - 点击 "New repository secret"
   - Name: `ACCESS_KEY_ID`
   - Value: 粘贴你的 AccessKey ID
   - 点击 "Add secret"
   - 重复上述步骤添加其他 3 个 Secrets

## 📦 第三步:检查阿里云 OSS 配置

### 1. 确认 Bucket 存在且配置正确

访问: https://oss.console.aliyun.com/

**检查项:**
- ✅ Bucket 名称是 `quanttide-test1`
- ✅ 所在区域是 `华东1(杭州)` 或与 `OSS_ENDPOINT` 匹配
- ✅ 读写权限设置为 **公共读**

**如果 Bucket 不是公共读:**
1. 进入 Bucket 概览
2. 点击 `权限管理` → `读写权限`
3. 设置为 `公共读`
4. 保存

### 2. 配置静态网站托管

1. 进入 `数据管理` → `静态页面`
2. 点击 `设置`
3. 配置:
   - 默认首页: `index.html`
   - 默认 404 页: `404.html`
4. 点击 `保存`

### 3. 获取正确的访问地址

访问方式:

**方式 1: OSS 默认域名**
```
https://quanttide-test1.oss-cn-hangzhou.aliyuncs.com/index.html
```

**方式 2: 静态网站托管域名**
```
http://quanttide-test1.oss-cn-hangzhou.aliyuncs.com
```

**注意:** 两种域名不同,静态网站托管域名通常以 `oss-website-` 开头

## 🧪 第四步:手动测试部署

如果需要手动触发部署:

### 方式 1: 使用 GitHub CLI (如果已安装)
```bash
gh workflow run deploy-oss.yml
```

### 方式 2: 空提交触发
```bash
# 在本地执行
git commit --allow-empty -m "触发部署"
git push origin main
```

### 方式 3: 修改工作流支持手动触发

编辑 `.github/workflows/deploy-oss.yml`:

```yaml
on:
  workflow_dispatch:  # 添加这一行
  push:
    branches:
      - main
```

然后在 GitHub 网站的 Actions 页面手动运行。

## 📊 第五步:查看部署日志

### 在 GitHub 查看详细日志

1. 访问: https://github.com/wabcyyyy/test/actions
2. 点击最新的 workflow 运行
3. 展开 "Upload files to OSS" 步骤
4. 查看详细输出

**关键日志信息:**
- ✅ 上传文件列表
- ✅ 是否启用增量上传
- ✅ 是否配置静态页面

### 成功的日志示例:
```
Run fangbinwei/aliyun-oss-website-action@v1
  with:
    accessKeyId: ***
    accessKeySecret: ***
    bucket: quanttide-test1
    endpoint: oss-cn-hangzhou.aliyuncs.com
    folder: .
    indexPage: index.html
    notFoundPage: 404.html
    incremental: true
✓ Upload index.html to OSS
✓ Upload 404.html to OSS
✓ Set static page configuration
```

## 🌐 第六步:验证部署结果

### 1. 直接访问 OSS 文件

在浏览器打开:
```
https://quanttide-test1.oss-cn-hangzhou.aliyuncs.com/index.html
```

### 2. 强制刷新浏览器缓存

- Windows: `Ctrl + F5` 或 `Ctrl + Shift + R`
- Mac: `Cmd + Shift + R`

### 3. 检查 OSS 控制台文件列表

1. 访问 OSS 控制台
2. 进入 Bucket → `文件管理`
3. 确认文件已上传且时间是最新的

## 🔧 常见问题 FAQ

### Q1: Workflow 显示成功,但访问 404?

**可能原因:**
- Bucket 权限不是公共读
- 使用了错误的访问域名
- 静态页面配置未生效

**解决方法:**
1. 检查 Bucket 读写权限
2. 确认访问域名正确
3. 重新配置静态页面设置

### Q2: 更改 Secrets 后需要重新推送代码吗?

**不需要!** Secrets 是即时生效的,只需要:
1. 在 GitHub 更新 Secrets
2. 空提交或推送新代码触发 workflow

### Q3: 如何查看当前使用的 AccessKey?

**在阿里云控制台:**
1. 访问: https://ram.console.aliyun.com/manage/ak
2. 查看 AccessKey 列表
3. 确认哪些 Key 是激活状态

### Q4: 增量上传是什么意思?

**增量上传:**
- 只上传有变化的文件
- 保留 `.actioninfo` 文件记录
- 删除 OSS 中多余的文件

**全量上传 (incremental: false):**
- 先清空 OSS 所有文件
- 再上传所有文件

### Q5: 如何排除某些文件不上传?

在 workflow 中添加 `exclude` 参数:

```yaml
exclude: |
  README.md
  .git/
  .github/
  *.md
```

## 📞 获取帮助

如果以上步骤都无法解决问题,请提供以下信息:

1. GitHub Actions 运行日志截图
2. GitHub Secrets 配置情况(隐藏具体值,只显示名称)
3. OSS Bucket 配置截图
4. 具体的错误信息

## ✅ 检查清单

部署前确保:
- [ ] GitHub Secrets 已正确配置(4 个)
- [ ] Bucket 已创建且权限为"公共读"
- [ ] 静态页面配置已完成
- [ ] 代码已推送到正确的分支(main/master)
- [ ] GitHub Actions 运行成功(绿色✅)
- [ ] 使用正确的 OSS 访问域名
