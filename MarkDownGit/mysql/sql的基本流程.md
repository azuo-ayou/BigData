- 解析，sql的语法解析和生成对应的AST抽象语法数
- 校验，通过元数据校验sql中使用的函数、表名、字段等
- 优化，优化sql对应的AST。合并计算，剪枝，谓词下推
- 执行，生成实行执行的物理代码和逻辑，并交由对应的执行引擎



mysql怎么保证数据的一致性