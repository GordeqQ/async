# План асинхронного выполнения задач

Работа делится между двумя исполнителями так, чтобы они могли делать изменения параллельно:

- исполнитель 1 отвечает за функцию `run` в `static/focus.js`;
- исполнитель 2 отвечает за объект `API` и функцию `sendRequest` в `static/focus.js`;
- серверные файлы `server.js` и `data/*` менять не нужно.

Перед началом оба исполнителя создают отдельные ветки:

```bash
git checkout -b executor-1-run
```

```bash
git checkout -b executor-2-request
```

После выполнения своей части каждый делает коммит:

```bash
git add static/focus.js
git commit -m "Solve async task part"
```

Если при слиянии будет конфликт в `static/focus.js`, нужно оставить итоговый вариант из задачи 4: `run` от исполнителя 1 и `API`/`sendRequest` от исполнителя 2.

## Задача 1 - решение - исполнитель 1

Заменить текущую функцию `run` в `static/focus.js` на async/await-вариант:

```js
async function run() {
    const orgOgrns = await sendRequest(API.organizationList);
    if (!orgOgrns) {
        return;
    }

    const ogrns = orgOgrns.join(",");

    const requisites = await sendRequest(`${API.orgReqs}?ogrn=${ogrns}`);
    if (!requisites) {
        return;
    }

    const orgsMap = reqsToMap(requisites);

    const analytics = await sendRequest(`${API.analytics}?ogrn=${ogrns}`);
    if (!analytics) {
        return;
    }

    addInOrgsMap(orgsMap, analytics, "analytics");

    const buh = await sendRequest(`${API.buhForms}?ogrn=${ogrns}`);
    if (!buh) {
        return;
    }

    addInOrgsMap(orgsMap, buh, "buhForms");
    render(orgsMap, orgOgrns);
}
```

Для этой задачи `sendRequest` уже должен возвращать промис. Если исполнитель 2 еще не замержен, временно заменить `sendRequest` на этот вариант:

```js
function sendRequest(url) {
    return new Promise((resolve) => {
        const xhr = new XMLHttpRequest();
        xhr.open("GET", url, true);

        xhr.onreadystatechange = function () {
            if (xhr.readyState === XMLHttpRequest.DONE) {
                if (xhr.status === 200) {
                    resolve(JSON.parse(xhr.response));
                } else {
                    resolve(null);
                }
            }
        };

        xhr.send();
    });
}
```

## Задача 2 - решение - исполнитель 2

Заменить функцию `sendRequest` в `static/focus.js` на вариант с `fetch`:

```js
async function sendRequest(url) {
    const response = await fetch(url);
    return await response.json();
}
```

После этой замены `sendRequest` возвращает промис автоматически, потому что функция объявлена как `async`.

## Задача 3 - решение - исполнитель 2

Для проверки ошибки временно изменить один адрес в объекте `API`.

Заменить:

```js
analytics: "/api3/analytics",
```

на:

```js
analytics: "/api3/analitics",
```

Сервер вернет ответ со статусом `404`, у объекта `response` поле `ok` будет равно `false`.

После проверки заменить `sendRequest` на итоговый вариант с обработкой ошибок:

```js
async function sendRequest(url) {
    try {
        const response = await fetch(url);

        if (!response.ok) {
            alert(`Ошибка запроса ${url}: ${response.status} ${response.statusText}`);
            return null;
        }

        return await response.json();
    } catch (error) {
        alert(`Ошибка запроса ${url}: ${error.message}`);
        return null;
    }
}
```

Важно: после проверки ошибки вернуть адрес обратно, иначе финальная таблица не отрисуется.

```js
analytics: "/api3/analytics",
```

## Задача 4 - решение - исполнитель 1

Заменить функцию `run` в `static/focus.js` на финальный вариант с `Promise.all`:

```js
async function run() {
    const orgOgrns = await sendRequest(API.organizationList);
    if (!orgOgrns) {
        return;
    }

    const ogrns = orgOgrns.join(",");
    const [requisites, analytics, buh] = await Promise.all([
        sendRequest(`${API.orgReqs}?ogrn=${ogrns}`),
        sendRequest(`${API.analytics}?ogrn=${ogrns}`),
        sendRequest(`${API.buhForms}?ogrn=${ogrns}`),
    ]);

    if (!requisites || !analytics || !buh) {
        return;
    }

    const orgsMap = reqsToMap(requisites);
    addInOrgsMap(orgsMap, analytics, "analytics");
    addInOrgsMap(orgsMap, buh, "buhForms");
    render(orgsMap, orgOgrns);
}
```

Финальный `static/focus.js` должен содержать:

```js
const API = {
    organizationList: "/orgsList",
    analytics: "/api3/analytics",
    orgReqs: "/api3/reqBase",
    buhForms: "/api3/buh",
};

async function run() {
    const orgOgrns = await sendRequest(API.organizationList);
    if (!orgOgrns) {
        return;
    }

    const ogrns = orgOgrns.join(",");
    const [requisites, analytics, buh] = await Promise.all([
        sendRequest(`${API.orgReqs}?ogrn=${ogrns}`),
        sendRequest(`${API.analytics}?ogrn=${ogrns}`),
        sendRequest(`${API.buhForms}?ogrn=${ogrns}`),
    ]);

    if (!requisites || !analytics || !buh) {
        return;
    }

    const orgsMap = reqsToMap(requisites);
    addInOrgsMap(orgsMap, analytics, "analytics");
    addInOrgsMap(orgsMap, buh, "buhForms");
    render(orgsMap, orgOgrns);
}

run();

async function sendRequest(url) {
    try {
        const response = await fetch(url);

        if (!response.ok) {
            alert(`Ошибка запроса ${url}: ${response.status} ${response.statusText}`);
            return null;
        }

        return await response.json();
    } catch (error) {
        alert(`Ошибка запроса ${url}: ${error.message}`);
        return null;
    }
}
```

Остальные функции ниже `sendRequest` остаются без изменений.

## Порядок слияния

1. Сначала мержится ветка исполнителя 2, потому что она дает финальную `sendRequest`.
2. Потом мержится ветка исполнителя 1 с финальной `run`.
3. Если Git покажет конфликт в верхней части `static/focus.js`, оставить блок из раздела "Финальный `static/focus.js`".
4. После слияния запустить проверку:

```bash
npm run server
```

Открыть `http://localhost:3000`. В нормальном финальном варианте таблица должна отрисоваться без alert. Для проверки задачи 3 временно вернуть `analytics: "/api3/analitics"` и убедиться, что появляется alert с `404 Not Found`, а код не падает.
