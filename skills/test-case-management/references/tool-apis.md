# Справочник по API Test IT — ручные тест-кейсы

В Test IT ручной тест-кейс хранится в библиотеке тестов как рабочий элемент типа `TestCases`. Пример ниже показывает, как выбрать раздел, создать кейс с предусловием и двумя шагами, а затем проверить, что сохранилось. Схему REST API для своей версии Test IT можно открыть по пути `/swagger` на адресе системы. [Работа с API Test IT](https://docs.testit.software/user-guide/work-with-api.html)

| Что нужно сделать | Метод или значение |
|---|---|
| Указать базовый URL | `https://<адрес-вашей-системы>/api/v2` |
| Передать приватный токен | Заголовок `Authorization: PrivateToken <токен>` |
| Получить проекты | `GET /api/v2/projects` |
| Получить разделы проекта | `GET /api/v2/projects/{projectId}/sections` |
| Создать ручной кейс вместе с шагами | `POST /api/v2/workItems`, тип `TestCases` |
| Прочитать сохранённый кейс | `GET /api/v2/workItems/{id}` |

## Шаг 1 — подключиться и выбрать раздел

Создай приватный API-токен в разделе **Настройки профиля → Безопасность**. Скопируй значение сразу после создания. Выполняй примеры в одной сессии Bash с установленным `curl`. Введи адрес своей системы Test IT без `/api/v2`, например `https://testit.example.com`. При вводе токена символы не отображаются. [Создание API-токена](https://docs.testit.software/user-guide/user-settings.html)

```bash
read -r -p 'Адрес Test IT: ' TESTIT_URL
read -r -s -p 'Приватный API-токен: ' TESTIT_TOKEN
printf '\n'
API_URL="${TESTIT_URL%/}/api/v2"

curl -sS -w '\nHTTP %{http_code}\n' \
  -H "Authorization: PrivateToken $TESTIT_TOKEN" \
  "$API_URL/projects"
```

В ответе найди нужный проект по имени и скопируй его `id` (UUID). Следующий запрос вернёт разделы именно этого проекта. Если список разбит на страницы, используй параметры `skip` и `take`. [Методы проектов](https://github.com/testit-tms/api-client-python/blob/main/docs/ProjectsApi.md#get_all_projects) · [Метод получения разделов](https://github.com/testit-tms/api-client-python/blob/main/docs/ProjectSectionsApi.md#get_sections_by_project_id)

```bash
read -r -p 'UUID проекта: ' PROJECT_ID
curl -sS -w '\nHTTP %{http_code}\n' \
  -H "Authorization: PrivateToken $TESTIT_TOKEN" \
  "$API_URL/projects/$PROJECT_ID/sections"
```

## Шаг 2 — создать кейс с отдельным результатом для каждого шага

Пример проверяет одно правило. При стоимости корзины 1 000 ₽ активный промокод `SAVE10` даёт скидку 10 %. Вставь `id` выбранного раздела. Команда создаст в нём один кейс.

```bash
read -r -p 'UUID раздела: ' SECTION_ID
curl -sS -w '\nHTTP %{http_code}\n' \
  -H "Authorization: PrivateToken $TESTIT_TOKEN" \
  -H 'Content-Type: application/json' \
  -X POST "$API_URL/workItems" \
  --data-binary @- <<JSON
{
  "projectId": "$PROJECT_ID",
  "sectionId": "$SECTION_ID",
  "name": "Корзина — промокод SAVE10 уменьшает итог с 1 000 ₽ до 900 ₽",
  "entityTypeName": "TestCases",
  "duration": 60000,
  "state": "NotReady",
  "priority": "Medium",
  "preconditionSteps": [
    {
      "action": "Открыть корзину с одним товаром стоимостью 1 000 ₽.",
      "testData": "Количество: 1. Скидок нет. Промокод SAVE10 активен и даёт скидку 10 %.",
      "expected": "В строке «Итого» указано 1 000 ₽."
    }
  ],
  "steps": [
    {
      "action": "Ввести промокод в поле «Промокод».",
      "testData": "SAVE10",
      "expected": "В поле «Промокод» отображается SAVE10."
    },
    {
      "action": "Нажать кнопку «Применить».",
      "expected": "В строке «Итого» указано 900 ₽."
    }
  ]
}
JSON
```

### Как устроено тело запроса

| Поле | Что передать |
|---|---|
| `projectId`, `sectionId` | UUID проекта и UUID раздела из первого шага. По схеме `sectionId` необязателен. В примере он задан явно, чтобы кейс попал в выбранный раздел. |
| `name`, `entityTypeName` | Название кейса и точное значение `TestCases` для ручного тест-кейса. |
| `duration` | Оценку длительности выполнения в миллисекундах. `60000` означает одну минуту. |
| `state`, `priority` | Состояние и приоритет. В примере новый кейс получает состояние `NotReady` и средний приоритет `Medium`. |
| `preconditionSteps`, `steps`, `postconditionSteps` | Массивы предусловий, основных шагов и постусловий. В примере действий после теста нет, поэтому поле постусловий опущено. |
| `action`, `expected`, `testData` | Внутри шага укажи действие и наблюдаемый результат. Если нужны конкретные входные данные, помести их в `testData`. |

Обязательные поля модели создания — `projectId`, `name`, `entityTypeName`, `duration`, `state` и `priority`. Для ручной проверки заполняй `steps` в том же POST. Держи `action`, `expected` и `testData` одного шага в одном объекте, чтобы при чтении кейса было видно, какое ожидание относится к действию. [Модель кейса](https://github.com/testit-tms/api-client-js/blob/main/docs/CreateWorkItemApiModel.md) · [Модель шага](https://github.com/testit-tms/api-client-js/blob/main/docs/CreateStepApiModel.md)

## Шаг 3 — проверить, что сохранилось

При успешном создании Test IT отвечает `201 Created` и возвращает рабочий элемент. Скопируй `id` из ответа. Это UUID созданного кейса. Поле `globalId` содержит его числовой идентификатор. Метод чтения принимает оба варианта. [Создание и чтение рабочих элементов](https://github.com/testit-tms/api-client-python/blob/main/docs/WorkItemsApi.md) · [Поля ответа](https://github.com/testit-tms/api-client-python/blob/main/docs/WorkItemApiResult.md)

```bash
read -r -p 'ID созданного кейса: ' WORK_ITEM_ID
curl -sS -w '\nHTTP %{http_code}\n' \
  -H "Authorization: PrivateToken $TESTIT_TOKEN" \
  "$API_URL/workItems/$WORK_ITEM_ID"
```

При ответе `200 OK` сравни `projectId` и `sectionId` с выбранными UUID. В `preconditionSteps` должно быть одно предусловие, а в `steps` — ровно два шага в исходном порядке. У первого шага проверь `testData` со значением `SAVE10`, у второго — ожидаемый итог **900 ₽**. Так можно обнаружить случай, когда карточка создалась, но данные или ожидание оказались не в том поле.

## Несколько кейсов и повторная отправка

Для действующего, просроченного и уже использованного промокода подготовь три самостоятельных кейса. Отправляй по одному JSON-объекту на `POST /api/v2/workItems` и сохраняй возвращённый `id` после каждого ответа `201`. Если соединение оборвалось после отправки, перед повтором проверь выбранный раздел. Кейс мог сохраниться, даже если ответ не дошёл. [Контракт метода создания](https://github.com/testit-tms/api-client-python/blob/main/docs/WorkItemsApi.md#api_v2_work_items_post)

## Как разобрать ошибку

| Ответ API | Что проверить |
|---|---|
| `400` или `422` | Прочитай тело ошибки. Сверь обязательные поля, типы значений и допустимые значения `entityTypeName`, `state` и `priority` со Swagger своего экземпляра. |
| `401` | Проверь значение токена и формат заголовка `Authorization: PrivateToken <токен>`, включая пробел после `PrivateToken`. |
| `403` | Проверь, есть ли у владельца токена доступ к проекту и право на нужное действие в библиотеке тестов. |
| `404` | Для запроса с идентификатором сверь адрес системы, путь `/api/v2` и идентификатор ресурса. |
