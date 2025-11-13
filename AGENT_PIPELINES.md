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
