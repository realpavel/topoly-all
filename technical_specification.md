# Описание
## Наименование
Topoly — веб-ориентированное приложение для исследования и анализа улично-дорожной сети города.
## Предметная область
Предметная область охватывает анализ городской топологии улиц, выявление самоорганизующихся городских пространств и районов с регулярной планировкой.
Система позволяет загружать картографические данные из OpenStreetMap, представлять улично-дорожную сеть в виде графа, вычислять метрики топологии (среднее число связей/пересечений улиц) как для города в целом, так и по отдельным районам.
Заказчик: Центр аналитики динамических процессов и систем СПбГУ
# Данные
## Логика проектирования базы данных
База данных проектировалась для хранения и анализа графовой структуры улично-дорожной сети города. <br/>
Основные задачи: <br/>
хранение информации о городах и их метаданных;
представление дорожной сети как графа (узлы — перекрёстки, рёбра — сегменты улиц);
сохранение исходных данных OSM (точки, пути) для возможности повторной обработки;
хранение произвольных тегов OSM через паттерн EAV. <br/>
## Основные сущности
Cities — список загруженных городов. Хранит название города и флаг завершённости загрузки.

CityProperties — метаданные города: координаты центра, население, плотность населения, часовой пояс, время создания записи.

AccessNodes — узлы графа доступности (перекрёстки, здания, точки интереса). Для каждого узла хранятся координаты, тип, название и теги OSM в формате JSON.

AccessEdges — рёбра графа доступности (сегменты улиц). Связывает два узла (id_src → id_dst), хранит тип дороги, длину в метрах, флаг связи со зданием.

Points — исходные точки (вершины) из OSM с координатами.

Ways — пути OSM, привязанные к городу.

Edges — сегменты путей, связывающие две точки.

Properties — справочник свойств для паттерна EAV.

WayProperties / PointProperties — значения свойств для путей и точек (паттерн EAV для хранения тегов OSM). <br/>

## Основные сценарии использования
* Загрузка нового города

Исследователь вводит название города; система загружает данные из OSM; парсит и сохраняет граф улиц в БД.


* Визуализация топологии

Пользователь выбирает город; система отображает граф на карте с возможностью масштабирования.


* Расчёт метрик

Исследователь запускает расчёт; система вычисляет среднюю степень узлов, betweenness centrality, коэффициент кластеризации.


* Анализ по районам

Фильтрация данных по районам города; сравнительный анализ метрик между районами.<br/>

## Основные связи
* Cities (1) ←→ (N) AccessNodes — город содержит множество узлов графа
* Cities (1) ←→ (N) AccessEdges — город содержит множество рёбер
* AccessNodes (1) ←→ (N) AccessEdges — узел может быть началом/концом многих рёбер
* Cities (1) ←→ (1) CityProperties — один город имеет одну запись свойств
* Ways (1) ←→ (N) Edges — путь состоит из сегментов
* Points (1) ←→ (N) Edges — точка может быть концом многих сегментов
* Properties (1) ←→ (N) WayProperties/PointProperties — одно свойство для многих значений 

## Для каждого элемента данных — ограничения

* Cities — города
* CityProperties — свойства города
* AccessNodes — узлы графа доступности
* AccessEdges — рёбра графа доступности
* Points — точки OSM
* Ways — пути OSM
* Edges — сегменты путей
* Properties — справочник свойств
* WayProperties — свойства путей (EAV)
* PointProperties — свойства точек (EAV)

### Cities — города
| Название поля | Тип | Ограничения |
|---------------|-----|-------------|
| id | BIGINT | PRIMARY KEY, AUTOINCREMENT |
| id\_property | INTEGER | FOREIGN KEY → CityProperties.id |
| city\_name | VARCHAR(30) | UNIQUE, NOT NULL |
| downloaded | BOOLEAN | INDEX, DEFAULT FALSE |

### CityProperties — свойства города
| Название поля | Тип | Ограничения |
|---------------|-----|-------------|
| id | INTEGER | PRIMARY KEY, AUTOINCREMENT |
| c\_latitude | FLOAT | NOT NULL |
| c\_longitude | FLOAT | NOT NULL |
| id\_district | INTEGER | Может отсутствовать |
| id\_start\_polygon | BIGINT | Может отсутствовать |
| population | INTEGER | Может отсутствовать |
| population\_density | FLOAT | DEFAULT 0, INDEX |
| time\_zone | VARCHAR(6) | Может отсутствовать |
| time\_created | TIMESTAMP WITH TIME ZONE | INDEX, DEFAULT NOW() |

### AccessNodes — узлы графа доступности
| Название поля | Тип | Ограничения |
|---------------|-----|-------------|
| id | BIGINT | PRIMARY KEY, AUTOINCREMENT |
| id\_city | BIGINT | FOREIGN KEY → Cities.id, ON DELETE CASCADE, NOT NULL |
| source\_type | VARCHAR(16) | NOT NULL, значения: osm\_node, building, intersection |
| source\_id | BIGINT | Может отсутствовать |
| node\_type | VARCHAR(16) | NOT NULL, значения: intersection, building, poi |
| longitude | FLOAT | NOT NULL, -180 ≤ x ≤ 180 |
| latitude | FLOAT | NOT NULL, -90 ≤ x ≤ 90 |
| name | VARCHAR(128) | Может отсутствовать |
| tags | TEXT | JSON-формат, может отсутствовать |

### AccessEdges — рёбра графа доступности
| Название поля | Тип | Ограничения |
|---------------|-----|-------------|
| id | BIGINT | PRIMARY KEY, AUTOINCREMENT |
| id\_city | BIGINT | FOREIGN KEY → Cities.id, ON DELETE CASCADE, NOT NULL |
| id\_src | BIGINT | FOREIGN KEY → AccessNodes.id, ON DELETE CASCADE, NOT NULL |
| id\_dst | BIGINT | FOREIGN KEY → AccessNodes.id, ON DELETE CASCADE, NOT NULL |
| source\_way\_id | BIGINT | Может отсутствовать |
| road\_type | VARCHAR(32) | NOT NULL |
| length\_m | FLOAT | ≥ 0, может отсутствовать |
| is\_building\_link | BOOLEAN | NOT NULL, DEFAULT FALSE |
| name | VARCHAR(128) | Может отсутствовать |

### Points — точки OSM
| Название поля | Тип | Ограничения |
|---------------|-----|-------------|
| id | BIGINT | PRIMARY KEY (ID из OSM) |
| longitude | FLOAT | NOT NULL, -180 ≤ x ≤ 180 |
| latitude | FLOAT | NOT NULL, -90 ≤ x ≤ 90 |

### Ways — пути OSM
| Название поля | Тип | Ограничения |
|---------------|-----|-------------|
| id | BIGINT | PRIMARY KEY (ID из OSM) |
| id\_city | BIGINT | FOREIGN KEY → Cities.id, ON UPDATE CASCADE, NOT NULL |

### Edges — сегменты путей
| Название поля | Тип | Ограничения |
|---------------|-----|-------------|
| id | INTEGER | PRIMARY KEY, AUTOINCREMENT |
| id\_way | BIGINT | FOREIGN KEY → Ways.id, ON UPDATE CASCADE, NOT NULL |
| id\_src | BIGINT | FOREIGN KEY → Points.id, ON UPDATE CASCADE, NOT NULL |
| id\_dist | BIGINT | FOREIGN KEY → Points.id, ON UPDATE CASCADE, NOT NULL |

### Properties — справочник свойств
| Название поля | Тип | Ограничения |
|---------------|-----|-------------|
| id | INTEGER | PRIMARY KEY, AUTOINCREMENT |
| property | VARCHAR(50) | NOT NULL |

### WayProperties — свойства путей (EAV)
| Название поля | Тип | Ограничения |
|---------------|-----|-------------|
| id | INTEGER | PRIMARY KEY, AUTOINCREMENT |
| id\_way | BIGINT | FOREIGN KEY → Ways.id, ON UPDATE CASCADE, NOT NULL |
| id\_property | BIGINT | FOREIGN KEY → Properties.id, NOT NULL |
| value | VARCHAR | NOT NULL |

### PointProperties — свойства точек (EAV)
| Название поля | Тип | Ограничения |
|---------------|-----|-------------|
| id | INTEGER | PRIMARY KEY, AUTOINCREMENT |
| id\_point | BIGINT | FOREIGN KEY → Points.id, ON UPDATE CASCADE, NOT NULL |
| id\_property | INTEGER | FOREIGN KEY → Properties.id, NOT NULL |
| value | VARCHAR | NOT NULL |


## Общие ограничения целостности
* Координаты должны быть валидными<br/>
longitude: -180 ≤ x ≤ 180; latitude: -90 ≤ x ≤ 90
* Ребро не может ссылаться на один и тот же узел<br/>
id_src ≠ id_dst в AccessEdges и Edges
* Каскадное удаление при удалении города<br/>
При удалении записи из Cities автоматически удаляются связанные AccessNodes и AccessEdges

* Уникальность названия города<br/>
city_name должен быть уникальным в таблице Cities
* Идентификаторы OSM используют BIGINT<br/>
OSM node ID — 64-битные числа, Integer недостаточен
* Город должен быть загружен перед расчётом метрик<br/>
Проверка Cities.downloaded = TRUE перед операциями с графом

# Пользовательские роли
| Роль | Ответственность | Количество пользователей |
|------|-----------------|--------------------------|
| **Гость** | Просмотр списка городов, базовая визуализация графа на карте. Без возможности модификации данных. | Неограниченно |
| **Исследователь** | Загрузка новых городов из OSM, запуск расчёта метрик топологии, фильтрация по районам, экспорт данных в CSV/JSON, сравнительный анализ районов. | 10–50 человек |
| **Администратор** | Управление пользователями, очистка кэша и данных, мониторинг системы, системные настройки. | 1–3 человека |
# UI / API 
## UI (Пользовательский интерфейс)

| Компонент | Путь | Описание |
|-----------|------|----------|
| `App.tsx` | `front/src/app/App.tsx` | Корневой компонент приложения |
| `TownsPage` | `front/src/pages/TownsPage/` | Страница со списком загруженных городов |
| `TownPage` | `front/src/pages/TownPage/` | Страница конкретного города с картой и метриками |
| `MapWidget` | `front/src/widgets/MapWidget/` | Интерактивная карта с визуализацией графа улиц |
| `RoadsWidget` | `front/src/widgets/RoadsWidget/` | Таблица с информацией об улицах и их метриках |

## API (REST эндпоинты)

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/api/cities` | Получить список всех городов |
| GET | `/api/cities/{id}` | Получить информацию о конкретном городе |
| POST | `/api/cities` | Добавить новый город (загрузка из OSM) |
| DELETE | `/api/cities/{id}` | Удалить город и все связанные данные |
| GET | `/api/cities/{id}/graph` | Получить граф города (узлы и рёбра) |
| GET | `/api/cities/{id}/metrics` | Получить метрики топологии города |
| GET | `/api/cities/{id}/districts` | Получить список районов города |
| GET | `/api/cities/{id}/districts/{dist_id}/metrics` | Получить метрики конкретного района |

## Структура backend

| Файл/Директория | Описание |
|-----------------|----------|
| `api/backend/main.py` | Точка входа FastAPI приложения |
| `api/backend/app/routes.py` | Определение API маршрутов |
| `api/backend/app/lifespan.py` | Управление жизненным циклом приложения |
| `api/backend/application/service_facade.py` | Фасад для доступа к сервисам |
| `api/backend/application/city_service.py` | Сервис управления городами |
| `api/backend/application/graph_service.py` | Сервис работы с графом |
| `api/backend/application/region_service.py` | Сервис работы с районами |
| `api/backend/infrastructure/database.py` | Конфигурация подключения к БД |
| `api/backend/infrastructure/models.py` | SQLAlchemy модели |
| `api/backend/infrastructure/repositories/` | Репозитории для работы с данными |
| `api/backend/domain/schemas.py` | Pydantic схемы для валидации |

## Структура frontend

| Файл/Директория | Описание |
|-----------------|----------|
| `front/src/main.tsx` | Точка входа React приложения |
| `front/src/app/` | Конфигурация приложения и провайдеры |
| `front/src/entities/city/` | Сущность "Город" |
| `front/src/entities/graph/` | Сущность "Граф" |
| `front/src/entities/region/` | Сущность "Район" |
| `front/src/shared/api/` | HTTP-клиент и API методы |
| `front/src/shared/ui/` | Переиспользуемые UI компоненты |


# Технологии разработки
## Язык программирования
Backend: Python 3.9+
Frontend: TypeScript, React
## Фреймворки и библиотеки
Backend: FastAPI, SQLAlchemy, databases (async), NetworkX, Pandas<br/>
Frontend: React, Vite, Leaflet/MapLibre
## СУБД
PostgreSQL
## Инфраструктура
Docker, Docker Compose
Nginx (для продакшн-сборки фронтенда)
# Тестирование
Unit-тесты с использованием pytest (см. директорию tests/)<br/>
Тестирование сервисов: test_city_service.py, test_graph_service.py, test_region_service.py<br/>
Тестирование репозиториев: test_ingestion_repository.py<br/>
Тестирование API routes: test_routes.py<br/>
Тестирование конфигурации БД: test_database_config.py<br/>
Тестирование парсинга OSM: test_osm_handler.py, test_street_name_parser.py<br/>
Интеграционные тесты жизненного цикла приложения: test_lifespan.py<br/>
