# Документация Payment API

Базовый URL: https://warren.su/api/

Аутентификация:
Все запросы требуют заголовки:
- X-Public-Key: Публичный ключ партнера
- X-Secret-Key: Секретный ключ партнера

=== 📊 ПОЛУЧЕНИЕ БАЛАНСА ===

Эндпоинт: GET /balance
Описание: Получение текущего баланса партнера в USDT

Запрос:
curl -X GET "https://warren.su/api/balance" \
  -H "X-Public-Key: ваш_публичный_ключ" \
  -H "X-Secret-Key: ваш_секретный_ключ"

Пример на PHP:
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
    return json_decode($response, true);
}

Успешный ответ:
{
    "status": "success",
    "balance": {
        "balance_usdt": "4750.832949",
        "deposit_usdt": "9999.881664"
    }
}

Описание полей:
- balance_usdt: Основной баланс в USDT (заработанные средства от платежей)
- deposit_usdt: Авансовый депозит в USDT (предоплаченные средства)

=== 💳 СОЗДАНИЕ ТРАНЗАКЦИИ ===

Эндпоинт: POST /transactions
Описание: Создание новой платежной транзакции и генерация QR-кода

Параметры запроса:
- amount (decimal, обязательный): Сумма платежа в RUB
- currency (string, обязательный): Всегда "RUB"
- internal_uuid (string, обязательный): Уникальный ID заказа в вашей системе
- client_id (string, обязательный): ID клиента в вашей системе

Запрос:
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

Пример на PHP:
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
    return json_decode($response, true);
}

Успешный ответ:
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

=== 🔍 ПРОВЕРКА СТАТУСА ТРАНЗАКЦИИ ===

Эндпоинт: GET /status?internal_uuid={ваш_internal_uuid}
Описание: Получение текущего статуса и деталей транзакции

Запрос:
curl -X GET "https://warren.su/api/status?internal_uuid=order-12345" \
  -H "X-Public-Key: ваш_публичный_ключ" \
  -H "X-Secret-Key: ваш_секретный_ключ"

Пример на PHP:
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
    return json_decode($response, true);
}

Успешный ответ:
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

Статусы транзакций:
- pending: Транзакция создана, ожидает генерации QR
- details_issued: QR-код сгенерирован, ожидает оплаты
- paid: Транзакция успешно оплачена
- timeout: Время оплаты истекло

=== 🩺 HEALTH CHECK ===

Эндпоинт: GET /health
Описание: Проверка работоспособности API и подключения к базе данных

Запрос:
curl -X GET "https://warren.su/api/health"

Пример на PHP:
function checkHealth() {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => 'https://warren.su/api/health',
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 10
    ]);

    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    if ($httpCode === 200) {
        return json_decode($response, true);
    }
    return ['status' => 'error', 'message' => 'Service unavailable'];
}

Успешный ответ:
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

=== ⚠️ ОБРАБОТКА ОШИБОК ===

Коды ошибок:
- 400: Неверные параметры запроса
- 401: Ошибка аутентификации
- 404: Транзакция не найдена
- 405: Метод не поддерживается
- 409: Дублирующийся internal_uuid
- 500: Внутренняя ошибка сервера

Формат ошибки:
{
    "status": "error",
    "message": "Описание ошибки"
}

=== 🔄 TYPICAL WORKFLOW ===

1. Создаем заказ в вашей системе
2. Вызываем API для создания транзакции
3. Сохраняем transaction_uuid в вашей БД
4. Показываем клиенту sbp_payment_link
5. Периодически проверяем статус через /status
6. При статусе 'paid' - засчитываем транзакцию

Важные замечания:
- internal_uuid должен быть уникальным для каждой транзакции
- Сохраняйте transaction_uuid для отслеживания
- Используйте webhook для мгновенных уведомлений
