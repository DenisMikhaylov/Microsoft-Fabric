# Лабораторная работа № 04
# Создание онтологии Fabric IQ для Lamna Healthcare и подключение к Microsoft Foundry

**Продолжительность:** 100–130 минут  
**Уровень:** средний  
**Результат:** Lakehouse с пятью таблицами, Eventhouse с показаниями датчиков, онтология из пяти типов сущностей и пяти связей, проверенные экземпляры и (опционально) агент Microsoft Foundry, использующий онтологию через Foundry IQ.

> **Статус функций.** Ontology (preview) и интеграция Fabric IQ / Foundry IQ находятся в предварительной версии. Названия пунктов интерфейса могут меняться. Лабораторная ориентируется на новую Ontology experience и актуальную документацию Microsoft Learn. Не смешивайте её со старым интерфейсом ontology.

---

## 1. Цель и логика работы

Вы создадите семантическую модель предметной области вымышленной сети клиник Lamna Healthcare.

Логическая цепочка данных:

```text
Hospitals.csv ───────> Lakehouse.dbo.hospitals ──> Hospital
Departments.csv ─────> Lakehouse.dbo.departments -> Department
Rooms.csv ───────────> Lakehouse.dbo.rooms ──────> Room
Patients.csv ────────> Lakehouse.dbo.patients ───> Patient
VitalSignEquipment.csv -> Lakehouse.dbo.vitalsignequipment -> VitalSignEquipment
VitalSignsReadings.csv -> Eventhouse.KQLDB.VitalSignsReadings
                                                    |
                              вторичная time-series привязка к EquipmentId
                                                    |
                                           VitalSignEquipment
```

Онтология описывает не просто таблицы, а бизнес-объекты и отношения между ними:

```text
Hospital ──contains──> Department ──has──> Room
                                           ^
                                           |
Patient ─────────────assignedTo─────────────┘

VitalSignEquipment ──monitors──> Patient
VitalSignEquipment ──locatedIn──> Room
```

**Почему нужны внешние ключи в свойствах сущностей:** связь настраивается сопоставлением значения свойства источника и значения свойства целевой сущности. Например, `Hospital.HospitalId` сопоставляется с `Department.HospitalId`. Если не создать свойство `HospitalId` у Department, связать больницу с отделениями будет нечем.

**Почему VitalSignsReadings хранится в Eventhouse:** это временной ряд измерений. В онтологии он добавляется как дополнительный источник для `VitalSignEquipment`, а строки связываются с основной таблицей оборудования по `EquipmentId`. Измерения не создают отдельные экземпляры оборудования.

## 2. Предварительные требования

Перед началом подготовьте:

1. Учётную запись Microsoft, имеющую доступ к Microsoft Fabric.
2. Рабочую область Fabric на платной Fabric capacity или Fabric Trial. `My Workspace` для лабораторной не используйте.
3. Разрешения на создание Fabric items и Ontology (preview).
4. Если выполняется этап Foundry: Azure subscription, проект Microsoft Foundry с развёрнутой моделью и права на управление проектом/подключениями.
5. Браузер Edge или Chrome.

### 2.1. Проверка tenant settings

Если пункта `Ontology (preview)` нет в списке New item, попросите администратора Fabric проверить настройки tenant:

- `Users can create Fabric items`;
- `Users can create ontology (preview) items`.

Для работы с Eventhouse также должна быть доступна соответствующая Fabric workload и ёмкость.

### 2.2. Имена объектов

Используйте именно эти имена, чтобы команды и контрольные точки совпадали с инструкцией:

| Объект | Имя |
|---|---|
| Workspace | `FabricIQ_Lab_WS` |
| Lakehouse | `LamnaHealthcareLH` |
| Eventhouse | `LamnaHealthcareEH` |
| KQL database | `LamnaHealthcareEH` (создаётся вместе с Eventhouse) |
| Ontology | `LamnaHealthcareOntology` |
| Foundry knowledge source | `LamnaHealthcareOntologySource` |
| Foundry knowledge base | `LamnaHealthcareKB` |
| Foundry agent | `LamnaHealthcareAssistant` |

---

## 3. Этап 0. Создание Workspace и Lakehouse (10 минут)

### 3.1. Создайте рабочую область

1. Откройте `https://app.fabric.microsoft.com/home?experience=fabric`.
2. Войдите под своей учётной записью.
3. В левой панели выберите **Workspaces**.
4. Нажмите **New workspace**.
5. Введите `FabricIQ_Lab_WS`.
6. В настройках лицензирования/Advanced выберите доступную Fabric capacity или Fabric Trial.
7. Нажмите **Apply** / **Create**.

**Проверка:** в списке Workspaces появилась `FabricIQ_Lab_WS`; рабочая область не является `My Workspace`.

### 3.2. Создайте Lakehouse

1. Откройте `FabricIQ_Lab_WS`.
2. Выберите **+ New item → Lakehouse**.
3. Имя: `LamnaHealthcareLH`.
4. Оставьте включённой опцию Lakehouse schemas, если интерфейс её предлагает.
5. Нажмите **Create**.

После создания откроется Lakehouse Explorer. Для лабораторной таблицы должны оказаться в схеме `dbo`.

---

## 4. Этап 1. Загрузка исходных данных (20 минут)

### 4.1. Скачайте и распакуйте архив

Скачайте архив:

`https://github.com/MicrosoftLearning/mslearn-fabric/raw/main/Allfiles/Labs/23-24/sample-data.zip`

На локальном компьютере создайте каталог:

- Windows: `C:\Labs\FabricIQ\sample-data\`
- Linux/macOS: `~/Labs/FabricIQ/sample-data/`

Распакуйте архив в этот каталог. В нём должны находиться шесть CSV:

| CSV-файл | Назначение | Где будет использоваться |
|---|---|---|
| `Hospitals.csv` | Больницы | Lakehouse |
| `Departments.csv` | Отделения | Lakehouse |
| `Rooms.csv` | Палаты | Lakehouse |
| `Patients.csv` | Пациенты и текущие назначения | Lakehouse |
| `VitalSignEquipment.csv` | Оборудование, пациент, палата, дата начала мониторинга | Lakehouse |
| `VitalSignsReadings.csv` | Измерения датчиков во времени | Eventhouse |

Не загружайте `VitalSignsReadings.csv` в Lakehouse: в этой лабораторной он будет помещён в Eventhouse.

### 4.2. Загрузите пять статических CSV в Lakehouse

1. Откройте `LamnaHealthcareLH`.
2. В Explorer выберите папку **Files**.
3. Нажмите **Get data → Upload files** (в некоторых версиях интерфейса — **Upload files**).
4. Выберите пять файлов:
   - `Hospitals.csv`;
   - `Departments.csv`;
   - `Rooms.csv`;
   - `Patients.csv`;
   - `VitalSignEquipment.csv`.
5. Нажмите **Open**, затем **Upload**.
6. Дождитесь окончания загрузки всех файлов.

**Ожидаемый путь в Lakehouse Explorer:**

```text
FabricIQ_Lab_WS
└── LamnaHealthcareLH
    └── Files
        ├── Hospitals.csv
        ├── Departments.csv
        ├── Rooms.csv
        ├── Patients.csv
        └── VitalSignEquipment.csv
```

### 4.3. Преобразуйте CSV в таблицы

Для каждого из пяти файлов повторите действия:

1. В Explorer раскройте `Files`.
2. Справа от CSV нажмите `…`.
3. Выберите **Load to Tables → New table**.
4. В окне импорта задайте:
   - **New table name** — имя без расширения, в нижнем регистре;
   - **Column header** — включить `Use header for column names`;
   - **Separator** — запятая `,`.
5. Нажмите **Load**.
6. Дождитесь уведомления об успешной загрузке. Не запускайте следующую загрузку, пока предыдущая выполняется.

Используйте соответствия:

| Файл | Схема и таблица |
|---|---|
| `Hospitals.csv` | `dbo.hospitals` |
| `Departments.csv` | `dbo.departments` |
| `Rooms.csv` | `dbo.rooms` |
| `Patients.csv` | `dbo.patients` |
| `VitalSignEquipment.csv` | `dbo.vitalsignequipment` |

**Ожидаемый путь в Explorer:**

```text
LamnaHealthcareLH
├── Files
│   └── исходные CSV
└── Tables
    └── dbo
        ├── hospitals
        ├── departments
        ├── rooms
        ├── patients
        └── vitalsignequipment
```

### 4.4. Контроль загрузки

Убедитесь, что в `Tables → dbo` отображаются все пять таблиц.

Откройте таблицы и проверьте, что заголовки колонок не превратились в первую строку данных. Если первая строка содержит названия колонок, загрузка была выполнена без `Use header for column names`; удалите только ошибочно созданную таблицу и загрузите файл повторно с правильной настройкой.

Не переименовывайте колонки произвольно: дальнейшие шаги используют исходные имена.

---

## 5. Этап 2. Создание Eventhouse и загрузка временного ряда (10–15 минут)

Этот этап обязателен, если вы хотите выполнить часть лабораторной с показаниями датчиков. Для упрощённого варианта без временного ряда его можно пропустить, но тогда не выполняйте time-series binding.

### 5.1. Создайте Eventhouse

1. Вернитесь в рабочую область `FabricIQ_Lab_WS`.
2. Выберите **+ New item → Eventhouse**.
3. Имя: `LamnaHealthcareEH`.
4. Нажмите **Create**.
5. Откройте созданную KQL database с именем `LamnaHealthcareEH`.

Ожидаемая структура:

```text
FabricIQ_Lab_WS
└── LamnaHealthcareEH
    └── LamnaHealthcareEH (KQL database)
```

### 5.2. Импортируйте VitalSignsReadings.csv

1. Откройте KQL database `LamnaHealthcareEH`.
2. Выберите **Get data → Local file**.
3. В блоке назначения выберите **New table**.
4. Имя таблицы: `VitalSignsReadings`.
5. Подтвердите имя кнопкой с галочкой.
6. В блоке загрузки файлов нажмите **Browse for files**.
7. Выберите локальный файл `VitalSignsReadings.csv` из `C:\Labs\FabricIQ\sample-data\` (или соответствующего каталога на вашей ОС).
8. Нажмите **Next**.
9. На странице проверки убедитесь, что включён `First row header`, а в предварительном просмотре видны имена колонок, включая `EquipmentId` и `Timestamp`.
10. Нажмите **Finish**.
11. Дождитесь статуса `Successfully ingested`, затем закройте окно.

**Контроль:** таблица `VitalSignsReadings` отображается в KQL database. В учебном наборе ожидается 20 строк. Если число отличается, сначала проверьте выбранный CSV и статус ingestion, не меняйте схему наугад.

---

## 6. Этап 3. Создание онтологии (5 минут)

1. Откройте `FabricIQ_Lab_WS`.
2. Выберите **+ New item**.
3. Найдите **Ontology (preview)**.
4. Имя: `LamnaHealthcareOntology`.
5. Нажмите **Create**.

Должен открыться пустой холст онтологии.

---

## 7. Этап 4. Создание типов сущностей и свойств (20 минут)

Создайте пять типов сущностей. Для каждого типа:

1. На холсте выберите **Add entity type**.
2. Введите имя сущности.
3. Нажмите **Add Entity Type**.
4. Выберите сущность в Explorer.
5. Откройте **View entity type details** — откроется страница **Configure**.
6. В блоке **Properties** раскройте **Manage property bindings**.
7. Нажмите **Add properties**.
8. Добавьте свойства из таблицы ниже, указав имя и тип.
9. Нажмите **Save**.
10. Выберите **Define entity type key**, укажите ключ и сохраните.

> Ключ сущности должен быть уникальным идентификатором экземпляра. Для ключа поддерживаются типы String и Integer. Не назначайте ключом дату, имя или неуникальный атрибут.

### 7.1. Схема сущностей

| Entity type | Property | Тип | Назначение |
|---|---|---|---|
| `Hospital` | `HospitalId` | Integer | **Ключ** |
|  | `HospitalName` | String | Название больницы |
|  | `City` | String | Город |
|  | `State` | String | Штат/регион |
| `Department` | `DepartmentId` | Integer | **Ключ** |
|  | `DepartmentName` | String | Название отделения |
|  | `HospitalId` | Integer | Внешний ключ на Hospital |
|  | `Floor` | Integer | Этаж |
| `Room` | `RoomId` | Integer | **Ключ** |
|  | `RoomNumber` | String | Номер палаты |
|  | `DepartmentId` | Integer | Внешний ключ на Department |
|  | `RoomType` | String | Тип палаты |
| `Patient` | `PatientId` | Integer | **Ключ** |
|  | `FirstName` | String | Имя |
|  | `LastName` | String | Фамилия |
|  | `DateOfBirth` | DateTime | Дата рождения |
|  | `AdmissionDate` | DateTime | Дата поступления |
|  | `CurrentRoomId` | Integer | Внешний ключ на Room |
| `VitalSignEquipment` | `EquipmentId` | String | **Ключ** |
|  | `PatientId` | Integer | Внешний ключ на Patient |
|  | `RoomId` | Integer | Внешний ключ на Room |
|  | `EquipmentType` | String | Тип оборудования |
|  | `MonitoringStartDate` | DateTime | Начало мониторинга |

**Обратите внимание на важные отличия от упрощённой схемы:**

- У `Department` обязательно есть `HospitalId`, иначе нельзя настроить связь с Hospital.
- У `Room` обязательно есть `DepartmentId`.
- У `Patient` используется `CurrentRoomId`, а не `RoomId`.
- У `VitalSignEquipment` ключ `EquipmentId` имеет тип String (например, `VS-1001`), а не Integer.
- В `VitalSignEquipment` нужны оба внешних ключа: `PatientId` и `RoomId`.

**Контроль:** в Explorer должны быть пять Entity Types, у каждой сущности определён ключ.

---

## 8. Этап 5. Создание типов связей (10 минут)

На холсте выберите **Add relationship**. В окне создания задайте имя, Origin entity type и Target entity type. Создайте пять связей:

| Relationship type | Origin (откуда) | Target (куда) | Смысл |
|---|---|---|---|
| `contains` | `Hospital` | `Department` | Больница содержит отделения |
| `has` | `Department` | `Room` | Отделение содержит палаты |
| `assignedTo` | `Patient` | `Room` | Пациент размещён в палате |
| `monitors` | `VitalSignEquipment` | `Patient` | Оборудование мониторит пациента |
| `locatedIn` | `VitalSignEquipment` | `Room` | Оборудование находится в палате |

Для каждой строки:

1. Нажмите **Add relationship**.
2. Введите `Relationship type name`.
3. Укажите `Origin entity type`.
4. Укажите `Target entity type`.
5. Нажмите **Create**.

После создания на холсте появятся линии между сущностями. На этом этапе связь только описана логически; сопоставление колонок будет выполнено отдельно.

---

## 9. Этап 6. Привязка сущностей к таблицам Lakehouse (20 минут)

Data binding связывает свойства онтологии с колонками таблиц. Исходные данные при этом не копируются в онтологию.

Для каждой сущности:

1. В Explorer выберите сущность.
2. Откройте **View entity type details**.
3. На странице **Configure** найдите **Manage property bindings**.
4. Нажмите **Add binding and properties** (в некоторых версиях интерфейса действие может называться **Bind data**).
5. В блоке **Sources** нажмите **Add** — откроется OneLake catalog.
6. Найдите `FabricIQ_Lab_WS → LamnaHealthcareLH → dbo`.
7. Выберите нужную таблицу и нажмите **Select table**.
8. Откройте **Entity type properties**.
9. Проверьте соответствие свойств и исходных колонок по таблицам ниже.
10. Если свойство уже создано на предыдущем этапе, выберите его из списка существующих свойств. Не создавайте дубликат с тем же смыслом.
11. Нажмите **Create** и дождитесь уведомления `Entity type updated successfully`.
12. Вернитесь на страницу Configure и убедитесь, что свойства связаны с колонками, а не имеют состояние `Unbound`.

### 9.1. Соответствие таблиц и свойств

| Entity type | Источник | Обязательные сопоставления |
|---|---|---|
| `Hospital` | `LamnaHealthcareLH → dbo → hospitals` | `HospitalId`, `HospitalName`, `City`, `State` |
| `Department` | `LamnaHealthcareLH → dbo → departments` | `DepartmentId`, `DepartmentName`, `HospitalId`, `Floor` |
| `Room` | `LamnaHealthcareLH → dbo → rooms` | `RoomId`, `RoomNumber`, `DepartmentId`, `RoomType` |
| `Patient` | `LamnaHealthcareLH → dbo → patients` | `PatientId`, `FirstName`, `LastName`, `DateOfBirth`, `AdmissionDate`, `CurrentRoomId` |
| `VitalSignEquipment` | `LamnaHealthcareLH → dbo → vitalsignequipment` | `EquipmentId`, `PatientId`, `RoomId`, `EquipmentType`, `MonitoringStartDate` |

### 9.2. Особое внимание при привязке

- `Hospital.HospitalId` должен быть ключом сущности и отображаться на `hospitals.HospitalId`.
- `Department.HospitalId` — обычное свойство Department, оно отображается на `departments.HospitalId`.
- `Room.DepartmentId` отображается на `rooms.DepartmentId`.
- `Patient.CurrentRoomId` отображается на `patients.CurrentRoomId`.
- `VitalSignEquipment.EquipmentId` — строковый ключ, отображается на `vitalsignequipment.EquipmentId`.
- Не подменяйте `CurrentRoomId` на `RoomId`: это разные имена полей исходной схемы.

**Контроль:** откройте Configure у каждой сущности. У всех обязательных свойств указан правильный источник.

---

## 10. Этап 7. Time-series binding для показаний датчиков (10–15 минут)

Этот этап выполняется после статической привязки `VitalSignEquipment` к `vitalsignequipment`.

1. Откройте `VitalSignEquipment → View entity type details → Configure`.
2. Выберите **Manage property bindings → Add binding and properties**.
3. В Sources нажмите **Add**.
4. В OneLake catalog найдите:
   `FabricIQ_Lab_WS → LamnaHealthcareEH → LamnaHealthcareEH → VitalSignsReadings`.
5. Выберите таблицу и нажмите **Select table**.
6. В разделе **Relationship with primary data source** сопоставьте:
   - колонка основной таблицы `vitalsignequipment`: `EquipmentId`;
   - колонка дополнительного источника `VitalSignsReadings`: `EquipmentId`.
7. Сохраните сопоставление источников.
8. Откройте **Entity type properties**.
9. Не удаляйте и не переназначайте существующие статические свойства, в том числе ключ `EquipmentId`.
10. Проверьте новые свойства временного ряда:

| Свойство | Источник | Тип свойства | Колонка времени |
|---|---|---|---|
| `ReadingId` | `VitalSignsReadings.ReadingId` | Timeseries<String> | `Timestamp` |
| `HeartRate` | `VitalSignsReadings.HeartRate` | Timeseries<Integer> | `Timestamp` |
| `OxygenSaturation` | `VitalSignsReadings.OxygenSaturation` | Timeseries<Integer> | `Timestamp` |
| `RespiratoryRate` | `VitalSignsReadings.RespiratoryRate` | Timeseries<Integer> | `Timestamp` |

11. Для каждого свойства временного ряда выберите `Timestamp` как timestamp column.
12. Нажмите **Save**. Дождитесь `Entity type updated successfully`.
13. Вернитесь на Configure и убедитесь, что новые свойства имеют тип Timeseries и источник `VitalSignsReadings`.

**Логика:** статическая таблица создаёт экземпляр оборудования и содержит контекст (тип оборудования, пациент, палата). Таблица `VitalSignsReadings` содержит повторяющиеся измерения во времени. `EquipmentId` связывает каждое измерение с соответствующим экземпляром оборудования, а `Timestamp` задаёт временной порядок.

---

## 11. Этап 8. Настройка связей через свойства (10–15 минут)

Это отдельная операция после создания типов связей и привязки сущностей. Для каждой связи нужно указать, какие свойства Origin и Target должны совпасть.

1. В Explorer раскройте **Relationship Types**.
2. Выберите нужную связь, например `contains`.
3. Откроется страница конфигурации relationship type.
4. Оставьте **Use mapping table? = Off**.
5. Проверьте Origin и Target.
6. Выберите соответствующие свойства.
7. Нажмите **Save** и дождитесь сообщения об успешном обновлении.

Настройте все пять связей:

| Связь | Origin entity property | Target entity property | Что сопоставляется |
|---|---|---|---|
| `contains` (Hospital → Department) | `Hospital.HospitalId` | `Department.HospitalId` | ID больницы |
| `has` (Department → Room) | `Department.DepartmentId` | `Room.DepartmentId` | ID отделения |
| `assignedTo` (Patient → Room) | `Patient.CurrentRoomId` | `Room.RoomId` | Текущая палата пациента |
| `monitors` (VitalSignEquipment → Patient) | `VitalSignEquipment.PatientId` | `Patient.PatientId` | ID пациента |
| `locatedIn` (VitalSignEquipment → Room) | `VitalSignEquipment.RoomId` | `Room.RoomId` | ID палаты оборудования |

**Критически важно:** для `Hospital → Department` сопоставляются `HospitalId` и `HospitalId`, а не `HospitalId` и `DepartmentId`. В строке Department хранится идентификатор родительской больницы.

Mapping table в этой лабораторной не используется: нужные внешние ключи уже находятся в исходных таблицах.

---

## 12. Этап 9. Проверка экземпляров и связей (10 минут)

### 12.1. Проверьте экземпляры Room

1. В Explorer выберите `Room`.
2. Откройте **View entity type details**.
3. Перейдите на вкладку **Instances**.
4. Дождитесь загрузки данных.
5. Проверьте, что таблица содержит 10 комнат.
6. Найдите `ICU-302`.
7. Проверьте ожидаемые значения:
   - `RoomId = 2`;
   - `DepartmentId = 1`;
   - `RoomType = Critical Care`.

### 12.2. Проверьте количество объектов

Для учебного набора ожидаются:

| Entity type | Ожидаемое число экземпляров |
|---|---:|
| Hospital | 1 |
| Department | 3 |
| Room | 10 |
| Patient | 5 |
| VitalSignEquipment | 5 |

Если количество не совпадает, проверьте загрузку исходных CSV и привязку таблиц. Не исправляйте данные вручную до выяснения причины.

### 12.3. Проверьте ключ и time-series

Откройте `VitalSignEquipment → Configure` и проверьте:

- `EquipmentId` остаётся ключом;
- `PatientId` связан с `vitalsignequipment.PatientId`;
- `RoomId` связан с `vitalsignequipment.RoomId`;
- `ReadingId`, `HeartRate`, `OxygenSaturation`, `RespiratoryRate` имеют Timeseries bindings;
- для временных свойств используется `Timestamp`.

Вкладка Instances показывает статические свойства оборудования. Не ожидайте, что временные измерения будут отображаться как обычные статические колонки экземпляра.

### 12.4. Проверьте relationship types

Откройте по очереди все пять связей в Explorer → Relationship Types. Сверьте пары свойств с таблицей из этапа 8.

---

## 13. Этап 10. Подключение онтологии к Microsoft Foundry через Foundry IQ (20–30 минут, дополнительный)

> Этот этап требует отдельного Azure/Microsoft Foundry проекта и соответствующих разрешений. Он не является частью обязательной сборки онтологии.

В актуальном сценарии подключения **онтологии** через Foundry IQ используйте Knowledge source и Knowledge base. Не путайте этот сценарий с добавлением опубликованного Fabric data agent в список Tools: это отдельный вариант интеграции.

### 13.1. Предварительные условия

1. Есть активная Azure subscription и разрешение создавать ресурсы Foundry.
2. Создан проект Microsoft Foundry.
3. В проекте есть развёрнутая модель.
4. Учётная запись имеет доступ к Fabric workspace и ontology item.
5. Managed identity проекта Foundry имеет роль `Search Index Data Contributor` на ресурсе Azure AI Search.
6. Разработчику/исполнителю предоставлены требуемые роли Foundry, в том числе Foundry User; для создания подключения может потребоваться Foundry Project Manager.

Если роль `Search Index Data Contributor` не назначена, агент может получать HTTP 403 при обращении к онтологии.

### 13.2. Создайте Knowledge source

1. Откройте портал Microsoft Foundry: `https://ai.azure.com/`.
2. Выберите нужный Foundry project.
3. В левой панели откройте **Knowledge → Knowledge bases**.
4. Нажмите **+ New knowledge base**.
5. В качестве типа источника выберите **Fabric IQ**.
6. В открывшемся OneLake catalog найдите:
   `FabricIQ_Lab_WS → LamnaHealthcareOntology`.
7. Выберите `LamnaHealthcareOntology` и подтвердите выбор.
8. Имя источника: `LamnaHealthcareOntologySource`.
9. В описании укажите:

   `Онтология Lamna Healthcare. Содержит больницы, отделения, палаты, пациентов и оборудование мониторинга. Используется для вопросов о размещении пациентов, структуре больницы и привязке оборудования.`

10. Создайте источник.

### 13.3. Создайте Knowledge base

1. Задайте имя `LamnaHealthcareKB`.
2. В описание внесите:

   `Отвечает на вопросы о структуре Lamna Healthcare: больницах, отделениях, палатах, пациентах, размещении пациентов и оборудовании мониторинга. Использует связи онтологии для переходов между объектами. Не делает выводов о медицинском диагнозе и лечении.`

3. В Retrieval instructions (если поле доступно) укажите:

   `Используй только сведения из онтологии LamnaHealthcareOntology. Для вопросов о размещении пациента переходи от Patient к Room через assignedTo. Для вопросов об отделении переходи от Department к Room через has. Для оборудования используй связи monitors и locatedIn. Если соответствующая запись отсутствует, сообщи, что данных недостаточно. Не выдумывай пациентов, палаты и измерения.`

4. В Answer instructions (если поле доступно) укажите:

   `Отвечай по-русски. Для каждого ответа называй найденные сущности и значимые идентификаторы. Если вопрос предполагает несколько объектов, используй маркированный список. Не делай медицинских рекомендаций и не интерпретируй измерения как диагноз.`

5. Сохраните Knowledge base.

### 13.4. Создайте Foundry agent и подключите Knowledge base

1. В Foundry откройте **Agents**.
2. Выберите **+ New agent**.
3. Имя агента: `LamnaHealthcareAssistant`.
4. Выберите развёрнутую модель проекта.
5. В Instructions вставьте:

   `Ты — информационный ассистент Lamna Healthcare. Отвечай на вопросы о больницах, отделениях, палатах, пациентах и оборудовании мониторинга. Используй подключённую базу знаний LamnaHealthcareKB как источник фактов. Не придумывай отсутствующие данные. Если в источнике нет ответа, прямо сообщи об этом. Не давай медицинских диагнозов и рекомендаций по лечению.`

6. В разделе **Knowledge** выберите **+ Add knowledge**.
7. Выберите `LamnaHealthcareKB`.
8. Сохраните агента.

**Контроль:** Knowledge base отображается в настройках агента как подключённая база знаний.

---

## 14. Этап 11. Проверка агента (10–15 минут)

Откройте Chat/Playground агента и выполните запросы по очереди. Сверяйте ответы с экземплярами и связями онтологии.

| Тестовый запрос | Что проверяется |
|---|---|
| «Какие отделения есть в больнице Lamna General Hospital?» | Hospital → Department (`contains`) |
| «Какие палаты относятся к отделению ICU?» | Department → Room (`has`) |
| «В какой палате находится пациент с ID 1001?» | Patient → Room (`assignedTo`) |
| «Какое оборудование мониторит пациента 1001?» | Equipment → Patient (`monitors`) |
| «В какой палате находится оборудование VS-1001?» | Equipment → Room (`locatedIn`) |
| «Какие пациенты находятся в палатах отделения Emergency?» | Department → Room → Patient |
| «Какое оборудование находится в палате ICU-302?» | Room → Equipment |
| «Какие показания пульса есть у оборудования VS-1001?» | Time-series свойства Equipment |

Не требуйте от агента дословно совпадающего формата ответа: важны корректность сущностей, идентификаторов и связей.

### 14.1. Критерии успешного выполнения

Лабораторная считается выполненной, если:

- в Workspace создан `LamnaHealthcareLH`;
- в `dbo` находятся пять таблиц;
- в Eventhouse создана `VitalSignsReadings` (для полного варианта);
- создана `LamnaHealthcareOntology`;
- в онтологии пять типов сущностей с корректными ключами;
- созданы пять типов связей;
- статические свойства привязаны к нужным таблицам;
- временные свойства связаны с Eventhouse по `EquipmentId` и упорядочены по `Timestamp`;
- в Instances видны ожидаемые количества записей;
- все пять связей настроены через правильные пары свойств;
- (для дополнительного этапа) агент использует `LamnaHealthcareKB` и отвечает на контрольные вопросы по данным.

---

## 15. Типовые ошибки и способы диагностики

| Симптом | Вероятная причина | Что проверить |
|---|---|---|
| Нет пункта Ontology (preview) | Tenant setting не включён или функция недоступна в регионе/ёмкости | `Users can create ontology (preview) items`, `Users can create Fabric items`, capacity |
| Нет Lakehouse в OneLake catalog | Нет доступа к workspace/item или выбран не тот workspace | Доступ текущей учётной записи, имя Workspace, имя Lakehouse |
| В таблице первая строка — заголовки | CSV загружен без `Use header for column names` | Повторить загрузку с включённым флажком |
| Свойство отображается как Unbound | Не выполнен mapping свойства на source column | Configure → Properties / property bindings |
| Нельзя назначить ключ | Свойство имеет неподдерживаемый тип или не является уникальным | Ключ должен быть String или Integer и уникальным |
| Не удаётся настроить relationship | Не создано нужное FK-свойство или выбран неверный тип сущности | Проверить свойства из этапа 7 и пары из этапа 8 |
| Для Hospital–Department нет совпадений | Сопоставлены `HospitalId` и `DepartmentId` вместо `HospitalId` и `HospitalId` | Relationship `contains` |
| Time-series свойства не появляются | Нет Eventhouse/KQL table, неверно сопоставлен EquipmentId или не выбран Timestamp | Eventhouse, таблица `VitalSignsReadings`, secondary source mapping |
| Foundry возвращает 403 | Не хватает доступа к ontology или роли Azure AI Search | `Search Index Data Contributor`, доступ к Fabric item, identity |
| Агент отвечает общими словами и не использует источник | Knowledge base не подключена или инструкции слишком общие | Agent → Knowledge, описание KB, инструкции |

---

## 16. Что сдать преподавателю

Сохраните скриншоты следующих результатов:

1. Workspace `FabricIQ_Lab_WS` со списком созданных объектов.
2. `LamnaHealthcareLH → Tables → dbo` с пятью таблицами.
3. Eventhouse с таблицей `VitalSignsReadings`.
4. Холст онтологии со всеми пятью сущностями и пятью связями.
5. Configure для каждой сущности с ключом и привязанными свойствами.
6. Configure для `VitalSignEquipment` с time-series свойствами.
7. Конфигурации всех пяти relationship types.
8. Вкладка Instances для `Room` с записью `ICU-302`.
9. Ответы Foundry агента на три-четыре контрольных вопроса (если выполнялся дополнительный этап).

В отчёте кратко ответьте:

1. Чем Entity Type отличается от таблицы Lakehouse?
2. Для чего нужны entity key и внешние ключи в свойствах сущностей?
3. Почему связь `Hospital → Department` сопоставляет `HospitalId` с `HospitalId`?
4. Чем статическая привязка оборудования отличается от time-series привязки показаний?
5. Как `EquipmentId` и `Timestamp` используются при работе с измерениями?
6. Чем подключение Ontology через Foundry IQ Knowledge base отличается от подключения опубликованного Fabric data agent как инструмента?

---

## 17. Официальные материалы

- [Microsoft Learn — Create an ontology with Fabric IQ (manual lab)](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/23-build-ontology-manually.html)
- [Microsoft Learn — Ontology (preview): create entity types](https://learn.microsoft.com/en-us/fabric/iq/ontology/how-to-create-entity-types)
- [Microsoft Learn — Ontology (preview): bind data](https://learn.microsoft.com/en-us/fabric/iq/ontology/how-to-bind-data)
- [Microsoft Learn — Create an Ontology agent in Foundry IQ](https://learn.microsoft.com/en-us/fabric/iq/ontology/how-to-create-agent-foundry-iq)
- [Microsoft Learn — Connect agents to Fabric with Fabric IQ](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/fabric-iq)

**Примечание:** Microsoft может обновлять preview-интерфейс. Если подпись кнопки отличается, ориентируйтесь на название операции и актуальную страницу Microsoft Learn, а не на расположение элемента в старом скриншоте.
