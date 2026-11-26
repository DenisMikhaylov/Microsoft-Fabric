Ниже представлена подробная лабораторная работа, в которой источник данных создаётся с нуля — вы загружаете CSV-файлы в Lakehouse, а затем строите на их основе Labeled Property Graph с помощью Neo4j Graph Intelligence в Microsoft Fabric.

---

# 🧪 Лабораторная работа: Построение и анализ Labeled Property Graph с Neo4j Graph Intelligence в Microsoft Fabric

**Продолжительность:** 120 минут  
**Уровень:** Средний  
**Среда:** Microsoft Fabric (Azure) + Neo4j Graph Intelligence (AuraDB Professional, пробный период)



---

## Этап 0. Создание источника данных: загрузка CSV в Lakehouse (20 минут)

На этом этапе мы **создаём источник данных с нуля** — загружаем CSV-файлы в Lakehouse в OneLake. Это те самые данные, которые затем будут превращены в граф.

### 0.1. Создайте Lakehouse

1. В вашей рабочей области Fabric выберите **+ New item → Lakehouse**.
2. Назовите Lakehouse `RecommendationsLH` и нажмите **Create**.
3. Убедитесь, что Lakehouse создан **с поддержкой схем (schema-enabled)** — это требование Neo4j Graph Intelligence.

### 0.2. Скачайте CSV-файлы

Скачайте набор данных для сценария розничных рекомендаций из репозитория Neo4j:

**`https://github.com/neo4j-product-examples/aura-graph-analytics/tree/main/better_recommendations/data`**

В репозитории находятся пять CSV-файлов:

| Файл | Содержимое |
| :--- | :--- |
| `order_history.csv` | История заказов: `user_id`, `order_id` |
| `baskets.csv` | Состав заказов: `order_id`, `product_id` |
| `products.csv` | Продукты: `product_id`, `product_name`, `aisle_id` |
| `aisles.csv` | Отделы магазина: `aisle_id`, `aisle` |
| `departments.csv` | Департаменты: `department_id`, `department` |

Эти данные описывают розничные покупки: пользователи размещают заказы, заказы содержат продукты, продукты относятся к отделам и департаментам.

### 0.3. Загрузите CSV в Lakehouse

1. Откройте Lakehouse `RecommendationsLH`.
2. Нажмите **Upload files** и загрузите все пять CSV-файлов.
3. После загрузки **преобразуйте каждый CSV в таблицу** (щелчок правой кнопкой → **Load to table**). Убедитесь, что в разделе **Tables** появились пять таблиц: `order_history`, `baskets`, `products`, `aisles`, `departments`.

> 💡 **Почему именно этот набор данных?** Он естественным образом образует граф: `User` → `Order` → `Product` → `Aisle` → `Department`. Это позволяет продемонстрировать все ключевые концепции LPG — узлы, метки, свойства и связи, включая связи со свойствами (например, количество товара в заказе).

---

## Этап 1. Создание LPG: AI-моделирование и трансформация (20 минут)

### 1.1. Создайте Neo4j Graph Dataset

1. В рабочей области Fabric выберите **+ New item**.
2. Найдите и выберите **Neo4j Graph Dataset**.
3. Задайте имя, например `RetailGraphLab`.
4. Нажмите **Create**.

### 1.2. Выберите источник данных

1. В процессе настройки выберите **New Graph Dataset** (поскольку данные уже есть в OneLake).
2. Укажите Lakehouse `RecommendationsLH`.
3. Выберите **все пять таблиц** и нажмите **Next**.

### 1.3. Запустите AI-моделирование

1. Fabric запустит **AI-ассистированное моделирование графа**. Система автоматически предложит модель на основе структуры таблиц.
2. Изучите предложенную модель. Для данного набора данных ожидаемая модель выглядит так:

**Узлы (Nodes):**

| Label (метка) | Свойства | Ключ |
| :--- | :--- | :--- |
| `User` | `user` | `user_id` |
| `Order` | `Order` | `order_id` |
| `Product` | `productId`, `name` | `product_id` |
| `Aisle` | `aisleId`, `name` | `aisle_id` |
| `Department` | `departmentId`, `department` | `department_id` |

**Связи (Relationships):**

| Тип связи | От узла | К узлу | Источник |
| :--- | :--- | :--- | :--- |
| `PLACED` | `User` | `Order` | `order_history` |
| `CONTAINS` | `Order` | `Product` | `baskets` |
| `IN_AISLE` | `Product` | `Aisle` | `products` |
| `IN_DEPARTMENT` | `Aisle` | `Department` | `products` |

> 💡 **Совет:** Убедитесь, что все ключевые свойства (ключи) имеют **целочисленный тип (`integer`)** — это важно для корректного связывания данных.

### 1.4. Отредактируйте модель (при необходимости)

Вы можете использовать **point-and-click интерфейс**, чтобы переименовать узлы или связи, добавить/удалить свойства или изменить направление связи.

### 1.5. Выполните трансформацию

1. Нажмите **Transform to graph**.
2. Дождитесь завершения пяти шагов трансформации. Это может занять несколько минут.
3. Вы получите уведомление по электронной почте, когда задача будет выполнена.

> **Что произошло?** Вы создали **Labeled Property Graph**, где:
> - **Узлы** — это экземпляры сущностей (конкретный пользователь, заказ, продукт).
> - **Метки** — это типы узлов (`User`, `Order`, `Product`).
> - **Свойства** — атрибуты узлов (`user`, `name`, `price`).
> - **Связи** — типизированные направленные рёбра (`PLACED`, `CONTAINS`).

---

## Этап 2. Загрузка данных и создание/проверка связей (15 минут)

После трансформации данные загружены, и связи созданы автоматически. Однако в реальных сценариях часто требуется **добавить или скорректировать связи** вручную с помощью Cypher.

### 2.1. Проверьте загруженные данные

Откройте вкладку **Query** (или **Cypher**) в интерфейсе Neo4j Graph Dataset и выполните:

```cypher
MATCH (n)
RETURN labels(n) AS NodeType, count(*) AS Count
ORDER BY Count DESC
```

Вы увидите количество узлов каждого типа, загруженных из OneLake.

### 2.2. Проверьте связи

```cypher
MATCH ()-[r]->()
RETURN type(r) AS RelationshipType, count(*) AS Count
ORDER BY Count DESC
```

Убедитесь, что присутствуют связи `PLACED`, `CONTAINS`, `IN_AISLE`, `IN_DEPARTMENT`.

### 2.3. Создайте связь вручную (пример для сценария рекомендаций)

Допустим, мы хотим создать связь `BOUGHT` между пользователем и продуктом напрямую (пропуская заказ). Это полезно для быстрых запросов рекомендаций:

```cypher
MATCH (u:User)-[:PLACED]->(:Order)-[:CONTAINS]->(p:Product)
MERGE (u)-[:BOUGHT]->(p)
```

**Разбор:**
- `MATCH` находит пары «пользователь — продукт» через цепочку заказов.
- `MERGE` создаёт связь `BOUGHT`, **только если её ещё нет** (в отличие от `CREATE`, который создал бы дубликат).

Проверьте результат:

```cypher
MATCH (u:User)-[:BOUGHT]->(p:Product)
RETURN u.user, p.name
LIMIT 10
```

> **Ключевой принцип LPG:** связи — это **полноценные объекты данных**, которые можно создавать, обновлять и удалять так же, как узлы. Это фундаментальное отличие от реляционной модели, где связь — это лишь результат `JOIN`.

---

## Этап 3. Выполнение базовых Cypher-запросов (25 минут)

Теперь, когда граф построен, исследуем его с помощью **Cypher**.

### 3.1. Простой запрос: найти пользователя и его заказы

```cypher
MATCH (u:User {user: 'Alice'})-[:PLACED]->(o:Order)
RETURN u.user, o.Order
LIMIT 10
```

**Разбор:** паттерн `(u)-[:PLACED]->(o)` описывает связь «пользователь разместил заказ». Запрос найдёт все заказы Alice.

> 💡 **Адаптация:** Если в ваших данных нет пользователя `Alice`, найдите существующего пользователя запросом `MATCH (u:User) RETURN u.user LIMIT 5` и подставьте его имя в запрос.

### 3.2. Запрос с агрегацией: сколько заказов у каждого пользователя

```cypher
MATCH (u:User)-[:PLACED]->(o:Order)
RETURN u.user, count(o) AS OrderCount
ORDER BY OrderCount DESC
LIMIT 10
```

### 3.3. Запрос со свойствами связи: какие продукты и в каком количестве входят в заказы

```cypher
MATCH (:Order)-[c:CONTAINS]->(p:Product)
RETURN p.name, count(c) AS TimesOrdered
ORDER BY TimesOrdered DESC
LIMIT 10
```

> 💡 **Свойства связей:** Если в таблице `baskets` есть колонка `quantity`, она может быть привязана как свойство связи `CONTAINS`. Тогда запрос `sum(c.quantity)` покажет общее количество проданных единиц.

### 3.4. Запрос с фильтрацией через WITH

```cypher
MATCH (:Order)-[c:CONTAINS]->(p:Product)
WITH p, count(c) AS TimesOrdered
WHERE TimesOrdered > 5
RETURN p.name, TimesOrdered
ORDER BY TimesOrdered DESC
```

**Разбор:** `WITH` используется как «конвейер» для фильтрации **после агрегации**. Без `WITH` нельзя было бы отфильтровать по `TimesOrdered`.

### 3.5. Text2Cypher (опционально)

Нажмите **Ctrl + I** (или используйте AI-панель) и введите на естественном языке:
> *«Покажи мне 5 самых популярных продуктов и сколько раз каждый из них заказывали»*

Text2Cypher сгенерирует Cypher-запрос автоматически. Сравните его с запросом, который вы написали бы вручную.

---

## Этап 4. Поиск путей и связанных сущностей (20 минут)

### 4.1. Поиск кратчайшего пути между двумя пользователями через общие продукты

```cypher
MATCH (u1:User {user: 'Alice'}), (u2:User {user: 'Bob'})
MATCH path = shortestPath((u1)-[:BOUGHT*..5]-(u2))
RETURN path, length(path) AS DegreesOfSeparation
```

**Разбор:** `shortestPath` находит **кратчайший путь** в графе связей `BOUGHT` (до 5 шагов). Результат покажет, через сколько «рукопожатий» (общих продуктов) связаны Alice и Bob.

> 💡 **Адаптация:** Если пользователей `Alice` и `Bob` нет в данных, выберите двух существующих пользователей.

### 4.2. Поиск всех продуктов, связанных с пользователем через цепочку «заказ → отдел → департамент»

```cypher
MATCH (u:User {user: 'Alice'})-[:PLACED]->(:Order)-[:CONTAINS]->(p:Product)
      -[:IN_AISLE]->(a:Aisle)-[:IN_DEPARTMENT]->(d:Department)
RETURN DISTINCT d.department AS Department, collect(p.name) AS Products
```

Этот запрос выполняет **многошаговый обход** (5 шагов) и агрегирует продукты по департаментам.

### 4.3. Поиск «похожих пользователей» (купивших те же продукты, но не напрямую связанных)

```cypher
MATCH (alice:User {user: 'Alice'})-[:BOUGHT]->(p:Product)<-[:BOUGHT]-(other:User)
WHERE other <> alice
RETURN other.user, count(p) AS SharedProducts
ORDER BY SharedProducts DESC
LIMIT 10
```

> **Почему это эффективно?** Благодаря **index-free adjacency** (в Neo4j) переход по связи не зависит от общего размера графа. Обход на 3–5 шагов выполняется за миллисекунды, тогда как в реляционной БД каждый шаг добавлял бы новый `JOIN` и увеличивал стоимость запроса экспоненциально.

---

## Этап 5. Аналитические результаты с помощью GDS-алгоритмов (15 минут)

Neo4j Graph Intelligence предоставляет доступ к **графовым алгоритмам** через библиотеку **Graph Data Science (GDS)**. Их можно запускать прямо из Query-интерфейса Fabric с помощью Cypher-процедур.

### 5.1. Спроецируйте граф в память

```cypher
CALL gds.graph.project(
  'retailGraph',
  ['User', 'Product'],
  { BOUGHT: { orientation: 'UNDIRECTED' } }
)
```

**Разбор:** `gds.graph.project` создаёт именованную проекцию `retailGraph`, включающую только узлы `User` и `Product` и связи `BOUGHT` (ненаправленные). Это ускоряет алгоритмы, так как они работают с подмножеством графа.

### 5.2. PageRank: найти самых влиятельных пользователей

```cypher
CALL gds.pageRank.stream('retailGraph', {
  maxIterations: 20,
  dampingFactor: 0.85
})
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).user AS User, score
ORDER BY score DESC
LIMIT 10
```

**Что показывает:** PageRank оценивает **влиятельность** узла на основе структуры связей. Пользователи с высоким PageRank — это «хабы», связанные с многими другими пользователями (потенциальные лидеры мнений или ключевые клиенты).

### 5.3. Louvain: обнаружение сообществ (кластеров пользователей)

```cypher
CALL gds.louvain.stream('retailGraph')
YIELD nodeId, communityId
RETURN communityId, collect(gds.util.asNode(nodeId).user) AS Members, count(*) AS Size
ORDER BY Size DESC
```

**Что показывает:** Louvain **группирует пользователей в сообщества** на основе плотности связей. Это может выявить группы клиентов со схожими вкусами — основа для **персонализированных рекомендаций**.

### 5.4. Node Similarity: найти похожих пользователей

```cypher
CALL gds.nodeSimilarity.stream('retailGraph')
YIELD node1, node2, similarity
RETURN gds.util.asNode(node1).user AS User1, gds.util.asNode(node2).user AS User2, similarity
ORDER BY similarity DESC
LIMIT 10
```

**Что показывает:** Алгоритм вычисляет **степень сходства** между пользователями на основе общих связей. Высокое сходство означает, что пользователи, вероятно, имеют схожие предпочтения.

### 5.5. Очистка

После завершения анализа удалите проекцию, чтобы освободить память:

```cypher
CALL gds.graph.drop('retailGraph')
```


4. **Cypher — декларативный язык паттерн-матчинга**. `MATCH` описывает искомую структуру, а не последовательность шагов.
5. **GDS-алгоритмы** (PageRank, Louvain, Node Similarity) превращают граф в источник аналитических инсайтов, которые **почти невозможно получить с помощью SQL** или традиционного BI.
6. **Text2Cypher** снижает порог входа: бизнес-пользователи могут формулировать вопросы на естественном языке, а система генерирует корректный Cypher.
