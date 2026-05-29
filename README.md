# Отчёт по лабораторной работе 3
X
**Среда:** LM Studio (локальная LLM)  
**Модель:** `openai/gpt-oss-20b`  
**Endpoint:** `http://localhost:1234/v1`  
**Инструменты:** Wikipedia REST API, OpenAlex API  

---

## 1. Постановка задачи

Реализована система построения краткого научно-аналитического обзора по заданной теме. Для каждой темы сравниваются три конфигурации:


| Конфигурация          | Описание                                          | Инструменты                                   |
| --------------------- | ------------------------------------------------- | --------------------------------------------- |
| **Baseline**          | Одношаговая генерация по Wikipedia-контексту      | Wikipedia + LLM                               |
| **Agent**             | Многошаговый пайплайн с состоянием и trace        | Wikipedia, OpenAlex, извлечение abstract, LLM |
| **Agent + evaluator** | Agent с последующей оценкой и регенерацией ответа | Agent + evaluator (LLM)                       |


**Структура итогового ответа:** определение темы, основные подходы, 3–5 ключевых работ, применения, ограничения, список источников.

---

## 2. Архитектура решения

### 2.1. Состояние агента (`AgentState`)

- `topic` — тема запроса  
- `history` — журнал шагов (action, payload, result_preview)  
- `sources` — сырые записи OpenAlex  
- `notes` — извлечённые аннотации (title, year, abstract)  
- `final_answer`, `status`, `stop_reason`

### 2.2. Логика agent-режима

1. `wikipedia` — общий справочный контекст
2. `openalex_search_page_N` — поиск публикаций (с пагинацией)
3. `extract_abstracts_page_N` — восстановление текста abstract из inverted index
4. `generate_final` — формирование структурированного обзора
5. (для eval) `regenerate_after_eval` — улучшение по рубрике evaluator

### 2.3. Evaluator

LLM оценивает ответ по 5 критериям (0–5): correctness, groundedness, completeness, coverage_of_required_fields, source_consistency. Интегральная метрика — **rubric_score** (среднее по 5 критериям).

---

## 3. Тестовый набор

Использованы 8 тем из методички:

- Agentic AI for customer support
- Graph RAG for enterprise knowledge systems
- LLM evaluation and process-aware metrics
- Tool-using language models in scientific search
- Retrieval-augmented generation in medicine
- Planning and reflection in LLM agents
- Human-in-the-loop AI systems
- Knowledge graphs for procedural reasoning

---

## 4. Результаты основных экспериментов

### 4.1. Сводная таблица (средние по режимам)


| Режим    | Correctness | Groundedness | Completeness | Coverage | Source consistency | Rubric | Шаги | Latency, с |
| -------- | ----------- | ------------ | ------------ | -------- | ------------------ | ------ | ---- | ---------- |
| baseline | 3.88        | 2.75         | 3.00         | 4.25     | 2.88               | 3.35   | 1.0  | 23.65      |
| agent    | 4.25        | 3.62         | 3.50         | 3.88     | 3.62               | 3.77   | 6.0  | 45.05      |
| eval     | 4.12        | 3.25         | 3.25         | 3.62     | 3.25               | 3.50   | 7.0  | 104.82     |


### 4.2. Детальные результаты по темам


| Тема                                      | Режим    | Rubric | Correctness | Groundedness | Completeness | Шаги | Latency, с |
| ----------------------------------------- | -------- | ------ | ----------- | ------------ | ------------ | ---- | ---------- |
| Agentic AI for customer support…          | agent    | 3.60   | 4           | 3            | 4            | 6    | 34.17      |
| Agentic AI for customer support…          | baseline | 4.80   | 5           | 5            | 4            | 1    | 17.81      |
| Agentic AI for customer support…          | eval     | 3.60   | 4           | 3            | 4            | 7    | 91.30      |
| Graph RAG for enterprise knowledge syste… | agent    | 3.40   | 4           | 3            | 3            | 6    | 43.94      |
| Graph RAG for enterprise knowledge syste… | baseline | 4.20   | 4           | 3            | 4            | 1    | 21.81      |
| Graph RAG for enterprise knowledge syste… | eval     | 3.40   | 4           | 3            | 3            | 7    | 95.36      |
| Human-in-the-loop AI systems…             | agent    | 2.20   | 3           | 2            | 3            | 6    | 49.16      |
| Human-in-the-loop AI systems…             | baseline | 3.20   | 4           | 2            | 3            | 1    | 25.10      |
| Human-in-the-loop AI systems…             | eval     | 3.60   | 4           | 3            | 4            | 7    | 112.28     |
| Knowledge graphs for procedural reasonin… | agent    | 3.60   | 4           | 3            | 3            | 6    | 45.62      |
| Knowledge graphs for procedural reasonin… | baseline | 3.20   | 4           | 3            | 2            | 1    | 20.48      |
| Knowledge graphs for procedural reasonin… | eval     | 2.80   | 4           | 3            | 3            | 7    | 100.21     |
| LLM evaluation and process-aware metrics… | agent    | 4.80   | 5           | 5            | 4            | 6    | 53.31      |
| LLM evaluation and process-aware metrics… | baseline | 3.80   | 4           | 3            | 3            | 1    | 26.52      |
| LLM evaluation and process-aware metrics… | eval     | 4.80   | 5           | 5            | 4            | 7    | 118.02     |
| Planning and reflection in LLM agents…    | agent    | 4.80   | 5           | 5            | 4            | 6    | 48.10      |
| Planning and reflection in LLM agents…    | baseline | 2.20   | 3           | 2            | 2            | 1    | 28.09      |
| Planning and reflection in LLM agents…    | eval     | 3.20   | 4           | 3            | 2            | 7    | 115.92     |
| Retrieval-augmented generation in medici… | agent    | 4.20   | 5           | 4            | 4            | 6    | 40.74      |
| Retrieval-augmented generation in medici… | baseline | 2.60   | 3           | 2            | 3            | 1    | 29.72      |
| Retrieval-augmented generation in medici… | eval     | 3.20   | 4           | 3            | 3            | 7    | 96.10      |
| Tool-using language models in scientific… | agent    | 3.60   | 4           | 4            | 3            | 6    | 45.36      |
| Tool-using language models in scientific… | baseline | 2.80   | 4           | 2            | 3            | 1    | 19.64      |
| Tool-using language models in scientific… | eval     | 3.40   | 4           | 3            | 3            | 7    | 109.33     |


---

## 5. Дополнительные эксперименты

### 5.1. Влияние числа источников (top-k)


| Режим      | Rubric | Latency, с | Шаги |
| ---------- | ------ | ---------- | ---- |
| agent_top3 | 3.40   | 44.67      | 8.0  |
| agent_top5 | 4.20   | 42.15      | 8.0  |
| agent_top8 | 2.40   | 43.66      | 6.0  |


### 5.2. Влияние max_steps


| Режим        | Rubric | Latency, с | Шаги |
| ------------ | ------ | ---------- | ---- |
| agent_steps4 | 3.90   | 37.58      | 4.0  |
| agent_steps6 | 4.30   | 47.60      | 6.0  |
| agent_steps8 | 3.00   | 47.18      | 8.0  |


---

## 6. Визуализация

Сравнение режимов

---

## 7. Разбор траекторий (trace)

**Agentic AI for customer support**

- Статус: finished / final_answer_generated
- Шаги: wikipedia → openalex_search_page_1 → extract_abstracts_page_1 → openalex_search_page_2 → extract_abstracts_page_2 → generate_final
- Источников с аннотациями: 7

**Graph RAG for enterprise knowledge systems**

- Статус: finished / final_answer_generated
- Шаги: wikipedia → openalex_search_page_1 → extract_abstracts_page_1 → openalex_search_page_2 → extract_abstracts_page_2 → generate_final
- Источников с аннотациями: 9

**LLM evaluation and process-aware metrics**

- Статус: finished / final_answer_generated
- Шаги: wikipedia → openalex_search_page_1 → extract_abstracts_page_1 → openalex_search_page_2 → extract_abstracts_page_2 → generate_final
- Источников с аннотациями: 10

**Типичные типы ошибок:**

- **Слабый поиск Wikipedia** — для узких тем summary может быть нерелевантным (fallback через search API частично помогает).
- **Пустые abstract в OpenAlex** — не все статьи содержат abstract_inverted_index; agent может остановиться с `no_sources`.
- **Потеря полноты в baseline** — модель генерирует «ключевые работы» без реальных публикаций.
- **Рост latency в eval** — дополнительные вызовы LLM (оценка + регенерация) увеличивают время в 1.5–2 раза относительно agent.

---

## 8. Выводы

1. **Лучший rubric_score** в среднем показал режим `**agent`** (3.77).
2. **Самый быстрый режим** — `**baseline`** (23.65 с в среднем).
3. **Baseline** (rubric 3.35) — 1 шаг, ~23.65 с; groundedness 2.75 — модель часто «додумывает» ключевые работы без OpenAlex.
4. **Agent** (rubric 3.77) — 6 шагов, ~45.05 с; groundedness 3.62, completeness 3.50 — лучший баланс качества и процесса.
5. **Agent + evaluator** (rubric 3.50) — 7 шагов, ~~104.82 с (~~2.3× медленнее agent); регенерация нестабильна: помогает отдельным темам (Human-in-the-loop: 2.2→3.6), но в среднем rubric ниже agent.
6. **Доп. эксперименты:** оптимально `per_page=5`, `max_steps=6`; top-8 и steps=8 ухудшают coverage из-за шума/лишних шагов.
7. **Границы применимости:** модель `openai/gpt-oss-20b` в LM Studio требует усечения контекста; при превышении лимита — ошибка 400 (исправлено truncation в коде).

---

## 9. Воспроизводимость

```bash
# 1. Запустите LM Studio и загрузите модель
# 2. Включите Local Server (порт 1234)
pip install pandas matplotlib requests
python run_experiments_lmstudio.py
python generate_report.py
```

Файлы результатов: `results.csv`, `results_extra.csv`, `summary.csv`, `comparison_plots.png`, `traces/*.json`.

---

## 10. Приложение: параметры LLM

- **temperature (генерация):** 0.3  
- **temperature (evaluator):** 0.0  
- **max_tokens:** 4000  
- **system prompt:** научный ассистент, ответ на языке запроса, без галлюцинаций источников
