---
name: js-bloat-analyzer
description: 分析 JavaScript/Node.js 项目的依赖膨胀问题，识别冗余包并推荐替代方案。适用于清理项目依赖、优化 bundle 大小、提升安全性和可维护性。

input_schema:
  project_path:
    type: string
    required: true
    description: 项目根目录路径
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
    description: 是否自动应用修复

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

---

# JS 依赖膨胀分析器

基于 "The Three Pillars of JavaScript Bloat" 方法论，系统性分析和清理项目依赖。

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

## 输入参数

| 参数名 | 类型 | 必填 | 描述 |
|--------|------|------|------|
| project_path | string | 是 | 项目根目录路径 |
| check_depth | string | 否 | 检查深度：basic(仅直接依赖) / deep(完整依赖树)，默认 deep |
| auto_fix | boolean | 否 | 是否自动应用修复，默认 false |

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
    },
    {
      "name": "arrify",
      "type": "pillar_2", 
      "reason": "原子化微包",
      "replacement": "Array.isArray(val) ? val : [val]",
      "action": "inline"
    },
    {
      "name": "globalthis",
      "type": "pillar_3",
      "reason": "过期 ponyfill",
      "replacement": "原生 globalThis",
      "action": "remove"
    }
  ],
  "recommendations": [
    "使用 knip 移除未使用的依赖",
    "使用 @e18e/cli migrate 自动替换",
    "考虑使用 empathic 替代 find-up"
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
