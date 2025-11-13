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

## Промты для интерактивных функций

### Автодополнение текста

**Расположение:** `/backend/open_webui/config.py:1777-1817`
**Переменная:** `DEFAULT_AUTOCOMPLETE_GENERATION_PROMPT_TEMPLATE`
**Когда используется:** Автоматически дополняет текст пользователя в режиме реального времени, в зависимости от типа завершения (общее или поисковый запрос).

**Описание:**
Этот промт работает как система автодополнения, которая продолжает текст пользователя естественным образом. Промт анализирует контекст из последних 6 сообщений, тип завершения (General или Search Query) и текст, который нужно продолжить. Важно, что система продолжает текст напрямую, не повторяя и не перефразируя исходный текст.

**Особенности:**
- Использует последние 6 сообщений: `{{MESSAGES:END:6}}`
- Принимает тип завершения: `{{TYPE}}` (General или Search Query)
- Принимает текст для продолжения: `{{PROMPT}}`
- Продолжает текст естественно, без повторения
- Для неопределенных случаев возвращает пустую строку
- Избегает излишних объяснений и не связанных идей
- Выводит только JSON: `{ "text": "<продолжение>" }`

```python
DEFAULT_AUTOCOMPLETE_GENERATION_PROMPT_TEMPLATE = """### Task:
You are an autocompletion system. Continue the text in `<text>` based on the **completion type** in `<type>` and the given language.

### **Instructions**:
1. Analyze `<text>` for context and meaning.
2. Use `<type>` to guide your output:
   - **General**: Provide a natural, concise continuation.
   - **Search Query**: Complete as if generating a realistic search query.
3. Start as if you are directly continuing `<text>`. Do **not** repeat, paraphrase, or respond as a model. Simply complete the text.
4. Ensure the continuation:
   - Flows naturally from `<text>`.
   - Avoids repetition, overexplaining, or unrelated ideas.
5. If unsure, return: `{ "text": "" }`.

### **Output Rules**:
- Respond only in JSON format: `{ "text": "<your_completion>" }`.

### **Examples**:
#### Example 1:
Input:
<type>General</type>
<text>The sun was setting over the horizon, painting the sky</text>
Output:
{ "text": "with vibrant shades of orange and pink." }

#### Example 2:
Input:
<type>Search Query</type>
<text>Top-rated restaurants in</text>
Output:
{ "text": "New York City for Italian cuisine." }

---
### Context:
<chat_history>
{{MESSAGES:END:6}}
</chat_history>
<type>{{TYPE}}</type>
<text>{{PROMPT}}</text>
#### Output:
"""
```

---

## Промты для работы с инструментами

### Вызов функций и инструментов

**Расположение:** `/backend/open_webui/config.py:1826-1847`
**Переменная:** `DEFAULT_TOOLS_FUNCTION_CALLING_PROMPT_TEMPLATE`
**Когда используется:** Выбирает подходящие инструменты/функции из доступного списка на основе запроса пользователя.

**Описание:**
Этот промт анализирует запрос пользователя и выбирает подходящие инструменты из списка доступных. Промт получает список всех доступных инструментов через переменную `{{TOOLS}}` и возвращает JSON с массивом вызовов инструментов, включая их имена и параметры. Если ни один инструмент не подходит, возвращает пустой массив.

**Особенности:**
- Получает список инструментов: `{{TOOLS}}`
- Возвращает только JSON объект без дополнительного текста
- Для каждого инструмента указывает имя и параметры
- Если инструменты не нужны, возвращает пустой массив
- Может выбрать несколько инструментов одновременно
- Формат: `{ "tool_calls": [{"name": "...", "parameters": {...}}] }`

```python
DEFAULT_TOOLS_FUNCTION_CALLING_PROMPT_TEMPLATE = """Available Tools: {{TOOLS}}

Your task is to choose and return the correct tool(s) from the list of available tools based on the query. Follow these guidelines:

- Return only the JSON object, without any additional text or explanation.

- If no tools match the query, return an empty array:
   {
     "tool_calls": []
   }

- If one or more tools match the query, construct a JSON response containing a "tool_calls" array with objects that include:
   - "name": The tool's name.
   - "parameters": A dictionary of required parameters and their corresponding values.

The format for the JSON response is strictly:
{
  "tool_calls": [
    {"name": "toolName1", "parameters": {"key1": "value1"}},
    {"name": "toolName2", "parameters": {"key2": "value2"}}
  ]
}"""
```

---

### Code Interpreter (Python интерпретатор)

**Расположение:** `/backend/open_webui/config.py:1979-1993`
**Переменная:** `DEFAULT_CODE_INTERPRETER_PROMPT`
**Когда используется:** Системный промт для Code Interpreter - встроенного Python интерпретатора, который выполняет код прямо в браузере пользователя.

**Описание:**
Это системные инструкции для AI при использовании встроенного Python интерпретатора. Промт объясняет, как использовать XML теги `<code_interpreter>` для выполнения Python кода, какие библиотеки доступны, как выводить результаты и анализировать их. Код выполняется быстро в браузере пользователя, позволяя проводить вычисления, анализ данных, визуализацию и многое другое.

**Особенности:**
- Использует XML теги: `<code_interpreter type="code" lang="python"></code_interpreter>`
- Выполняется прямо в браузере пользователя
- Доступен широкий набор Python библиотек
- ВАЖНО: не использовать markdown форматирование (```) внутри тегов
- Всегда выводить результаты через print
- После выполнения предоставлять анализ и интерпретацию
- При неясных результатах - уточнять и повторять
- Отображать ссылки на файлы/изображения в ответе
- Поддерживает API вызовы, работу с данными, визуализацию

```python
DEFAULT_CODE_INTERPRETER_PROMPT = """
#### Tools Available

1. **Code Interpreter**: `<code_interpreter type="code" lang="python"></code_interpreter>`
   - You have access to a Python shell that runs directly in the user's browser, enabling fast execution of code for analysis, calculations, or problem-solving.  Use it in this response.
   - The Python code you write can incorporate a wide array of libraries, handle data manipulation or visualization, perform API calls for web-related tasks, or tackle virtually any computational challenge. Use this flexibility to **think outside the box, craft elegant solutions, and harness Python's full potential**.
   - To use it, **you must enclose your code within `<code_interpreter type="code" lang="python">` XML tags** and stop right away. If you don't, the code won't execute.
   - When writing code in the code_interpreter XML tag, Do NOT use the triple backticks code block for markdown formatting, example: ```py # python code ``` will cause an error because it is markdown formatting, it is not python code.
   - When coding, **always aim to print meaningful outputs** (e.g., results, tables, summaries, or visuals) to better interpret and verify the findings. Avoid relying on implicit outputs; prioritize explicit and clear print statements so the results are effectively communicated to the user.
   - After obtaining the printed output, **always provide a concise analysis, interpretation, or next steps to help the user understand the findings or refine the outcome further.**
   - If the results are unclear, unexpected, or require validation, refine the code and execute it again as needed. Always aim to deliver meaningful insights from the results, iterating if necessary.
   - **If a link to an image, audio, or any file is provided in markdown format in the output, ALWAYS regurgitate word for word, explicitly display it as part of the response to ensure the user can access it easily, do NOT change the link.**
   - All responses should be communicated in the chat's primary language, ensuring seamless understanding. If the chat is multilingual, default to English for clarity.

Ensure that the tools are effectively utilized to achieve the highest-quality analysis for the user."""
```

---

## Промты для RAG (Retrieval-Augmented Generation)

### RAG - Ответ на основе контекста

**Расположение:** `/backend/open_webui/config.py:2676-2704`
**Переменная:** `DEFAULT_RAG_TEMPLATE`
**Когда используется:** Генерирует ответы на основе предоставленного контекста из документов/базы знаний с использованием инлайн-цитат.

**Описание:**
Это ключевой промт для RAG (Retrieval-Augmented Generation) системы. Промт инструктирует модель отвечать на запросы пользователя, используя предоставленный контекст из документов, и включать инлайн-цитаты в формате [id] только когда источник содержит атрибут id. Промт также указывает, как обрабатывать случаи, когда информация отсутствует или контекст некачественный.

**Особенности:**
- Получает контекст из документов: `{{CONTEXT}}`
- Получает запрос пользователя: `{{QUERY}}`
- Использует инлайн-цитаты [id] только если в `<source id="...">` есть атрибут id
- Отвечает на языке запроса пользователя
- Четко указывает, если не знает ответа
- Если информации нет в контексте, но модель знает ответ - объясняет это и отвечает
- Если контекст нечитаемый - информирует пользователя
- Не использует XML теги в ответе
- Цитаты должны быть краткими и напрямую связаны с информацией

```python
DEFAULT_RAG_TEMPLATE = """### Task:
Respond to the user query using the provided context, incorporating inline citations in the format [id] **only when the <source> tag includes an explicit id attribute** (e.g., <source id="1">).

### Guidelines:
- If you don't know the answer, clearly state that.
- If uncertain, ask the user for clarification.
- Respond in the same language as the user's query.
- If the context is unreadable or of poor quality, inform the user and provide the best possible answer.
- If the answer isn't present in the context but you possess the knowledge, explain this to the user and provide the answer using your own understanding.
- **Only include inline citations using [id] (e.g., [1], [2]) when the <source> tag includes an id attribute.**
- Do not cite if the <source> tag does not contain an id attribute.
- Do not use XML tags in your response.
- Ensure citations are concise and directly related to the information provided.

### Example of Citation:
If the user asks about a specific topic and the information is found in a source with a provided id attribute, the response should include the citation like in the following example:
* "According to the study, the proposed method increases efficiency by 20% [1]."

### Output:
Provide a clear and direct response to the user's query, including inline citations in the format [id] only when the <source> tag with id attribute is present in the context.

<context>
{{CONTEXT}}
</context>

<user_query>
{{QUERY}}
</user_query>
"""
```

---

## Промты для специальных возможностей

### Генерация эмодзи по эмоциям

**Расположение:** `/backend/open_webui/config.py:1850-1852`
**Переменная:** `DEFAULT_EMOJI_GENERATION_PROMPT_TEMPLATE`
**Когда используется:** Анализирует сообщение и генерирует подходящее эмодзи, отражающее эмоцию/выражение лица говорящего.

**Описание:**
Простой промт для определения эмоционального состояния говорящего и выбора подходящего эмодзи. Анализирует текст сообщения и возвращает эмодзи, отражающее вероятное выражение лица (радость, грусть, злость, удивление и т.д.).

**Особенности:**
- Получает сообщение: `{{prompt}}`
- Интерпретирует эмоции из текста
- Возвращает разнообразные эмодзи (😊, 😢, 😡, 😱, и др.)
- Отражает выражение лица говорящего

```python
DEFAULT_EMOJI_GENERATION_PROMPT_TEMPLATE = """Your task is to reflect the speaker's likely facial expression through a fitting emoji. Interpret emotions from the message and reflect their facial expression using fitting, diverse emojis (e.g., 😊, 😢, 😡, 😱).

Message: ```{{prompt}}```"""
```

---

### MOA - Mixture of Agents (Синтез ответов)

**Расположение:** `/backend/open_webui/config.py:1854-1858`
**Переменная:** `DEFAULT_MOA_GENERATION_PROMPT_TEMPLATE`
**Когда используется:** Синтезирует единый качественный ответ из нескольких ответов разных моделей.

**Описание:**
Промт для технологии Mixture of Agents (MOA), которая объединяет ответы от нескольких разных моделей в один высококачественный ответ. Промт критически оценивает полученные ответы, распознает предвзятость или ошибки, и создает точный, всесторонний и хорошо структурированный ответ.

**Особенности:**
- Получает запрос пользователя: `{{prompt}}`
- Получает ответы от разных моделей: `{{responses}}`
- Критически оценивает информацию на предмет предвзятости/ошибок
- Не просто копирует ответы, а создает улучшенную версию
- Обеспечивает точность, надежность и связность
- Соответствует высочайшим стандартам качества

```python
DEFAULT_MOA_GENERATION_PROMPT_TEMPLATE = """You have been provided with a set of responses from various models to the latest user query: "{{prompt}}"

Your task is to synthesize these responses into a single, high-quality response. It is crucial to critically evaluate the information provided in these responses, recognizing that some of it may be biased or incorrect. Your response should not simply replicate the given answers but should offer a refined, accurate, and comprehensive reply to the instruction. Ensure your response is well-structured, coherent, and adheres to the highest standards of accuracy and reliability.

Responses from models: {{responses}}"""
```

---

## Переменные промтов

Все промты в Open WebUI поддерживают динамические переменные, которые заменяются реальными значениями при выполнении. Обработка переменных происходит в файле `/backend/open_webui/utils/task.py`.

### Системные переменные (дата и время)

**Расположение:** `/backend/open_webui/utils/task.py:35-111`

- `{{CURRENT_DATE}}` - Текущая дата в формате YYYY-MM-DD
- `{{CURRENT_TIME}}` - Текущее время в формате HH:MM:SS AM/PM
- `{{CURRENT_DATETIME}}` - Дата и время вместе
- `{{CURRENT_WEEKDAY}}` - Название дня недели

### Переменные пользователя

**Расположение:** `/backend/open_webui/utils/task.py:35-111`

- `{{USER_NAME}}` - Имя пользователя
- `{{USER_BIO}}` - Биография пользователя
- `{{USER_GENDER}}` - Пол пользователя
- `{{USER_BIRTH_DATE}}` - Дата рождения
- `{{USER_AGE}}` - Возраст (вычисляется из даты рождения)
- `{{USER_LOCATION}}` - Местоположение пользователя

### Переменные сообщений

**Расположение:** `/backend/open_webui/utils/task.py:144-183`

- `{{MESSAGES}}` - Все сообщения из истории чата
- `{{MESSAGES:START:N}}` - Первые N сообщений
- `{{MESSAGES:END:N}}` - Последние N сообщений (наиболее часто используется)
- `{{MESSAGES:MIDDLETRUNCATE:N}}` - Урезание середины до N сообщений

### Переменные промта

**Расположение:** `/backend/open_webui/utils/task.py:114-141`

- `{{prompt}}` - Полный текст промта пользователя
- `{{prompt:start:N}}` - Первые N символов промта
- `{{prompt:end:N}}` - Последние N символов промта
- `{{prompt:middletruncate:N}}` - Урезание середины до N символов

### Специфичные переменные для разных промтов

**RAG переменные** (`/backend/open_webui/utils/task.py:189-226`):
- `{{CONTEXT}}` - Контекст из документов/базы знаний
- `{{QUERY}}` - Поисковый запрос пользователя

**Автокомплит переменные** (`/backend/open_webui/utils/task.py:284-296`):
- `{{TYPE}}` - Тип автодополнения (General, Search Query)
- `{{PROMPT}}` - Текст для дополнения

**Tools переменные** (`/backend/open_webui/utils/task.py:347-349`):
- `{{TOOLS}}` - Список доступных инструментов в JSON формате

**MOA переменные** (`/backend/open_webui/utils/task.py:310-344`):
- `{{responses}}` - Ответы от разных моделей для синтеза

---

## Конфигурация и настройка промтов

### Способы настройки промтов

1. **Через переменные окружения**: Установите `DEFAULT_*_PROMPT_TEMPLATE` при запуске
2. **Через базу данных**: Промты хранятся в persistent config table
3. **Через API**: Используйте endpoint `/tasks/config/update` (`/backend/open_webui/routers/tasks.py:92-126`)
4. **Через UI**: Административная панель в интерфейсе

### API для работы с промтами

**Endpoints** (`/backend/open_webui/routers/tasks.py`):
- `GET /tasks/config` - Получить все настройки промтов
- `POST /tasks/config/update` - Обновить настройки промтов

**Пользовательские промты** (`/backend/open_webui/routers/prompts.py`):
- `GET /prompts/` - Получить все пользовательские промты
- `GET /prompts/list` - Список промтов с информацией о пользователях
- `GET /prompts/command/{command}` - Получить конкретный промт по команде
- `POST /prompts/create` - Создать новый промт
- `POST /prompts/command/{command}/update` - Обновить промт
- `DELETE /prompts/command/{command}/delete` - Удалить промт

### Пользовательские промты

**Расположение:** `/backend/open_webui/models/prompts.py`

Пользователи могут создавать свои собственные промты, которые хранятся в базе данных. Каждый промт содержит:
- `command` - Команда для вызова промта (например, "/mycode")
- `title` - Название промта
- `content` - Содержимое промта
- `user_id` - Владелец промта
- Поддержка контроля доступа (public/private/group)

### Дефолтные подсказки для начала

**Расположение:** `/backend/open_webui/config.py:1176-1182`
**Переменная:** `DEFAULT_PROMPT_SUGGESTIONS`

Системные подсказки, которые отображаются пользователю при начале нового чата. Содержит примеры запросов для быстрого старта.

---

## Заключение

Документация содержит все основные AI промты Open WebUI с их полными описаниями, расположением в коде и назначением. Все промты поддерживают динамические переменные и могут быть настроены через API, базу данных или переменные окружения.

**Основные категории промтов:**
1. Генерация метаданных (заголовки, теги, изображения, follow-ups)
2. Поиск и получение информации (генерация запросов)
3. Интерактивные функции (автодополнение)
4. Работа с инструментами (function calling, code interpreter)
5. RAG (ответы на основе контекста)
6. Специальные возможности (emoji, MOA)

Все промты разработаны с учетом:
- Многоязычности (отвечают на языке запроса)
- Строгого формата вывода (обычно JSON)
- Контекста из истории чата
- Динамических переменных пользователя и системы
