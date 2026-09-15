# Smart Trip Assistant: минимальный агентный прототип

Один notebook демонстрирует детерминированный обмен сообщениями между Trip
Manager и Travel Search Agent на LangGraph. Он не использует LLM, API-ключи,
внешние данные и реальные операции бронирования.

## Локальный запуск

Из каталога `hw/07-hw/smart-trip-assistant` выполните:

```bash
uv sync
uv run jupyter lab
```

Откройте `smart_trip_agents.ipynb` и выполните `Run All Cells`. UV создаст
локальное окружение `.venv` в текущем каталоге.

Для неинтерактивной проверки:

```bash
uv run jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.timeout=180 smart_trip_agents.ipynb
```

## Запуск в Google Colab

Локальный Jupyter notebook и Colab notebook: это один и тот же файл.

1. Откройте <https://colab.research.google.com/>.
2. Выберите `File → Upload notebook` и загрузите `smart_trip_agents.ipynb`.
3. Выполните `Runtime → Run all`.
4. Для сдачи выберите `File → Save a copy in Drive`.
5. Нажмите `Share`, включите доступ по ссылке и передайте ссылку преподавателю.

Первая кодовая ячейка автоматически устанавливает `langgraph==1.2.11`, если
этой версии нет в текущей Colab VM. GPU и подключение Google Drive не нужны.

## Ожидаемый результат

Notebook печатает две трассы `User → Manager → Searcher → Manager → User`:

- предложения найдены и отфильтрованы по бюджету;
- подходящих предложений нет, но граф завершается корректным ответом.

Последняя строка подтверждает успешное выполнение всех встроенных проверок.
