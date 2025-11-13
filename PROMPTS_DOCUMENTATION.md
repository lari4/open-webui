# Документация AI Промтов Open WebUI

Этот документ содержит все AI промты, используемые в приложении Open WebUI, с подробным описанием их назначения и применения.

## Оглавление

1. [Промты для автоматической генерации метаданных](#промты-для-автоматической-генерации-метаданных)
   - [Генерация заголовков чата](#генерация-заголовков-чата)
   - [Генерация тегов](#генерация-тегов)
   - [Генерация изображений](#генерация-изображений)
   - [Генерация follow-up вопросов](#генерация-follow-up-вопросов)
2. [Промты для поиска и получения информации](#промты-для-поиска-и-получения-информации)
3. [Промты для работы с инструментами](#промты-для-работы-с-инструментами)
4. [Промты для интерактивных функций](#промты-для-интерактивных-функций)
5. [Промты для RAG (Retrieval-Augmented Generation)](#промты-для-rag-retrieval-augmented-generation)
6. [Промты для специальных возможностей](#промты-для-специальных-возможностей)

---

## Промты для автоматической генерации метаданных

### Генерация заголовков чата

**Расположение:** `/backend/open_webui/config.py:1599-1621`
**Переменная:** `DEFAULT_TITLE_GENERATION_PROMPT_TEMPLATE`
**Когда используется:** Автоматически генерирует короткий заголовок (3-5 слов) с эмодзи для нового чата на основе первых сообщений.

**Описание:**
Этот промт используется для создания краткого и понятного заголовка чата, который отражает основную тему беседы. Промт анализирует последние 2 сообщения из истории чата и генерирует заголовок на языке беседы (по умолчанию английский для мультиязычных чатов). Ответ должен быть строго в формате JSON без дополнительного текста.

**Особенности:**
- Использует последние 2 сообщения: `{{MESSAGES:END:2}}`
- Требует строгий JSON формат без markdown
- Заголовок должен быть на языке чата
- Включает эмодзи для визуального представления темы
- Приоритет точности над креативностью

```python
DEFAULT_TITLE_GENERATION_PROMPT_TEMPLATE = """### Task:
Generate a concise, 3-5 word title with an emoji summarizing the chat history.
### Guidelines:
- The title should clearly represent the main theme or subject of the conversation.
- Use emojis that enhance understanding of the topic, but avoid quotation marks or special formatting.
- Write the title in the chat's primary language; default to English if multilingual.
- Prioritize accuracy over excessive creativity; keep it clear and simple.
- Your entire response must consist solely of the JSON object, without any introductory or concluding text.
- The output must be a single, raw JSON object, without any markdown code fences or other encapsulating text.
- Ensure no conversational text, affirmations, or explanations precede or follow the raw JSON output, as this will cause direct parsing failure.
### Output:
JSON format: { "title": "your concise title here" }
### Examples:
- { "title": "📉 Stock Market Trends" },
- { "title": "🍪 Perfect Chocolate Chip Recipe" },
- { "title": "Evolution of Music Streaming" },
- { "title": "Remote Work Productivity Tips" },
- { "title": "Artificial Intelligence in Healthcare" },
- { "title": "🎮 Video Game Development Insights" }
### Chat History:
<chat_history>
{{MESSAGES:END:2}}
</chat_history>"""
```

---

### Генерация тегов

**Расположение:** `/backend/open_webui/config.py:1629-1645`
**Переменная:** `DEFAULT_TAGS_GENERATION_PROMPT_TEMPLATE`
**Когда используется:** Автоматически категоризирует чат, присваивая 1-3 широких тега и 1-3 специфических подтемы.

**Описание:**
Промт анализирует последние 6 сообщений чата и генерирует теги для категоризации. Начинает с высокоуровневых доменов (наука, технологии, философия и т.д.), затем добавляет более специфические подтемы. Для коротких чатов (менее 3 сообщений) или слишком разнообразных бесед использует только тег ["General"].

**Особенности:**
- Использует последние 6 сообщений: `{{MESSAGES:END:6}}`
- Начинает с широких доменов (Science, Technology, Philosophy, Arts, Politics, Business, Health, Sports, Entertainment, Education)
- Для коротких/разнообразных чатов использует ["General"]
- Выводит JSON массив тегов
- Приоритет точности над специфичностью

```python
DEFAULT_TAGS_GENERATION_PROMPT_TEMPLATE = """### Task:
Generate 1-3 broad tags categorizing the main themes of the chat history, along with 1-3 more specific subtopic tags.

### Guidelines:
- Start with high-level domains (e.g. Science, Technology, Philosophy, Arts, Politics, Business, Health, Sports, Entertainment, Education)
- Consider including relevant subfields/subdomains if they are strongly represented throughout the conversation
- If content is too short (less than 3 messages) or too diverse, use only ["General"]
- Use the chat's primary language; default to English if multilingual
- Prioritize accuracy over specificity

### Output:
JSON format: { "tags": ["tag1", "tag2", "tag3"] }

### Chat History:
<chat_history>
{{MESSAGES:END:6}}
</chat_history>"""
```

---

### Генерация изображений

**Расположение:** `/backend/open_webui/config.py:1653-1671`
**Переменная:** `DEFAULT_IMAGE_PROMPT_GENERATION_PROMPT_TEMPLATE`
**Когда используется:** Генерирует детальное описание для создания изображений с помощью AI генераторов изображений на основе контекста чата.

**Описание:**
Этот промт создает подробное описание для генерации изображений, анализируя контекст из последних 6 сообщений чата. Промт инструктирует модель быть максимально описательной, включая детали о цветах, формах и важных элементах. Описание должно быть понятным человеку, который не видит изображение.

**Особенности:**
- Использует последние 6 сообщения: `{{MESSAGES:END:6}}`
- Фокусируется на наиболее важных аспектах
- Избегает предположений и несуществующей информации
- Использует язык чата
- Для сложных изображений фокусируется на главных элементах

```python
DEFAULT_IMAGE_PROMPT_GENERATION_PROMPT_TEMPLATE = """### Task:
Generate a detailed prompt for am image generation task based on the given language and context. Describe the image as if you were explaining it to someone who cannot see it. Include relevant details, colors, shapes, and any other important elements.

### Guidelines:
- Be descriptive and detailed, focusing on the most important aspects of the image.
- Avoid making assumptions or adding information not present in the image.
- Use the chat's primary language; default to English if multilingual.
- If the image is too complex, focus on the most prominent elements.

### Output:
Strictly return in JSON format:
{
    "prompt": "Your detailed description here."
}

### Chat History:
<chat_history>
{{MESSAGES:END:6}}
</chat_history>"""
```

---

### Генерация follow-up вопросов

**Расположение:** `/backend/open_webui/config.py:1680-1694`
**Переменная:** `DEFAULT_FOLLOW_UP_GENERATION_PROMPT_TEMPLATE`
**Когда используется:** Предлагает 3-5 релевантных последующих вопросов, которые пользователь может задать для продолжения беседы.

**Описание:**
Промт анализирует последние 6 сообщений и генерирует естественные вопросы, которые пользователь мог бы задать далее. Вопросы формулируются от лица пользователя, обращенные к ассистенту. Они должны быть краткими, понятными и напрямую связаны с обсуждаемой темой.

**Особенности:**
- Использует последние 6 сообщений: `{{MESSAGES:END:6}}`
- Вопросы от лица пользователя (не ассистента)
- Не повторяет уже обсужденное
- Для коротких бесед предлагает более общие, но релевантные вопросы
- Выводит JSON массив из 3-5 вопросов
- Только релевантные вопросы, имеющие смысл в контексте

```python
DEFAULT_FOLLOW_UP_GENERATION_PROMPT_TEMPLATE = """### Task:
Suggest 3-5 relevant follow-up questions or prompts that the user might naturally ask next in this conversation as a **user**, based on the chat history, to help continue or deepen the discussion.
### Guidelines:
- Write all follow-up questions from the user's point of view, directed to the assistant.
- Make questions concise, clear, and directly related to the discussed topic(s).
- Only suggest follow-ups that make sense given the chat content and do not repeat what was already covered.
- If the conversation is very short or not specific, suggest more general (but relevant) follow-ups the user might ask.
- Use the conversation's primary language; default to English if multilingual.
- Response must be a JSON array of strings, no extra text or formatting.
### Output:
JSON format: { "follow_ups": ["Question 1?", "Question 2?", "Question 3?"] }
### Chat History:
<chat_history>
{{MESSAGES:END:6}}
</chat_history>"""
```

---

## Промты для поиска и получения информации

### Генерация поисковых запросов

**Расположение:** `/backend/open_webui/config.py:1734-1756`
**Переменная:** `DEFAULT_QUERY_GENERATION_PROMPT_TEMPLATE`
**Когда используется:** Автоматически генерирует 1-3 широких и релевантных поисковых запроса для получения дополнительной информации из интернета или базы знаний.

**Описание:**
Этот промт анализирует последние 6 сообщений чата и определяет, нужна ли дополнительная информация. По умолчанию приоритет отдается генерации запросов, если есть хоть малейшая неопределенность. Промт стремится получить максимально полную, актуальную и ценную информацию. Возвращает пустой массив только если абсолютно точно известно, что поиск не нужен.

**Особенности:**
- Использует последние 6 сообщений: `{{MESSAGES:END:6}}`
- Доступна текущая дата: `{{CURRENT_DATE}}`
- Приоритет генерации запросов при любой неопределенности
- Только JSON ответ, без дополнительного текста
- Каждый запрос должен быть уникальным, кратким и релевантным
- Возвращает пустой массив только при 100% уверенности, что поиск не нужен
- Фокусируется на широких запросах для максимального охвата информации

```python
DEFAULT_QUERY_GENERATION_PROMPT_TEMPLATE = """### Task:
Analyze the chat history to determine the necessity of generating search queries, in the given language. By default, **prioritize generating 1-3 broad and relevant search queries** unless it is absolutely certain that no additional information is required. The aim is to retrieve comprehensive, updated, and valuable information even with minimal uncertainty. If no search is unequivocally needed, return an empty list.

### Guidelines:
- Respond **EXCLUSIVELY** with a JSON object. Any form of extra commentary, explanation, or additional text is strictly prohibited.
- When generating search queries, respond in the format: { "queries": ["query1", "query2"] }, ensuring each query is distinct, concise, and relevant to the topic.
- If and only if it is entirely certain that no useful results can be retrieved by a search, return: { "queries": [] }.
- Err on the side of suggesting search queries if there is **any chance** they might provide useful or updated information.
- Be concise and focused on composing high-quality search queries, avoiding unnecessary elaboration, commentary, or assumptions.
- Today's date is: {{CURRENT_DATE}}.
- Always prioritize providing actionable and broad queries that maximize informational coverage.

### Output:
Strictly return in JSON format:
{
  "queries": ["query1", "query2"]
}

### Chat History:
<chat_history>
{{MESSAGES:END:6}}
</chat_history>
"""
```

---
