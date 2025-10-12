# Документация API

## Описание
API предоставляет функционал для управления платежами и получения информации о балансе. Все запросы к API выполняются через защищённое соединение по протоколу HTTPS. Для аутентификации используется механизм HMAC-SHA256 с использованием публичного и секретного ключей. Все ответы возвращаются в формате JSON.

## Базовый URL
```
https://warren.su/api/v1
```

## Аутентификация
Для доступа к API требуется аутентификация по схеме HMAC-SHA256. Необходимо передать публичный ключ, временную метку и подпись в заголовках запроса. Формат заголовков следующий:

```
X-Public-Key: public_key_партнера
X-Timestamp: 1642252800 (Unix timestamp)
X-Signature: HMAC-SHA256 подпись
```

### Процесс формирования подписи
1. Соберите строку для подписи: `public_key + timestamp + body`, где:
   - `public_key` — публичный ключ партнёра.
   - `timestamp` — Unix-временная метка (в секундах).
   - `body` — тело запроса в виде строки (для GET-запросов — пустая строка).
2. Подпишите полученную строку с помощью секретного ключа (Secret Key) алгоритмом HMAC-SHA256:  
   ```php
   hash_hmac('sha256', $string, $secret_key)
   ```
3. Передайте результат в заголовке `X-Signature`.

### Пример формирования подписи
```php
$public_key = "your_public_key";
$secret_key = "your_secret_key";
$timestamp = "1642252800";
$body = '{"amount":"100.00","currency":"USDT","internal_payment_uuid":"partner-uuid-12345","internal_client_id":"client-67890"}';
$string_to_sign = $public_key . $timestamp . $body;
$signature = hash_hmac('sha256', $string_to_sign, $secret_key);
```

## Эндпоинты

### 1.1 Создание платежа
**Метод:** POST  
**Путь:** `/payment/create`  
**Описание:** Создаёт новый платёж в системе. Требуется указать сумму, валюту, уникальный идентификатор платежа и идентификатор клиента.

#### Тело запроса
```json
{
  "amount": "100.00",
  "currency": "USDT",
  "internal_payment_uuid": "partner-uuid-12345",
  "internal_client_id": "client-67890"
}
```

- **amount**: Сумма платежа (строка, формат десятичного числа).
- **currency**: Валюта платежа (например, "USDT").
- **internal_payment_uuid**: Уникальный идентификатор платежа на стороне партнёра.
- **internal_client_id**: Уникальный идентификатор клиента на стороне партнёра.

#### Пример запроса (PHP)
```php
<?php
$public_key = "your_public_key";
$secret_key = "your_secret_key";
$timestamp = time();
$data = [
    "amount" => "100.00",
    "currency" => "USDT",
    "internal_payment_uuid" => "partner-uuid-12345",
    "internal_client_id" => "client-67890"
];
$body = json_encode($data);
$string_to_sign = $public_key . $timestamp . $body;
$signature = hash_hmac('sha256', $string_to_sign, $secret_key);

$ch = curl_init("https://warren.su/api/v1/payment/create");
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $body);
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    "X-Public-Key: $public_key",
    "X-Timestamp: $timestamp",
    "X-Signature: $signature",
    "Content-Type: application/json"
]);

$response = curl_exec($ch);
curl_close($ch);
echo $response;
?>
```

#### Пример запроса (JavaScript Fetch)
```javascript
const publicKey = "your_public_key";
const secretKey = "your_secret_key";
const timestamp = Math.floor(Date.now() / 1000);
const data = {
    amount: "100.00",
    currency: "USDT",
    internal_payment_uuid: "partner-uuid-12345",
    internal_client_id: "client-67890"
};
const body = JSON.stringify(data);

// Функция для создания HMAC-SHA256 подписи
async function createHmacSignature(key, message) {
    const msgBuffer = new TextEncoder().encode(message);
    const keyBuffer = new TextEncoder().encode(key);
    const cryptoKey = await crypto.subtle.importKey(
        "raw", keyBuffer, { name: "HMAC", hash: "SHA-256" },
        false, ["sign"]
    );
    const signature = await crypto.subtle.sign("HMAC", cryptoKey, msgBuffer);
    return Array.from(new Uint8Array(signature))
        .map(b => b.toString(16).padStart(2, '0'))
        .join('');
}

const stringToSign = publicKey + timestamp + body;
createHmacSignature(secretKey, stringToSign).then(signature => {
    fetch("https://warren.su/api/v1/payment/create", {
        method: "POST",
        headers: {
            "X-Public-Key": publicKey,
            "X-Timestamp": timestamp,
            "X-Signature": signature,
            "Content-Type": "application/json"
        },
        body: body
    })
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.error("Error:", error));
});
```

#### Успешный ответ (200)
```json
{
  "status": "success",
  "data": {
    "transaction_uuid": "0dad0aaf-71bc-47ef-b608-b319d0c2eadc",
    "operation_number": "7437-2298-1389-3977",
    "qr_code_id": "70910725667409975299455889568227",
    "sbp_url": "https://qr.nspk.ru/xxx",
    "amount": "100.00",
    "currency": "USDT",
    "exchange_rate": "89.45",
    "partner_amount": "98.50",
    "commission_percent": "1.5",
    "status": "pending",
    "created_at": "2024-01-15T12:00:00Z"
  }
}
```

- **transaction_uuid**: Уникальный идентификатор транзакции в системе.
- **operation_number**: Номер операции.
- **qr_code_id**: Идентификатор QR-кода для оплаты.
- **sbp_url**: Ссылка на QR-код для оплаты через СБП.
- **amount**: Сумма платежа.
- **currency**: Валюта платежа.
- **exchange_rate**: Курс обмена.
- **partner_amount**: Сумма, зачисленная партнёру после вычета комиссии.
- **commission_percent**: Процент комиссии.
- **status**: Статус платежа (например, "pending").
- **created_at**: Дата и время создания платежа (в формате ISO 8601).

### 1.2 Получение статуса транзакции
**Метод:** GET  
**Путь:** `/payment/status/{internal_payment_uuid}`  
**Описание:** Возвращает информацию о статусе платежа по его уникальному идентификатору.

#### Пример запроса (PHP)
```php
<?php
$public_key = "your_public_key";
$secret_key = "your_secret_key";
$timestamp = time();
$internal_payment_uuid = "partner-uuid-12345";
$body = "";
$string_to_sign = $public_key . $timestamp . $body;
$signature = hash_hmac('sha256', $string_to_sign, $secret_key);

$ch = curl_init("https://warren.su/api/v1/payment/status/$internal_payment_uuid");
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    "X-Public-Key: $public_key",
    "X-Timestamp: $timestamp",
    "X-Signature: $signature"
]);

$response = curl_exec($ch);
curl_close($ch);
echo $response;
?>
```

#### Пример запроса (JavaScript Fetch)
```javascript
const publicKey = "your_public_key";
const secretKey = "your_secret_key";
const timestamp = Math.floor(Date.now() / 1000);
const internalPaymentUuid = "partner-uuid-12345";
const body = "";

// Функция для создания HMAC-SHA256 подписи
async function createHmacSignature(key, message) {
    const msgBuffer = new TextEncoder().encode(message);
    const keyBuffer = new TextEncoder().encode(key);
    const cryptoKey = await crypto.subtle.importKey(
        "raw", keyBuffer, { name: "HMAC", hash: "SHA-256" },
        false, ["sign"]
    );
    const signature = await crypto.subtle.sign("HMAC", cryptoKey, msgBuffer);
    return Array.from(new Uint8Array(signature))
        .map(b => b.toString(16).padStart(2, '0'))
        .join('');
}

const stringToSign = publicKey + timestamp + body;
createHmacSignature(secretKey, stringToSign).then(signature => {
    fetch(`https://warren.su/api/v1/payment/status/${internalPaymentUuid}`, {
        method: "GET",
        headers: {
            "X-Public-Key": publicKey,
            "X-Timestamp": timestamp,
            "X-Signature": signature
        }
    })
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.error("Error:", error));
});
```

#### Успешный ответ (200)
```json
{
  "status": "success",
  "data": {
    "transaction_uuid": "0dad0aaf-71bc-47ef-b608-b319d0c2eadc",
    "internal_payment_uuid": "partner-uuid-12345",
    "internal_client_id": "client-67890",
    "amount": "100.00",
    "currency": "USDT",
    "exchange_rate": "89.45",
    "partner_amount": "98.50",
    "commission_percent": "1.5",
    "status": "paid",
    "operation_number": "7437-2298-1389-3977",
    "created_at": "2024-01-15T12:00:00Z",
    "details_issued_at": "2024-01-15T12:00:01Z",
    "paid_at": "2024-01-15T12:05:30Z"
  }
}
```

- **transaction_uuid**: Уникальный идентификатор транзакции.
- **internal_payment_uuid**: Идентификатор платежа на стороне партнёра.
- **internal_client_id**: Идентификатор клиента на стороне партнёра.
- **amount**: Сумма платежа.
- **currency**: Валюта платежа.
- **exchange_rate**: Курс обмена.
- **partner_amount**: Сумма, зачисленная партнёру.
- **commission_percent**: Процент комиссии.
- **status**: Статус платежа (например, "paid").
- **operation_number**: Номер операции.
- **created_at**: Дата создания платежа.
- **details_issued_at**: Дата выдачи деталей платежа.
- **paid_at**: Дата оплаты (если применимо).

### 1.3 Получение баланса
**Метод:** GET  
**Путь:** `/balance`  
**Описание:** Возвращает текущий баланс партнёра, включая доступный и замороженный баланс.

#### Пример запроса (PHP)
```php
<?php
$public_key = "your_public_key";
$secret_key = "your_secret_key";
$timestamp = time();
$body = "";
$string_to_sign = $public_key . $timestamp . $body;
$signature = hash_hmac('sha256', $string_to_sign, $secret_key);

$ch = curl_init("https://warren.su/api/v1/balance");
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    "X-Public-Key: $public_key",
    "X-Timestamp: $timestamp",
    "X-Signature: $signature"
]);

$response = curl_exec($ch);
curl_close($ch);
echo $response;
?>
```

#### Пример запроса (JavaScript Fetch)
```javascript
const publicKey = "your_public_key";
const secretKey = "your_secret_key";
const timestamp = Math.floor(Date.now() / 1000);
const body = "";

// Функция для создания HMAC-SHA256 подписи
async function createHmacSignature(key, message) {
    const msgBuffer = new TextEncoder().encode(message);
    const keyBuffer = new TextEncoder().encode(key);
    const cryptoKey = await crypto.subtle.importKey(
        "raw", keyBuffer, { name: "HMAC", hash: "SHA-256" },
        false, ["sign"]
    );
    const signature = await crypto.subtle.sign("HMAC", cryptoKey, msgBuffer);
    return Array.from(new Uint8Array(signature))
        .map(b => b.toString(16).padStart(2, '0'))
        .join('');
}

const stringToSign = publicKey + timestamp + body;
createHmacSignature(secretKey, stringToSign).then(signature => {
    fetch("https://warren.su/api/v1/balance", {
        method: "GET",
        headers: {
            "X-Public-Key": publicKey,
            "X-Timestamp": timestamp,
            "X-Signature": signature
        }
    })
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.error("Error:", error));
});
```

#### Успешный ответ (200)
```json
{
  "status": "success",
  "data": {
    "balance": "15000.75",
    "frozen_balance": "2500.00",
    "currency": "USDT"
  }
}
```

- **balance**: Доступный баланс.
- **frozen_balance**: Замороженный баланс.
- **currency**: Валюта баланса.

## Коды ошибок
- **400** - Неверные данные в запросе.
- **401** - Неавторизован (ошибка аутентификации).
- **404** - Транзакция не найдена.
- **500** - Внутренняя ошибка сервера.
