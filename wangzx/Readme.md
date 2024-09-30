## 2024-09-10
1. 按照 https://duckdb.org/docs/dev/building/build_instructions 的说明，可以 构建成功
   ```shell
    GEN=ninja make
   ```
2. 在 Clion 可以调试代码。   
   target: duckdb_main 
   入口: tools/shell/shell.c
   可以正确运行，调试。
   ```bash
   duckdb wangzx/demo.duck "select *, current_date() from users"
   
   # 调试断点： duckdb/src/core_functions/scalar/date/current.cpp:: CurrentDateFunction
   ```
   从这个过程，可以查看代码的调用堆栈，帮助理解 SQL 的执行过程。
   
```
duckdb::CurrentDateFunction(duckdb::DataChunk &, duckdb::ExpressionState &, duckdb::Vector &) current.cpp:25
duckdb::ExpressionExecutor::Execute(const duckdb::BoundFunctionExpression &, duckdb::ExpressionState *, const duckdb::SelectionVector *, unsigned long long, duckdb::Vector &) execute_function.cpp:78
duckdb::ExpressionExecutor::Execute(const duckdb::Expression &, duckdb::ExpressionState *, const duckdb::SelectionVector *, unsigned long long, duckdb::Vector &) expression_executor.cpp:211
duckdb::ExpressionExecutor::ExecuteExpression(unsigned long long, duckdb::Vector &) expression_executor.cpp:102
duckdb::ExpressionExecutor::ExecuteExpression(duckdb::Vector &) expression_executor.cpp:96
duckdb::ExpressionExecutor::EvaluateScalar(duckdb::ClientContext &, const duckdb::Expression &, bool) expression_executor.cpp:112
duckdb::ExpressionExecutor::TryEvaluateScalar(duckdb::ClientContext &, const duckdb::Expression &, duckdb::Value &) expression_executor.cpp:122
duckdb::ConstantFoldingRule::Apply(duckdb::LogicalOperator &, duckdb::vector<…> &, bool &, bool) constant_folding.cpp:36
duckdb::ExpressionRewriter::ApplyRules(duckdb::LogicalOperator &, const duckdb::vector<…> &, duckdb::unique_ptr<…>, bool &, bool) expression_rewriter.cpp:19
duckdb::ExpressionRewriter::VisitExpression(duckdb::unique_ptr<…> *) expression_rewriter.cpp:78
(function scope)::$_22::operator()(duckdb::unique_ptr<…> *) const logical_operator_visitor.cpp:114
duckdb::LogicalOperatorVisitor::EnumerateExpressions(duckdb::LogicalOperator &, const std::__1::function<…> &) logical_operator_visitor.cpp:109
duckdb::LogicalOperatorVisitor::VisitOperatorExpressions(duckdb::LogicalOperator &) logical_operator_visitor.cpp:114
duckdb::ExpressionRewriter::VisitOperator(duckdb::LogicalOperator &) expression_rewriter.cpp:65
(function scope)::$_12::operator()() const optimizer.cpp:104
duckdb::Optimizer::RunOptimizer(duckdb::OptimizerType, const std::__1::function<…> &) optimizer.cpp:78
duckdb::Optimizer::RunBuiltInOptimizers() optimizer.cpp:104
duckdb::Optimizer::Optimize(duckdb::unique_ptr<…>) optimizer.cpp:236
  -- 要理解 Optimizer 的输入和输出数据结构 LogicOperator
duckdb::ClientContext::CreatePreparedStatementInternal(duckdb::ClientContextLock &, const std::__1::basic_string<…> &, duckdb::unique_ptr<…>, duckdb::optional_ptr<…>) client_context.cpp:357
duckdb::ClientContext::CreatePreparedStatement(duckdb::ClientContextLock &, const std::__1::basic_string<…> &, duckdb::unique_ptr<…>, duckdb::optional_ptr<…>, duckdb::PreparedStatementMode) client_context.cpp:424
(function scope)::$_2::operator()() const client_context.cpp:659
duckdb::ClientContext::RunFunctionInTransactionInternal(duckdb::ClientContextLock &, const std::__1::function<…> &, bool) client_context.cpp:1082
duckdb::ClientContext::PrepareInternal(duckdb::ClientContextLock &, duckdb::unique_ptr<…>) client_context.cpp:658
duckdb::ClientContext::Prepare(duckdb::unique_ptr<…>) client_context.cpp:671
duckdb::Connection::Prepare(duckdb::unique_ptr<…>) connection.cpp:152
::duckdb_shell_sqlite3_prepare_v2(sqlite3 *, const char *, int, sqlite3_stmt **, const char **) sqlite3_api_wrapper.cpp:204
  -- 并不仅仅是 prepare，而是直接执行
shell_exec shell.c:12989
main shell.c:20154
```

-[ ] understand std::move
- 断点： PhysicalTableScan::GetData 这个应该是物理执行层了。

# 2024-09-24

```sql
select name, count(freight), sum(freight) 
from sale_orders so 
left join customers c on c.customer_id = so.customer_id 
where name = 'IB89Nf23kAom' 
group by name;
```
1. 画出 pipeline 树
   pipeline 1: PhysicalTableScan(c) -> PhysicalFilter(customer_id <= 999999) -> PhysicalHashJoin
      - parents: pipeline 2: PhysicalTableScan(so) -> PhysicalHashJoin -> PhysicalProjection -> PhysicalHashAggregate
        - parents: pipeline 3: PhysicalHashAggregate -> PhysicalMaterializedCollector 
          - parents: pipeline 4: PhysicalMaterializedCollector -> null
2. 阅读这几个 PhysicalOperator 的源代码，理解其执行逻辑
   - PhysicalTableScan
   - PhysicalHashJoin as sink and operator
   - PhysicalHashAggregate
   - PhysicalFilter
   - PhysicalProjection
3. 单个 Pipeline 是如何拆分为多个 Task 执行的？
4. Join 的 Filter 左移 的执行流程 JoinFilterPushdownInfo::PushFilters
5. 这个执行计划中的 PhysicalHashAggregate 是如何执行的，是否足够高效？(主要是group by 是否需要维护一个很大的内存？)
6. 1 个 Pipeline 示例会创建多个 PipelineTask, 应该是进行分区处理？估计其共享信息会保存在 pipeline 内。
7. DuckDB 的 Operator 逻辑上区分为 Source, Operator, Sink 三个接口，但又都揉在一起，为什么不拆分为3个不同的职能？
   - Operator: 在执行计划中的。 
   - 在 Pipeline 中命名为 PipeOperator: Source, Filter, Collector.
8. 如何查看 DataChunk 的数据？阅读熟悉 Vector 的数据结构。
   - Rust 的 struct + impl 的方式，阅读代码显然比 C++ 的方式更加清晰。
9. [ ] 阅读 read_csv 表函数的代码，评估后续提供一个类似的函数 read_jdbc(url, user, password, sql)
10. 如果使用 rust 来实现，会怎么做？
11. 窗口函数是如何执行的？
12. 子查询是如何执行的？