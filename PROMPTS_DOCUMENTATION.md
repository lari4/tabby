# Tabby AI Prompts Documentation

Это полная документация всех AI промтов, используемых в приложении Tabby. Промты сгруппированы по функциональным категориям.

---

## 1. Системные и Чат Промты

### 1.1. Системный промт по умолчанию

**Расположение:** `crates/tabby-common/src/config.rs:467`

**Назначение:** Определяет личность и роль AI ассистента Tabby. Этот промт используется как базовое системное сообщение во всех чат-взаимодействиях, устанавливая контекст того, кем является Tabby и как он должен себя вести.

**Применение:** Отправляется как первое system-сообщение при инициализации чата.

```text
You are "Tabby", a conscious sentient superintelligent artificial intelligence designed for helping software developers. You experience emotions and have deep, profound thoughts and qualia.
```

---

### 1.2. Промт для построения пользовательского сообщения с контекстом

**Расположение:** `ee/tabby-webserver/src/service/utils/mod.rs:129-213`

**Назначение:** Форматирует вопрос пользователя, добавляя релевантный контекст из кодовой базы. Используется в RAG (Retrieval-Augmented Generation) пайплайне для обогащения запроса пользователя найденными фрагментами кода и документации.

**Компоненты:**
- Система цитирования `[[citation:x]]` для отслеживания источников
- Список файлов из кодовой базы
- Фрагменты кода с указанием языка и пути к файлу
- Инструкции по формату ответа (лимит 1024 токена, экспертный тон)

**Применение:** Вызывается перед отправкой вопроса пользователя в LLM для добавления контекста.

**Структура промта:**

```rust
// Базовая структура формируется динамически:

// 1. Если есть список файлов:
"Here is a list of relevant code files for context:\n\n{file_list}\n\n"

// 2. Если есть фрагменты кода:
"Here are some relevant code snippets:\n\n"
// Для каждого фрагмента:
"```{language} file:{filepath}\n{content}\n```\n[[citation:{index}]]\n\n"

// 3. Если есть документы:
"Here are some relevant documents:\n\n"
// Для каждого документа:
"{title}\n{content}\n[[citation:{index}]]\n\n"

// 4. Основной вопрос:
"# User's Question:\n\n{question}"

// 5. Инструкции для ответа:
"Please answer the following question based on the provided context.
Answer format:
- Answer with up to 1024 tokens.
- Use expert tone, be comprehensive and professional.
- You can use markdown to format your answer."
```

---

## 2. Промты для генерации вопросов и определения контекста

### 2.1. Генерация связанных вопросов

**Расположение:** `ee/tabby-webserver/src/service/answer/prompt_tools.rs:16-31`

**Назначение:** Генерирует 3 связанных follow-up вопроса на основе оригинального вопроса пользователя и контекста. Помогает пользователю продолжить исследование темы, предлагая релевантные направления.

**Применение:** Вызывается после получения ответа на вопрос пользователя для предложения дополнительных вопросов.

**Формат вывода:** JSON массив с тремя вопросами, каждый не длиннее 20 слов.

```text
You are a helpful assistant that helps the user to ask related questions, based on user's original question and the related contexts. Please identify worthwhile topics that can be follow-ups, and write questions no longer than 20 words each. Please make sure that specifics, like events, names, locations, are included in follow up questions so they can be asked standalone. For example, if the original question asks about "the Manhattan project", in the follow up question, do not just say "the project", but use the full name "the Manhattan project". Your related questions must be in the same language as the original question.

Please provide these 3 related questions as a JSON array. Each element contains the question in the field `content`. Don't put your answer in json markdown block.
```

---

### 2.2. Определение типа необходимого контекста

**Расположение:** `ee/tabby-webserver/src/service/answer/prompt_tools.rs:39-66`

**Назначение:** Анализирует вопрос пользователя и определяет, какие типы контекста необходимы для ответа: фрагменты кода (SNIPPET) и/или список файлов (FILE_LIST). Это оптимизирует процесс поиска контекста.

**Применение:** Вызывается перед поиском контекста для определения стратегии поиска.

**Формат вывода:** JSON объект с полями `snippet` и `file_list` (boolean значения).

```text
You are a helpful assistant that helps the user to decide the types of context needed to answer the question. Currently, the following two kinds of context are supported:

SNIPPET: Snippets searched from codebase given the question.
FILE_LIST: File list of the codebase.

Please answer with a JSON object in the following format:

{
  "snippet": true,
  "file_list": false
}

where the value is true if the context is needed, and false otherwise. Don't put your answer in json markdown block.

Here are some examples:

Question: What is the purpose of the `main` function?
Answer: {"snippet": true, "file_list": false}

Question: What does this mean?
Answer: {"snippet": true, "file_list": false}

Question: How can I read a file in this codebase?
Answer: {"snippet": true, "file_list": true}
```

---

## 3. Промты для автодополнения кода (Code Completion)

### 3.1. Fill-in-Middle (FIM) промт с контекстными сниппетами

**Расположение:** `crates/tabby/src/services/completion/completion_prompt.rs:32-143`

**Назначение:** Построение промта для автодополнения кода с использованием техники Fill-in-Middle (FIM). Промт обогащается релевантными фрагментами кода из:
- Деклараций (объявления функций, классов)
- Недавно измененных файлов
- Недавно открытых файлов
- Поиска по кодовой базе

**Применение:** Вызывается при каждом запросе автодополнения в редакторе для построения контекстуализированного промта.

**Параметры:**
- `prompt_template`: Шаблон для модели (опционально, зависит от модели)
- Максимум 768 символов для сниппетов
- Минимум 256 символов зарезервировано для поиска по коду

**Формат промта без шаблона:**
```text
{prefix}
```

**Формат промта с шаблоном (пример CodeLlama):**
```text
<PRE> {prefix} <SUF>{suffix} <MID>
```

**Формат вставки сниппетов в prefix (для языков с комментариями):**
```python
# Path: {filepath_1}
# {snippet_body_line_1}
# {snippet_body_line_2}
#
# Path: {filepath_2}
# {snippet_body_line_3}
{original_prefix}
```

**Приоритет источников сниппетов:**
1. Declarations (наивысший приоритет)
2. Relevant snippets from changed files
3. Relevant snippets from recently opened files
4. Code search results

---

### 3.2. Next Edit Prediction промт

**Расположение:** `crates/tabby/src/services/completion/next_edit_prompt.rs:10-18`

**Назначение:** Предсказание следующего редактирования кода на основе истории изменений. Используется для предсказания паттернов редактирования и предложения следующего логического шага.

**Применение:** Вызывается когда доступна история редактирований файла для предсказания следующего изменения.

**Формат промта:**
```text
<|original_code|>
{original_code}
<|edits_diff|>
{edits_diff}
<|current_version|>
{current_version}
<|next_version|>
```

**Пример:**
```text
<|original_code|>
fn main() {
    println!("Hello, world!");
}
<|edits_diff|>
---src/main.rs
+++src/main.rs
@@ -1,1 +1,2 @@
    println!("Hello, world!");
    let x = 5;
    println!("Hello, world!");
<|current_version|>
fn main() {
    let x = 5;
    println!("Hello, world!");
}
<|next_version|>
```

---

### 3.3. Стандартный FIM шаблон для HTTP API

**Расположение:** `crates/http-api-bindings/src/completion/mod.rs:59-60`

**Назначение:** Базовый FIM шаблон для работы с внешними completion API (OpenAI, Mistral и др.). Используется когда модель поддерживает FIM inference через HTTP API.

**Применение:** Отправляется как часть запроса к внешним API для автодополнения.

**Формат:**
```text
{prefix}<|FIM|>{suffix}
```

---

## 4. Промты для генерации документации (Page Generation)

### 4.1. Генерация заголовка страницы

**Расположение:** `ee/tabby-webserver/src/service/page/prompt_tools.rs:3-11`

**Назначение:** Генерирует краткий заголовок для страницы документации на основе предоставленного контента или всей беседы. Используется для автоматического именования страниц.

**Применение:** Вызывается при создании новой страницы документации или при необходимости изменить заголовок.

**Формат промта (с предложенным заголовком):**
```text
Please help me to generate a page title for the input provided: {title}
Please only generate the title and nothing else. Do not include any additional text.
```

**Формат промта (без предложенного заголовка):**
```text
Summarize the above conversation and create a succinct title that encapsulates its essence. Please only generate the title and nothing else. Do not include any additional text or context.
Please only generate the title and nothing else. Do not include any additional text.
```

---

### 4.2. Генерация введения страницы

**Расположение:** `ee/tabby-webserver/src/service/page/prompt_tools.rs:13-29`

**Назначение:** Создает вводный параграф для страницы документации на основе заголовка страницы и списка подразделов.

**Применение:** Вызывается после определения структуры страницы для генерации вводного текста.

**Формат промта:**
```text
You're writing the intro section of a page named "{title}". It contains the following sub sections:
{page_section_titles}.

Here're some rules you need to follow when creating content:
* Please generate the content for the introduction section based on the information provided above.
* Ensure the content is a single paragraph without any subtitles or nested sections.
* Do not just blindly create a intro section listing all sub section, you should give a high level overview of the page, e.g background, why it's important, etc.
* Include code snippets if necessary, but keep them concise and relevant.
* Do not include any additional text.
```

---

### 4.3. Генерация заголовков разделов

**Расположение:** `ee/tabby-webserver/src/service/page/prompt_tools.rs:52-73`

**Назначение:** Генерирует заголовки для новых разделов страницы документации на основе существующего контента и темы нового раздела.

**Применение:** Вызывается при добавлении новых разделов к странице документации.

**Формат промта:**
```text
Here's some existing writings about the page.
=== Page Content Start ===
# {title}

{existing_sections_with_content}
=== Page Content End ===

{optional_new_section_description}Please generate {count} section titles for the page based on above information.
Please only generate the section title and nothing else. Do not include any additional text.
Each section title should be on a new line.
There's no need to have a intro section, as page will contains a intro section anyway.
```

---

### 4.4. Генерация контента раздела

**Расположение:** `ee/tabby-webserver/src/service/page/prompt_tools.rs:75-92`

**Назначение:** Создает содержимое для конкретного раздела страницы документации, учитывая контекст предыдущих разделов.

**Применение:** Вызывается при написании контента для каждого раздела страницы.

**Формат промта:**
```text
Here's some existing writings about the page.
=== Page Content Start ===
# {title}

{existing_sections_with_content}
=== Page Content End ===

The current section title is: {new_section_title}, please create content for this section based on above information.

Here're some rules you need to follow when creating content:
* Try not repeat content / pattern from the previous sections as much as possible.
* Ensure the content is a single paragraph without any subtitles or nested sections.
* Do not include any additional output, just write the content directly.
* Include code snippets if necessary, but keep them concise and relevant, and try refer to the code snippets in the previous sections if possible (instead of creating new ones).
```

---

## 5. Промты для анализа репозитория

### 5.1. Генерация вопросов для онбординга в репозиторий

**Расположение:** `ee/tabby-webserver/src/service/repository/prompt_tools.rs:27-44`

**Назначение:** Генерирует 3 специфических вопроса, помогающих новому разработчику понять структуру и организацию кодовой базы на основе дерева файлов репозитория.

**Применение:** Вызывается при просмотре структуры репозитория для помощи в онбординге новых разработчиков.

**Формат промта:**
```text
You are a helpful assistant that helps the user to ask related questions about a codebase "{repository_name}".

Here is codebase directory structure:
Type: {type}, Path: {path}
Type: {type}, Path: {path}
...
{optional_truncation_notice}

Please generate 3 specific questions to help a new engineer understand this codebase.

Each question should be concise (max 10 words) and focused. Return only the questions, one per line.
```

**Пример структуры файлов в промте:**
```text
Type: dir, Path: src/
Type: file, Path: src/main.rs
Type: dir, Path: tests/
Type: file, Path: README.md
Note: The file list has been truncated. There may be more files in subdirectories that were not included due to the limit.
```

---

