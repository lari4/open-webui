# Документация схем работы агента и пайплайнов Open WebUI

Этот документ содержит подробное описание всех пайплайнов и схем работы AI агента в Open WebUI, включая описание потока данных, используемых промтов и обработки результатов.

## Оглавление

1. [Пайплайн генерации заголовка чата](#пайплайн-генерации-заголовка-чата)
2. [Пайплайн генерации тегов](#пайплайн-генерации-тегов)
3. [Пайплайн генерации follow-up вопросов](#пайплайн-генерации-follow-up-вопросов)
4. [Пайплайн генерации поисковых запросов](#пайплайн-генерации-поисковых-запросов)
5. [Пайплайн RAG (Retrieval-Augmented Generation)](#пайплайн-rag-retrieval-augmented-generation)
6. [Пайплайн веб-поиска](#пайплайн-веб-поиска)
7. [Пайплайн вызова инструментов](#пайплайн-вызова-инструментов)
8. [Пайплайн Code Interpreter](#пайплайн-code-interpreter)
9. [Пайплайн автодополнения](#пайплайн-автодополнения)
10. [Пайплайн MOA (Mixture of Agents)](#пайплайн-moa-mixture-of-agents)

---

## Пайплайн генерации заголовка чата

**Назначение:** Автоматически генерирует краткий заголовок (3-5 слов) с эмодзи для нового чата на основе первых сообщений.

**Файлы:**
- `/backend/open_webui/routers/tasks.py` - endpoint `generate_title()`
- `/backend/open_webui/utils/task.py` - функция `title_generation_template()`
- `/backend/open_webui/config.py` - `DEFAULT_TITLE_GENERATION_PROMPT_TEMPLATE`

**Конфигурация:**
- `ENABLE_TITLE_GENERATION` - включить/выключить функцию
- `TITLE_GENERATION_PROMPT_TEMPLATE` - кастомный шаблон промта
- `TASK_MODEL` / `TASK_MODEL_EXTERNAL` - модель для задач

### Схема работы

```
┌──────────────┐
│   Frontend   │
│  (New Chat)  │
└──────┬───────┘
       │
       │ POST /tasks/title/completions
       │ Body: { chat_id, user_message, messages }
       ▼
┌──────────────────────────────────────┐
│  generate_title() endpoint           │
│  /backend/open_webui/routers/tasks.py│
└──────┬───────────────────────────────┘
       │
       │ 1. Извлекает последнее сообщение пользователя
       │ 2. Получает историю чата (последние 2 сообщения)
       │
       ▼
┌──────────────────────────────────────┐
│  title_generation_template()         │
│  /backend/open_webui/utils/task.py   │
└──────┬───────────────────────────────┘
       │
       │ Заменяет переменные в промте:
       │ - {{MESSAGES:END:2}} → последние 2 сообщения
       │ - {{USER_NAME}} → имя пользователя
       │ - {{CURRENT_DATE}} → текущая дата
       │
       ▼
┌──────────────────────────────────────┐
│  Сформированный промт                │
│                                      │
│  ### Task:                           │
│  Generate a concise, 3-5 word title  │
│  with an emoji...                    │
│                                      │
│  ### Chat History:                   │
│  User: How does photosynthesis work? │
│  Assistant: Photosynthesis is...     │
└──────┬───────────────────────────────┘
       │
       │ Отправляется в generate_chat_completion()
       │
       ▼
┌──────────────────────────────────────┐
│  TASK_MODEL (легкая модель)          │
│  Например: llama3.2:3b               │
└──────┬───────────────────────────────┘
       │
       │ Генерирует JSON ответ
       │
       ▼
┌──────────────────────────────────────┐
│  Результат (JSON)                    │
│  {                                   │
│    "title": "🌱 Photosynthesis       │
│              Explained"              │
│  }                                   │
└──────┬───────────────────────────────┘
       │
       │ Парсинг JSON
       │
       ▼
┌──────────────────────────────────────┐
│  Обновление чата в БД                │
│  chat.title = "🌱 Photosynthesis..."│
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────┐
│   Frontend   │
│  (Отображает │
│   заголовок) │
└──────────────┘
```

### Поток данных

**Вход:**
- `chat_id` - ID чата
- `user_message` - последнее сообщение пользователя
- `messages` - история сообщений

**Промежуточные данные:**
- Последние 2 сообщения из истории
- Отформатированный промт с заменёнными переменными

**Выход:**
- JSON объект с полем `title`
- Пример: `{ "title": "🌱 Photosynthesis Explained" }`

**Используемый промт:** `DEFAULT_TITLE_GENERATION_PROMPT_TEMPLATE` (см. PROMPTS_DOCUMENTATION.md)

---

## Пайплайн генерации тегов

**Назначение:** Автоматически категоризирует чат, присваивая 1-3 широких тега и 1-3 специфических подтемы для организации и поиска.

**Файлы:**
- `/backend/open_webui/routers/tasks.py` - endpoint `generate_chat_tags()`
- `/backend/open_webui/utils/task.py` - функция `tags_generation_template()`
- `/backend/open_webui/config.py` - `DEFAULT_TAGS_GENERATION_PROMPT_TEMPLATE`

**Конфигурация:**
- `ENABLE_TAGS_GENERATION` - включить/выключить функцию
- `TAGS_GENERATION_PROMPT_TEMPLATE` - кастомный шаблон промта

### Схема работы

```
┌──────────────┐
│   Frontend   │
│ (Chat ended  │
│  or request) │
└──────┬───────┘
       │
       │ POST /tasks/tags/completions
       │ Body: { chat_id, user_message, messages }
       ▼
┌──────────────────────────────────────┐
│  generate_chat_tags() endpoint       │
│  /backend/open_webui/routers/tasks.py│
└──────┬───────────────────────────────┘
       │
       │ 1. Получает полную историю чата
       │ 2. Использует последние 6 сообщений для контекста
       │
       ▼
┌──────────────────────────────────────┐
│  tags_generation_template()          │
│  /backend/open_webui/utils/task.py   │
└──────┬───────────────────────────────┘
       │
       │ Заменяет: {{MESSAGES:END:6}}
       │
       ▼
┌──────────────────────────────────────┐
│  Сформированный промт                │
│                                      │
│  ### Task:                           │
│  Generate 1-3 broad tags...          │
│                                      │
│  ### Chat History:                   │
│  [Последние 6 сообщений о Python]    │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  TASK_MODEL                          │
└──────┬───────────────────────────────┘
       │
       │ Анализирует темы разговора
       │
       ▼
┌──────────────────────────────────────┐
│  Результат (JSON)                    │
│  {                                   │
│    "tags": [                         │
│      "Technology",                   │
│      "Programming",                  │
│      "Python",                       │
│      "Web Development"               │
│    ]                                 │
│  }                                   │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  Сохранение тегов в БД               │
│  chat.tags = ["Technology", ...]     │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────┐
│   Frontend   │
│ (Отображает  │
│    теги)     │
└──────────────┘
```

### Поток данных

**Вход:**
- `chat_id` - ID чата
- `user_message` - последнее сообщение
- `messages` - полная история чата

**Промежуточные данные:**
- Последние 6 сообщений для контекста
- Промт с категориями (Science, Technology, Philosophy, Arts, etc.)

**Выход:**
- JSON массив тегов
- Пример: `{ "tags": ["Technology", "Programming", "Python"] }`

**Используемый промт:** `DEFAULT_TAGS_GENERATION_PROMPT_TEMPLATE`

---

## Пайплайн генерации follow-up вопросов

**Назначение:** Предлагает 3-5 релевантных вопросов, которые пользователь может задать для продолжения беседы.

**Файлы:**
- `/backend/open_webui/routers/tasks.py` - endpoint `generate_follow_ups()`
- `/backend/open_webui/utils/task.py` - функция `follow_up_generation_template()`
- `/backend/open_webui/config.py` - `DEFAULT_FOLLOW_UP_GENERATION_PROMPT_TEMPLATE`

**Конфигурация:**
- `ENABLE_FOLLOW_UP_GENERATION` - включить/выключить функцию
- `FOLLOW_UP_GENERATION_PROMPT_TEMPLATE` - кастомный шаблон промта

### Схема работы

```
┌──────────────┐
│   Frontend   │
│  (After AI   │
│   response)  │
└──────┬───────┘
       │
       │ POST /tasks/follow_up/completions
       │ Body: { messages }
       ▼
┌──────────────────────────────────────┐
│  generate_follow_ups() endpoint      │
│  /backend/open_webui/routers/tasks.py│
└──────┬───────────────────────────────┘
       │
       │ Получает последние 6 сообщений
       │
       ▼
┌──────────────────────────────────────┐
│  follow_up_generation_template()     │
│  /backend/open_webui/utils/task.py   │
└──────┬───────────────────────────────┘
       │
       │ Заменяет: {{MESSAGES:END:6}}
       │
       ▼
┌──────────────────────────────────────┐
│  Сформированный промт                │
│                                      │
│  ### Task:                           │
│  Suggest 3-5 relevant follow-up      │
│  questions...                        │
│                                      │
│  ### Chat History:                   │
│  User: Explain recursion             │
│  AI: Recursion is when a function... │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  TASK_MODEL                          │
└──────┬───────────────────────────────┘
       │
       │ Генерирует релевантные вопросы
       │
       ▼
┌──────────────────────────────────────┐
│  Результат (JSON)                    │
│  {                                   │
│    "follow_ups": [                   │
│      "Can you show me an example     │
│       of recursion in Python?",      │
│      "What's the difference between  │
│       recursion and iteration?",     │
│      "How do I avoid stack overflow  │
│       with recursion?",              │
│      "When should I use recursion?", │
│      "What is tail recursion?"       │
│    ]                                 │
│  }                                   │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────┐
│   Frontend   │
│ (Показывает  │
│  кнопки с    │
│  вопросами)  │
└──────────────┘
```

### Поток данных

**Вход:**
- `messages` - история чата (последние 6 сообщений)

**Промежуточные данные:**
- Контекст разговора
- Последние темы обсуждения

**Выход:**
- JSON массив вопросов (3-5 штук)
- Вопросы от лица пользователя, направленные ассистенту

**Используемый промт:** `DEFAULT_FOLLOW_UP_GENERATION_PROMPT_TEMPLATE`

---

## Пайплайн RAG (Retrieval-Augmented Generation)

**Назначение:** Предоставляет AI доступ к документам и базам знаний, извлекая релевантную информацию и добавляя её в контекст разговора с цитированием источников.

**Файлы:**
- `/backend/open_webui/routers/retrieval.py` - загрузка документов, управление векторной БД
- `/backend/open_webui/utils/middleware.py` - `chat_completion_with_files_handler()` (строка ~915-1500)
- `/backend/open_webui/retrieval/` - векторная БД, embeddings, reranking
- `/backend/open_webui/routers/knowledge.py` - управление базами знаний
- `/backend/open_webui/models/knowledge.py` - модель данных знаний

**Конфигурация:**
- `RAG_EMBEDDING_MODEL` - модель для embeddings
- `RAG_RERANKING_MODEL` - модель для reranking (опционально)
- `RAG_TEMPLATE` - шаблон для инжекции контекста
- `RAG_CHUNK_SIZE` / `RAG_CHUNK_OVERLAP` - размер и перекрытие чанков
- `RAG_TOP_K` - количество документов для извлечения
- `ENABLE_RAG_HYBRID_SEARCH` - гибридный поиск (BM25 + semantic)

### Схема работы (Часть 1: Загрузка документов)

```
┌──────────────┐
│   Frontend   │
│ Upload File  │
└──────┬───────┘
       │
       │ POST /files/
       │ FormData: { file, collection_id }
       ▼
┌─────────────────────────────────────────────┐
│  process_file() endpoint                    │
│  /backend/open_webui/routers/retrieval.py   │
└──────┬──────────────────────────────────────┘
       │
       │ 1. Определяет тип файла
       │    (PDF, DOCX, XLSX, CSV, TXT, MD, JSON, etc.)
       │
       ▼
┌─────────────────────────────────────────────┐
│  Парсинг документа (LangChain loaders)      │
│  - PDFLoader для PDF                        │
│  - UnstructuredWordDocumentLoader для DOCX  │
│  - CSVLoader для CSV                        │
│  - и т.д.                                   │
└──────┬──────────────────────────────────────┘
       │
       │ Извлеченный текст
       │
       ▼
┌─────────────────────────────────────────────┐
│  Text Splitting (чанкирование)              │
│  - RecursiveCharacterTextSplitter           │
│  - TokenTextSplitter                        │
│  - MarkdownTextSplitter                     │
│                                             │
│  Параметры:                                 │
│  - chunk_size = RAG_CHUNK_SIZE (400)        │
│  - chunk_overlap = RAG_CHUNK_OVERLAP (100)  │
└──────┬──────────────────────────────────────┘
       │
       │ Массив текстовых чанков
       │ [chunk1, chunk2, chunk3, ...]
       │
       ▼
┌─────────────────────────────────────────────┐
│  Embeddings Generation                      │
│  RAG_EMBEDDING_MODEL                        │
│  (sentence-transformers/all-MiniLM-L6-v2)   │
└──────┬──────────────────────────────────────┘
       │
       │ Векторы embeddings
       │ [[0.123, -0.456, ...], ...]
       │
       ▼
┌─────────────────────────────────────────────┐
│  Vector Database Storage                    │
│  - Chroma (default)                         │
│  - Qdrant, Milvus, PGVector, Pinecone, etc. │
│                                             │
│  Metadata для каждого чанка:                │
│  - file_id                                  │
│  - collection_id                            │
│  - source (file name)                       │
│  - page_number                              │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│  Document    │
│   Indexed    │
└──────────────┘
```

### Схема работы (Часть 2: Поиск и инжекция контекста)

```
┌──────────────┐
│   Frontend   │
│   User asks  │
│   question   │
└──────┬───────┘
       │
       │ POST /chat/completions
       │ Body: { messages, files: [file_ids] }
       ▼
┌─────────────────────────────────────────────┐
│  chat_completion_with_files_handler()       │
│  /backend/open_webui/utils/middleware.py    │
└──────┬──────────────────────────────────────┘
       │
       │ 1. Проверяет наличие файлов в запросе
       │ 2. Если files пусты, использует Knowledge Base
       │
       ▼
┌─────────────────────────────────────────────┐
│  Генерация поисковых запросов               │
│                                             │
│  generate_queries() →                       │
│  QUERY_GENERATION_PROMPT_TEMPLATE           │
│                                             │
│  Вход: последнее сообщение пользователя    │
│  Выход: ["query1", "query2", "query3"]      │
└──────┬──────────────────────────────────────┘
       │
       │ Поисковые запросы
       │
       ▼
┌─────────────────────────────────────────────┐
│  Vector Search                              │
│                                             │
│  Для каждого запроса:                       │
│  1. Генерирует embedding запроса            │
│  2. Ищет похожие векторы в БД               │
│     (cosine similarity)                     │
│                                             │
│  Если ENABLE_RAG_HYBRID_SEARCH:             │
│  - Semantic search (embeddings)             │
│  - BM25 keyword search                      │
│  - Объединение результатов                  │
└──────┬──────────────────────────────────────┘
       │
       │ Найденные документы (chunks)
       │
       ▼
┌─────────────────────────────────────────────┐
│  Reranking (опционально)                    │
│                                             │
│  Если RAG_RERANKING_MODEL установлена:      │
│  - CrossEncoder переранжирует результаты    │
│  - Учитывает контекст запроса               │
│  - Сортирует по релевантности               │
└──────┬──────────────────────────────────────┘
       │
       │ Top-K документов (RAG_TOP_K)
       │
       ▼
┌─────────────────────────────────────────────┐
│  Форматирование контекста                   │
│  RAG_TEMPLATE                               │
│                                             │
│  Переменные:                                │
│  {{CONTEXT}} = найденные документы с ID     │
│  {{QUERY}} = вопрос пользователя            │
│                                             │
│  Пример контекста:                          │
│  <source id="1">                            │
│  Python is a programming language...        │
│  </source>                                  │
│  <source id="2">                            │
│  Django is a web framework...               │
│  </source>                                  │
└──────┬──────────────────────────────────────┘
       │
       │ Отформатированный промт с контекстом
       │
       ▼
┌─────────────────────────────────────────────┐
│  Инжекция в системное сообщение             │
│                                             │
│  add_or_update_system_message()             │
│                                             │
│  Добавляет RAG промт в начало messages:     │
│  [                                          │
│    {role: "system", content: RAG_PROMPT},   │
│    {role: "user", content: "question"},     │
│    ...                                      │
│  ]                                          │
└──────┬──────────────────────────────────────┘
       │
       │ Обогащенные сообщения
       │
       ▼
┌─────────────────────────────────────────────┐
│  LLM (OpenAI, Ollama, etc.)                 │
│                                             │
│  Модель получает:                           │
│  - Контекст из документов                   │
│  - Инструкции по цитированию [id]           │
│  - Вопрос пользователя                      │
└──────┬──────────────────────────────────────┘
       │
       │ Ответ с цитатами
       │
       ▼
┌─────────────────────────────────────────────┐
│  Ответ с источниками                        │
│                                             │
│  "Python is a high-level programming        │
│  language [1]. Django is a popular web      │
│  framework for Python [2]."                 │
│                                             │
│  Источники:                                 │
│  [1] intro_to_python.pdf                    │
│  [2] django_tutorial.md                     │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│   Frontend   │
│  (Показывает │
│   ответ и    │
│  источники)  │
└──────────────┘
```

### Поток данных

**Часть 1 - Индексация:**

**Вход:**
- Файл (PDF, DOCX, TXT, etc.)
- collection_id (опционально)

**Промежуточные данные:**
- Извлеченный текст
- Текстовые чанки (chunks)
- Векторы embeddings

**Выход:**
- Индексированный документ в векторной БД
- file_id для последующего использования

**Часть 2 - Retrieval:**

**Вход:**
- Вопрос пользователя
- file_ids или knowledge_base_id

**Промежуточные данные:**
- Поисковые запросы (1-3 штуки)
- Найденные документы/чанки
- Переранжированные результаты

**Выход:**
- Контекст для LLM в формате RAG_TEMPLATE
- Ответ с инлайн-цитатами [id]
- Список источников

**Используемые промты:**
- `DEFAULT_QUERY_GENERATION_PROMPT_TEMPLATE` - для генерации запросов
- `DEFAULT_RAG_TEMPLATE` - для форматирования контекста

---

## Пайплайн вызова инструментов

**Назначение:** Позволяет AI использовать внешние инструменты и функции (Tools/Function Calling) для выполнения действий: API вызовы, поиск в интернете, вычисления и т.д.

**Файлы:**
- `/backend/open_webui/utils/middleware.py` - `chat_completion_tools_handler()` (строка ~285)
- `/backend/open_webui/utils/tools.py` - загрузка и управление инструментами
- `/backend/open_webui/functions.py` - function pipes и manifolds
- `/backend/open_webui/routers/tools.py` - управление инструментами
- `/backend/open_webui/utils/mcp/client.py` - поддержка MCP (Model Context Protocol)

**Типы инструментов:**
1. **Local Tools** - Python функции в Open WebUI
2. **OpenAPI Tools** - Внешние API через OpenAPI specs
3. **MCP Tools** - Инструменты через Model Context Protocol
4. **Direct Tools** - Кастомные backend реализации

**Конфигурация:**
- `TOOLS_FUNCTION_CALLING_PROMPT_TEMPLATE` - промт для выбора инструментов
- Tool definitions в JSON формате (OpenAI Function Calling)

### Схема работы

```
┌──────────────┐
│   Frontend   │
│  User message│
└──────┬───────┘
       │
       │ POST /chat/completions
       │ Body: { messages, tools_enabled: true }
       ▼
┌─────────────────────────────────────────────┐
│  chat_completion_tools_handler()            │
│  /backend/open_webui/utils/middleware.py    │
└──────┬──────────────────────────────────────┘
       │
       │ 1. Проверяет, включены ли инструменты
       │ 2. Собирает доступные инструменты
       │
       ▼
┌─────────────────────────────────────────────┐
│  Tool Discovery & Schema Generation         │
│                                             │
│  get_tool_specs()                           │
│                                             │
│  Для каждого инструмента:                   │
│  - Извлекает docstring                      │
│  - Создает JSON schema параметров           │
│  - Формирует OpenAI function spec           │
│                                             │
│  Пример:                                    │
│  {                                          │
│    "name": "web_search",                    │
│    "description": "Search the web",         │
│    "parameters": {                          │
│      "type": "object",                      │
│      "properties": {                        │
│        "query": {"type": "string"}          │
│      },                                     │
│      "required": ["query"]                  │
│    }                                        │
│  }                                          │
└──────┬──────────────────────────────────────┘
       │
       │ Список tool specs
       │
       ▼
┌─────────────────────────────────────────────┐
│  Генерация промта для выбора инструментов   │
│                                             │
│  TOOLS_FUNCTION_CALLING_PROMPT_TEMPLATE     │
│                                             │
│  Переменные:                                │
│  {{TOOLS}} = JSON список всех инструментов  │
│                                             │
│  Добавляет к сообщениям пользователя        │
└──────┬──────────────────────────────────────┘
       │
       │ Сообщения + промт с инструментами
       │
       ▼
┌─────────────────────────────────────────────┐
│  LLM (с function calling)                   │
│                                             │
│  Модель анализирует запрос и решает:        │
│  - Нужны ли инструменты?                    │
│  - Какие инструменты вызвать?               │
│  - Какие параметры передать?                │
└──────┬──────────────────────────────────────┘
       │
       │ Ответ LLM
       │
       ▼
       ┌─────────────────┐
       │ Инструменты     │
       │ не нужны?       │
       └────┬───────┬────┘
            │ НЕТ   │ ДА
            │       │
            │       └──────────┐
            │                  │
            │                  ▼
            │        ┌─────────────────┐
            │        │  Обычный ответ  │
            │        │  без вызова     │
            │        │  инструментов   │
            │        └─────────────────┘
            │
            ▼
┌─────────────────────────────────────────────┐
│  Парсинг tool_calls из ответа LLM           │
│                                             │
│  JSON формат:                               │
│  {                                          │
│    "tool_calls": [                          │
│      {                                      │
│        "name": "web_search",                │
│        "parameters": {                      │
│          "query": "latest news AI"          │
│        }                                    │
│      }                                      │
│    ]                                        │
│  }                                          │
└──────┬──────────────────────────────────────┘
       │
       │ Список вызовов инструментов
       │
       ▼
┌─────────────────────────────────────────────┐
│  Выполнение инструментов                    │
│                                             │
│  Для каждого tool_call:                     │
│  1. Находит функцию по имени                │
│  2. Валидирует параметры                    │
│  3. Выполняет функцию                       │
│  4. Собирает результаты                     │
│                                             │
│  Параллельное выполнение если возможно      │
└──────┬──────────────────────────────────────┘
       │
       │ Результаты выполнения
       │
       ▼
┌─────────────────────────────────────────────┐
│  Форматирование результатов                 │
│                                             │
│  Создает tool_response сообщения:           │
│  [                                          │
│    {                                        │
│      "role": "tool",                        │
│      "name": "web_search",                  │
│      "content": "AI News: ..."              │
│    }                                        │
│  ]                                          │
└──────┬──────────────────────────────────────┘
       │
       │ Обновленная история сообщений
       │
       ▼
┌─────────────────────────────────────────────┐
│  Повторный вызов LLM                        │
│                                             │
│  История теперь включает:                   │
│  - Оригинальный запрос пользователя         │
│  - Решение LLM вызвать инструменты          │
│  - Результаты выполнения инструментов       │
│                                             │
│  LLM формулирует финальный ответ            │
└──────┬──────────────────────────────────────┘
       │
       │ Финальный ответ с учетом tool results
       │
       ▼
┌──────────────┐
│   Frontend   │
│ (Показывает  │
│  итоговый    │
│   ответ)     │
└──────────────┘
```

### Поток данных

**Вход:**
- Сообщения пользователя
- Флаг включения инструментов
- Доступные инструменты

**Промежуточные данные:**
- JSON specs инструментов
- Решение LLM о вызове инструментов
- Параметры для вызова
- Результаты выполнения

**Выход:**
- Ответ, обогащенный результатами инструментов
- История с tool_calls и tool_responses

**Используемый промт:** `DEFAULT_TOOLS_FUNCTION_CALLING_PROMPT_TEMPLATE`

---

## Пайплайн Code Interpreter

**Назначение:** Позволяет AI писать и выполнять Python код прямо в браузере пользователя для вычислений, анализа данных, визуализации и других задач.

**Файлы:**
- `/backend/open_webui/utils/code_interpreter.py` - интеграция с Jupyter
- `/backend/open_webui/config.py` - `DEFAULT_CODE_INTERPRETER_PROMPT`
- `/backend/open_webui/utils/middleware.py` - обработка code_interpreter тегов

**Конфигурация:**
- `CODE_INTERPRETER_BLOCKED_MODULES` - запрещенные модули (безопасность)
- System prompt добавляется автоматически при включении

### Схема работы

```
┌──────────────┐
│   Frontend   │
│  User asks to│
│  run Python  │
└──────┬───────┘
       │
       │ POST /chat/completions
       │ Body: { messages, code_interpreter: true }
       ▼
┌─────────────────────────────────────────────┐
│  Инжекция Code Interpreter промта           │
│                                             │
│  add_or_update_system_message()             │
│                                             │
│  Добавляет DEFAULT_CODE_INTERPRETER_PROMPT  │
│  в начало сообщений                         │
│                                             │
│  Промт объясняет:                           │
│  - Как использовать XML теги                │
│    <code_interpreter type="code"            │
│     lang="python">                          │
│  - Какие библиотеки доступны                │
│  - Как выводить результаты                  │
│  - НЕ использовать markdown ```             │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│  LLM генерирует код                         │
│                                             │
│  Пример ответа:                             │
│  "I'll calculate the factorial:             │
│                                             │
│  <code_interpreter type="code"              │
│   lang="python">                            │
│  def factorial(n):                          │
│      if n <= 1:                             │
│          return 1                           │
│      return n * factorial(n-1)              │
│                                             │
│  result = factorial(5)                      │
│  print(f'Factorial of 5 is {result}')       │
│  </code_interpreter>"                       │
└──────┬──────────────────────────────────────┘
       │
       │ Ответ с XML тегами кода
       │
       ▼
┌─────────────────────────────────────────────┐
│  Парсинг <code_interpreter> тегов           │
│                                             │
│  Извлекает:                                 │
│  - Язык программирования (python)           │
│  - Код для выполнения                       │
│                                             │
│  Код отправляется в frontend                │
└──────┬──────────────────────────────────────┘
       │
       │ Code block + metadata
       │
       ▼
┌─────────────────────────────────────────────┐
│  Frontend: Pyodide (Python in Browser)      │
│                                             │
│  1. Инициализирует Jupyter kernel           │
│  2. Проверяет запрещенные модули            │
│  3. Выполняет код в изолированной среде     │
│                                             │
│  Доступные библиотеки:                      │
│  - numpy, pandas, matplotlib                │
│  - requests, beautifulsoup4                 │
│  - и многие другие                          │
└──────┬──────────────────────────────────────┘
       │
       │ Результат выполнения
       │
       ▼
┌─────────────────────────────────────────────┐
│  Вывод результата                           │
│                                             │
│  - stdout (print statements)                │
│  - stderr (errors)                          │
│  - Возвращаемые значения                    │
│  - Графики (matplotlib → base64 images)     │
│  - Таблицы (pandas → HTML)                  │
│                                             │
│  Пример вывода:                             │
│  "Factorial of 5 is 120"                    │
│                                             │
│  Если ошибка:                               │
│  "NameError: name 'foo' is not defined"     │
└──────┬──────────────────────────────────────┘
       │
       │ Результат отображается в чате
       │
       ▼
┌─────────────────────────────────────────────┐
│  Опционально: LLM анализирует результат     │
│                                             │
│  Результат может быть отправлен обратно     │
│  в LLM для интерпретации и объяснения       │
│                                             │
│  LLM может:                                 │
│  - Объяснить результаты                     │
│  - Предложить улучшения                     │
│  - Исправить ошибки                         │
│  - Написать новый код                       │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│   Frontend   │
│ (Показывает  │
│  код и его   │
│ результаты)  │
└──────────────┘
```

### Поток данных

**Вход:**
- Запрос пользователя (например: "Calculate factorial of 5")
- Флаг code_interpreter включен

**Промежуточные данные:**
- System prompt с инструкциями
- LLM генерирует Python код в XML тегах
- Извлеченный код

**Выход:**
- Результаты выполнения кода (stdout, stderr, визуализации)
- Интерпретация результатов от LLM

**Особенности:**
- Код выполняется в браузере (безопасно, быстро)
- Поддержка визуализации (matplotlib графики)
- Изолированное выполнение (Pyodide sandbox)
- Блокировка опасных модулей (os, subprocess, etc.)

**Используемый промт:** `DEFAULT_CODE_INTERPRETER_PROMPT`

---

## Пайплайн автодополнения

**Назначение:** Автоматически дополняет текст пользователя в режиме реального времени, предлагая естественное продолжение для обычного текста или поисковых запросов.

**Файлы:**
- `/backend/open_webui/routers/tasks.py` - endpoint `generate_autocompletion()`
- `/backend/open_webui/utils/task.py` - функция `autocomplete_generation_template()`
- `/backend/open_webui/config.py` - `DEFAULT_AUTOCOMPLETE_GENERATION_PROMPT_TEMPLATE`

**Конфигурация:**
- `ENABLE_AUTOCOMPLETE_GENERATION` - включить/выключить функцию
- `AUTOCOMPLETE_GENERATION_INPUT_MAX_LENGTH` - макс длина текста
- `AUTOCOMPLETE_GENERATION_PROMPT_TEMPLATE` - кастомный шаблон

### Схема работы

```
┌──────────────┐
│   Frontend   │
│ User typing  │
│ in input...  │
└──────┬───────┘
       │
       │ POST /tasks/autocompletion/completions
       │ Body: {
       │   prompt: "The best restaurants in",
       │   type: "Search Query",
       │   messages: [...]
       │ }
       ▼
┌─────────────────────────────────────────────┐
│  generate_autocompletion() endpoint         │
│  /backend/open_webui/routers/tasks.py       │
└──────┬──────────────────────────────────────┘
       │
       │ 1. Проверяет длину текста
       │ 2. Определяет тип автодополнения
       │    - "General": обычный текст
       │    - "Search Query": поисковый запрос
       │
       ▼
┌─────────────────────────────────────────────┐
│  autocomplete_generation_template()         │
│  /backend/open_webui/utils/task.py          │
└──────┬──────────────────────────────────────┘
       │
       │ Заменяет переменные:
       │ - {{MESSAGES:END:6}} → контекст
       │ - {{TYPE}} → тип автодополнения
       │ - {{PROMPT}} → текст для продолжения
       │
       ▼
┌─────────────────────────────────────────────┐
│  Сформированный промт                       │
│                                             │
│  ### Task:                                  │
│  You are an autocompletion system...        │
│                                             │
│  <type>Search Query</type>                  │
│  <text>The best restaurants in</text>       │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│  TASK_MODEL (быстрая модель)                │
│  Генерирует продолжение                     │
└──────┬──────────────────────────────────────┘
       │
       │ Анализирует контекст и тип
       │
       ▼
┌─────────────────────────────────────────────┐
│  Результат (JSON)                           │
│  {                                          │
│    "text": "New York City for Italian      │
│             cuisine"                        │
│  }                                          │
│                                             │
│  Если неопределенность:                     │
│  { "text": "" }                             │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│   Frontend   │
│ (Показывает  │
│  продолжение │
│  серым       │
│  текстом)    │
└──────────────┘
    Tab → принять
```

### Поток данных

**Вход:**
- `prompt` - текст, который нужно продолжить
- `type` - тип ("General" или "Search Query")
- `messages` - контекст чата (последние 6)

**Промежуточные данные:**
- Анализ контекста разговора
- Определение намерения пользователя

**Выход:**
- JSON с продолжением текста
- Пустая строка если неопределенность

**Особенности:**
- Очень быстрая генерация (легкая модель)
- Разные режимы для разных типов текста
- Не повторяет исходный текст
- Учитывает контекст разговора

**Используемый промт:** `DEFAULT_AUTOCOMPLETE_GENERATION_PROMPT_TEMPLATE`

---

## Пайплайн MOA (Mixture of Agents)

**Назначение:** Синтезирует единый высококачественный ответ из нескольких ответов разных моделей, критически оценивая и объединяя информацию.

**Файлы:**
- `/backend/open_webui/routers/tasks.py` - endpoint `generate_moa_response()`
- `/backend/open_webui/utils/task.py` - функция `moa_response_generation_template()`
- `/backend/open_webui/config.py` - `DEFAULT_MOA_GENERATION_PROMPT_TEMPLATE`

**Конфигурация:**
- `MOA_GENERATION_PROMPT_TEMPLATE` - кастомный шаблон синтеза

### Схема работы

```
┌──────────────┐
│   Frontend   │
│  User query  │
└──────┬───────┘
       │
       │ "Explain quantum computing"
       │
       ▼
┌─────────────────────────────────────────────┐
│  Параллельный запрос к нескольким моделям   │
│                                             │
│  Модель 1 (GPT-4):                          │
│  → Response 1                               │
│                                             │
│  Модель 2 (Claude):                         │
│  → Response 2                               │
│                                             │
│  Модель 3 (Llama):                          │
│  → Response 3                               │
│                                             │
│  Модель 4 (Gemini):                         │
│  → Response 4                               │
└──────┬──────────────────────────────────────┘
       │
       │ Собранные ответы от всех моделей
       │
       ▼
┌─────────────────────────────────────────────┐
│  POST /tasks/moa/completions                │
│                                             │
│  generate_moa_response() endpoint           │
│  /backend/open_webui/routers/tasks.py       │
└──────┬──────────────────────────────────────┘
       │
       │ Body: {
       │   prompt: "Explain quantum computing",
       │   responses: [response1, response2, ...]
       │ }
       │
       ▼
┌─────────────────────────────────────────────┐
│  moa_response_generation_template()         │
│  /backend/open_webui/utils/task.py          │
└──────┬──────────────────────────────────────┘
       │
       │ Заменяет переменные:
       │ - {{prompt}} → оригинальный запрос
       │ - {{responses}} → все ответы моделей
       │
       ▼
┌─────────────────────────────────────────────┐
│  Сформированный промт для синтеза           │
│                                             │
│  You have been provided with a set of       │
│  responses from various models to:          │
│  "Explain quantum computing"                │
│                                             │
│  Response 1: [GPT-4 answer]                 │
│  Response 2: [Claude answer]                │
│  Response 3: [Llama answer]                 │
│  Response 4: [Gemini answer]                │
│                                             │
│  Your task is to synthesize these           │
│  responses into a single, high-quality      │
│  response...                                │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│  Aggregator Model (обычно самая сильная)    │
│                                             │
│  Критически анализирует:                    │
│  - Точность информации                      │
│  - Предвзятость или ошибки                  │
│  - Полноту ответов                          │
│  - Противоречия между ответами              │
│                                             │
│  Синтезирует:                               │
│  - Берет лучшее из каждого ответа           │
│  - Дополняет недостающую информацию         │
│  - Структурирует логично                    │
│  - Обеспечивает связность                   │
└──────┬──────────────────────────────────────┘
       │
       │ Синтезированный ответ
       │
       ▼
┌─────────────────────────────────────────────┐
│  Финальный высококачественный ответ         │
│                                             │
│  "Quantum computing is a revolutionary      │
│  paradigm that leverages quantum            │
│  mechanical phenomena...                    │
│                                             │
│  [Объединенная и улучшенная информация      │
│   от всех моделей, без предвзятости,        │
│   с исправленными ошибками]"                │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│   Frontend   │
│ (Показывает  │
│   итоговый   │
│    синтез)   │
└──────────────┘
```

### Поток данных

**Вход:**
- `prompt` - оригинальный запрос пользователя
- `responses` - массив ответов от разных моделей

**Промежуточные данные:**
- Анализ каждого ответа
- Выявление сильных/слабых сторон
- Обнаружение ошибок и предвзятости

**Выход:**
- Единый синтезированный ответ
- Более точный и полный, чем любой отдельный ответ

**Преимущества MOA:**
- Снижает галлюцинации (ошибки одной модели корректируются другими)
- Объединяет различные перспективы
- Повышает точность и надежность
- Использует сильные стороны каждой модели

**Используемый промт:** `DEFAULT_MOA_GENERATION_PROMPT_TEMPLATE`

---

## Общая архитектура и взаимодействие пайплайнов

### Главный обработчик чата

Все пайплайны координируются через главный middleware в `/backend/open_webui/utils/middleware.py`. Вот как они взаимодействуют:

```
┌────────────────────────────────────────────────────────┐
│             POST /chat/completions                     │
│              (Frontend request)                        │
└───────────────────────┬────────────────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────┐
        │    Middleware Pipeline        │
        │  (utils/middleware.py)        │
        └───┬───────────────────────┬───┘
            │                       │
    ┌───────▼──────┐        ┌──────▼────────┐
    │  Memory      │        │  Files/RAG    │
    │  Handler     │        │  Handler      │
    └───────┬──────┘        └──────┬────────┘
            │                      │
            │    ┌─────────────────┘
            │    │
    ┌───────▼────▼──────┐
    │  Web Search       │
    │  Handler          │
    └───────┬───────────┘
            │
    ┌───────▼───────────┐
    │  Tools/Function   │
    │  Calling Handler  │
    └───────┬───────────┘
            │
    ┌───────▼───────────┐
    │  Code Interpreter │
    │  Handler          │
    └───────┬───────────┘
            │
            ▼
    ┌───────────────────┐
    │  LLM API Call     │
    │  (OpenAI/Ollama)  │
    └───────┬───────────┘
            │
            ▼
    ┌───────────────────┐
    │  Response Stream  │
    └───────┬───────────┘
            │
            ▼
    ┌───────────────────┐
    │  Post-processing  │
    │  (Title, Tags,    │
    │   Follow-ups)     │
    └───────┬───────────┘
            │
            ▼
    ┌───────────────────┐
    │  Frontend Display │
    └───────────────────┘
```

### Последовательность выполнения

1. **Pre-processing** (перед вызовом LLM):
   - Memory Handler: добавляет долговременную память
   - RAG Handler: инжектирует контекст из документов
   - Web Search Handler: добавляет результаты поиска
   - Tools Handler: подготавливает спецификации инструментов
   - Code Interpreter: добавляет system prompt

2. **LLM Call**: Вызов модели с обогащенным контекстом

3. **Post-processing** (после получения ответа):
   - Title Generation: генерация заголовка
   - Tags Generation: присвоение тегов
   - Follow-ups Generation: предложение следующих вопросов

### Ключевые принципы

**Модульность:**
- Каждый пайплайн независим
- Можно включать/выключать отдельно
- Легко добавлять новые пайплайны

**Композиция:**
- Пайплайны могут работать вместе
- RAG + Web Search + Tools одновременно
- Результаты одного влияют на другие

**Асинхронность:**
- Потоковая передача ответов (streaming)
- WebSocket для real-time обновлений
- Параллельное выполнение где возможно

**Расширяемость:**
- Кастомные промты для каждого пайплайна
- Пользовательские функции и инструменты
- MCP протокол для сторонних расширений

---

## Заключение

Open WebUI реализует сложную многоуровневую архитектуру агентов с модульными пайплайнами:

**Основные пайплайны:**
1. Title/Tags/Follow-ups - метаданные и навигация
2. RAG - работа с документами и знаниями
3. Web Search - доступ к актуальной информации
4. Tools/Function Calling - внешние действия
5. Code Interpreter - выполнение Python кода
6. Autocomplete - помощь при вводе
7. MOA - синтез ответов от множества моделей

**Все пайплайны:**
- Используют динамические промты с переменными
- Настраиваются через конфигурацию
- Работают в координации через middleware
- Поддерживают потоковую передачу данных
- Могут комбинироваться для сложных задач

Эта архитектура обеспечивает гибкость, расширяемость и высокое качество работы AI агента.
