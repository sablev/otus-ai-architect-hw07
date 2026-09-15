# Умный помощник для оформления командировок

Архитектурное проектирование мультиагентной системы с RAG-пайплайном для
подсистемы «Умный помощник» оформления командировок: C4-модель в LikeC4 и
минимальный исполняемый прототип на LangGraph. Учебный проект курса
OTUS «AI Architect» (ДЗ 07).

## Онлайн-версия

Интерактивные диаграммы (LikeC4), прямые ссылки на каждое представление:

- [Диаграмма системного контекста (C1)](https://sablev.github.io/otus-ai-architect-hw07/#/view/index)
- [Диаграмма контейнеров (C2)](https://sablev.github.io/otus-ai-architect-hw07/#/view/smart-assistant--containers)
- [Компоненты Agent Orchestrator (C3)](https://sablev.github.io/otus-ai-architect-hw07/#/view/smart-assistant--agent-orchestrator--components)
- [Сценарий «Планирование поездки» (диаграмма последовательности)](https://sablev.github.io/otus-ai-architect-hw07/#/view/smart-assistant--plan-trip--dynamic?dynamic=sequence)
- [Сценарий «Гибридный поиск Policy RAG» (диаграмма последовательности)](https://sablev.github.io/otus-ai-architect-hw07/#/view/smart-assistant--policy-rag--dynamic?dynamic=sequence)
- [Сценарий «Подготовка индексов Policy RAG» (диаграмма последовательности)](https://sablev.github.io/otus-ai-architect-hw07/#/view/smart-assistant--policy-ingestion--dynamic?dynamic=sequence)

Исполняемый прототип: детерминированный обмен сообщениями между Trip Manager
и Travel Search Agent на LangGraph, без LLM, API-ключей и внешних данных.

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sablev/otus-ai-architect-hw07/blob/main/smart-trip-assistant/smart_trip_agents.ipynb)

## Скоуп

Архитектурная модель покрывает полный контур оформления командировки: сбор
параметров поездки, применение корпоративной политики через Policy RAG,
параллельный подбор транспорта и гостиницы, проверку бюджета, согласование
отклонений и бронирование после явного подтверждения сотрудника.

Минимальный прототип намеренно сужен до одной задачи, которую Manager поручает
Searcher на локальном каталоге предложений: LLM, Policy RAG, реальные
провайдеры и бронирование в него не входят, чтобы обмен агентов был
воспроизводимым и не требовал API-ключей.

## Отклонения от задания

- Диаграммы сделаны в LikeC4 (DSL как код, сборка сайта в CI), а не в
  Draw.io / Structurizr / Holst из примеров задания.
- Прототип выполнен на LangGraph (допустимо формулировкой задания
  «LangChain/LangGraph») и запускается и локально через UV, и в
  Google Colab: это один и тот же файл notebook.

## Состав репозитория

| Путь | Содержимое |
|---|---|
| `likec4/` | Модель LikeC4: системный контекст (C1), контейнеры (C2), компоненты Agent Orchestrator (C3) и три динамических сценария |
| `likec4/exports/` | Необязательные статические PNG-снимки; актуальные диаграммы GitHub Pages собираются непосредственно из исходников `.c4` |
| `smart-trip-assistant/smart_trip_agents.ipynb` | Минимальный исполняемый прототип: Manager поручает поиск агенту Searcher и формирует итоговый ответ |
| `smart-trip-assistant/` | UV-проект прототипа: `pyproject.toml`, `uv.lock`, инструкция по локальному запуску и публикации в Colab |

## Локальный запуск

**Интерактивные диаграммы** (dev-сервер LikeC4 на http://localhost:5174,
на хосте нужен только Docker):

```bash
docker compose -f likec4/docker-compose.yaml up -d
```

**Исполняемый прототип** (на хосте нужен только
[UV](https://docs.astral.sh/uv/)):

```bash
cd smart-trip-assistant
uv sync
uv run jupyter lab
```

Откройте `smart_trip_agents.ipynb` и выполните `Run All Cells`. Для
неинтерактивной проверки:

```bash
uv run jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.timeout=180 smart_trip_agents.ipynb
```

Notebook проходит два сценария: предложения найдены в бюджете и подходящих
предложений нет; встроенные проверки подтверждают корректность трассы
`User → Manager → Searcher → Manager → User` в обоих случаях.

## Ключевые архитектурные решения

- **Детерминированный Supervisor вместо свободного диалога агентов.** Trip
  Supervisor ведёт фиксированный граф процесса и выбирает следующий узел;
  агенты-исполнители (Policy, Transport, Hotel, Budget) получают только
  нужные типизированные секции Trip Dossier, а не всю историю.
- **Гибридный Policy RAG.** До ранжирования фрагменты фильтруются по
  метаданным: дате, стране, категории и ACL. Кандидаты из векторного и
  BM25-поиска объединяются через RRF и переранжируются моделью CrossEncoder;
  результат: PolicyConstraints с цитатами на разделы политик. BM25-индекс
  отдельно отвечает за точные термины, лимиты и коды политик.
- **Политики загружаются офлайн.** Policy Ingestion Worker разбирает версии
  документов из корпоративного репозитория, режет их на фрагменты по
  структуре разделов и обновляет оба индекса вне критического пути запроса.
- **Бронирование отделено от выбора и защищено подтверждением.** Выбор
  варианта не равен подтверждению: Booking Executor вызывается только после
  явного подтверждения сотрудника и отправляет идемпотентный запрос по
  выбранным offer_id. Отклонения от политики уходят в систему согласований.
- **Schema & Loop Guard.** Схемы результатов агентов-исполнителей
  валидируются, retry_count ограничен, а глобальный `recursion_limit` графа
  гарантирует, что процесс завершится с сохранённым состоянием и
  структурированной ошибкой.

## Публикация

Сайт на GitHub Pages собирает workflow
[.github/workflows/pages.yml](.github/workflows/pages.yml): команда
`likec4 build` проходит по исходникам `.c4`, а базовый путь репозитория и
hash-маршруты дают прямые ссылки на представления.
