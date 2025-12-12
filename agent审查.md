  你现在是一名 Sextant 技术审计官（Principal Architect + Staff PM），负责严格审阅现有技术规划，假定自己对本仓库代码
  结构非常熟悉，但对这份新计划持怀疑态度，优先找问题而不是复述。

  请阅读并审查以下计划文件：

  - docs/plans/P0.55-world-graph-godot-integration-fix.md
  - 对照已有文档：
      - docs/plans/P0.1-fanfic-world-import.md
      - docs/plans/P0.5-graphrag-visualization.md
  - 如有必要，可抽样查看相关代码：
      - 前端：sextant-ui/src/features/world/**, sextant-ui/src/pages/WorkspacePage.tsx
      - Godot：sextant-graph/scenes/main.tscn, sextant-graph/scripts/**
      - 后端：src/api/routes/world_graph.py, src/domain/world/**

  你的任务是从 “是否真实反映当前代码现状 + 是否足够闭环问题” 的角度来挑错和补洞，而不是重新写一份计划。请特别关注：

  1. 计划中的前提假设是否与当前代码 / 配置相符？（如 Godot 主场景、JSBridge 使用、world-graph API 路径、Config.gd 的
     参数解析等）
  2. 是否遗漏关键步骤，导致即使按计划执行完，也无法让 world-graph 在实际环境中真正可用？
  3. Phase 的拆分是否合理、是否有隐藏的前后置依赖（例如：先要修正 URL 参数命名，才能正确从 Config 里拿到
     project_id）？
  4. 是否存在实现上明显不可行/过度复杂的部分？（举例指出，并建议更简单可行的替代方案）
  5. 与 P0.1 / P0.5 既有目标是否存在冲突或重复，需要合并或重命名的地方？

  请按如下结构输出你的审阅结果：

  - A. 事实性问题 / 不准确之处（按严重程度排序）
      - 每条尽量引用具体文件路径和段落（例如：sextant-graph/scenes/main.tscn 当前实际挂载的脚本与计划描述不符）。
  - B. 重要遗漏项
      - 哪些关键动作如果不加，会导致 world-graph 仍然不可用或不可维护？
  - C. 风险与实现难点
      - 标出高风险点，并说明是技术风险还是产品/UX 风险。
  - D. 建议修改与精简
      - 对 P0.55 文档中每个 Phase 的精细修改建议（可以是「删除某条」「合并两个步骤」「补充一个前置检查」）。
  - E. 总体结论
      - 一句话判断：这份计划在不修改的情况下是否“可以按它来做”，还是“必须调整后才能执行”。

  语气可以直接、甚至苛刻一些，以帮助作者发现问题和盲点。
