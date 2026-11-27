

# 🧪 Лабораторная работа: AI-агент в Microsoft Foundry на базе онтологии Fabric IQ



## Этап 0. Создание Lakehouse и загрузка данных из OneLake (20 минут)

### 0.1. Создайте рабочую область и Lakehouse

1. Откройте `https://app.fabric.microsoft.com` и войдите с учётными данными Fabric.
2. Создайте **новую рабочую область** (например, `FabricIQ_Lab_WS`) и назначьте ей **Fabric capacity или Trial**. Убедитесь, что это **не** `My Workspace`.
3. В рабочей области выберите **+ New item → Lakehouse**.
4. Назовите Lakehouse `LamnaHealthcareLH` и нажмите **Create**.

### 0.2. Загрузите CSV-файлы

1. Скачайте архив с sample-данными:  
   **`https://github.com/MicrosoftLearning/mslearn-fabric/raw/main/Allfiles/Labs/23-24/sample-data.zip`**
2. Распакуйте архив. В нём **шесть CSV-файлов**:

| Файл | Содержимое |
| :--- | :--- |
| `Hospitals.csv` | Больницы: `HospitalId`, `HospitalName`, `City`, `State` |
| `Departments.csv` | Отделения: `DepartmentId`, `DepartmentName`, `HospitalId` |
| `Rooms.csv` | Палаты: `RoomId`, `RoomNumber`, `DepartmentId` |
| `Patients.csv` | Пациенты: `PatientId`, `PatientName`, `RoomId` |
| `VitalSignEquipment.csv` | Оборудование мониторинга: `EquipmentId`, `EquipmentType`, `PatientId`, `MonitoringStart` |
| `VitalSignsReadings.csv` | Показания датчиков (time series) — **в этой лабораторной не используется**, оставьте для расширения |

3. В Lakehouse `LamnaHealthcareLH` нажмите **Upload files** и загрузите **пять** файлов (все, кроме `VitalSignsReadings.csv`).
4. После загрузки **преобразуйте каждый CSV в таблицу**: щёлкните правой кнопкой → **Load to Tables → New table**. Имя таблицы — имя файла в нижнем регистре (например, `hospitals`, `departments`). Убедитесь, что в разделе **Tables** появились пять таблиц.

---

## Этап 1. Создание онтологии: типы сущностей, свойства, ключи (25 минут)

### 1.1. Создайте пустую онтологию

1. В рабочей области выберите **+ New item**.
2. Найдите **Ontology (preview)** и создайте элемент с именем `LamnaHealthcareOntology`.

### 1.2. Создайте тип сущности «Hospital»

1. На ленте онтологии нажмите **Add entity type**.
2. Введите имя `Hospital`.
3. На панели справа перейдите на вкладку **Properties** и нажмите **Add properties**.
4. Добавьте свойства (после каждого нажимайте **+ Add**):

| Property Name | Data Type | Property Type |
| :--- | :--- | :--- |
| `HospitalId` | Integer | Static |
| `HospitalName` | String | Static |
| `City` | String | Static |
| `State` | String | Static |

5. Назначьте `HospitalId` **ключом**: откройте контекстное меню свойства и выберите **Set as key**.

### 1.3. Создайте остальные четыре типа сущностей

Повторите шаг 1.2 для каждого типа, используя таблицу ниже. **Ключ** указан в первой строке каждого блока.

**Department**  
| `DepartmentId` (Integer, **key**) | `DepartmentName` (String) |

**Room**  
| `RoomId` (Integer, **key**) | `RoomNumber` (String) |

**Patient**  
| `PatientId` (Integer, **key**) | `PatientName` (String) |

**VitalSignEquipment**  
| `EquipmentId` (Integer, **key**) | `EquipmentType` (String) | `MonitoringStart` (DateTime) |

> 💡 **Ключи обязательны:** без ключа онтология не сможет однозначно идентифицировать экземпляры сущности и корректно связать их с данными.

---

## Этап 2. Создание связей (relationships) между сущностями (15 минут)

Связи описывают, **как сущности относятся друг к другу**. В нашем сценарии — четыре связи.

### 2.1. Создайте связь «Hospital содержит Department»

1. На ленте нажмите **Add relationship**.
2. Заполните:
   - **Relationship type name:** `contains`
   - **Source entity type:** `Hospital`
   - **Target entity type:** `Department`
3. Нажмите **Add relationship type**.

### 2.2. Создайте остальные три связи

| Relationship Name | Source Entity Type | Target Entity Type | Смысл |
| :--- | :--- | :--- | :--- |
| `contains` | `Department` | `Room` | Отделение содержит палаты |
| `assignedTo` | `Patient` | `Room` | Пациент назначен в палату |
| `monitors` | `VitalSignEquipment` | `Patient` | Оборудование мониторит пациента |

> 💡 **Направление имеет значение:** `monitors` идёт **от оборудования к пациенту**. Это семантика: «оборудование мониторит пациента», а не наоборот.

---

## Этап 3. Привязка онтологии к данным OneLake (20 минут)

**Data binding** соединяет схему онтологии с конкретными таблицами в Lakehouse. Данные **не копируются** — онтология ссылается на источник.

### 3.1. Привяжите статические сущности (Hospital, Department, Room, Patient)

Для каждой сущности выполните:

1. Выберите тип сущности на холсте.
2. На панели **Entity type configuration** перейдите на вкладку **Bindings**.
3. Нажмите **Add data to entity type**.
4. В каталоге OneLake выберите `LamnaHealthcareLH` → **Connect**.
5. Выберите соответствующую таблицу (например, `hospitals` для `Hospital`) → **Next**.
6. Оставьте **Binding type = Static**.
7. Проверьте автоматическое сопоставление колонок со свойствами (например, `HospitalId → HospitalId`). При необходимости скорректируйте.
8. Нажмите **Save**.

**Соответствие таблиц:**

| Entity Type | Таблица в Lakehouse |
| :--- | :--- |
| `Hospital` | `hospitals` |
| `Department` | `departments` |
| `Room` | `rooms` |
| `Patient` | `patients` |

### 3.2. Привяжите `VitalSignEquipment`

1. Выберите `VitalSignEquipment` → **Bindings** → **Add data to entity type**.
2. Выберите `LamnaHealthcareLH` → таблицу `vitalsignequipment` → **Next**.
3. Оставьте **Static** и сопоставьте свойства. Нажмите **Save**.
4. **Повторно** нажмите **Add data to entity type** — на этот раз выберите **Eventhouse** `LamnaHealthcareEH` (если создали его для time-series данных) или просто пропустите этот шаг, если используете только статические данные из Lakehouse. Time-series привязка требует наличия Eventhouse и таблицы `VitalSignsReadings`.

### 3.3. Настройте связи (relationships) на данных

Каждая связь должна «знать», из какой таблицы брать информацию о соединении.

1. На холсте онтологии щёлкните **линию связи** между `Hospital` и `Department`.
2. На панели **Relationship configuration** укажите:
   - **Workspace:** ваша рабочая область
   - **Lakehouse:** `LamnaHealthcareLH`
   - **Schema:** `dbo`
   - **Table:** `departments` (эта таблица содержит ключи и для `Hospital`, и для `Department`)
3. Сопоставьте колонки: `HospitalId → HospitalId`, `DepartmentId → DepartmentId`.
4. Нажмите **Create**.

Повторите для остальных связей:

| Relationship | Source Data (Table) |
| :--- | :--- |
| `Department contains Room` | `rooms` |
| `Patient assignedTo Room` | `patients` |
| `VitalSignEquipment monitors Patient` | `vitalsignequipment` |

---

## Этап 4. Подключение Foundry-агента к онтологии через Fabric IQ tool (25 минут)

Теперь онтология готова стать **инструментом** для AI-агента.

### 4.1. Откройте проект Foundry и создайте агента

1. Войдите в **портал Microsoft Foundry** и выберите свой проект.
2. Перейдите в **Build → Agents** и создайте нового агента (Prompt agent).
3. В качестве модели выберите развёрнутую модель (например, `gpt-4o-mini`).
4. В **Instructions** задайте роль агента, например:
   > *«Ты — ассистент сети больниц Lamna Healthcare. Отвечай на вопросы о больницах, отделениях, палатах, пациентах и оборудовании, используя доступные инструменты Fabric IQ. Если данных недостаточно, скажи об этом.»*

### 4.2. Добавьте Fabric IQ tool (OneLake Catalog)

1. На странице агента перейдите на вкладку **Tools**.
2. Нажмите **Add → Browse all tools**.
3. Выберите **Fabric IQ (OneLake Catalog)** и нажмите **Add tool**.
4. В каталоге OneLake отфильтруйте по **Ontology** (или по Data agent, если вы его публиковали).
5. Выберите созданную онтологию `LamnaHealthcareOntology` и нажмите **Add**.

> 💡 **Альтернативный путь:** если вы опубликовали **Fabric data agent** на основе онтологии, можно добавить именно его через фильтр **Data agent** в каталоге OneLake.

### 4.3. (Для администратора) Настройте разрешения

Убедитесь, что:
- Ресурс **Azure AI Search** (на котором работает Foundry IQ) имеет роль **Search Index Data Contributor** для managed identity проекта Foundry. Без этого агент получит **403** при обращении к онтологии.
- Пользователь, вызывающий агента, имеет **лицензию Fabric** и доступ на чтение к онтологии и её источникам данных.

---

## Этап 5. Тестирование агента: вопросы на естественном языке (15 минут)

Откройте **Playground** агента в Foundry и задайте вопросы. Агент должен **делегировать** их Fabric IQ через MCP-endpoint, а Fabric IQ — транслировать бизнес-термины в структурированные запросы через слой **NL2Ontology**.

### 5.1. Базовые вопросы (проверка связей)

| Вопрос | Что проверяет |
| :--- | :--- |
| *«Какие отделения есть в больнице Lamna General Hospital?»* | Связь `Hospital contains Department` |
| *«Какие палаты относятся к отделению ICU?»* | Связь `Department contains Room` |
| *«В какой палате находится пациент с ID 1001?»* | Связь `Patient assignedTo Room` |
| *«Какое оборудование мониторит пациента 1001?»* | Связь `VitalSignEquipment monitors Patient` |

### 5.2. Многошаговые вопросы (проверка обхода)

| Вопрос | Что проверяет |
| :--- | :--- |
| *«Какое оборудование мониторит пациентов в отделении ICU больницы Lamna General?»* | Обход: Hospital → Department → Room → Patient → Equipment |
| *«Перечисли всех пациентов, которые находятся в палатах отделения Emergency, и укажите, какое оборудование к ним привязано»* | Обход на 4 шага с агрегацией |

### 5.3. Вопросы с фильтрацией и агрегацией

| Вопрос | Что проверяет |
| :--- | :--- |
| *«Сколько палат в каждом отделении больницы Lamna General?»* | Агрегация `count` с группировкой |
| *«Какие пациенты находятся в палатах с номером, начинающимся на "1"?»* | Фильтрация по строковому шаблону |

> 💡 **Ключевой момент:** агент **не генерирует SQL или Cypher напрямую**. Он формулирует запрос к Fabric IQ на естественном языке, а Fabric IQ **сам** определяет, какие сущности, свойства и связи онтологии использовать, и возвращает синтезированный ответ.

---
