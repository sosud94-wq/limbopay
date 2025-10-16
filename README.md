```markdown
# Документация Payment API

## 🔗 Базовый URL
```
https://warren.su/api/
```

## 🔐 Аутентификация
Все запросы требуют HTTP заголовки:

| Заголовок | Описание |
|-----------|----------|
| `X-Public-Key` | Публичный ключ партнера |
| `X-Secret-Key` | Секретный ключ партнера |

---

## 📊 Получение баланса

### Эндпоинт
**GET** `/balance`

### Описание
Получение текущего баланса партнера в USDT.

### Пример запроса (cURL)
```bash
curl -X GET "https://warren.su/api/balance" \
  -H "X-Public-Key: ваш_публичный_ключ" \
  -H "X-Secret-Key: ваш_секретный_ключ"
```

### Пример на PHP
```php
<?php
function getBalance($publicKey, $secretKey) {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => 'https://warren.su/api/balance',
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'X-Public-Key: ' . $publicKey,
            'X-Secret-Key: ' . $secretKey,
            'Content-Type: application/json'
        ]
    ]);
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($httpCode === 200) {
        $data = json_decode($response, true);
        if ($data['status'] === 'success') {
            echo "Основной баланс: " . $data['balance']['balance_usdt'] . " USDT\n";
            echo "Авансовый депозит: " . $data['balance']['deposit_usdt'] . " USDT\n";
        }
        return $data;
    }
    return ['status' => 'error', 'message' => 'HTTP ' . $httpCode];
}

// Использование
$balance = getBalance('ваш_публичный_ключ', 'ваш_секретный_ключ');
?>
```

### Успешный ответ
```json
{
    "status": "success",
    "balance": {
        "balance_usdt": "4750.832949",
        "deposit_usdt": "9999.881664"
    }
}
```

### Описание полей
| Поле | Тип | Описание |
|------|-----|----------|
| `balance_usdt` | `decimal` | Основной баланс в USDT (заработанные средства от успешных платежей) |
| `deposit_usdt` | `decimal` | Авансовый депозит в USDT (предоплаченные средства для мгновенных выплат) |

---

## 💳 Создание транзакции

### Эндпоинт
**POST** `/transactions`

### Описание
Создание новой платежной транзакции и автоматическая генерация QR-кода для оплаты через СБП.

### Параметры запроса
| Параметр | Тип | Обязательный | Описание |
|----------|-----|--------------|----------|
| `amount` | `decimal` | ✅ | Сумма платежа в RUB |
| `currency` | `string` | ✅ | Валюта (всегда `"RUB"`) |
| `internal_uuid` | `string` | ✅ | Уникальный ID заказа в вашей системе |
| `client_id` | `string` | ✅ | ID клиента в вашей системе |

### Пример запроса (cURL)
```bash
curl -X POST "https://warren.su/api/transactions" \
  -H "X-Public-Key: ваш_публичный_ключ" \
  -H "X-Secret-Key: ваш_секретный_ключ" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "1000.00",
    "currency": "RUB",
    "internal_uuid": "order-12345",
    "client_id": "client-001"
  }'
```

### Пример на PHP
```php
<?php
function createTransaction($publicKey, $secretKey, $orderData) {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => 'https://warren.su/api/transactions',
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($orderData),
        CURLOPT_HTTPHEADER => [
            'X-Public-Key: ' . $publicKey,
            'X-Secret-Key: ' . $secretKey,
            'Content-Type: application/json'
        ]
    ]);
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if ($result['status'] === 'success') {
        $transaction = $result['transaction'];
        // Сохраняем в вашу БД
        saveTransactionToDB($orderData['internal_uuid'], $transaction['transaction_uuid']);
        // Возвращаем ссылку для оплаты
        return [
            'success' => true,
            'payment_link' => $transaction['sbp_payment_link'],
            'transaction_uuid' => $transaction['transaction_uuid']
        ];
    }
    
    return ['success' => false, 'error' => $result['message'] ?? 'Unknown error'];
}

// Использование
$orderData = [
    'amount' => '1000.00',
    'currency' => 'RUB',
    'internal_uuid' => 'order-' . uniqid(),
    'client_id' => 'client-001'
];

$result = createTransaction('ваш_публичный_ключ', 'ваш_секретный_ключ', $orderData);
if ($result['success']) {
    echo "Ссылка для оплаты: " . $result['payment_link'];
}
?>
```

### Успешный ответ
```json
{
    "status": "success",
    "transaction": {
        "transaction_uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "amount": "1000.00",
        "partner_amount": "980.00",
        "partner_amount_usd": "11.200000",
        "usdt_rate": "87.5000",
        "partner_fee_percent": "2.00",
        "status": "details_issued",
        "internal_uuid": "order-12345",
        "operation_number": "OP123456789",
        "operation_id": "op_abc123",
        "qr_code_id": "qr_xyz789",
        "sbp_payment_link": "https://qrpay.link/pay/op_abc123",
        "webhook_setup": true
    }
}
```

### Важные моменты
- **`internal_uuid`** должен быть **уникальным** для каждой транзакции
- При успешном создании **автоматически генерируется QR-код**
- Статус `details_issued` означает готовность платежных данных
- **Webhook настраивается автоматически** для уведомлений об оплате
- Сохраняйте `transaction_uuid` для отслеживания платежа

---

## 🔍 Проверка статуса транзакции

### Эндпоинт
**GET** `/status?internal_uuid={ваш_internal_uuid}`

### Описание
Получение текущего статуса и детальной информации о транзакции.

### Пример запроса (cURL)
```bash
curl -X GET "https://warren.su/api/status?internal_uuid=order-12345" \
  -H "X-Public-Key: ваш_публичный_ключ" \
  -H "X-Secret-Key: ваш_секретный_ключ"
```

### Пример на PHP
```php
<?php
function getTransactionStatus($publicKey, $secretKey, $internalUUID) {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => 'https://warren.su/api/status?internal_uuid=' . urlencode($internalUUID),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'X-Public-Key: ' . $publicKey,
            'X-Secret-Key: ' . $secretKey,
            'Content-Type: application/json'
        ]
    ]);
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    $status = json_decode($response, true);
    
    if ($status['status'] === 'success') {
        $transaction = $status['transaction'];
        switch ($transaction['status']) {
            case 'paid':
                echo "✅ Платеж оплачен: " . $transaction['paid_at'] . "\n";
                completeOrder($internalUUID);
                break;
            case 'details_issued':
                echo "⏳ Ожидает оплаты\n";
                echo "Ссылка: " . $transaction['sbp_payment_link'] . "\n";
                break;
            case 'timeout':
                echo "❌ Время оплаты истекло\n";
                break;
        }
    }
    
    return $status;
}

// Использование
$status = getTransactionStatus('ваш_публичный_ключ', 'ваш_секретный_ключ', 'order-12345');
?>
```

### Успешный ответ
```json
{
    "status": "success",
    "transaction": {
        "transaction_uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "amount": "1000.00",
        "partner_amount": "980.00",
        "partner_amount_usd": "11.200000",
        "usdt_rate": "87.5000",
        "status": "paid",
        "internal_uuid": "order-12345",
        "client_id": "client-001",
        "sbp_payment_link": "https://qrpay.link/pay/op_abc123",
        "created_at": "2024-01-15 10:30:00",
        "paid_at": "2024-01-15 10:35:22"
    }
}
```

### Статусы транзакций
| Статус | Описание |
|--------|----------|
| `pending` | Транзакция создана, ожидает генерации QR-кода |
| `details_issued` | QR-код сгенерирован, ожидает оплаты |
| `paid` | ✅ Транзакция успешно оплачена |
| `timeout` | ❌ Время оплаты истекло |

---

## 🩺 Health Check

### Эндпоинт
**GET** `/health`

### Описание
Проверка работоспособности API и подключения к базе данных.

### Пример запроса (cURL)
```bash
curl -X GET "https://warren.su/api/health"
```

### Пример на PHP
```php
<?php
function checkHealth() {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => 'https://warren.su/api/health',
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 10,
        CURLOPT_HTTPHEADER => ['Content-Type: application/json']
    ]);
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($httpCode === 200) {
        $health = json_decode($response, true);
        if ($health['status'] === 'success') {
            echo "✅ API работает\n";
            echo "База данных: " . $health['database'] . "\n";
            return $health;
        }
    }
    return ['status' => 'error', 'message' => 'Service unavailable'];
}

// Использование для мониторинга
$health = checkHealth();
?>
```

### Успешный ответ
```json
{
    "status": "success",
    "message": "API is working",
    "database": "connected",
    "tables": {
        "partners": "exists",
        "transactions": "exists"
    },
    "timestamp": "2024-01-15T10:30:00+00:00",
    "environment": "production"
}
```

---

## ⚠️ Обработка ошибок

### Коды HTTP ошибок
| Код | Описание |
|-----|----------|
| `400` | Неверные параметры запроса |
| `401` | ❌ Ошибка аутентификации |
| `404` | Транзакция не найдена |
| `405` | Метод не поддерживается |
| `409` | Дублирующийся `internal_uuid` |
| `500` | Внутренняя ошибка сервера |

### Формат ответа об ошибке
```json
{
    "status": "error",
    "message": "Описание конкретной ошибки"
}
```

---

## 🔄 Типичный workflow оплаты

### Пошаговый процесс
1. **Создайте заказ** в вашей системе с уникальным `internal_uuid`
2. **Вызовите** `POST /transactions` для создания платежной транзакции
3. **Сохраните** `transaction_uuid` в вашей БД для отслеживания
4. **Покажите клиенту** `sbp_payment_link` для оплаты
5. **Мониторьте статус** через `GET /status` или webhook
6. **При статусе `paid`** засчитайте транзакцию и


```

### 🚨 Важные замечания
- **Уникальность**: `internal_uuid` должен быть уникальным для каждой транзакции
- **Сохранение**: Обязательно сохраняйте `transaction_uuid` для отслеживания
- **Webhook**: Используйте webhook для мгновенных уведомлений об оплате
- **Таймаут**: Учитывайте время ожидания оплаты (обычно 5-10 минут) время жини QR кода 15 минут
- **Проверка статуса**: Регулярно проверяйте статус через API или webhook



