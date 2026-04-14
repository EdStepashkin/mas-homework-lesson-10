# Мультиагентна дослідницька система 🤖🔬

Мультиагентна AI-система, побудована на базі **LangChain** та **LangGraph**, яка координує трьох спеціалізованих суб-агентів за патерном **Plan → Research → Critique**.

Supervisor оркеструє ітеративний цикл дослідження: Planner декомпозує запит, Researcher виконує глибокий аналіз, а Critic верифікує результати і може повернути на доопрацювання. Збереження звіту захищене через **Human-in-the-Loop (HITL)** — користувач затверджує, редагує або відхиляє фінальний документ.

Система покрита автоматизованими тестами через **DeepEval** з golden dataset, компонентними тестами та e2e evaluation pipeline.

> Розширення мультиагентної системи (homework-lesson-8) + тестовий пакет (homework-lesson-10).

---

## 🏗 Архітектура

```
User (REPL)
  │
  ▼
Supervisor Agent
  │
  ├── 1. plan(request)       → Planner Agent      → structured ResearchPlan
  │
  ├── 2. research(plan)      → Research Agent      → findings (web + knowledge base)
  │
  ├── 3. critique(findings)  → Critic Agent        → structured CritiqueResult
  │       │
  │       ├── verdict: "APPROVE"  → step 4
  │       └── verdict: "REVISE"   → back to step 2 with feedback (max 2 rounds)
  │
  └── 4. save_report(...)    → HITL gated          → approve / edit / reject
```

### Суб-агенти

| Агент | Роль | Інструменти | Structured Output |
|-------|------|-------------|-------------------|
| **Planner** | Декомпозиція запиту у план дослідження | `web_search`, `knowledge_search` | `ResearchPlan` |
| **Researcher** | Глибоке дослідження за планом | `web_search`, `read_url`, `knowledge_search` | — |
| **Critic** | Верифікація якості (freshness, completeness, structure) | `web_search`, `read_url`, `knowledge_search` | `CritiqueResult` |
| **Supervisor** | Оркестрація циклу Plan→Research→Critique→Save | `plan`, `research`, `critique`, `save_report` | — |

---

## 🌟 Ключові можливості

- **Мультиагентна оркестрація**: Supervisor координує 3 спеціалізованих агенти через `create_agent` з `langchain.agents`
- **Structured Output**: Planner і Critic повертають валідовані Pydantic-моделі (`ResearchPlan`, `CritiqueResult`) через `response_format`
- **Ітеративне дослідження**: Critic може повернути Researcher на доопрацювання з конкретним зворотним зв'язком (evaluator-optimizer патерн)
- **HITL (Human-in-the-Loop)**: `HumanInTheLoopMiddleware` перехоплює `save_report` — користувач затверджує, редагує або відхиляє звіт
- **RAG з гібридним пошуком**: FAISS (семантичний) + BM25 (лексичний) + CrossEncoder реранкінг
- **Стрімування**: Реальний час виводу через `stream_mode=["updates", "messages"]`
- **Автоматизовані тести**: DeepEval із GEval метриками, ToolCorrectness та e2e evaluation

---

## 🛠 Технологічний стек

- **LLM**: Google Gemini (`gemini-3-flash-preview`) через `ChatGoogleGenerativeAI`
- **Агентний фреймворк**: `langchain.agents.create_agent` + `HumanInTheLoopMiddleware`
- **Персистентність**: `langgraph.checkpoint.memory.InMemorySaver`
- **RAG-пайплайн**: `FAISS`, `OpenAIEmbeddings` (`text-embedding-3-small`), `BM25Retriever`, `HuggingFaceCrossEncoder` (`BAAI/bge-reranker-base`), `EnsembleRetriever`
- **Structured Output**: Pydantic `BaseModel` через `response_format` параметр `create_agent`
- **Тестування**: `DeepEval` (GEval, AnswerRelevancy, ToolCorrectness), `pytest`
- **Інструменти**:
  - `knowledge_search` — пошук у локальній базі знань (RAG)
  - `web_search` — пошук в інтернеті (DuckDuckGo)
  - `read_url` — витягування тексту веб-сторінки (trafilatura)
  - `save_report` — збереження звіту (HITL-захищений)
- **Конфігурація**: Pydantic `BaseSettings` + `.env`

---

## 📁 Структура проєкту

```
homework-lesson-10/
├── main.py                  # REPL з HITL interrupt/resume loop
├── supervisor.py            # Supervisor Agent + HITL middleware
├── agents/
│   ├── __init__.py
│   ├── planner.py           # Planner Agent (response_format=ResearchPlan)
│   ├── research.py          # Research Agent
│   └── critic.py            # Critic Agent (response_format=CritiqueResult)
├── tests/
│   ├── golden_dataset.json  # 15 golden examples (happy path + edge + failure)
│   ├── test_planner.py      # GEval Plan Quality
│   ├── test_researcher.py   # GEval Groundedness
│   ├── test_critic.py       # GEval Critique Quality
│   ├── test_tools.py        # ToolCorrectnessMetric (3 cases)
│   └── test_e2e.py          # E2E evaluation на повному golden dataset
├── schemas.py               # Pydantic-моделі: ResearchPlan, CritiqueResult
├── tools.py                 # web_search, read_url, knowledge_search, save_report
├── retriever.py             # Hybrid search: FAISS + BM25 + CrossEncoder reranking
├── ingest.py                # Ingestion pipeline: PDF → chunks → FAISS index
├── config.py                # System prompts (4 агенти) + Settings
├── requirements.txt         # Залежності
├── data/                    # Вхідні PDF-документи для RAG
├── index/                   # Згенеровані індекси (не в Git)
├── example_output/          # Приклади згенерованих звітів
└── .env                     # API-ключі (не в Git)
```

---

## 🚀 Встановлення та запуск

### 1. Клонування репозиторію
```bash
git clone https://github.com/EdStepashkin/mas-homework-lesson-10.git
cd homework-lesson-10
```

### 2. Створення віртуального середовища
```bash
python -m venv venv
source venv/bin/activate
```

### 3. Встановлення залежностей
```bash
pip install -r requirements.txt
```

### 4. Налаштування змінних середовища (.env)
Створіть файл `.env` у кореневій директорії:
```env
GEMINI_API_KEY="AIzaSyYourApiKeyHere..."
OPENAI_API_KEY="sk-proj-YourOpenAiKey..."
```
*(`.env` додано до `.gitignore`).*

### 5. Індексація документів (Ingestion)
Помістіть PDF-документи у папку `data/` і запустіть:
```bash
python ingest.py
```

### 6. Запуск системи
```bash
python main.py
```

---

## 🧪 Тестування (DeepEval)

### Передумови

DeepEval використовує **`gpt-4o-mini` як judge-модель** для оцінки GEval метрик.  
Перед запуском тестів переконайтеся, що:

1. `OPENAI_API_KEY` задано у `.env` (або как змінна середовища)
2. `GEMINI_API_KEY` задано у `.env` (для запуску самих агентів)
3. Віртуальне середовище активоване:
   ```bash
   source venv/bin/activate
   ```

> **Важливо:** тести запускаються з кореневої директорії проєкту, оскільки агенти імпортуються відносно неї.

---

### Запуск тестів

#### Усі тести
```bash
deepeval test run tests/
```

#### Окремі файли з verbose-виводом
```bash
# Компонентні тести
deepeval test run tests/test_planner.py -v
deepeval test run tests/test_researcher.py -v
deepeval test run tests/test_critic.py -v

# Tool correctness
deepeval test run tests/test_tools.py -v

# E2E на повному golden dataset (найдовший — запускає реальний pipeline)
deepeval test run tests/test_e2e.py -v
```

#### Корисні флаги

| Флаг | Опис |
|------|------|
| `-v` | Verbose: показує score та причину для кожного тесту |
| `-c` | Використовує кеш попередніх оцінок (економить токени при перезапусках) |
| `-x` | Зупиняється після першого провалу |
| `-n 4` | Паралельний запуск у 4 процеси |
| `--display failing` | Показує тільки тести, що провалились |

```bash
# Приклад: verbose + кеш + тільки failing
deepeval test run tests/ -v -c --display failing
```

---

### Структура тестів

| Файл | Що тестує | Метрика | Поріг |
|------|-----------|---------|-------|
| `test_planner.py` | Якість плану (конкретні запити, джерела, формат) | `GEval("Plan Quality")` | 0.7 |
| `test_researcher.py` | Обґрунтованість відповіді на джерелах | `GEval("Groundedness")` | 0.7 |
| `test_critic.py` | Конкретність критики та actionability | `GEval("Critique Quality")` | 0.7 |
| `test_tools.py` | Правильність викликів інструментів | `ToolCorrectnessMetric` | 0.5 |
| `test_e2e.py` | Повний pipeline на golden dataset | `GEval("Correctness")` + `AnswerRelevancyMetric` | 0.6 / 0.7 |

---

### Golden Dataset

`tests/golden_dataset.json` містить **15 прикладів** у трьох категоріях:

| Категорія | Кількість | Опис |
|-----------|-----------|------|
| `happy_path` | 5 | Типові дослідницькі запити |
| `edge_cases` | 5 | Неоднозначні, мультимовні, логічно некоректні запити |
| `failure_cases` | 5 | Безглузді, заборонені, нездійсненні запити |

---

### Очікуваний вивід

```
$ deepeval test run tests/ -v

Running 5 test files...

tests/test_planner.py
  ✅ test_plan_quality       (Plan Quality: 0.85, threshold: 0.7)
  ✅ test_plan_has_queries   (Plan Quality: 0.90, threshold: 0.7)

tests/test_researcher.py
  ✅ test_research_grounded  (Groundedness: 0.78, threshold: 0.7)
  ❌ test_research_edge_case (Groundedness: 0.45, threshold: 0.7)

tests/test_critic.py
  ✅ test_critique_approve   (Critique Quality: 0.92, threshold: 0.7)
  ✅ test_critique_revise    (Critique Quality: 0.88, threshold: 0.7)

tests/test_tools.py
  ✅ test_planner_tools      (Tool Correctness: 1.0, threshold: 0.5)
  ✅ test_researcher_tools   (Tool Correctness: 1.0, threshold: 0.5)
  ✅ test_supervisor_save    (Tool Correctness: 1.0, threshold: 0.5)

tests/test_e2e.py
  ✅ test_golden_dataset [happy_path cases passed]
  ✅ test_golden_dataset [edge_cases passed]
  ❌ test_golden_dataset [some failure_cases failed — expected behavior]

======================================================
Overall: ~19/22 passed
```

> Деякі тести можуть fail — це нормально. Мета не 100% pass rate, а зафіксувати **baseline** і поступово покращувати систему.

---

## 💬 Приклад роботи

```
🔬 Multi-Agent Research System (type 'exit' to quit)
   Supervisor → Planner → Researcher → Critic → HITL
------------------------------------------------------------

You: Compare RAG approaches: naive, sentence-window, and parent-child. Write a report.

🔧 plan("Compare RAG approaches: naive, sentence-window, parent-child")
  📎 plan → ResearchPlan(goal="Compare three RAG retrieval strategies", ...)

🔧 research("Research these topics: 1) naive RAG approach 2) sentence-window ...")
  📎 research → [detailed findings with sources]

🔧 critique("Findings: ... [research results] ...")
  📎 critique → CritiqueResult(verdict="REVISE", gaps=["Outdated benchmarks", ...])

🔧 research("Find: 1) 2025-2026 benchmarks 2) parent-child details")
  📎 research → [updated findings]

🔧 critique("Updated findings: ...")
  📎 critique → CritiqueResult(verdict="APPROVE", strengths=["Up-to-date", ...])

🔧 save_report({"filename": "rag_comparison.md", "content": "# Comparison of RAG..."})

============================================================
⏸️  ACTION REQUIRES APPROVAL
============================================================
  Tool:  save_report
  File:  rag_comparison.md

👉 approve / edit / reject: approve

✅ Approved!

🤖 Supervisor: Звіт збережено у example_output/rag_comparison.md
```

---

*Оригінальне завдання доступне у файлі `ASSIGNMENT.md`.*