@startuml
skinparam maxMessageSize 150
skinparam sequenceMessageAlign center
autonumber

title Оформление заказа

actor "Покупатель" as buyer
participant "Веб-интерфейс\n(React SPA)" as web
participant "API Gateway\n(Nginx)" as gw
participant "Cart Service\n(Go)" as cart
database "Redis\n(Кеш корзины)" as redis
participant "Order Service\n(Go)" as order
database "Orders DB\n(PostgreSQL)" as orderdb
participant "Payment Service\n(Go)" as pay
database "Payment DB\n(PostgreSQL)" as paydb
participant "Платёжный шлюз\n(ЮKassa)" as psp
queue "RabbitMQ" as mq
participant "Catalog Service\n(Go)" as catalog
database "Catalog DB\n(PostgreSQL)" as catalogdb
participant "Media Service\n(Go)" as media
database "Media Storage\n(MinIO/S3)" as s3

== 1. Добавление товара в корзину ==
buyer -> web: Добавить товар в корзину
web -> gw: POST-запрос\n(Товар, количество)
gw -> gw: Проверка Rate Limit
gw -> cart: Проксирование запроса\n(Добавить товар)
cart -> redis: Обновить корзину
redis --> cart: Кеш обновлен
cart --> gw: Товар добавлен
gw --> web: JSON (Корзина обновлена)
web --> buyer: Товар в корзине

== 2. Оформление заказа ==
buyer -> web: Оформить заказ
web -> gw: POST-запрос\n(Данные заказа, номер карты)
gw -> order: Проксирование запроса\n(Создать заказ, оплатить)
order -> catalog: Получить данные товара
catalog -> catalogdb: Чтение цены, остатка и URL медиа
catalogdb --> catalog: Цена, остаток, URL медиа
catalog --> order: Данные товара\n(цена, остаток, URL медиа)

alt Товара нет в наличии
    order --> gw: Ошибка: Товара нет в наличии
    gw --> web: HTTP 409 Conflict\n(Нет в наличии)
    web --> buyer: Сообщение: "К сожалению, товар закончился"
else Товар доступен
    order -> pay: Инициализация оплаты\n(Сумма, номер карты)
    pay -> psp: Проведение платежа (HTTPS)
    
    alt Оплата отклонена
        psp --> pay: Отказ в платеже
        pay -> paydb: Сохранить транзакцию\n(Статус: FAILED)
        paydb --> pay: Транзакция сохранена
        pay --> order: Оплата не прошла
        order --> gw: Ошибка: Оплата отклонена
        gw --> web: HTTP 402 Payment Required
        web --> buyer: Сообщение: "Оплата не прошла.\nПроверьте данные карты."
    else Оплата успешна
        psp --> pay: Успешная оплата
        pay -> paydb: Сохранить транзакцию\n(Статус: SUCCESS)
        paydb --> pay: Транзакция сохранена
        pay --> order: Оплата подтверждена
        order -> orderdb: Создать заказ\n
        orderdb --> order: Заказ создан
        
        group Обмен событиями RabbitMQ
            order ->> mq: publish: OrderCreated\n
            
            mq ->> cart: consume: OrderCreated\n(Убрать товар из корзины)
            cart -> redis: Обновить кеш корзины
            redis --> cart: Кеш обновлен
            
            mq ->> catalog: consume: OrderCreated\n(Уменьшить остаток товара)
            catalog -> catalogdb: Сократить позицию товара
            catalogdb --> catalog: Остаток обновлен
            catalog ->> mq: publish: StockReserved\n
        end
        
        order --> gw: Заказ успешно создан
        gw --> web: JSON (Детали заказа)
        web --> buyer: Экран "Спасибо за заказ!"
    end
end

== 3. Просмотр истории заказов ==
buyer -> web: Просмотр истории заказов
web -> gw: GET-запрос (Список заказов)
gw -> gw: Проверка Rate Limit
gw -> order: Проксирование запроса\n(Получить заказы)
order -> orderdb: Запрос списка заказов
orderdb --> order: Список заказов\n
order --> gw: JSON (Список заказов)
gw --> web: JSON (Список заказов)
web --> buyer: Список заказов
buyer -> media: Запрос медиа по URL
media -> s3: Чтение файла
s3 --> media: Файл
media --> buyer: Фото/видео
@enduml
