---
name: js-bloat-analyzer
description: 分析 JavaScript/Node.js 项目的依赖膨胀问题，识别冗余包并推荐替代方案。

**触发场景：**
- 项目依赖过多，需要清理优化
- bundle 体积过大，需要找出冗余依赖
- 升级项目前评估依赖健康度
- 新接手项目，快速了解依赖状况

**关键词：** npm、dependency、bloat、node_modules、optimize、bundle、cleanup、冗余依赖

triggers:
  - type: keyword
    patterns: ["依赖优化", "node_modules 清理", "bloat", "npm audit", "依赖清理", "包优化"]
  - type: intent
    patterns: ["分析依赖", "清理冗余包", "优化 bundle", "减小体积"]
  - type: context
    condition: "file_exists('package.json') and has_node_modules"

input_schema:
  project_path:
    type: string
    required: true
    description: 项目根目录路径（需包含 package.json）
  check_depth:
    type: string
    required: false
    default: "deep"
    enum: ["basic", "deep"]
    description: 检查深度
  auto_fix:
    type: boolean
    required: false
    default: false
    description: 是否自动应用修复（建议先 review）

output_schema:
  summary:
    type: object
    properties:
      total_packages: { type: integer }
      redundant_packages: { type: integer }
      estimated_savings: { type: string }
  redundant_deps:
    type: array
    items:
      type: object
      properties:
        name: { type: string }
        type: { type: string, enum: ["pillar_1", "pillar_2", "pillar_3"] }
        reason: { type: string }
        replacement: { type: string }
        action: { type: string, enum: ["inline", "remove", "replace"] }
  recommendations:
    type: array
    items: { type: string }

verification:
  - check: "output.summary.total_packages >= 0"
    severity: error
  - check: "len(output.redundant_deps) == output.summary.redundant_packages"
    severity: error
  - check: "all dep in output.redundant_deps has dep.name and dep.replacement"
    severity: error
  - check: "output.summary.estimated_savings matches /\\d+\\.\\d+MB|\\d+KB/"
    severity: warning
  - check: "file_exists(input.project_path + '/package.json')"
    severity: error

---

# JS 依赖膨胀分析器

基于 "The Three Pillars of JavaScript Bloat" 方法论，系统性分析和清理项目依赖。

## Capabilities

- 扫描并分析项目依赖树
- 识别三大类型冗余依赖（老旧运行时、原子化微包、过期 ponyfills）
- 推荐替代方案和迁移路径
- 估算清理后的体积节省

## Constraints

- 仅支持 **JavaScript/Node.js 项目**（需存在 package.json）
- 不处理**私有/企业内部包**（无法获取元数据分析）
- 大型项目（>1000 个依赖）分析可能需要较长时间
- 推荐的替代方案基于社区最佳实践，需测试验证兼容性
- **auto_fix=true 时需谨慎**，建议先 review 再应用

## When NOT to Use

- ❌ 非 JavaScript/Node.js 项目（无 package.json）
- ❌ 项目依赖 <10 个（优化收益不明显）
- ❌ 生产环境紧急修复时（分析需要稳定环境）
- ❌ 不经过 review 直接应用 auto_fix（可能有破坏性变更）
- ❌ 私有包未配置 registry 访问权限

## Anti-patterns

- ❌ 直接删除依赖而不验证替代方案兼容性
- ❌ 在生产环境直接运行 auto_fix=true
- ❌ 忽略大型项目的分析时间警告
- ❌ 不测试就应用所有替换建议
- ❌ 处理 Monorepo 时不指定子包路径

## 三大膨胀类型

### 1. 老旧运行时支持
为支持 ES3/旧 Node 版本而存在的包：
- `is-string` → 用 `typeof val === 'string'`
- `hasown` → 用 `Object.hasOwn()`
- 各种 `Array.prototype` polyfills

### 2. 原子化架构
过度拆分的微包（一个函数一个包）：
- `arrify` → `Array.isArray(val) ? val : [val]`
- `slash` → `path.replace(/\\/g, '/')`
- `path-key` → `process.platform === 'win32' ? 'Path' : 'PATH'`
- `is-wsl` / `is-windows` → 检查 `process.platform`

### 3. 过期的 Ponyfills
平台已原生支持但仍在使用的填充：
- `globalthis` → 原生 `globalThis` (2019+)
- `object.entries` → 原生 `Object.entries` (2017+)
- `indexof` → 原生 `Array.prototype.indexOf` (2010+)

## 执行步骤

1. **扫描依赖树**
   ```bash
   npm ls --all > deps-tree.txt
   # 或使用 npmgraph 可视化分析
   ```

2. **检测冗余依赖**
   - 运行 `npx knip` 找出未使用的依赖
   - 运行 `npx @e18e/cli analyze` 识别可替换的包

3. **分类膨胀类型**
   - 检查是否有 Pillar 1 类型（老旧运行时支持）
   - 检查是否有 Pillar 2 类型（原子化微包）
   - 检查是否有 Pillar 3 类型（过期 ponyfills）

4. **生成替换建议**
   参考 module-replacements 数据集：
   - https://e18e.dev/docs/replacements/

5. **输出报告**
   - 冗余依赖列表
   - 推荐的替代方案
   - 预估节省空间

## 输出格式

```json
{
  "summary": {
    "total_packages": 150,
    "redundant_packages": 23,
    "estimated_savings": "2.3MB"
  },
  "redundant_deps": [
    {
      "name": "is-string",
      "type": "pillar_1",
      "reason": "老旧运行时支持",
      "replacement": "typeof val === 'string'",
      "action": "inline"
    }
  ],
  "recommendations": [
    "使用 knip 移除未使用的依赖",
    "使用 @e18e/cli migrate 自动替换"
  ]
}
```

## 工具链

- **knip**: 找出未使用的依赖和死代码
- **@e18e/cli**: 分析和自动迁移冗余依赖
- **npmgraph**: 可视化依赖树
- **module-replacements**: 替代方案数据库

## 常用替换参考

| 冗余包 | 替换方案 | 类型 |
|--------|----------|------|
| chalk | picocolors / util.styleText | P2 |
| find-up | empathic | P2 |
| is-* 系列 | 原生类型检查 | P1/P2 |
| globalthis | 原生 globalThis | P3 |
| object.* 系列 | 原生 Object 方法 | P3 |

## 质量检查

- [ ] 是否识别了未使用的直接依赖
- [ ] 是否检查了完整依赖树（不只是顶层）
- [ ] 是否提供了具体的替换代码示例
- [ ] 是否评估了向后兼容性风险
