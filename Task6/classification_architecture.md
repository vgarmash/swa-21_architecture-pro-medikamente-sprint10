# Архитектура движка классификации данных

## 1. Введение

### 1.1. Назначение документа

Данный документ описывает архитектуру движка классификации данных, построенного на базе OpenSource-компонентов. Движок предназначен для автоматической классификации данных перед их загрузкой в аналитическое хранилище (DWH).

### 1.2. Контекст

Движок классификации является частью аналитического слоя (BI-платформы), спроектированного в Task 2. Он обеспечивает:

- Автоматическое обнаружение конфиденциальных данных
- Присвоение тегов классификации (PII, MEDICAL, FINANCIAL, etc.)
- Интеграцию с ETL-пайплайнами
- Поддержку принципов Privacy By Design

### 1.3. Требования

**Функциональные требования:**
- Классификация данных по 4 категориям тегов (тип, чувствительность, законодательство, хранение)
- Поддержка 35+ категорий данных из реестра Task 1
- Автоматическое обнаружение изменений схемы данных
- Версионирование классификации
- Интеграция с Apache Airflow

**Нефункциональные требования:**
- Производительность: >100 GB/час
- Точность классификации: Precision >0.95, Recall >0.90
- Доступность: >99.5%
- Масштабируемость: горизонтальное масштабирование

---

## 2. Архитектурный обзор

### 2.1. Диаграмма контекста (C4 Level 1)

```plantuml
@startuml C4_Classification_Context
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4.puml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

title C4 Context Diagram: Data Classification Engine

Person(etl_engineer, "ETL Engineer", "Инженер данных, управляющий пайплайнами ETL")
Person(data_steward, "Data Steward", "Ответственный за качество и классификацию данных")
Person(bi_analyst, "BI Analyst", "Аналитик, работающий с данными DWH")

System_Boundary(sources, "Источники данных") {
    SystemDb(postgres_operational, "PostgreSQL", "Operational DB", "Операционная БД с бизнес-данными")
    SystemDb(onec_accounting, "1С:Бухгалтерия", "1C:Enterprise", "Бухгалтерский учёт (клиент-сервер)")
    SystemDb(onec_inventory, "1С:Торговля и склад", "1C:Enterprise", "Учёт ТМЦ (клиент-сервер)")
    SystemDb(files_storage, "Excel/CSV Files", "File Storage", "Файловые данные (Excel, CSV)")
}

System(classification_engine, "Data Classification Engine", "Классификация данных", "Автоматическая классификация данных перед загрузкой в DWH")

System_Boundary(dwh, "Data Warehouse") {
    System(dwh_clickhouse, "ClickHouse DWH", "DWH", "Аналитическое хранилище с 4-слойной архитектурой")
}

System_Boundary(bi, "BI Platform") {
    System(superset, "Apache Superset", "BI Platform", "Отчёты и дашборды для бизнеса")
}

System_Boundary(orchestration, "Оркестрация") {
    System(airflow, "Apache Airflow", "ETL Orchestrator", "Оркестрация пайплайнов ETL и классификации")
}

Rel(etl_engineer, airflow, "Управляет пайплайнами", "HTTP")
Rel(data_steward, classification_engine, "Настраивает правила классификации", "YAML/Git")
Rel(bi_analyst, superset, "Просматривает отчёты", "HTTPS")

Rel(postgres_operational, airflow, "Извлечение данных", "JDBC")
Rel(onec_accounting, airflow, "Извлечение данных", "ODBC")
Rel(onec_inventory, airflow, "Извлечение данных", "ODBC")
Rel(files_storage, airflow, "Загрузка файлов", "File read")

Rel(airflow, classification_engine, "Запускает классификацию", "HTTPS")

Rel(classification_engine, dwh_clickhouse, "Загрузка классифицированных данных", "JDBC")
Rel(dwh_clickhouse, superset, "Предоставляет данные для отчётов", "SQL")

UpdateRelStyle(etl_engineer, airflow, $offsetY="-60")
UpdateRelStyle(data_steward, classification_engine, $offsetX="-80")
UpdateRelStyle(bi_analyst, superset, $offsetY="60")

UpdateElementStyle(classification_engine, $bgColor="#4ECDC4", $borderColor="#2C7A7B")
UpdateElementStyle(dwh_clickhouse, $bgColor="#95E1D3")
UpdateElementStyle(superset, $bgColor="#FFE66D")

@enduml
```

### 2.2. Принципы проектирования

| Принцип | Реализация |
|---------|------------|
| **Minimal Complexity** | Использование готовых OpenSource-компонентов |
| **Privacy By Design** | Классификация до загрузки в DWH |
| **Scalability** | Stateless компоненты, Kubernetes |
| **Extensibility** | Плагины для новых правил классификации |
| **Observability** | Метрики, логи, трассировка |

---

## 3. Компоненты системы

### 3.1. Обзор компонентов

| Компонент | Технология | Назначение |
|-----------|------------|------------|
| **Classification API** | FastAPI (Python) | REST API для классификации |
| **Rule Engine** | Custom (Python + YAML) | Применение правил классификации |
| **ML Classifier** | Microsoft Presidio | ML-распознавание PII |
| **Tag Registry** | PostgreSQL | Хранение тегов классификации |
| **Schema Analyzer** | Custom (Python) | Анализ изменений схемы |
| **Airflow Operators** | Apache Airflow | Интеграция с ETL |

### 3.2. Диаграмма контейнеров (C4 Level 2)

```plantuml
@startuml C4_Classification_Engine_Container
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4.puml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

title Контейнерная диаграмма: Data Classification Engine

Person(etl_user, "ETL Engineer", "Инженер данных, управляющий пайплайнами")
Person(data_steward, "Data Steward", "Ответственный за качество и классификацию данных")

System_Boundary(boundary, "Data Classification System") {
    
    Container(api, "Classification API", "Python/FastAPI", "REST API для классификации данных")
    Container(rule_engine, "Rule Engine", "Python + YAML", "Движок правил классификации")
    Container(ml_classifier, "ML Classifier", "Microsoft Presidio", "ML-распознавание PII и конфиденциальных данных")
    Container(schema_analyzer, "Schema Analyzer", "Python", "Анализ изменений схемы данных")
    
    ContainerDb(tag_registry, "Tag Registry", "PostgreSQL 15+", "Реестр тегов классификации")
    ContainerDb(classification_rules, "Classification Rules", "YAML Files + Git", "Правила классификации (версионирование)")
    
    Container_Boundary(airflow, "Apache Airflow") {
        Container(etl_orchestrator, "ETL Orchestrator", "Airflow DAGs", "Оркестрация пайплайнов ETL и классификации")
        Container(classification_task, "Classification Task", "Airflow Operator", "Задача классификации в DAG")
    }
}

System_Boundary(dwh, "Data Warehouse (ClickHouse)") {
    ContainerDb(raw_storage, "Raw Storage", "ClickHouse", "Слой 1: Исходные данные (зашифровано)")
    ContainerDb(classified_storage, "Classified Storage", "ClickHouse", "Слой 2: Данные с тегами")
    ContainerDb(anonymized_storage, "Anonymized Storage", "ClickHouse", "Слой 3: Обезличенные данные")
    ContainerDb(data_marts, "Data Marts", "ClickHouse", "Слой 4: Бизнес-витрины")
}

System_Boundary(sources, "Источники данных") {
    SystemDb(postgres_operational, "PostgreSQL", "Operational DB", "Операционная БД (PostgreSQL)")
    SystemDb(onec_accounting, "1С:Бухгалтерия", "1C:Enterprise", "Бухгалтерский учёт")
    SystemDb(files_excel, "Excel/CSV Files", "File Storage", "Файловые данные")
}

Rel(etl_user, etl_orchestrator, "Управляет", "HTTP")
Rel(data_steward, tag_registry, "Просматривает/редактирует", "SQL")
Rel(data_steward, classification_rules, "Редактирует", "Git")

Rel(etl_orchestrator, classification_task, "Запускает", "Python")
Rel(classification_task, api, "Вызывает", "HTTPS")

Rel(api, rule_engine, "Применяет правила", "Python call")
Rel(api, ml_classifier, "Запускает ML-классификацию", "Python call")
Rel(api, schema_analyzer, "Проверяет схему", "Python call")

Rel(rule_engine, classification_rules, "Загружает правила", "File read")
Rel(api, tag_registry, "Сохраняет теги", "JDBC")

Rel(postgres_operational, etl_orchestrator, "Извлечение данных", "JDBC")
Rel(onec_accounting, etl_orchestrator, "Извлечение данных", "ODBC")
Rel(files_excel, etl_orchestrator, "Загрузка файлов", "File read")

Rel(etl_orchestrator, raw_storage, "Загрузка сырых данных", "JDBC")
Rel(raw_storage, classified_storage, "Классифицированные данные", "JDBC")
Rel(classified_storage, anonymized_storage, "Обезличенные данные", "JDBC")
Rel(anonymized_storage, data_marts, "Агрегированные данные", "JDBC")

UpdateRelStyle(etl_user, etl_orchestrator, $offsetY="-60")
UpdateRelStyle(data_steward, tag_registry, $offsetX="40")
UpdateRelStyle(api, rule_engine, $offsetY="-30")
UpdateRelStyle(api, ml_classifier, $offsetY="30")

@enduml
```


## 4. Метрики и мониторинг

### 4.1. Метрики качества классификации

| Метрика | Описание | Формула | Целевое значение | Источник |
|---------|----------|---------|------------------|----------|
| **Precision (Точность)** | Доля правильно классифицированных колонок среди всех классифицированных | TP / (TP + FP) | > 0.95 | Classification API |
| **Recall (Полнота)** | Доля правильно классифицированных колонок среди всех конфиденциальных | TP / (TP + FN) | > 0.90 | Classification API |
| **F1-Score** | Гармоническое среднее точности и полноты | 2 × (Precision × Recall) / (Precision + Recall) | > 0.92 | Classification API |
| **Coverage (Покрытие)** | Доля классифицированных колонок от общего числа | Classified / Total | > 0.95 | Tag Registry |
| **Avg Confidence** | Средняя уверенность классификации | Σ confidence / N | > 0.85 | Classification API |
| **Manual Review Rate** | Доля колонок, требующих ручной проверки | Manual / Total | < 0.05 | Tag Registry |

**Обозначения:**
- **TP (True Positive):** Правильно классифицированные конфиденциальные данные
- **FP (False Positive):** Ложно классифицированные как конфиденциальные
- **FN (False Negative):** Пропущенные конфиденциальные данные (критично!)

### 4.2. Метрики производительности

| Метрика | Описание | Целевое значение | Источник |
|---------|----------|------------------|----------|
| **Classification Latency** | Время классификации одной таблицы | < 5 секунд | Prometheus |
| **Throughput** | Пропускная способность (таблиц в минуту) | > 100 таблиц/мин | Prometheus |
| **Data Volume** | Объём обработанных данных (GB/час) | > 100 GB/час | Airflow |
| **API Response Time** | Время ответа API (p50, p95, p99) | p95 < 500ms | Prometheus |
| **Error Rate** | Доля ошибочных запросов к API | < 0.1% | Prometheus |
| **ETL Pipeline Duration** | Общая длительность ETL-пайплайна | < 30 минут | Airflow |

### 4.3. Метрики безопасности

| Метрика | Описание | Целевое значение | Источник |
|---------|----------|------------------|----------|
| **Tagged Data Ratio** | Доля данных с тегами классификации | 100% | Tag Registry |
| **Anonymized Data Ratio** | Доля обезличенных данных в DWH | 100% | DWH Audit |
| **Security Incidents** | Количество инцидентов утечки данных | 0 | Security Log |
| **Anomaly Detection Time** | Время обнаружения аномалии доступа | < 5 минут | Audit Log Service |
| **Unauthorized Access Attempts** | Попытки несанкционированного доступа | 0 | Audit Log Service |
| **Encryption Coverage** | Доля зашифрованных конфиденциальных данных | 100% | Tag Registry |

### 4.4. Метрики системы

| Метрика | Описание | Целевое значение | Источник |
|---------|----------|------------------|----------|
| **API Availability** | Доступность Classification API | > 99.5% | Prometheus |
| **CPU Usage** | Использование CPU подом классификации | < 70% | Kubernetes Metrics |
| **Memory Usage** | Использование памяти подом | < 80% | Kubernetes Metrics |
| **Pod Replicas** | Количество реплик Classification API | 3-10 (auto-scale) | Kubernetes |
| **Database Connections** | Активные подключения к Tag Registry | < 100 | PostgreSQL |
| **Queue Depth** | Глубина очереди задач классификации | < 100 | Redis/Airflow |

### 4.5. Метрики тегов

| Метрика | Описание | Целевое значение | Источник |
|---------|----------|------------------|----------|
| **Total Tags** | Общее количество активных тегов | 35+ | Tag Registry |
| **Tags by Category** | Количество тегов по категориям | N/A | Tag Registry |
| **Tags by Sensitivity** | Распределение по уровням чувствительности | N/A | Tag Registry |
| **Tag Changes (24h)** | Изменения тегов за последние 24 часа | < 10 | Tag Registry |
| **Confidence Distribution** | Распределение уверенности классификации | N/A | Tag Registry |
| **Classification Method** | Распределение по методам (rule/ml/manual) | N/A | Tag Registry |

### 4.6. Дашборды мониторинга

| Дашборд | Аудитория | Метрики | Частота обновления |
|---------|-----------|---------|-------------------|
| **Classification Health** | ETL Engineer, Data Steward | Precision, Recall, F1, Coverage | Real-time (1 мин) |
| **Performance Overview** | ETL Engineer | Latency, Throughput, Error Rate | Real-time (1 мин) |
| **Security Monitor** | Security Team | Incidents, Anomalies, Unauthorized Access | Real-time (30 сек) |
| **Tag Registry Stats** | Data Steward | Total Tags, Changes, Distribution | Hourly |
| **System Resources** | DevOps | CPU, Memory, Pod Replicas, Availability | Real-time (30 сек) |
| **ETL Pipeline Status** | ETL Engineer | Pipeline Duration, Success Rate, Volume | Per DAG run |

### 4.7. Алерты

| Алерт | Условие | Приоритет | Канал уведомления |
|-------|---------|-----------|-------------------|
| **Low Precision** | Precision < 0.90 | High | Slack, Email |
| **Low Recall** | Recall < 0.85 | High | Slack, Email |
| **Low Coverage** | Coverage < 0.90 | Medium | Slack |
| **High Error Rate** | Error Rate > 1% | High | Slack, PagerDuty |
| **High Latency** | p95 Latency > 1s | Medium | Slack |
| **Security Incident** | Security Incidents > 0 | Critical | PagerDuty, Phone |
| **API Down** | Availability < 99% | Critical | PagerDuty, Phone |
| **Pod CrashLoop** | Pod restarts > 3 в час | High | Slack, PagerDuty |
| **Queue Backlog** | Queue Depth > 500 | Medium | Slack |
| **Tag Registry Down** | DB connection failed | Critical | PagerDuty, Phone |

---

## 5. Выводы

### 5.1. Резюме архитектуры

**Спроектированная архитектура включает:**

1. **Classification API** — REST API на FastAPI для внешней интеграции
2. **Rule Engine** — Движок правил на YAML с приоритетами
3. **ML Classifier** — Microsoft Presidio для распознавания PII
4. **Schema Analyzer** — Обнаружение изменений схемы
5. **Tag Manager** — Управление жизненным циклом тегов
6. **Tag Registry** — PostgreSQL для хранения тегов
7. **Airflow Integration** — DAG для автоматизации классификации

### 5.2. Соответствие требованиям

| Требование | Реализация |
|------------|------------|
| **Автоматическая классификация** | Rule Engine + ML Classifier |
| **35+ категорий данных** | YAML правила для всех категорий |
| **Обнаружение изменений схемы** | Schema Analyzer с хэшированием |
| **Версионирование** | Tag Registry с версионированием |
| **Интеграция с ETL** | Airflow DAG + Custom Operator |
| **Производительность >100 GB/час** | Горизонтальное масштабирование (K8s HPA) |
| **Precision >0.95** | Комбинация rule-based + ML |
| **Scalability** | Kubernetes + шардирование ClickHouse |

### 5.3. Метрики эффективности

| Категория метрик | Ключевые метрики | Целевые значения |
|------------------|------------------|------------------|
| **Качество классификации** | Precision, Recall, F1-Score | > 0.95, > 0.90, > 0.92 |
| **Производительность** | Latency, Throughput | < 5 сек, > 100 таблиц/мин |
| **Безопасность** | Security Incidents, Encryption Coverage | 0, 100% |
| **Доступность** | API Availability | > 99.5% |

### 5.4. Следующие шаги

| Этап | Срок | Задачи |
|------|------|--------|
| **1. Разработка прототипа** | 2 недели | Настройка API, базовые правила |
| **2. Создание правил классификации** | 2 недели | YAML правила для 35+ категорий |
| **3. Интеграция с ETL** | 1 неделя | Airflow DAG, тестирование |
| **4. Тестирование и валидация** | 2 недели | Валидация метрик, нагрузочное тестирование |
| **5. Промышленное развёртывание** | 1 неделя | Kubernetes, мониторинг, алерты |
