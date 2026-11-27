Поскольку нативный движок Graph в Microsoft Fabric **не поддерживает RDF** (он работает исключительно с моделью LPG), для выполнения 
этой лабораторной работы мы будем использовать **Python-ноутбуки Fabric** с библиотеками `rdflib` и `owlready2`. Это позволит нам создавать 
RDF-тройки, строить графы, проектировать онтологии, применять ограничения SHACL, выполнять SPARQL-запросы и проверять данные — 
и всё это в среде Azure Fabric, используя данные из OneLake.

---

# 🧪 Лабораторная работа: Создание и анализ семантического графа RDF в Microsoft Fabric




---

## Этап 0. Подготовка среды: установка библиотек и настройка ноутбука (10 минут)

### 0.1. Создайте Python-ноутбук в Fabric

1. В вашей рабочей области Fabric выберите **+ New item → Notebook**.
2. Назовите ноутбук `RDF_Semantic_Graph_Lab`.
3. Убедитесь, что язык ноутбука — **Python** (переключатель в верхней панели).

### 0.2. Установите необходимые библиотеки

В первой ячейке выполните установку библиотек. `rdflib` — для работы с RDF и SPARQL, `owlready2` — для онтологий OWL, `pyshacl` — для валидации SHACL.

```python
%pip install rdflib==7.1.1 owlready2==0.47 pyshacl==0.30.1
```

> 💡 **Примечание:** `rdflib` — это стандартная Python-библиотека для работы с RDF. `owlready2` позволяет загружать и манипулировать OWL-онтологиями. `pyshacl` — это валидатор SHACL.

### 0.3. Импортируйте библиотеки и настройте пространства имён

```python
from rdflib import Graph, Namespace, Literal, URIRef, RDF, RDFS, OWL, XSD
from rdflib.namespace import FOAF, SKOS
from owlready2 import get_ontology
from pyshacl import validate
import pandas as pd

# Определяем пространства имён для нашей онтологии
EX = Namespace("http://example.org/lab/")
SCHEMA = Namespace("https://schema.org/")

print("Библиотеки успешно импортированы!")
```

> 💡 **Почему именно эти пространства имён?** `EX` — это ваше собственное пространство имён для лабораторной работы. `SCHEMA` и `FOAF` — это широко используемые публичные словари, которые мы будем переиспользовать. Это соответствует лучшей практике RDF — не изобретать новые термины там, где уже есть стандартные.

---

## Этап 1. Создание RDF Triples и построение RDF Graph (20 минут)

### 1.1. Создайте пустой RDF-граф

```python
# Создаём пустой RDF-граф
g = Graph()

# Привязываем префиксы для удобства сериализации
g.bind("ex", EX)
g.bind("schema", SCHEMA)
g.bind("foaf", FOAF)

print("Пустой RDF-граф создан.")
```

### 1.2. Добавьте первые тройки вручную

**RDF-тройка** — это утверждение «субъект — предикат — объект». Каждая тройка добавляется отдельно. Создадим информацию о человеке по имени Alice.

```python
# Тройка 1: Alice — это Person (используем schema.org)
g.add((EX.Alice, RDF.type, SCHEMA.Person))

# Тройка 2: у Alice есть имя "Alice"
g.add((EX.Alice, SCHEMA.name, Literal("Alice")))

# Тройка 3: Alice знает Bob (используем FOAF)
g.add((EX.Alice, FOAF.knows, EX.Bob))

# Тройка 4: Bob — это Person
g.add((EX.Bob, RDF.type, SCHEMA.Person))

# Тройка 5: у Bob есть имя "Bob"
g.add((EX.Bob, SCHEMA.name, Literal("Bob")))

print(f"Добавлено троек: {len(g)}")
```

### 1.3. Проверьте и визуализируйте граф

```python
# Вывод всех троек в читаемом формате
print("Содержимое RDF-графа:")
for s, p, o in g:
    print(f"  {s.n3(g.namespace_manager)} {p.n3(g.namespace_manager)} {o.n3(g.namespace_manager)}")
```

**Ожидаемый вывод:**
```
Содержимое RDF-графа:
  ex:Alice schema:name "Alice"
  ex:Alice rdf:type schema:Person
  ex:Alice foaf:knows ex:Bob
  ex:Bob schema:name "Bob"
  ex:Bob rdf:type schema:Person
```

> 💡 **Ключевой момент:** В RDF нет «таблиц» или «колонок». Есть только **множество троек**. Каждая тройка — это независимое утверждение. Именно поэтому RDF-граф называют «самоописываемым»: он содержит не только данные, но и их типы (`rdf:type`).

---

## Этап 2. Создание онтологии (T-Box): классы, свойства, иерархии (25 минут)

**Онтология (T-Box)** — это формальное описание понятий предметной области и их взаимосвязей. Мы создадим онтологию для сценария «Компания»: сотрудники, отделы, проекты.

### 2.1. Определите классы и иерархию

```python
# Создаём отдельный граф для онтологии (T-Box)
onto = Graph()
onto.bind("ex", EX)
onto.bind("rdfs", RDFS)
onto.bind("owl", OWL)

# Определяем классы
onto.add((EX.Person, RDF.type, OWL.Class))
onto.add((EX.Employee, RDF.type, OWL.Class))
onto.add((EX.Manager, RDF.type, OWL.Class))
onto.add((EX.Department, RDF.type, OWL.Class))
onto.add((EX.Project, RDF.type, OWL.Class))

# Определяем иерархию: Employee — это Person, Manager — это Employee
onto.add((EX.Employee, RDFS.subClassOf, EX.Person))
onto.add((EX.Manager, RDFS.subClassOf, EX.Employee))

print("Классы и иерархия определены.")
```

### 2.2. Определите свойства (Object Properties и Datatype Properties)

```python
# Object Properties (связи между сущностями)
onto.add((EX.worksFor, RDF.type, OWL.ObjectProperty))
onto.add((EX.worksFor, RDFS.domain, EX.Employee))
onto.add((EX.worksFor, RDFS.range, EX.Department))

onto.add((EX.manages, RDF.type, OWL.ObjectProperty))
onto.add((EX.manages, RDFS.domain, EX.Manager))
onto.add((EX.manages, RDFS.range, EX.Department))

onto.add((EX.worksOn, RDF.type, OWL.ObjectProperty))
onto.add((EX.worksOn, RDFS.domain, EX.Employee))
onto.add((EX.worksOn, RDFS.range, EX.Project))

# Datatype Properties (атрибуты-литералы)
onto.add((EX.name, RDF.type, OWL.DatatypeProperty))
onto.add((EX.name, RDFS.domain, EX.Person))
onto.add((EX.name, RDFS.range, XSD.string))

onto.add((EX.age, RDF.type, OWL.DatatypeProperty))
onto.add((EX.age, RDFS.domain, EX.Person))
onto.add((EX.age, RDFS.range, XSD.integer))

onto.add((EX.projectName, RDF.type, OWL.DatatypeProperty))
onto.add((EX.projectName, RDFS.domain, EX.Project))
onto.add((EX.projectName, RDFS.range, XSD.string))

print("Свойства определены.")
```

### 2.3. Добавьте ограничения (аксиомы OWL)

OWL позволяет задавать **логические ограничения** на классы. Например, можно указать, что у каждого сотрудника должен быть хотя бы один отдел, или что некоторые классы несовместимы.

```python
# Ограничение: Employee и Manager не могут быть одним и тем же (дизъюнкция)
onto.add((EX.Employee, OWL.disjointWith, EX.Manager))

# Ограничение: Manager должен управлять хотя бы одним Department
# (используем owl:Restriction через blank node)
from rdflib import BNode
restriction = BNode()
onto.add((restriction, RDF.type, OWL.Restriction))
onto.add((restriction, OWL.onProperty, EX.manages))
onto.add((restriction, OWL.someValuesFrom, EX.Department))
onto.add((EX.Manager, RDFS.subClassOf, restriction))

print("Ограничения OWL добавлены.")
```

> ⚠️ **Важно:** OWL-ограничения — это **аксиомы логики**, а не «валидаторы». Они позволяют reasoner'у выводить новые факты, но не проверяют данные в смысле «валидно/невалидно». Для валидации данных используется SHACL (см. Этап 4).

---

## Этап 3. Загрузка данных (A-Box) и связывание с онтологией (15 минут)

**A-Box (assertional box)** — это факты о конкретных индивидах. Мы создадим экземпляры наших классов и свяжем их с онтологией.

### 3.1. Создайте индивидов (экземпляры классов)

```python
# Используем тот же граф g, что и в Этапе 1, или создаём новый
data = Graph()
data.bind("ex", EX)
data.bind("schema", SCHEMA)

# Создаём сотрудников
data.add((EX.Alice, RDF.type, EX.Employee))
data.add((EX.Alice, EX.name, Literal("Alice")))
data.add((EX.Alice, EX.age, Literal(30, datatype=XSD.integer)))
data.add((EX.Alice, EX.worksFor, EX.EngineeringDept))
data.add((EX.Alice, EX.worksOn, EX.ProjectAlpha))

data.add((EX.Bob, RDF.type, EX.Manager))
data.add((EX.Bob, EX.name, Literal("Bob")))
data.add((EX.Bob, EX.age, Literal(45, datatype=XSD.integer)))
data.add((EX.Bob, EX.worksFor, EX.EngineeringDept))
data.add((EX.Bob, EX.manages, EX.EngineeringDept))

# Создаём отдел и проект
data.add((EX.EngineeringDept, RDF.type, EX.Department))
data.add((EX.EngineeringDept, EX.name, Literal("Engineering")))

data.add((EX.ProjectAlpha, RDF.type, EX.Project))
data.add((EX.ProjectAlpha, EX.projectName, Literal("Alpha")))

print(f"Загружено троек (A-Box): {len(data)}")
```

### 3.2. Объедините онтологию и данные в один граф

В RDF онтология (T-Box) и данные (A-Box) могут храниться **в одном графе**. Объединим их для дальнейшего анализа.

```python
# Объединяем онтологию и данные
combined = Graph()
combined.bind("ex", EX)
combined.bind("rdfs", RDFS)
combined.bind("owl", OWL)

for triple in onto:
    combined.add(triple)
for triple in data:
    combined.add(triple)

print(f"Объединённый граф содержит {len(combined)} троек.")
```

### 3.3. (Опционально) Сохраните граф в Lakehouse

Вы можете сериализовать граф в файл Turtle и сохранить его в Lakehouse для дальнейшего использования.

```python
# Сохраняем в файл Turtle
combined.serialize(destination="/lakehouse/default/Files/semantic_graph.ttl", format="turtle")
print("Граф сохранён в /lakehouse/default/Files/semantic_graph.ttl")
```

> 💡 **Зачем сохранять в Lakehouse?** Это позволяет использовать граф в других ноутбуках и интегрировать его с остальными данными в Fabric. Путь `/lakehouse/default/Files/` — это стандартный путь к файлам в привязанном Lakehouse.

---

## Этап 4. Применение ограничений SHACL и валидация (20 минут)

**SHACL (Shapes Constraint Language)** — это W3C-стандарт для **валидации RDF-данных** на основе **закрытого мира** (Closed World Assumption). В отличие от OWL, SHACL позволяет сказать: «у каждого Person **обязательно** должно быть имя».

### 4.1. Создайте SHACL-форму для класса Person

```python
from pyshacl import validate
from rdflib import Graph, Namespace, Literal, URIRef
from rdflib.namespace import SH, RDF, XSD

# Создаём граф для SHACL-формы
shacl_graph = Graph()
shacl_graph.bind("ex", EX)
shacl_graph.bind("sh", SH)

# Определяем форму для Person
person_shape = EX.PersonShape
shacl_graph.add((person_shape, RDF.type, SH.NodeShape))
shacl_graph.add((person_shape, SH.targetClass, EX.Person))

# Ограничение: name обязателен (minCount 1) и должен быть строкой
name_prop = EX.NamePropertyShape
shacl_graph.add((person_shape, SH.property, name_prop))
shacl_graph.add((name_prop, SH.path, EX.name))
shacl_graph.add((name_prop, SH.minCount, Literal(1)))
shacl_graph.add((name_prop, SH.datatype, XSD.string))

# Ограничение: age должен быть целым числом >= 0
age_prop = EX.AgePropertyShape
shacl_graph.add((person_shape, SH.property, age_prop))
shacl_graph.add((age_prop, SH.path, EX.age))
shacl_graph.add((age_prop, SH.datatype, XSD.integer))
shacl_graph.add((age_prop, SH.minInclusive, Literal(0)))

print("SHACL-форма для Person создана.")
```

### 4.2. Выполните валидацию данных

```python
# Валидируем данные (combined) с помощью SHACL-формы
conforms, results_graph, results_text = validate(
    combined,
    shacl_graph=shacl_graph,
    inference='rdfs',  # Включаем вывод RDFS для проверки иерархии
    abort_on_first=False,
    allow_infos=True,
    allow_warnings=True
)

print(f"Данные соответствуют SHACL-форме: {conforms}")
if not conforms:
    print("Нарушения:")
    print(results_text)
```

**Ожидаемый результат:** Поскольку у Alice и Bob есть имя и возраст, данные должны пройти валидацию (`conforms = True`).

### 4.3. Добавьте нарушение и проверьте, что SHACL его обнаружит

```python
# Создадим «плохие» данные: Person без имени
bad_data = Graph()
bad_data.add((EX.Charlie, RDF.type, EX.Person))
bad_data.add((EX.Charlie, EX.age, Literal(25, datatype=XSD.integer)))

# Валидируем «плохие» данные
conforms_bad, _, results_text_bad = validate(
    bad_data,
    shacl_graph=shacl_graph,
    inference='rdfs'
)

print(f"«Плохие» данные соответствуют SHACL-форме: {conforms_bad}")
if not conforms_bad:
    print("Обнаруженные нарушения:")
    print(results_text_bad)
```

**Ожидаемый результат:** `conforms_bad = False`, и SHACL сообщит, что у `ex:Charlie` отсутствует обязательное свойство `ex:name`.

> 💡 **Ключевой вывод:** SHACL — это **валидатор**, а не reasoner. Он проверяет данные на соответствие форме и сообщает о нарушениях. Именно поэтому SHACL используется для обеспечения качества данных, тогда как OWL — для логического вывода.

---

## Этап 5. Выполнение SPARQL-запросов (20 минут)

**SPARQL** — это W3C-стандарт языка запросов к RDF. Он позволяет формулировать вопросы к графу, используя паттерны троек с переменными.

### 5.1. Простой запрос: найти всех сотрудников и их отделы

```python
# SPARQL-запрос
query1 = """
PREFIX ex: <http://example.org/lab/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?employeeName ?departmentName
WHERE {
    ?employee a ex:Employee ;
              ex:name ?employeeName ;
              ex:worksFor ?dept .
    ?dept ex:name ?departmentName .
}
ORDER BY ?employeeName
"""

results = combined.query(query1)
print("Сотрудники и их отделы:")
for row in results:
    print(f"  {row.employeeName} работает в {row.departmentName}")
```

**Ожидаемый вывод:**
```
Сотрудники и их отделы:
  Alice работает в Engineering
  Bob работает в Engineering
```

### 5.2. Запрос с использованием иерархии классов (вывод RDFS)

Благодаря тому, что `Manager` является подклассом `Employee`, запрос по `ex:Employee` должен вернуть и `Manager` (если включён вывод). Проверим это.

```python
query2 = """
PREFIX ex: <http://example.org/lab/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?personName ?type
WHERE {
    ?person a ?type ;
            ex:name ?personName .
    ?type rdfs:subClassOf* ex:Person .
}
"""

results2 = combined.query(query2)
print("Все люди (включая подклассы):")
for row in results2:
    print(f"  {row.personName} — {row.type}")
```

**Ожидаемый вывод:** Alice и Bob должны появиться как `Employee` и `Manager` соответственно. Благодаря `rdfs:subClassOf*`, запрос находит все подклассы `Person`.

### 5.3. Запрос: найти всех, кто работает над проектом Alpha

```python
query3 = """
PREFIX ex: <http://example.org/lab/>

SELECT ?employeeName ?projectName
WHERE {
    ?employee ex:worksOn ?project ;
              ex:name ?employeeName .
    ?project ex:projectName ?projectName .
    FILTER (?projectName = "Alpha")
}
"""

results3 = combined.query(query3)
print("Сотрудники, работающие над проектом Alpha:")
for row in results3:
    print(f"  {row.employeeName} работает над {row.projectName}")
```

---

## Этап 6. Проверка и анализ графовых данных (10 минут)

### 6.1. Проверьте целостность графа (наличие обязательных троек)

```python
# Проверяем, что у каждого Person есть имя
query_check = """
PREFIX ex: <http://example.org/lab/>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>

SELECT ?person
WHERE {
    ?person a ex:Person .
    FILTER NOT EXISTS { ?person ex:name ?name }
}
"""

missing_names = combined.query(query_check)
missing_list = [str(row.person) for row in missing_names]
print(f"Люди без имени: {missing_list if missing_list else 'Нет — все данные корректны.'}")
```

### 6.2. Подсчитайте статистику по графу

```python
# Общее количество троек
print(f"Всего троек в графе: {len(combined)}")

# Количество экземпляров каждого класса
query_stats = """
PREFIX ex: <http://example.org/lab/>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>

SELECT ?class (COUNT(?instance) AS ?count)
WHERE {
    ?instance a ?class .
    FILTER (?class IN (ex:Person, ex:Employee, ex:Manager, ex:Department, ex:Project))
}
GROUP BY ?class
ORDER BY DESC(?count)
"""

stats = combined.query(query_stats)
print("\nСтатистика по классам:")
for row in stats:
    print(f"  {row['class']}: {row.count} экземпляров")
```

### 6.3. (Опционально) Визуализируйте граф

Вы можете использовать `networkx` и `matplotlib` для простой визуализации, или сохранить граф в формате, совместимом с инструментами визуализации (например, Graphviz).

```python
# Простая визуализация с networkx (если установлен)
try:
    import networkx as nx
    import matplotlib.pyplot as plt
    
    G = nx.DiGraph()
    for s, p, o in combined:
        if isinstance(o, URIRef):  # только связи между ресурсами
            G.add_edge(str(s).split('/')[-1], str(o).split('/')[-1], label=str(p).split('/')[-1])
    
    pos = nx.spring_layout(G)
    plt.figure(figsize=(10, 6))
    nx.draw(G, pos, with_labels=True, node_color='lightblue', node_size=2000, arrows=True)
    edge_labels = nx.get_edge_attributes(G, 'label')
    nx.draw_networkx_edge_labels(G, pos, edge_labels=edge_labels)
    plt.title("Визуализация RDF-графа")
    plt.show()
except ImportError:
    print("networkx не установлен. Пропускаем визуализацию.")
```


---

Если вам нужны **дополнительные упражнения** (например, по созданию более сложных SHACL-форм или SPARQL-запросов с `OPTIONAL` и `UNION`), я могу их подготовить.
