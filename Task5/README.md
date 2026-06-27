# Проектирование GraphQL API для сервиса client-info

## 1. Анализ существующего REST API

### Ключевые ресурсы и операции

| Эндпоинт | Операция | Возвращает |
|---|---|---|
| `GET /clients/{id}` | Получить клиента по ID | `Client` |
| `GET /clients/{id}/documents` | Список документов клиента | `[Document]` |
| `GET /clients/{id}/relatives` | Список родственников клиента | `[Relative]` |

### Сущности контракта

- **Client**: `id`, `name`, `age`
- **Document**: `id`, `type`, `number`, `issueDate`, `expiryDate`
- **Relative**: `id`, `relationType`, `name`, `age`

### Проблемы текущего REST-подхода

1. **N+1 запросов на сценарий.** Для отображения карточки клиента с документами и родственниками нужно минимум 3 отдельных HTTP-запроса → рост RPS на `client-info`.
2. **Over-fetching.** Каждый ресурс отдаёт все свои поля целиком, даже если потребителю нужны 2–3 атрибута из 500.
3. **Under-fetching.** Нет единого ресурса для всех данных — потребитель вынужден делать множество вызовов.
4. **Жёсткость.** Любой новый сценарий с другим набором данных требует либо новых ресурсов, либо нескольких вызовов существующих.

GraphQL решает все четыре проблемы: клиент в **одном запросе** запрашивает **ровно те поля**, которые ему нужны.

---

## 2. GraphQL-схема

```graphql
"""
Клиент — корневая сущность карточки.
Поля documents и relatives загружаются только если запрошены потребителем.
"""
type Client {
  id: ID!
  name: String!
  age: Int

  "Документы клиента (загружаются по требованию)"
  documents: [Document!]!

  "Родственники клиента (загружаются по требованию)"
  relatives: [Relative!]!
}

"Документ клиента (паспорт, СНИЛС, водительское удостоверение и т.д.)"
type Document {
  id: ID!
  type: String!
  number: String!
  issueDate: String
  expiryDate: String
}

"Родственник клиента"
type Relative {
  id: ID!
  relationType: String!
  name: String!
  age: Int
}

type Query {
  "Получить клиента по ID. Эквивалент GET /clients/{id} + вложенные ресурсы."
  client(id: ID!): Client

  "Получить документы конкретного клиента. Эквивалент GET /clients/{id}/documents."
  clientDocuments(clientId: ID!): [Document!]!

  "Получить родственников конкретного клиента. Эквивалент GET /clients/{id}/relatives."
  clientRelatives(clientId: ID!): [Relative!]!
}
```

---

## 3. Соответствие REST → GraphQL

| REST | GraphQL |
|---|---|
| `GET /clients/{id}` | `query { client(id: "...") { id name age } }` |
| `GET /clients/{id}/documents` | `query { clientDocuments(clientId: "...") { ... } }` или `client(id) { documents { ... } }` |
| `GET /clients/{id}/relatives` | `query { clientRelatives(clientId: "...") { ... } }` или `client(id) { relatives { ... } }` |

Вложенные поля `documents` и `relatives` внутри `Client` покрывают сценарий «всё в одном запросе», а отдельные query `clientDocuments` / `clientRelatives` сохраняют возможность точечного запроса (паритет с исходным REST).

---

## 4. Преимущества перехода

| Проблема REST | Решение в GraphQL |
|---|---|
| N+1 запросов на сценарий | Один запрос с вложенными полями |
| Over-fetching (500 атрибутов) | Клиент выбирает только нужные поля |
| Under-fetching | Вложенные связи в одном запросе |
| Рост RPS на client-info | Снижение числа запросов в разы |
| Жёсткость под новые сценарии | Новый сценарий = новый набор полей без изменения API |
