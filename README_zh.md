# docs-to-mcp-kb

把开源项目文档（Sphinx rst/md）构建为 **MCP 可访问的检索知识库** 的技能 + 参考实现：
云端 Embedding + BM25F 双通道融合、父子双层 Chunk、五个 MCP 工具、markdown 直读兜底。

> 沉淀自一次真实生产重构——建成三套在线知识库：**MuJoCo**（936 父块 / 2,808 子块）、
> **mjlab**（430 / 890）、**RSL-RL**（106 / 139）。仓库里每条坑都真实踩过，
> 每个默认值都有消融实验数据支撑。

## 工作原理

```
文档源(rst/md)
  │【构建期】子代理精读代表性文档 → 分类型 profile（人审冻结）
  ▼
提取扩展(Sphinx doctree-resolved 取 AST) → 结构化块树 JSON
  ▼
build_ir: 按 profile 分类型选父块 → 切子块 → 五字段(title/path/body/code/symbols)
  ├─→ MD 物化（确定性序列化，行号稳定）→ kb_read / kb_grep / kb_read_path
  ├─→ Whoosh BM25F 分类型索引（每类型独立参数）
  └─→ 云端 Embedding(text_type=document) → ChromaDB
  ▼
检索核心: 子块级融合(归一化加权) → norm_sum 聚合到父块 → 可选 Reranker 精排
  ▼
MCP Server(FastMCP): search / kb_read / kb_read_path / kb_grep / kb_structure
```

## 五个 MCP 工具

| 工具 | 职责 |
|---|---|
| `search_knowledge_base` | 主入口。子块级融合（keywords→BM25F，query→稠密）聚合到父节；命中自带节全文+子块目录；可选 qwen3-rerank 精排 |
| `kb_read_path` | 按 search 命中的 canonical path 直读某节（50KB 预算，`max_lines` 限流巨节） |
| `kb_read` | 物化 markdown 的行区间读取（opencode 风格 `N: content`，行长 2000 截断，50KB 预算，续读 footer） |
| `kb_grep` | MD 库的 Python 正则内容搜索（按文件分组输出，100 条上限，零命中自助诊断） |
| `kb_structure` | 语料地图 / 单文档章节大纲 |

## 仓库结构

```
docs-to-mcp-kb/
├── SKILL.md                  # 技能本体：9 阶段工作流 + 踩坑速查表
├── references/
│   ├── pitfalls.md           # 42 条踩坑（现象 → 根因 → 解法 → 预防）
│   ├── decisions.md          # 12 条 ADR（引擎/Embedding/融合选型及数据依据）
│   ├── templates.md          # 14 个方法模版（probe/freeze/build_ir/analyzer/...）
│   └── mcp_design.md         # MCP 工具层规范（描述规则、限制、降级矩阵）
├── README.md / README_zh.md
└── kbcore/                   # 参考实现（源仓库位于 kb_v2/src/kbcore/）
    ├── identifier_tokenizer.py   # 大小写保留的标识符 + 版本号分词器
    ├── build_ir.py               # profile 驱动的父块选择与子块切分
    ├── bm25_backend_whoosh.py    # 分类型 BM25F 索引
    ├── embed_client.py           # qwen3.7 embedding 客户端（批量/重试/断点）
    ├── rerank_client.py          # qwen3-rerank 客户端（token 预算分批）
    ├── retrieve.py               # 融合核心（norm_sum 聚合、降级矩阵）
    └── mcp_server.py             # FastMCP 五工具 server（KB_NAME 多知识库）
```

## 快速开始

1. **采用技能**：把 `docs-to-mcp-kb/` 复制进你的技能目录，按 `SKILL.md` 的
   9 阶段工作流处理你的文档语料。
2. **参考实现**：`PYTHONPATH` 指向 `kbcore/`（或直接 vendor），驱动管线：
   - Sphinx 构建产出提取 JSON 后：`build_ir.py --kb <name>`
   - `bm25_backend_whoosh.py build --kb <name>`
   - `vectorize.py --kb <name>`（云端向量化，断点续跑）
3. **接入宿主**：每个语料一条 `mcpServers` 配置（`KB_NAME` 选择
   路径/collection/profiles/examples）。

环境要求：Python ≥3.10，`sphinx`（提取）、`whoosh-reloaded`、`chromadb`、
`mcp>=1.2,<2`、`requests`；DashScope 风格的 embedding 端点
（`EMBEDDING_API_KEY`，模型 `qwen3.7-text-embedding`）与可选精排 `qwen3-rerank`。

## 为什么不是普通 RAG？

因为文档语料对朴素 RAG 极不友好：API 页、概念指南和 changelog 需要不同的
切块方式；符号需要子词匹配（从 `mjTEXROLE_USER` 要能检回 `mjtGeom`）；巨型
结构体让定长切分失效；纯语义搜索会漏掉精确标识符。本仓库的答案：分类型
profile、带字段权重和标识符分词的 BM25F、返回父节全文保证上下文，以及
42 条踩坑记录让你不必再吃一遍苦。从 [SKILL.md](SKILL.md) 开始。

## License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
