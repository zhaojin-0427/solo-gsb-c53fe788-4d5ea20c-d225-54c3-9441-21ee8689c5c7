# 关系查询溯源与反事实解释 API

客户端一次请求提交 **多张带主键的关系表快照** 和一个 **受限关系代数计划**，
服务执行查询，并为每条输出返回：

* **全部极小来源行见证集**（minimal witness sets / why-provenance）；
* **布尔溯源表达式**（DNF：见证内部为合取 `∧`，见证之间为析取 `∨`）；
* 可选地，对指定输出计算 **令其消失的全部最小源行删除集**，并对每个删除集
  **在删除后的快照上重算整条计划**，返回重算结果。

技术栈：Python 3.11 · FastAPI · SQLite（幂等响应持久化）· 无外部服务依赖。

---

## 1. 一键启动（Docker Compose）

```bash
docker compose up --build
```

启动后：

| 用途 | 地址 |
| --- | --- |
| HTTP API | http://localhost:8000 |
| 交互式文档（Swagger） | http://localhost:8000/docs |
| 健康检查 | http://localhost:8000/health |

幂等响应持久化在 Docker 卷 `provenance-data`（容器内 `/data/provenance.db`）。

### 本地直接运行（不使用 Docker）

```bash
python3 -m pip install -r requirements.txt
PROVENANCE_DB=./data/provenance.db \
  python3 -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

运行测试：

```bash
python3 -m pip install pytest
python3 -m pytest -q
```

---

## 2. API

### `POST /query`

请求体（JSON）：

```jsonc
{
  "idempotency_key": "readme-demo-1",        // 必填，绑定本次规范化请求
  "tables": [ ... ],                         // 1..n 张表快照（见下）
  "plan": { ... },                           // 1 个关系代数计划
  "explain": { "index": 0 }                  // 可选：对某条输出做反事实解释
}
```

**表快照**

```jsonc
{
  "table": "orders",
  "primary_key": ["oid"],                    // 必填，非空；支持复合主键
  "columns": [                               // 可省略（从行推断）
    {"name": "oid", "type": "int"},          // 类型: int/float/string/bool/any
    {"name": "cid", "type": "int"},
    {"name": "amount", "type": "int"}
  ],
  "rows": [ {"oid": 1, "cid": 10, "amount": 50}, ... ]
}
```

**关系代数计划节点**

| 节点 | 形式 |
| --- | --- |
| 扫描 | `{"type":"scan","table":"orders"}` |
| 选择/过滤 | `{"type":"filter","input":<node>,"predicate":<谓词>}` |
| 投影 | `{"type":"project","input":<node>,"columns":["a",{"from":"b","as":"c"}]}` |
| 等值连接 | `{"type":"join","left":<node>,"right":<node>,"on":[{"left":"cid","right":"cid"}]}` |
| 分组聚合 | `{"type":"aggregate","input":<node>,"group_by":["cid"],"aggregates":[{"fn":"sum","column":"amount","as":"total"}]}` |

* 谓词树：叶子 `{"column":"amount","op":">=","value":40}`；
  算子 `= <> < <= > >=` 以及 `is_null` / `is_not_null`；
  布尔组合 `{"and":[...]}`、`{"or":[...]}`、`{"not":{...}}`。
  谓词按 SQL 三值逻辑求值：与 `NULL` 比较得到 UNKNOWN，不满足选择条件。
* 聚合函数：`count`（`column` 可写 `"*"` 或具体列，后者忽略 NULL）、
  `sum` / `min` / `max`（自动忽略 NULL；全 NULL 时 `sum/min/max` 为 `null`）。
* 等值连接的连接列按 USING 语义合并；其余右表列名冲突时自动限定为
  `right.<列>`（可用 `right_alias` 指定限定名），或在投影中重命名。
* 计划只能引用已提交的表与其字段，否则返回 `ILLEGAL_REFERENCE` / `UNKNOWN_TABLE`。

**反事实目标选择器**（`explain`，二选一）：

* `{"index": 0}` —— 输出按全部列全序排序后的稳定下标；
* `{"row": {...}}` —— 与输出列完全相等的一行。

### 响应约定

* 所有响应都是 **规范化（canonical）JSON**：对象键按码点排序、无多余空白、
  无尾随换行；成功响应附带 `X-Idempotent-Replay: true/false`。
* 幂等：`(idempotency_key, 规范化请求指纹)` 绑定。重复请求 **逐字节一致**；
  同一键但规范化计划/快照/目标不同 → `409 IDEMPOTENCY_KEY_CONFLICT`。
  JSON 中对象键的书写顺序不影响绑定（指纹基于规范化 JSON 计算）。

---

## 3. 实际请求 / 响应示例

### 请求：过滤 + 等值连接 + 投影，并解释输出行 0

```bash
curl -s -X POST http://localhost:8000/query \
  -H 'Content-Type: application/json' \
  -d '{
  "idempotency_key": "readme-demo-1",
  "tables": [
    {"table": "orders", "primary_key": ["oid"],
     "columns": [{"name":"oid","type":"int"},{"name":"cid","type":"int"},{"name":"amount","type":"int"}],
     "rows": [{"oid":1,"cid":10,"amount":50},{"oid":2,"cid":10,"amount":30}]},
    {"table": "customers", "primary_key": ["cid"],
     "columns": [{"name":"cid","type":"int"},{"name":"name","type":"string"}],
     "rows": [{"cid":10,"name":"Alice"}]}
  ],
  "plan": {
    "type": "project",
    "columns": ["name", {"from":"oid","as":"order_id"}, "amount"],
    "input": {
      "type": "filter",
      "predicate": {"column":"amount","op":">=","value":40},
      "input": {
        "type": "join",
        "left": {"type":"scan","table":"orders"},
        "right": {"type":"scan","table":"customers"},
        "on": [{"left":"cid","right":"cid"}]
      }
    }
  },
  "explain": {"index": 0}
}'
```

实际响应（线上传输为单行紧凑 JSON；这里为可读性格式化）：

```json
{
  "status": "ok",
  "symbol_table": [
    {"symbol": "s1", "source": "customers[10]"},
    {"symbol": "s2", "source": "orders[1]"}
  ],
  "result": {
    "columns": ["name", "order_id", "amount"],
    "row_count": 1,
    "rows": [
      {
        "row": {"name": "Alice", "order_id": 1, "amount": 50},
        "witness_count": 1,
        "minimal_witnesses": [
          {
            "labels": ["customers[10]", "orders[1]"],
            "symbols": ["s1", "s2"],
            "conjunction": "s1 ∧ s2"
          }
        ],
        "provenance": {
          "dnf": "(s1 ∧ s2)",
          "dnf_structured": [["s1", "s2"]],
          "labels_structured": [["customers[10]", "orders[1]"]]
        }
      }
    ]
  },
  "counterfactual": {
    "target_index": 0,
    "target": {"name": "Alice", "order_id": 1, "amount": 50},
    "target_provenance": {
      "dnf": "(s1 ∧ s2)",
      "dnf_structured": [["s1", "s2"]],
      "labels_structured": [["customers[10]", "orders[1]"]]
    },
    "minimal_deletion_count": 2,
    "minimal_deletion_sets": [
      ["customers[10]"],
      ["orders[1]"]
    ],
    "scenarios": [
      {
        "delete": ["customers[10]"],
        "target_present": false,
        "result": {"columns": ["name", "order_id", "amount"], "row_count": 0, "rows": []}
      },
      {
        "delete": ["orders[1]"],
        "target_present": false,
        "result": {"columns": ["name", "order_id", "amount"], "row_count": 0, "rows": []}
      }
    ]
  }
}
```

解读：输出行 `(Alice,1,50)` 的唯一极小见证是 `orders[1] ∧ customers[10]`
（两条源行合取）；让它消失的最小删除集有两个：删客户 `customers[10]`
或删订单 `orders[1]`。两种删除都会在重算后使目标消失。

### 分组聚合的数值溯源（count / sum / min / max）

聚合输出行 = 组键 + 各聚合值。每个聚合表达式都为“这一确切数值”贡献一个
见证组件，与“组至少剩一行”的存在性组件合取（逐组件取并集）后极小化：

* `count(*)` / `count(col)`：覆盖 **所有参与计数的行**（后者忽略 NULL）
  —— 一个合取；
* `sum(col)`：枚举 **和为目标值的全部极小非空行组合**（按行而非按值，
  同值行产生不同组合并完整保留；空组合不是见证 —— 零行的和不由任何
  来源行产生）；
* `min(col)` / `max(col)`：枚举 **仍能产生目标极值的极小组合** —— 每个
  等于极值的行各成一个见证。

因此同组订单金额为 50 和 30 时，`count=2`、`sum=80` 只在两行都在时成立，
溯源是单个合取 `s1 ∧ s2`（而不是两条单行见证的析取）；删任一行数值即变，
最小删除集是两个单例，与删除后重算完全一致。

### 请求：分组聚合 + 解释组 `cid=10`

```bash
curl -s -X POST http://localhost:8000/query \
  -H 'Content-Type: application/json' \
  -d '{
  "idempotency_key": "readme-agg-1",
  "tables": [
    {"table": "orders", "primary_key": ["oid"],
     "columns": [{"name":"oid","type":"int"},{"name":"cid","type":"int"},{"name":"amount","type":"int"}],
     "rows": [
       {"oid":1,"cid":10,"amount":50},
       {"oid":2,"cid":10,"amount":30},
       {"oid":3,"cid":20,"amount":10}
     ]}
  ],
  "plan": {
    "type": "aggregate",
    "input": {"type":"scan","table":"orders"},
    "group_by": ["cid"],
    "aggregates": [
      {"fn":"count","column":"*","as":"n"},
      {"fn":"sum","column":"amount","as":"total"}
    ]
  },
  "explain": {"row": {"cid":10,"n":2,"total":80}}
}'
```

实际响应（线上传输为单行紧凑 JSON；这里为可读性格式化）：

```json
{
  "status": "ok",
  "symbol_table": [
    {"symbol": "s1", "source": "orders[1]"},
    {"symbol": "s2", "source": "orders[2]"},
    {"symbol": "s3", "source": "orders[3]"}
  ],
  "result": {
    "columns": ["cid", "n", "total"],
    "row_count": 2,
    "rows": [
      {
        "row": {"cid": 10, "n": 2, "total": 80},
        "witness_count": 1,
        "minimal_witnesses": [
          {
            "labels": ["orders[1]", "orders[2]"],
            "symbols": ["s1", "s2"],
            "conjunction": "s1 ∧ s2"
          }
        ],
        "provenance": {
          "dnf": "(s1 ∧ s2)",
          "dnf_structured": [["s1", "s2"]],
          "labels_structured": [["orders[1]", "orders[2]"]]
        }
      },
      {
        "row": {"cid": 20, "n": 1, "total": 10},
        "witness_count": 1,
        "minimal_witnesses": [
          {"labels": ["orders[3]"], "symbols": ["s3"], "conjunction": "s3"}
        ],
        "provenance": {
          "dnf": "(s3)",
          "dnf_structured": [["s3"]],
          "labels_structured": [["orders[3]"]]
        }
      }
    ]
  },
  "counterfactual": {
    "target_index": 0,
    "target": {"cid": 10, "n": 2, "total": 80},
    "target_provenance": {
      "dnf": "(s1 ∧ s2)",
      "dnf_structured": [["s1", "s2"]],
      "labels_structured": [["orders[1]", "orders[2]"]]
    },
    "minimal_deletion_count": 2,
    "minimal_deletion_sets": [
      ["orders[1]"],
      ["orders[2]"]
    ],
    "scenarios": [
      {
        "delete": ["orders[1]"],
        "target_present": false,
        "result": {
          "columns": ["cid", "n", "total"],
          "row_count": 2,
          "rows": [
            {"cid": 10, "n": 1, "total": 30},
            {"cid": 20, "n": 1, "total": 10}
          ]
        }
      },
      {
        "delete": ["orders[2]"],
        "target_present": false,
        "result": {
          "columns": ["cid", "n", "total"],
          "row_count": 2,
          "rows": [
            {"cid": 10, "n": 1, "total": 50},
            {"cid": 20, "n": 1, "total": 10}
          ]
        }
      }
    ]
  }
}
```

解读：目标行 `(10,2,80)` 的唯一极小见证是 `orders[1] ∧ orders[2]`；
令它消失的最小删除集是删 `orders[1]` 或 `orders[2]`。两个场景都在删除后
重算了整条计划：组 `cid=10` 仍在，但变成 `(10,1,30)` / `(10,1,50)`，
目标行消失（`target_present: false`）——布尔溯源、最小删除集与重算结果
三者一致。

多见证（DNF 多析取项）在 `sum`/`min`/`max` 出现多种极小组合时产生。例如
同组金额为 `30, 30, 60, -60` 时 `sum=60` 的极小组合是 `{30,30}` 与 `{60}`，
溯源为 `(s1 ∧ s2) ∨ (s3)`；该组含正负混合值，求和是非单调的 —— 删除
**任意一行** 都会改变总和（去掉 30 剩 30、去掉 60 剩 0、去掉 -60 剩 120），
因此最小删除集是四个单例 `["orders[1]"]`、`["orders[2]"]`、`["orders[3]"]`、
`["orders[4]"]`（由删除后重算判定，见下节）；`min`/`max` 则对每个等于
极值的行各给出一个单例见证。

### 混合正负金额的 sum（非单调求和）

同组金额一正一负时，`sum` 对删除不是单调的：删掉一行不属于任何极小
组合的来源行，总和照样会变。此时“令目标行消失的删除集”**不能**由见证
的命中集刻画（命中集是单调近似，会漏报甚至误报），服务先对目标组做候选
约简（见下节），再对剩余候选删除集逐个在删除后的快照上重算整条计划，
以完整目标行是否消失为准，返回全部按包含关系极小的方案。

下面的请求里，组 `cid=10` 的两行金额为 `5` 和 `-5`，合计 `total=0`：

```bash
curl -s -X POST http://localhost:8000/query \
  -H 'Content-Type: application/json' \
  -d '{
  "idempotency_key": "readme-mixed-1",
  "tables": [
    {"table": "orders", "primary_key": ["oid"],
     "columns": [{"name":"oid","type":"int"},{"name":"cid","type":"int"},{"name":"amount","type":"int"}],
     "rows": [
       {"oid":1,"cid":10,"amount":5},
       {"oid":2,"cid":10,"amount":-5},
       {"oid":3,"cid":20,"amount":7}
     ]}
  ],
  "plan": {
    "type": "aggregate",
    "input": {"type":"scan","table":"orders"},
    "group_by": ["cid"],
    "aggregates": [{"fn":"sum","column":"amount","as":"total"}]
  },
  "explain": {"row": {"cid":10,"total":0}}
}'
```

实际响应（线上传输为单行紧凑 JSON；这里为可读性格式化）：

```json
{
  "status": "ok",
  "symbol_table": [
    {"symbol": "s1", "source": "orders[1]"},
    {"symbol": "s2", "source": "orders[2]"},
    {"symbol": "s3", "source": "orders[3]"}
  ],
  "result": {
    "columns": ["cid", "total"],
    "row_count": 2,
    "rows": [
      {
        "row": {"cid": 10, "total": 0},
        "witness_count": 1,
        "minimal_witnesses": [
          {
            "labels": ["orders[1]", "orders[2]"],
            "symbols": ["s1", "s2"],
            "conjunction": "s1 ∧ s2"
          }
        ],
        "provenance": {
          "dnf": "(s1 ∧ s2)",
          "dnf_structured": [["s1", "s2"]],
          "labels_structured": [["orders[1]", "orders[2]"]]
        }
      },
      {
        "row": {"cid": 20, "total": 7},
        "witness_count": 1,
        "minimal_witnesses": [
          {"labels": ["orders[3]"], "symbols": ["s3"], "conjunction": "s3"}
        ],
        "provenance": {
          "dnf": "(s3)",
          "dnf_structured": [["s3"]],
          "labels_structured": [["orders[3]"]]
        }
      }
    ]
  },
  "counterfactual": {
    "target_index": 0,
    "target": {"cid": 10, "total": 0},
    "target_provenance": {
      "dnf": "(s1 ∧ s2)",
      "dnf_structured": [["s1", "s2"]],
      "labels_structured": [["orders[1]", "orders[2]"]]
    },
    "minimal_deletion_count": 2,
    "minimal_deletion_sets": [
      ["orders[1]"],
      ["orders[2]"]
    ],
    "scenarios": [
      {
        "delete": ["orders[1]"],
        "target_present": false,
        "result": {
          "columns": ["cid", "total"],
          "row_count": 2,
          "rows": [
            {"cid": 10, "total": -5},
            {"cid": 20, "total": 7}
          ]
        }
      },
      {
        "delete": ["orders[2]"],
        "target_present": false,
        "result": {
          "columns": ["cid", "total"],
          "row_count": 2,
          "rows": [
            {"cid": 10, "total": 5},
            {"cid": 20, "total": 7}
          ]
        }
      }
    ]
  }
}
```

解读：`total=0` 的唯一极小见证是非空组合 `orders[1] ∧ orders[2]`（空
组合不会被当作见证，否则溯源会退化成“组存在即可”）。令目标消失的
最小删除集是两个单例：删 `orders[1]` 总和变 `-5`，删 `orders[2]` 总和
变 `5` —— 与删除后重算逐一吻合；另一组 `cid=20` 不受影响。

再如单组金额为 `5, 2, -2`、`sum=5` 时，唯一极小组合是 `{5}`（溯源
`(s1)`），但删 `2` 总和变 `3`、删 `-2` 总和变 `7`，所以最小删除集是
三个单例 `["orders[1]"]`、`["orders[2]"]`、`["orders[3]"]` —— 包括不
出现在任何极小组合中的行。注意非单调求和的删除集不具备向上封闭性：
上例中删除 `{2,-2}` 两行反而让总和回到 `5`，目标行重新出现；每个返回
的方案都经过重算验证且按包含关系极小。

#### 候选约简与搜索预算（金额 `5, 2, -2` 加十条 `0`，共 13 行）

历史行为里，只要目标组的候选行超过 12 行，精确重算搜索就会放弃并退回
见证命中集近似 —— 对 `5, 2, -2` 加十条 `0`、`total=5` 的请求只报告
删除 `5`（唯一极小求和组合），漏掉删 `2` / `-2` 两个同样极小的方案。
现在候选池先做 **候选约简**，再按预算做精确枚举：

* **候选约简（保极小性，不丢方案）**：对目标组的每个聚合表达式，删除后
  不可能改变目标值的组成员称为“惰性成员”——
  `count(*)` 无惰性行；`count(col)` 的 `col IS NULL` 行为惰性；
  `sum(col)` 的 `col IS NULL` 或 `col = 0` 行为惰性（删掉零行总和不变，
  十条补位 `0` 全部约简掉）；`min/max(col)` 中非 NULL 且不等于目标极值
  的行为惰性。只在惰性成员见证中出现的来源行标签移出候选池 —— 但约简
  依据标签与成员见证的依赖关系：聚合输入为连接（尤其自连接）时，零值
  成员可能与非零成员共享标签；存在“危险核心”时（见下节）零值成员的标
  签保留在候选池中，一对一连接的零成员仍照常约简。
* **精确枚举（不利用失败剪枝）**：按删除集大小递增枚举；**只有已找到
  解的超集才跳过**（超集必不极小），某个删除集没让目标消失，绝不据此
  跳过它的超集 —— 混合正负求和下，多删一行可能正好把总和改离目标值。
* **两道搜索预算**：约简后候选池上限 **20 行**（否则整体 2^n 不可行）；
  单次反事实搜索的整计划重算次数上限 **10 000 次**。任一预算耗尽都返回
  结构化错误 `422 COUNTERFACTUAL_SEARCH_BUDGET_EXCEEDED`，**不会**以成功
  响应输出命中集近似或部分方案；该错误同样绑定幂等键，重放逐字节一致。

下面就是上面那条 13 行输入的实际请求与响应（十条 `0` 被约简，三个单例
删除集按标签顺序返回，重算总和分别为 `0`、`3`、`7`）：

```bash
curl -s -X POST http://localhost:8000/query \
  -H 'Content-Type: application/json' \
  -d '{
  "idempotency_key": "readme-mixed-padded-g1",
  "tables": [
    {"table": "orders", "primary_key": ["oid"],
     "columns": [{"name":"oid","type":"int"},{"name":"cid","type":"int"},{"name":"amount","type":"int"}],
     "rows": [
       {"oid":1,"cid":10,"amount":5},
       {"oid":2,"cid":10,"amount":2},
       {"oid":3,"cid":10,"amount":-2},
       {"oid":4,"cid":10,"amount":0},{"oid":5,"cid":10,"amount":0},
       {"oid":6,"cid":10,"amount":0},{"oid":7,"cid":10,"amount":0},
       {"oid":8,"cid":10,"amount":0},{"oid":9,"cid":10,"amount":0},
       {"oid":10,"cid":10,"amount":0},{"oid":11,"cid":10,"amount":0},
       {"oid":12,"cid":10,"amount":0},{"oid":13,"cid":10,"amount":0}
     ]}
  ],
  "plan": {
    "type": "aggregate",
    "input": {"type":"scan","table":"orders"},
    "group_by": ["cid"],
    "aggregates": [{"fn":"sum","column":"amount","as":"total"}]
  },
  "explain": {"row": {"cid":10,"total": 5}}
}'
```

实际响应（线上传输为单行紧凑 JSON；这里为可读性格式化）：

```json
{
  "status": "ok",
  "symbol_table": [
    {"symbol": "s1", "source": "orders[1]"}
  ],
  "result": {
    "columns": ["cid", "total"],
    "row_count": 1,
    "rows": [
      {
        "row": {"cid": 10, "total": 5},
        "witness_count": 1,
        "minimal_witnesses": [
          {
            "labels": ["orders[1]"],
            "symbols": ["s1"],
            "conjunction": "s1"
          }
        ],
        "provenance": {
          "dnf": "(s1)",
          "dnf_structured": [["s1"]],
          "labels_structured": [["orders[1]"]]
        }
      }
    ]
  },
  "counterfactual": {
    "target_index": 0,
    "target": {"cid": 10, "total": 5},
    "target_provenance": {
      "dnf": "(s1)",
      "dnf_structured": [["s1"]],
      "labels_structured": [["orders[1]"]]
    },
    "minimal_deletion_count": 3,
    "minimal_deletion_sets": [
      ["orders[1]"],
      ["orders[2]"],
      ["orders[3]"]
    ],
    "scenarios": [
      {
        "delete": ["orders[1]"],
        "target_present": false,
        "result": {"columns": ["cid", "total"], "row_count": 1,
                   "rows": [{"cid": 10, "total": 0}]}
      },
      {
        "delete": ["orders[2]"],
        "target_present": false,
        "result": {"columns": ["cid", "total"], "row_count": 1,
                   "rows": [{"cid": 10, "total": 3}]}
      },
      {
        "delete": ["orders[3]"],
        "target_present": false,
        "result": {"columns": ["cid", "total"], "row_count": 1,
                   "rows": [{"cid": 10, "total": 7}]}
      }
    ]
  }
}
```

解读：十条 `0` 行既不进入任何极小求和组合（符号表只有 `s1`），也被候选
约简移出重算搜索池；但 `2` 与 `-2` 是非零行，删任一行都会改变总和，
所以仍是极小删除集。若同一计划再选 `count(*)`，零行就不再是惰性行
（删它们会改变计数），约简会保留全部 13 行，十三个单例删除集全部返回。

#### 连接（自连接）来源标签依赖：零值成员的标签不能一律约简

候选约简按 **来源行标签与成员见证的依赖关系** 决定零值成员的标签能否移
出候选池，而不是只看成员金额。聚合输入可以是任意子计划（连接、过滤、
投影），一个输入成员由其极小见证的合取支撑；当零值成员与非零成员 **共
享来源标签** 时（自连接的典型情形），删掉共享标签会同时移除两类成员，
零值成员的标签可能进入极小删除集。

判定规则：对非零（活跃）成员的见证做合取覆盖，取其极小命中集 `A`，若

1. 删掉 `A` 后所有活跃成员都消失，而存活的惰性（零值）成员仍复现目标行
   （总和等聚合值与组键不变）；并且
2. 对任意 `a ∈ A`，只删 `A ∖ {a}` 时目标行仍在（否则 `A` 自身或其真子
   集已是更小的删除集）；

则 `A` 是“危险核心”，存活零值成员见证中不属于活跃标签池的标签必须保留
在候选池里 —— 再删这些零值标签才能让目标真正消失（其重算会使聚合输入
为空，场景保留 `EMPTY_AGGREGATE` 结构化错误）。若连接是一对一的（各成
员见证互不相交），零值成员总能被独立删干净，条件 2 不成立，任意多条零
成员仍被完整约简，候选池不会膨胀，也就不会误报预算耗尽。

下面的请求把同一张表按 `lk = rk` 自连接后再过滤出三条成员：来源行 1 分
别与行 2（金额 `5`）、行 3（金额 `-5`）连接，行 4 自连接产生金额 `0`；
行 5 的自连接被过滤掉。右表金额合计 `total = 0`，两条极小见证是

* 零值自连接成员本身：`t[4]`；
* `5 + (-5)` 相消：`t[1] ∧ t[2] ∧ t[3]`（行 1 是两条非零成员的共同来源）。

```bash
curl -s -X POST http://localhost:8000/query \
  -H 'Content-Type: application/json' \
  -d '{
  "idempotency_key": "readme-selfjoin-mixed-zero-1",
  "tables": [
    {
      "table": "t",
      "primary_key": ["id"],
      "columns": [
        {"name": "id", "type": "int"},
        {"name": "lk", "type": "int"},
        {"name": "rk", "type": "int"},
        {"name": "amount", "type": "int"}
      ],
      "rows": [
        {"id": 1, "lk": 1, "rk": null, "amount": 0},
        {"id": 2, "lk": null, "rk": 1, "amount": 5},
        {"id": 3, "lk": null, "rk": 1, "amount": -5},
        {"id": 4, "lk": 2, "rk": 2, "amount": 0},
        {"id": 5, "lk": 3, "rk": 3, "amount": 100}
      ]
    }
  ],
  "plan": {
    "type": "aggregate",
    "input": {
      "type": "filter",
      "predicate": {
        "or": [
          {"column": "right.id", "op": "=", "value": 2},
          {"column": "right.id", "op": "=", "value": 3},
          {"column": "right.id", "op": "=", "value": 4}
        ]
      },
      "input": {
        "type": "join",
        "left": {"type": "scan", "table": "t"},
        "right": {"type": "scan", "table": "t"},
        "on": [{"left": "lk", "right": "rk"}]
      }
    },
    "aggregates": [{"fn": "sum", "column": "right.amount", "as": "total"}]
  },
  "explain": {"row": {"total": 0}}
}'
```

实际响应（线上传输为单行紧凑 JSON；这里为可读性格式化）：

```json
{
  "status": "ok",
  "symbol_table": [
    {"symbol": "s1", "source": "t[1]"},
    {"symbol": "s2", "source": "t[2]"},
    {"symbol": "s3", "source": "t[3]"},
    {"symbol": "s4", "source": "t[4]"}
  ],
  "result": {
    "columns": ["total"],
    "row_count": 1,
    "rows": [
      {
        "row": {"total": 0},
        "witness_count": 2,
        "minimal_witnesses": [
          {
            "labels": ["t[1]", "t[2]", "t[3]"],
            "symbols": ["s1", "s2", "s3"],
            "conjunction": "s1 ∧ s2 ∧ s3"
          },
          {
            "labels": ["t[4]"],
            "symbols": ["s4"],
            "conjunction": "s4"
          }
        ],
        "provenance": {
          "dnf": "(s1 ∧ s2 ∧ s3) ∨ (s4)",
          "dnf_structured": [["s1", "s2", "s3"], ["s4"]],
          "labels_structured": [["t[1]", "t[2]", "t[3]"], ["t[4]"]]
        }
      }
    ]
  },
  "counterfactual": {
    "target_index": 0,
    "target": {"total": 0},
    "target_provenance": {
      "dnf": "(s1 ∧ s2 ∧ s3) ∨ (s4)",
      "dnf_structured": [["s1", "s2", "s3"], ["s4"]],
      "labels_structured": [["t[1]", "t[2]", "t[3]"], ["t[4]"]]
    },
    "minimal_deletion_count": 3,
    "minimal_deletion_sets": [
      ["t[1]", "t[4]"],
      ["t[2]"],
      ["t[3]"]
    ],
    "scenarios": [
      {
        "delete": ["t[1]", "t[4]"],
        "target_present": false,
        "recompute_error": {
          "code": "EMPTY_AGGREGATE",
          "message": "aggregate input is empty: no groups can be formed"
        }
      },
      {
        "delete": ["t[2]"],
        "target_present": false,
        "result": {"columns": ["total"], "row_count": 1,
                   "rows": [{"total": -5}]}
      },
      {
        "delete": ["t[3]"],
        "target_present": false,
        "result": {"columns": ["total"], "row_count": 1,
                   "rows": [{"total": 5}]}
      }
    ]
  }
}
```

解读：删 `t[2]`（剩 `-5`）或删 `t[3]`（剩 `5`）是两个单例方案。只删
`t[1]` 时两条非零成员都消失，但零值自连接成员 `t[4]` 仍在，总和还是
`0`、目标行重新出现，所以 `t[1]` 不是删除集；必须与 `t[4]` 一起删 ——
此时三条连接成员全部消失，聚合输入为空，重算返回结构化错误
`EMPTY_AGGREGATE`，目标消失。双标签方案 `["t[1]", "t[4]"]` 正是只按成员
金额约简候选池时被漏掉的极小删除集。每个方案都经过聚合与删除搜索后的
**整条计划重算**，以完整目标行是否消失裁决。

一对一连接则不同：右表金额为 `5`、`-5` 和九条 `0` 的自连接共 11 个成员、
22 个来源标签（九个零成员的见证与非零成员互不相交、可独立删净，不构成
危险核心），候选池约简到 4 个非零标签，`minimal_deletion_sets` 仍是四个
单例 `["t[1]"]`、`["t[2]"]`、`["t[3]"]`、`["t[4]"]`（重算总和分别为
`-5`、`5`、`-5`、`5`），不会因候选池膨胀而误报预算耗尽；真正耗尽预算
（约简后候选池超过 20 行或重算超过 10 000 次）仍返回结构化 `422`。

#### 预算耗尽：结构化错误，不输出近似方案

目标组约简后仍超过 20 个有效行时（例如 20 个 `1` 加一个 `-1`，共 21 行
非零、`total=19`），服务返回 HTTP `422` 和规范化错误体，响应中没有
`counterfactual` 字段；用同一个幂等键重试返回逐字节相同的错误体：

```json
{"error":{"code":"COUNTERFACTUAL_SEARCH_BUDGET_EXCEEDED",
 "message":"The exact counterfactual search for this non-monotone (mixed positive/negative) sum could not finish within the configured budget; no approximate deletion sets are returned. Retry with a smaller target group or split the request.",
 "details":{"active_member_count":21,"dropped_inert_label_count":0,
            "full_pool_size":21,"group_member_count":21,
            "pool_limit":20,"reduced_pool_size":21}}}
```

### 幂等重放逐字节一致

```bash
curl -s ... -d @req.json > a.json
curl -s ... -d @req.json > b.json
cmp a.json b.json && echo "byte-identical"   # byte-identical
# 第二次响应头: X-Idempotent-Replay: true
```

---

## 4. 结构化错误

错误统一为 `4xx` + 规范化 JSON：

```json
{"error": {"code": "DUPLICATE_PRIMARY_KEY",
           "message": "Table 't': duplicate primary key [1]",
           "details": {"table": "t", "primary_key": [1]}}}
```

| code | 触发条件 |
| --- | --- |
| `MALFORMED_REQUEST` | JSON 非法、请求不符合 schema、行含未声明列、表重名等 |
| `MALFORMED_PLAN` | 节点类型未知、谓词结构非法、投影/聚合别名冲突等 |
| `UNKNOWN_TABLE` | 计划引用未提交的表 |
| `ILLEGAL_REFERENCE` | 计划/谓词/分组/聚合引用不存在的字段 |
| `DUPLICATE_PRIMARY_KEY` | 同一表快照内主键重复 |
| `NULL_PRIMARY_KEY` | 主键的任一分量为 null |
| `EMPTY_AGGREGATE` | 聚合输入为空（无法形成任何组；含标量聚合） |
| `TYPE_ERROR` | 声明类型不符、跨类型比较/连接、对非数值求 `sum`、对布尔值做有序比较 |
| `TARGET_NOT_FOUND` | `explain` 的 index 越界或 row 不匹配任何输出 |
| `COUNTERFACTUAL_SEARCH_BUDGET_EXCEEDED` | 混合正负 `sum` 的精确反事实搜索耗尽预算（约简后候选池超过 20 行，或整计划重算超过 10 000 次；HTTP 422，不返回近似方案） |
| `IDEMPOTENCY_KEY_CONFLICT` | 幂等键已绑定到另一个规范化请求（HTTP 409） |

---

## 5. 溯源语义说明

* 来源行标签为 `表名[主键值,...]`，复合主键按顺序拼接。符号表把标签稳定映射为
  `s1, s2, ...`（按表名、主键全序排序）。
* 见证集经过包含极小化：投影去列、自连接等造成的冗余派生会被剔除；
  同一标签集合只保留一份，输出按标签全序稳定排序。
* 选择、等值连接、投影对源行单调；聚合输出的见证把“组存在性”（至少一名
  成员存活）与每个聚合表达式的数值组件合取：`count` 覆盖所有参与计数的行，
  `sum`/`min`/`max` 枚举产生目标值的极小非空行组合。因此“令输出消失的最小
  删除集”等于该输出全部极小见证的 **极小命中集**（hypergraph transversal）：
  命中所有见证后，组键或某个聚合值必然改变。
* 例外是 **含正负混合值的 `sum`**（非单调求和）：删除不属于任何极小组合的
  行也可能改变总和，且删除更多行可能让目标值重新出现，命中集刻画不再成立。
  此时服务先对目标组做 **候选约简**（`sum` 的零值/NULL 行、`min/max` 的非
  极值行、`count(col)` 的 NULL 行不可能出现在任何极小删除集中，被移出候选
  池），再对剩余候选行按删除集大小递增枚举，**按删除后重算的完整目标行**
  判定，返回全部按包含关系极小的方案；枚举中只有已找到解的超集才被跳过，
  未命中的删除集不会剪枝其超集。约简依据 **来源标签与成员见证的依赖关系**：
  聚合输入可以是连接子计划，零值成员可能与非零成员共享来源标签（自连接）。
  若存在活跃标签的极小命中集 `A` 删掉全部非零成员后，存活的零值成员仍复现
  目标行、且 `A` 的每个真子集都不能让目标消失，则这些零值成员自身的标签会
  保留在候选池中（于是出现 `{共享标签, 零值标签}` 这样的双标签删除集，其重算
  使聚合输入为空并保留 `EMPTY_AGGREGATE`）；一对一（见证互不相交）连接的零
  成员仍被完整约简，候选池不膨胀。约简后候选池超过 20 行、或整计划重算超过
  10 000 次时返回结构化错误 `COUNTERFACTUAL_SEARCH_BUDGET_EXCEEDED`（HTTP
  422），绝不回退为命中集近似。无论走哪条路径，每个删除场景都会 **重算整条
  计划** 并显式给出 `target_present`，以重算结果为准；删除后聚合输入为空时，
  场景返回结构化的 `recompute_error`（如 `EMPTY_AGGREGATE`）。
