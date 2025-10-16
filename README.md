# Документация Payment API

## Базовый URL
https://warren.su/api/

## Аутентификация
Все запросы требуют заголовки:
- X-Public-Key: Публичный ключ партнера
- X-Secret-Key: Секретный ключ партнера

---

## 1. Получение баланса
GET /balance

Получение текущего баланса партнера в USDT.

### Пример запроса (cURL)
curl -X GET "https://warren.su/api/balance" \
  -H "X-Public-Key: ваш_публичный_ключ" \
  -H "X-Secret-Key: ваш_секретный_ключ"

### JavaScript (fetch API)
/**
 * Получение баланса партнера
 * @param {string} publicKey - Публичный ключ
 * @param {string} secretKey - Секретный ключ
 * @returns {Promise<Object>} Промис с данными баланса
 */
async function getBalance(publicKey, secretKey) {
    try {
        const response = await fetch('https://warren.su/api/balance', {
            method: 'GET',
            headers: {
                'X-Public-Key': publicKey,
                'X-Secret-Key': secretKey,
                'Content-Type': 'application/json'
            }
        });

        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();

        if (data.status === 'success') {
            const balance = data.balance;
            console.log('Основной баланс:', balance.balance_usdt, 'USDT');
            console.log('Авансовый депозит:', balance.deposit_usdt, 'USDT');
            return data;
        } else {
            throw new Error(data.message || 'Неизвестная ошибка');
        }
    } catch (error) {
        console.error('Ошибка получения баланса:', error.message);
        throw error;
    }
}

// Использование
getBalance('ваш_публичный_ключ', 'ваш_секретный_ключ')
    .then(result => console.log('Баланс получен:', result))
    .catch(error => console.error('Ошибка:', error));

### Успешный ответ
{
    "status": "success",
    "balance": {
        "balance_usdt": "4750.832949",
        "deposit_usdt": "9999.881664"
    }
}

### Описание полей
Поле | Тип | Описание
-----|-----|----------
balance_usdt | decimal | Основной баланс в USDT (заработанные средства от успешных платежей)
deposit_usdt | decimal | Авансовый депозит в USDT (предоплаченные средства для мгновенных выплат)

---

## 2. Создание транзакции
POST /transactions

Создание новой платежной транзакции и генерация QR-кода.

### Параметры запроса
Параметр | Тип | Обязательный | Описание
---------|-----|--------------|----------
amount | decimal | Да | Сумма платежа в RUB
currency | string | Да | Всегда "RUB"
internal_uuid | string | Да | Уникальный ID заказа в вашей системе
client_id | string | Да | ID клиента в вашей системе

### Пример запроса (cURL)
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

### JavaScript (fetch API)
/**
 * Создание новой платежной транзакции
 * @param {string} publicKey - Публичный ключ
 * @param {string} secretKey - Секретный ключ
 * @param {Object} orderData - Данные заказа
 * @returns {Promise<Object>} Промис с данными транзакции
 */
async function createTransaction(publicKey, secretKey, orderData) {
    try {
        const transactionData = {
            ...orderData,
            internal_uuid: orderData.internal_uuid || `order-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
            currency: 'RUB'
        };

        const response = await fetch('https://warren.su/api/transactions', {
            method: 'POST',
            headers: {
                'X-Public-Key': publicKey,
                'X-Secret-Key': secretKey,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(transactionData)
        });

        if (!response.ok) {
            const errorData = await response.json();
            throw new Error(errorData.message || `HTTP error! status: ${response.status}`);
        }

        const result = await response.json();

        if (result.status === 'success') {
            const transaction = result.transaction;
            console.log('Транзакция создана:', transaction.transaction_uuid);
            
            return {
                success: true,
                paymentLink: transaction.sbp_payment_link,
                transactionId: transaction.transaction_uuid,
                status: transaction.status,
                partnerAmount: transaction.partner_amount_usd
            };
        } else {
            throw new Error(result.message || 'Ошибка создания транзакции');
        }
    } catch (error) {
        console.error('Ошибка создания транзакции:', error);
        return { success: false, error: error.message };
    }
}

// Использование
const orderData = {
    amount: '1000.00',
    client_id: 'client-001'
};

createTransaction('ваш_публичный_ключ', 'ваш_секретный_ключ', orderData)
    .then(result => {
        if (result.success) {
            console.log('Ссылка для оплаты:', result.paymentLink);
            window.open(result.paymentLink, '_blank');
        } else {
            alert('Ошибка: ' + result.error);
        }
    });

### Успешный ответ
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

### Важные моменты
- internal_uuid должен быть уникальным для каждой транзакции
- При успешном создании сразу генерируется QR-код
- Статус details_issued означает что платежные данные готовы
- Webhook настроен автоматически для уведомлений об оплате

---

## 3. Проверка статуса транзакции
GET /status?internal_uuid={ваш_internal_uuid}

Получение текущего статуса и деталей транзакции.

### Пример запроса (cURL)
curl -X GET "https://warren.su/api/status?internal_uuid=order-12345" \
  -H "X-Public-Key: ваш_публичный_ключ" \
  -H "X-Secret-Key: ваш_секретный_ключ"

### JavaScript (fetch API)
/**
 * Проверка статуса транзакции
 * @param {string} publicKey - Публичный ключ
 * @param {string} secretKey - Секретный ключ
 * @param {string} internalUUID - Internal UUID транзакции
 * @returns {Promise<Object>} Промис с данными статуса
 */
async function getTransactionStatus(publicKey, secretKey, internalUUID) {
    try {
        const response = await fetch(
            `https://warren.su/api/status?internal_uuid=${encodeURIComponent(internalUUID)}`,
            {
                method: 'GET',
                headers: {
                    'X-Public-Key': publicKey,
                    'X-Secret-Key': secretKey,
                    'Content-Type': 'application/json'
                }
            }
        );

        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }

        const status = await response.json();

        if (status.status === 'success') {
            const transaction = status.transaction;
            console.log('Статус платежа:', transaction.status);
            
            switch (transaction.status) {
                case 'paid':
                    console.log('Платеж успешно оплачен:', transaction.paid_at);
                    break;
                case 'details_issued':
                    console.log('Ожидает оплаты');
                    console.log('Ссылка:', transaction.sbp_payment_link);
                    break;
                case 'pending':
                    console.log('Генерируется QR-код');
                    break;
                case 'timeout':
                    console.log('Время оплаты истекло');
                    break;
            }
            return status;
        } else {
            throw new Error(status.message || 'Транзакция не найдена');
        }
    } catch (error) {
        console.error('Ошибка проверки статуса:', error);
        throw error;
    }
}

// Периодическая проверка
function checkTransactionPeriodically(publicKey, secretKey, internalUUID, interval = 5000) {
    const checkInterval = setInterval(async () => {
        try {
            const result = await getTransactionStatus(publicKey, secretKey, internalUUID);
            if (result.status === 'success' && result.transaction.status === 'paid') {
                clearInterval(checkInterval);
                console.log('Платеж завершен!');
            }
        } catch (error) {
            console.error('Ошибка проверки:', error);
            clearInterval(checkInterval);
        }
    }, interval);
    return () => clearInterval(checkInterval);
}

### Успешный ответ
{
    "status": "success",
    "transaction": {
        "transaction_uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "amount": "1000.00",
        "partner_amount": "980.00",
        "partner_amount_usd": "11.200000",
        "status": "paid",
        "internal_uuid": "order-12345",
        "client_id": "client-001",
        "sbp_payment_link": "https://qrpay.link/pay/op_abc123",
        "created_at": "2024-01-15 10:30:00",
        "paid_at": "2024-01-15 10:35:22"
    }
}

### Статусы транзакций
Статус | Описание
-------|----------
pending | Транзакция создана, ожидает генерации QR
details_issued | QR-код сгенерирован, ожидает оплаты
paid | Транзакция успешно оплачена
timeout | Время оплаты истекло

---

## 4. Health Check
GET /health

Проверка работоспособности API и подключения к базе данных.

### Пример запроса (cURL)
curl -X GET "https://warren.su/api/health"

### JavaScript (fetch API)
/**
 * Проверка работоспособности API
 */
async function checkHealth() {
    try {
        const response = await fetch('https://warren.su/api/health', {
            method: 'GET',
            headers: { 'Content-Type': 'application/json' }
        });

        if (!response.ok) {
            throw new Error(`HTTP ${response.status}: Service unavailable`);
        }

        const health = await response.json();

        if (health.status === 'success') {
            console.log('API работает нормально');
            console.log('База данных:', health.database);
            return health;
        } else {
            throw new Error(health.message || 'Health check failed');
        }
    } catch (error) {
        console.error('Проблемы с API:', error.message);
        throw error;
    }
}

### Успешный ответ
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

---

## Обработка ошибок

### Коды ошибок
Код | Описание
----|----------
400 | Неверные параметры запроса
401 | Ошибка аутентификации
404 | Транзакция не найдена
405 | Метод не поддерживается
409 | Дублирующийся internal_uuid
500 | Внутренняя ошибка сервера

### Формат ошибки
{
    "status": "error",
    "message": "Описание ошибки"
}

---

## Типичный workflow оплаты

class PaymentService {
    constructor(publicKey, secretKey) {
        this.publicKey = publicKey;
        this.secretKey = secretKey;
    }

    async processPayment(orderData) {
        try {
            const result = await createTransaction(this.publicKey, this.secretKey, orderData);
            
            if (!result.success) {
                throw new Error(result.error);
            }

            localStorage.setItem(`tx_${orderData.internal_uuid}`, JSON.stringify(result));
            checkTransactionPeriodically(this.publicKey, this.secretKey, orderData.internal_uuid);
            window.open(result.paymentLink, '_blank');
            
            return result;
        } catch (error) {
            console.error('Ошибка платежа:', error);
            alert('Ошибка: ' + error.message);
            return { success: false, error: error.message };
        }
    }
}

// Использование
const paymentService = new PaymentService('ваш_публичный_ключ', 'ваш_секретный_ключ');
const order = {
    amount: '1500.00',
    client_id: 'user-123',
    internal_uuid: `order-${Date.now()}`
};
paymentService.processPayment(order);
