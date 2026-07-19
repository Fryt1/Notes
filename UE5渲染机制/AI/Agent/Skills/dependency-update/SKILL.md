---
name: dependency-update
description: 检查和更新项目依赖。分析过期依赖，评估更新风险，提供更新建议
disable-model-invocation: false
allowed-tools: Bash, Read, Write, Grep
---

# 依赖更新 Skill

检查项目依赖，分析更新风险，安全地更新依赖包。

## 工作流程

### 1. 检查依赖状态

**Node.js 项目**
```bash
# 检查过期依赖
npm outdated

# 查看依赖树
npm list --depth=0

# 检查安全漏洞
npm audit
```

**Python 项目**
```bash
# 检查过期依赖
pip list --outdated

# 查看已安装包
pip list
```

**其他工具**
```bash
# 使用 npm-check-updates
npx npm-check-updates

# 使用 yarn
yarn outdated
```

### 2. 分析依赖信息

**依赖分类**：
- 🔴 **主要依赖** (dependencies): 生产环境必需
- 🟡 **开发依赖** (devDependencies): 仅开发时使用
- 🟢 **可选依赖** (optionalDependencies): 可选功能

**版本类型**：
- **Major**: 1.0.0 → 2.0.0 (破坏性更新)
- **Minor**: 1.0.0 → 1.1.0 (新功能)
- **Patch**: 1.0.0 → 1.0.1 (bug 修复)

### 3. 评估更新风险

**风险等级**：

🟢 **低风险** (安全更新)
- Patch 版本更新
- 安全漏洞修复
- Bug 修复
- 建议：直接更新

🟡 **中风险** (功能更新)
- Minor 版本更新
- 新增功能
- 性能改进
- 建议：测试后更新

🔴 **高风险** (破坏性更新)
- Major 版本更新
- API 变更
- 移除功能
- 建议：仔细评估，充分测试

### 4. 生成更新计划

```markdown
## 依赖更新报告

### 🔴 安全漏洞 (立即修复)
- `lodash` 4.17.15 → 4.17.21
  - 漏洞: CVE-2021-23337
  - 严重程度: High
  - 修复: `npm update lodash`

### 🟡 建议更新
- `axios` 0.21.0 → 0.27.2
  - 类型: Minor
  - 变更: 新增功能，性能优化
  - 风险: 低
  - 测试: 检查 API 调用

### 🟢 可选更新
- `eslint` 7.32.0 → 8.50.0
  - 类型: Major
  - 变更: 新规则，配置变更
  - 风险: 中
  - 建议: 查看迁移指南

### ⏸️ 暂不更新
- `react` 17.0.2 → 18.2.0
  - 原因: 需要大量代码改动
  - 计划: 下个版本更新
```

## 更新策略

### 策略 1: 保守更新
```bash
# 只更新 patch 版本
npm update

# 指定包更新
npm update package-name
```

### 策略 2: 积极更新
```bash
# 更新到最新版本
npx npm-check-updates -u
npm install
```

### 策略 3: 分批更新
```bash
# 先更新开发依赖
npm update --dev

# 测试通过后更新生产依赖
npm update --save
```

## 更新步骤

### 安全更新流程

1. **备份**
   ```bash
   # 提交当前代码
   git add .
   git commit -m "chore: backup before dependency update"

   # 备份 package-lock.json
   cp package-lock.json package-lock.json.backup
   ```

2. **更新依赖**
   ```bash
   npm update package-name
   ```

3. **运行测试**
   ```bash
   npm test
   npm run build
   ```

4. **检查应用**
   ```bash
   npm start
   # 手动测试关键功能
   ```

5. **提交更新**
   ```bash
   git add package.json package-lock.json
   git commit -m "chore: update package-name to x.x.x"
   ```

### Major 版本更新流程

1. **查看变更日志**
   ```bash
   # 访问项目 GitHub 查看 CHANGELOG
   # 或使用 npm 查看
   npm view package-name versions
   ```

2. **阅读迁移指南**
   - 查找官方迁移文档
   - 了解破坏性变更
   - 准备代码修改

3. **创建分支**
   ```bash
   git checkout -b update/package-name-v2
   ```

4. **更新并修改代码**
   ```bash
   npm install package-name@latest
   # 根据迁移指南修改代码
   ```

5. **全面测试**
   ```bash
   npm test
   npm run e2e
   # 手动测试所有功能
   ```

6. **代码审查**
   - 创建 Pull Request
   - 团队审查
   - 合并到主分支

## 依赖管理最佳实践

### ✅ 推荐做法

1. **锁定版本**
   ```json
   {
     "dependencies": {
       "express": "4.18.2"  // 精确版本
     }
   }
   ```

2. **定期检查**
   - 每周检查一次依赖更新
   - 每月进行一次安全审计

3. **自动化检查**
   ```yaml
   # GitHub Actions
   - name: Check dependencies
     run: npm audit
   ```

4. **使用工具**
   - Dependabot (GitHub)
   - Renovate Bot
   - Snyk

### ❌ 避免做法

1. **盲目更新**
   ```bash
   # 不要直接运行
   npx npm-check-updates -u && npm install
   ```

2. **忽略锁文件**
   ```bash
   # 不要删除 package-lock.json
   rm package-lock.json  # ❌
   ```

3. **混用包管理器**
   ```bash
   npm install  # ❌
   yarn add     # ❌
   # 项目中只用一种
   ```

## 常见问题处理

### 依赖冲突
```bash
# 查看冲突
npm ls package-name

# 解决方案
npm install package-name@version --force
# 或使用 overrides (npm 8.3+)
```

### 安装失败
```bash
# 清理缓存
npm cache clean --force

# 删除 node_modules
rm -rf node_modules package-lock.json

# 重新安装
npm install
```

### 版本不兼容
```bash
# 查看兼容版本
npm view package-name versions

# 安装特定版本
npm install package-name@x.x.x
```

## 输出格式

```
📦 依赖更新报告

🔍 检查结果：
- 总依赖数: 45
- 过期依赖: 12
- 安全漏洞: 2 (High)

🔴 紧急更新 (2)
1. lodash 4.17.15 → 4.17.21
   风险: 安全漏洞 CVE-2021-23337
   操作: npm update lodash

2. axios 0.21.0 → 0.21.4
   风险: 安全漏洞
   操作: npm update axios

🟡 建议更新 (5)
- express 4.17.1 → 4.18.2 (patch)
- jest 27.0.0 → 27.5.1 (minor)
...

🟢 可选更新 (5)
- eslint 7.x → 8.x (major, 需要配置调整)
...

💡 更新建议：
1. 先修复安全漏洞
2. 运行测试确保功能正常
3. Major 版本更新建议单独处理

📝 更新命令：
npm update lodash axios express jest
```

## 自动化更新

### 配置 Dependabot
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

### 配置 Renovate
```json
{
  "extends": ["config:base"],
  "packageRules": [
    {
      "updateTypes": ["patch", "pin", "digest"],
      "automerge": true
    }
  ]
}
```
