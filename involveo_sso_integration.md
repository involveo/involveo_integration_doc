## 1\. Подготовка токенов на стороне клиента

При загрузке страницы клиент должен сгенерировать пару JWT-токенов:

* access-токен — используется для авторизации в виджете
* refresh-токен — для безопасного обновления `access-токена` при его истечении




**Установка access-токена**

Клиент вставляет токен в атрибут "data-sso-token", в тег подключения виджета:

```html
<!-- Involveo widget -->
<script src="https://involveo.ru/widget.js"
  data-token="<YOUR_PROJECT_TOKEN>"
  data-sso-token="<YOUR_SSO_ACCESS_TOKEN>"
  data-sso-refresh-token-url="<YOUR_REFRESH_TOKEN_API>"
  data-sso-login-url="<YOUR_LOGIN_URL>"
  id="involveo_widget" type="module" defer>
</script>
<!-- /Involveo widget -->
```

> Расшифровка атрибутов:
>
> * data-token: токен проекта
> * data-sso-token: jwt-токен для авторизации
> * data-sso-refresh-token-url: API-эндпоинт для обмена токенов
> * data-sso-login-url: URL страницы с авторизацией на стороне клиента



**Хранение refresh-токена**

* HttpOnly cookie
* Local Storage, требует postMessage




## 2\. Генерация JWT-токенов

Для корректной работы виджета, токен должен содержать следующие данные:

| Параметр | Обязательный | Описание |
| --- | --- | --- |
| id | Да | Уникальный идентификатор пользователя в системе клиента. |
| iat | Нет | Время выдачи токена (Unix timestamp). |
| exp | Да | Время истечения токена в формате Unix timestamp, в секундах. |
| jti | Да | Уникальный ID токена (UUID). Обязателен для logout. |
| Другие поля (custom) | Нет | По желанию (например, `email`, `name`, `company`). Эти поля можно сопоставить с полями в ЛК виджета. |



**Пример payload:**

```json
{
  "id": 12345,
  "email": "user@example.com",
  "name": "Иван Петров",
  "company": "ООО ТекстильЦемент",
  "exp": 1735689600,
  "jti": "a1b2c3d4-e5f6-7890-g1h2-i3j4k5l6m7n8"
}
```



**Custom поля**

Виджет автоматически распознаёт следующие поля в payload JWT-токена:

* `name` — отображается как имя пользователя
* `company` — отображается как название компании

Если в токене присутствуют другие пользовательские поля (например, `email`, `phone` и т.д.), они сохраняются в системе и могут использоваться в дальнейшем — например, при формировании данных для отправки в Вебхук.

> Исключение: сервисные поля из списка: ['iat', 'exp', 'iss', 'sub', 'aud', 'jti']



### **Требования к заголовку (JWT Header)**

* `alg`: `HS256` — должен использоваться алгоритм HMAC с SHA-256
* `typ`: `JWT` — указывает тип токена




**Пример** заголовка\*\*:\*\*

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```



**Подпись токена:**

При генерации, токен должен быть подписан с использованием секретного ключа \<**YOUR_SSO_SECRET_KEY**\>, который можно получить в ЛК Involveo.



**Пример генерации токена - нативный (без лишних зависимостей)**

```javascript
<script>
async function generateTokenManually(user) {
  const header = { alg: 'HS256', typ: 'JWT' };
  const payload = {
    id: user.id,
    email: user.email,
    name: user.name,
    company: user.company,
    exp: Math.floor(Date.now() / 1000) + 1800, // 30 минут
    jti: crypto.randomUUID() 
  };

  const encoder = new TextEncoder();

  // Упорядочивание ключей и кодирование в Base64URL
  function base64UrlEncode(data) {
    const json = JSON.stringify(data, Object.keys(data).sort());
    const binString = Array.from(new Uint8Array(encoder.encode(json)))
      .map(b => String.fromCharCode(b)).join('');
    return btoa(binString)
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/, '');
  }

  const encodedHeader = base64UrlEncode(header);
  const encodedPayload = base64UrlEncode(payload);
  const signatureInput = `${encodedHeader}.${encodedPayload}`;

  // Ключ должен совпадать с тем, что получен в ЛК Involveo
  const keyData = encoder.encode('YOUR_SSO_SECRET_KEY');
  const key = await crypto.subtle.importKey(
    'raw',
    keyData,
    { name: 'HMAC', hash: { name: 'SHA-256' } },
    false,
    ['sign']
  );

  // Генерация подписи
  const signatureBuffer = await crypto.subtle.sign('HMAC', key, encoder.encode(signatureInput));
  const signatureArray = Array.from(new Uint8Array(signatureBuffer))
    .map(b => String.fromCharCode(b)).join('');
  const encodedSignature = btoa(signatureArray)
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '');

  return `${encodedHeader}.${encodedPayload}.${encodedSignature}`;
}

// Пример вызова
generateTokenManually({
  id: 12345,
  email: 'user@example.com',
  name: 'Иван Петров',
  company: 'ООО ТекстильЦемент'
}).then(token => {
  console.log('Generated token:', token); // Вставьте этот токен в data-sso-token
});
</script>
```



**Пример генерации токена через библиотеку `jsonwebtoken` (Node.js)**

> npm install jsonwebtoken uuid

```
const jwt = require('jsonwebtoken');
const { v4: uuidv4 } = require('uuid');

function generateToken(user) {
  const payload = {
    id: user.id,
    email: user.email,
    name: user.name,
    company: user.company,
    exp: Math.floor(Date.now() / 1000) + 1800, // 30 минут
    jti: uuidv4()
  };

  // Ключ должен совпадать с тем, что получен в ЛК Involveo
  return jwt.sign(payload, 'YOUR_SSO_SECRET_KEY', {
    algorithm: 'HS256',
    header: { alg: 'HS256', typ: 'JWT' }
  });
}

// Пример вызова
const token = generateToken({
  id: 12345,
  email: 'user@example.com',
  name: 'Иван Петров',
  company: 'ООО ТекстильЦемент'
}); // Вставьте этот токен в data-sso-token
console.log(token);
```



## 3\. Обновление JWT-токенов

Когда access-токен становится недействительным (истёк, повреждён или отсутствует), виджет сделает попытку обновления токенов.

Для этого на стороне клиента:

* должен быть реализован API-эндпоинт обновления токенов (например: `POST /api/auth/refresh_involveo_sso_token)`
* URL эндпоинта должен быть вставлен в атрибут "data-sso-refresh-token-url", в тег подключения виджета

  > ⚠️ URL должен быть указан вместе со схемой и хостом




**API для обновления токенов**

Метод: POST

Заголовки:

```text
Content-Type: application/json
```

Тело запроса: (В случае, если refresh-токен хранится в LocalStorage)

```json
{
  "refresh": "<YOUR_REFRESH_TOKEN>"
}
```

> ⚠️ Сервер должен проверить `refresh-токен` (на подпись, срок действия, статус) и при успехе выдать новую пару токенов



Ожидаемый ответ при успехе (200 ОК):

```json
{
  "access": "<NEW_ACCESS_TOKEN>",
  "refresh": "<NEW_REFRESH_TOKEN>" (опционально)
}
```

Ошибки:

* `401 Unauthorized` — если refresh-токен недействителен или истёк
* `400 Bad Request` — если refresh-токен не передан




**Способы передачи refresh-токена в запрос**

В зависимости от того, где хранится refresh-токен, механизм его передачи в API отличается.



1. Через HttpOnly cookie

   Если токен хранится в HttpOnly **cookie**, он **автоматически отправляется** с каждым запросом к вашему API.

   * Виджет делает запрос на `POST /api/auth/refresh_involveo_sso_token` с `credentials: include`
   * Сервер извлекает `refresh-токен` из cookie
   * Проверяет его и возвращает новые токены

2. Через postMessage

   Если токен хранится в клиентском хранилище (**LocalStorage**), виджет **не может напрямую его прочитать** из-за политики кросс-доменной безопасности. В этом случае используется механизм `window.postMessage`.

   > ⚠️ Этот скрипт **обязательно** должен быть размещён **сразу после тега виджета**
   >
   > ⚠️ Внимание:
   >
   > * \<YOUR_REFRESH_TOKEN_KEY\> нужно заменить на ваш ключ хранения refresh-токена в хранилище
   > * \<YOUR_ORIGIN\> нужно заменить на ваш хост, откуда будут уходить запросы на обновления токена

   ```html
   <!-- Скрипт для передачи refresh-токена -->
   <script>
   (function registerInvolveoRefreshTokenHandler() {
     // Разрешённый origin (Ваш домен, на котором работает виджет)
     const ALLOWED_ORIGIN = '<YOUR_ORIGIN>';
   
     // Ключ refresh токена в вашем хранилище (например, localStorage)
     const REFRESH_TOKEN_KEY = '<YOUR_REFRESH_TOKEN_KEY>'; // ← Замените на свой
   
     // Прослушивание запросов от виджета
     window.addEventListener('message', (event) => {
       if (event.origin !== ALLOWED_ORIGIN) {
         console.warn('[Involveo] Ошибка origin:', event.origin);
         return;
       }
       if (event.data?.type !== 'request-refresh-token-involveo') return;
   
       // Получение refresh-токена
       try {
         const refreshToken = localStorage.getItem(REFRESH_TOKEN_KEY);
         // Если IndexedDB — вызовите асинхронную функцию (см. примечание ниже)
   
         if (!refreshToken) return;
         
         // Отправка токена обратно виджету
         (event.source as Window)?.postMessage(
           {
             type: 'refresh-token-involveo',
             refreshToken,
           },
           event.origin
         );
       } catch (e) {
         console.warn('[Involveo] Не удалось получить refresh-токен из хранилища', e);
       }
     });
   })()
   
   </script>
   ```

   > ⚠️ Если вы используете `IndexedDB` или асинхронное хранилище, оберните получение токена в `Promise` и отправьте ответ после `await`. Виджет ожидает ответ в течение 3 секунд.




**Настройка CORS**

Убедитесь, что на вашем API-эндпоинте настроены заголовки:

```text
Access-Control-Allow-Origin: https://involveo.ru 
Access-Control-Allow-Credentials: true
```

> ⚠️ `Access-Control-Allow-Credentials: true` требуется, если refresh-токен передаётся через HttpOnly cookie



## 4\. Ошибка обновления токенов

В случае неуспешного сценария обновления токенов, виджет прекращает попытки авторизации и отображает пользователю один из двух вариантов сообщений (выбор настраивается в ЛК Involveo):

* Предложение перезагрузить страницу (предполагается, что при этом будет заново сгенерирован действующий токен), выбран по-умолчанию
* Предложение пройти на страницу авторизации на клиентском сайте




**URL авторизации на клиентском сайте**

Чтобы отправить пользователя на страницу авторизации, в виджете должен быть установлен атрибут "**data-sso-login-url**".

> ⚠️ URL должен быть указан вместе со схемой и хостом

При редиректе пользователя по этому адресу, виджет добавляет url-параметр "**return_url**", таким образом можно отследить источник перехода пользователя и после успешной авторизации вернуть его обратно на исходную страницу.



## 5\. Механизм отзыва токенов

При завершении пользовательской сессии на стороне клиента рекомендуется отозвать ранее выданный `access`-токен, чтобы предотвратить его дальнейшее использование.



Для этого необходимо вызвать событие `involveo.`ssoLogout:

```javascript
window.involveo.ssoLogout()
```



Виджет автоматически отправит текущий `access`-токен на сервер и добавит его в чёрный список. После этого токен станет недействительным для всех последующих запросов.
