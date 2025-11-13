# Tabby Agent Pipelines Documentation

Это полная документация всех пайплайнов и схем работы AI агентов в приложении Tabby. Каждый пайплайн описан с указанием потока данных, используемых промтов и форматов вывода.

---

## 1. Answer/Chat Pipeline - Пайплайн ответов на вопросы

### Назначение
Обработка вопросов пользователя с использованием RAG (Retrieval-Augmented Generation) для предоставления ответов с контекстом из кодовой базы и документации.

### Точка входа
`ee/tabby-webserver/src/service/answer.rs:67` - метод `AnswerService::answer()`

### Схема потока данных

```
┌─────────────────────────────────────────────────────────────────┐
│ ВХОД: User Message + Conversation History + Options             │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│ ШАГ 1: Определение необходимости контекста из кодовой базы    │
│ Файл: answer.rs:93-133                                         │
│ Промт: prompt_tools.rs:39-66 (decide_context_prompt)          │
├────────────────────────────────────────────────────────────────┤
│ → LLM анализирует вопрос                                       │
│ → Определяет нужны ли: file_list и/или snippet                │
│ → Возвращает: { snippet: bool, file_list: bool }              │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│ ШАГ 2: Сбор контекста из кода (если необходимо)               │
│ Файл: answer.rs:93-133                                         │
├────────────────────────────────────────────────────────────────┤
│ Если file_list == true:                                        │
│   → RetrievalService::collect_file_list()                      │
│   → Максимум 300 файлов                                        │
│                                                                 │
│ Если snippet == true:                                          │
│   → RetrievalService::collect_relevant_code()                  │
│   → Поиск с использованием BM25 + embeddings (RRF scoring)    │
│   → Возвращает релевантные фрагменты кода                     │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│ ШАГ 3: Сбор контекста из документации                         │
│ Файл: answer.rs:136-173                                        │
├────────────────────────────────────────────────────────────────┤
│ → RetrievalService::collect_relevant_docs()                    │
│ → Источники: Issues, PRs, Commits, Pages, Web Docs            │
│ → Поиск через Tantivy index + опционально Serper (web)        │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│ ШАГ 4: Генерация связанных вопросов (опционально)             │
│ Файл: answer.rs:176-191                                        │
│ Промт: prompt_tools.rs:16-31 (related_questions_prompt)       │
├────────────────────────────────────────────────────────────────┤
│ → LLM генерирует 3 follow-up вопроса                          │
│ → Каждый вопрос максимум 20 слов                              │
│ → Возвращает: JSON массив вопросов                            │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│ ШАГ 5: Формирование финального запроса к LLM                  │
│ Файл: answer.rs:194-214                                        │
│ Промт: utils/mod.rs:129-213 (build_user_prompt)               │
├────────────────────────────────────────────────────────────────┤
│ → Добавление системного промта (личность Tabby)               │
│ → Форматирование пользовательского промта с:                  │
│   • Список файлов (если есть)                                  │
│   • Фрагменты кода с цитированием [[citation:x]]             │
│   • Релевантные документы                                      │
│   • Вопрос пользователя                                        │
│   • Инструкции по формату ответа                              │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│ ШАГ 6: Потоковая генерация ответа                             │
│ Файл: answer.rs:216-245                                        │
├────────────────────────────────────────────────────────────────┤
│ → chat.chat_stream(request)                                    │
│ → Стриминг ответа по частям (deltas)                          │
│ → Эмиссия событий ThreadRunItem                               │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│ ВЫХОД: Stream of ThreadRunItem Events                          │
├────────────────────────────────────────────────────────────────┤
│ • ThreadAssistantMessageReadingCode                            │
│ • ThreadAssistantMessageAttachmentsCodeFileList                │
│ • ThreadAssistantMessageAttachmentsCode                        │
│ • ThreadAssistantMessageAttachmentsDoc                         │
│ • ThreadRelevantQuestions                                      │
│ • ThreadAssistantMessageContentDelta (streaming content)       │
│ • ThreadAssistantMessageCompleted                              │
└────────────────────────────────────────────────────────────────┘
```

### Используемые промты

1. **Системный промт** (`config.rs:467`)
   - Определяет личность Tabby
   - Отправляется как system message

2. **Промт определения контекста** (`prompt_tools.rs:39-66`)
   - Определяет нужен ли snippet и/или file_list
   - Возвращает JSON: `{"snippet": bool, "file_list": bool}`

3. **Промт построения пользовательского запроса** (`utils/mod.rs:129-213`)
   - Добавляет код-сниппеты с цитированием
   - Форматирует список файлов
   - Включает документы
   - Добавляет инструкции по формату ответа

4. **Промт генерации связанных вопросов** (`prompt_tools.rs:16-31`)
   - Генерирует 3 follow-up вопроса
   - Максимум 20 слов каждый
   - Возвращает JSON массив

### Потоки данных между шагами

```
User Message → Decide Context → { snippet: true, file_list: false }
                ↓
Code Search (BM25+Embeddings) → [CodeHit, CodeHit, ...]
                ↓
Doc Search (Tantivy) → [DocHit, DocHit, ...]
                ↓
Build Prompt → "Here are some relevant code snippets:\n```rust\n..."
                ↓
LLM Stream → "Based on the code..." (deltas)
                ↓
Related Questions → ["How does...", "What is...", "Where can..."]
```

### Форматы данных

**Входные данные:**
```rust
struct AnswerRequest {
    messages: Vec<Message>,
    options: AnswerOptions {
        code_query: Option<CodeQuery>,
        doc_query: Option<DocQuery>,
        generate_relevant_questions: bool,
    }
}
```

**Выходные события (stream):**
```rust
enum ThreadRunItem {
    ThreadAssistantMessageReadingCode,
    ThreadAssistantMessageAttachmentsCodeFileList { file_list: Vec<String> },
    ThreadAssistantMessageAttachmentsCode { hits: Vec<CodeSearchHit> },
    ThreadAssistantMessageAttachmentsDoc { hits: Vec<DocSearchHit> },
    ThreadRelevantQuestions { questions: Vec<String> },
    ThreadAssistantMessageContentDelta { delta: String },
    ThreadAssistantMessageCompleted { id: String },
}
```

### Оптимизации и кэширование

- **Поиск по коду:** Использует RRF (Reciprocal Rank Fusion) для объединения BM25 и embedding scores
- **Лимиты:** Максимум 300 файлов в file_list
- **Цитирование:** Система `[[citation:x]]` для отслеживания источников в ответе
- **Потоковая передача:** Ответ отправляется по мере генерации для улучшения UX

---

