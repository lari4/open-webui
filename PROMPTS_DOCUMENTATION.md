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
