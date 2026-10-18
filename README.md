[README.md](https://github.com/user-attachments/files/33625357/README.md)
# Food Delivery Analytics Assistant · 外卖经营分析助手

A Text-to-SQL personal project: business users ask questions in plain language, and the system writes SQL, runs it on a read-only database, fixes failed queries automatically, and explains the result. Evaluated on 30 questions with known answers; **accuracy improved from 70% to 90%** after iterating on the failure patterns.

[中文说明见下方](#中文说明)

| | English | 中文 |
|---|---|---|
| Showcase page | [en.html](https://tanyiwende19-source.github.io/text2sql-demo/en.html) | [index.html](https://tanyiwende19-source.github.io/text2sql-demo/) |
| Live demo | [Try it live](https://udify.app/workflow/s7PovPgswjjiKq9P) | [在线试用](https://udify.app/workflow/6XZuq5W0SQ9OhCxX) |

> All data is synthetic and does not represent any real company.

## How it works

```
Question
 → 1. LLM generates SQL (schema, metric definitions and SQL rules in the prompt)
 → 2. Code step cleans the SQL (strips code fences and semicolons)
 → 3. HTTP call to a read-only Supabase function, run_sql
 → 4. Success?
       yes → 7
       no  → 5. LLM rewrites the SQL using the error message → 6. runs it once more
 → 7. LLM explains the result: conclusion, data table, business insight, SQL used
```

**Stack:** Dify (workflow), Supabase / PostgreSQL, Python + duckdb (evaluation)

**Safety by design:** the `run_sql` database function only allows `SELECT` / `WITH`, blocks write keywords and multiple statements, times out after 5 seconds and returns at most 200 rows. Row-level security is on, so the public key cannot read tables directly. Safety does not depend on the model behaving well.

## Data

| Table | Rows | Description |
|---|---|---|
| users | 2,000 | City, sign-up date, user tier |
| merchants | 300 | Category, city, rating |
| orders | 13,657 | Jul–Sep 2026; status, amount, discount, delivery time |
| ab_assignments | 286 | New-customer coupon experiment, treatment vs control |

Generated with a fixed random seed, so the dataset is reproducible.

## Evaluation

- **30 test questions:** 10 easy, 12 medium, 8 hard. Hard questions cover A/B test metrics, window functions, month-over-month growth and retention, plus one question the data cannot answer.
- **Answer key:** reference SQL run with duckdb on the local CSVs, cross-checked against the database.
- **Scoring:** compares query results, not SQL text (one question can have several correct queries). Numbers allow ±0.01; column names and order are ignored; row order is checked when the question asks for a ranking.

| Difficulty | v1 | v2 |
|---|---|---|
| Easy | 8/10 | 10/10 |
| Medium | 11/12 | 12/12 |
| Hard | 2/8 | 5/8 |
| **Total** | **21/30 = 70%** | **27/30 = 90%** |

## What changed from v1 to v2

| Failure type | Cause | Fix |
|---|---|---|
| Date boundary | `<= '2026-09-30'` missed orders on the last day (timestamps include time of day) | Half-open date ranges |
| Experiment denominator | Inner join dropped assigned users with zero orders, inflating per-user averages | Start from the assignment table with a left join |
| Workflow bug | The retry branch ran, but the variable aggregator kept the first result | Reordered the aggregator inputs |
| SQL structure | Missing subquery, alias in HAVING, undefined alias | Rules for subqueries, window functions and retention; alias self-check |
| Explanation | Read only the first row of the result | Explanations must cover all rows |

## Limitations

- v2 rules were written from v1 failures and retested on the same questions, so 90% is optimistic. A held-out test set is the next step.
- Rules can over-generalise (one question applied a left join where it did not belong).
- Each question ran once; model output varies between runs.
- The evaluation used gpt-4o-mini. The live demo runs on Gemini 3.8 Flash, so answers may differ slightly.

## Next steps

- A searchable metric dictionary injected on demand, instead of hard-coding all rules in the prompt
- Ask a clarifying question when a request is ambiguous (e.g. "growth" as amount or rate)
- A held-out test set with three runs per question

---

## 中文说明

业务同学用中文提问，系统用大模型生成 SQL，在只读数据库里查询，出错时自动修正，最后给出带数字的业务结论。我用 30 道带标准答案的测试题评测，按错题原因迭代 Prompt 和工作流，**准确率从 70% 提升到 90%**。

- 展示页：<https://tanyiwende19-source.github.io/text2sql-demo/>
- 在线试用：<https://udify.app/workflow/6XZuq5W0SQ9OhCxX>

### 架构

生成 SQL → 清洗 SQL → 只读执行 → 失败时带着报错修正并重试一次 → 合并结果 → 解读（结论、数据表、业务解读、所用 SQL）。

**安全设计**：数据库函数 `run_sql` 只允许 SELECT / WITH，拦截写操作关键字和多条语句，5 秒超时，最多返回 200 行；表开启 RLS，公开 key 无法直接读表。

### 评测

30 道题（简单 10、中等 12、困难 8），用 duckdb 算出标准答案；判分比较查询结果而不是 SQL 文本。v1 准确率 70%，v2 提升到 90%，困难题从 25% 提升到 63%。

### 主要改进

- **时间边界**：改用左闭右开的时间条件，不再漏掉月末当天的订单
- **实验分母**：从分组表出发 left join，零订单用户也计入分母
- **工作流 bug**：修正分支的结果曾被变量聚合器丢掉，对调了变量顺序
- **SQL 结构与解读**：增加子查询、窗口函数、留存的写法规则，并要求解读覆盖全部行

### 局限

v2 是在同一套题上调优的，90% 偏乐观；每题只跑了一次；评测用的是 gpt-4o-mini，在线演示改用 Gemini 3.8 Flash，结果可能略有差异。

> 所有数据均为模拟数据，不代表任何真实公司。
