---
layout: post
title: "[P0001DP] 2. Генерация данных для БД (пробуем SQL и терпим фейл на большем объёме)"
categories: homelab
---

## \[P0001DP\] 2. Генерация данных для БД (пробуем SQL и терпим фейл на большем объёме)

Для генерации я написал несколько скриптов SQL, один для создания таблиц и несколько для генерации данных для них. 
В дальнейшем эти скрипты наврятли будут применяться, так как для дальнейших экспериментов понадобиться постаянный поток случайных данных в БД.

Скрипт генерации имее следующий вид:

```sql
CREATE SEQUENCE client_id_seq START 1;
CREATE SEQUENCE product_id_seq START 1;
CREATE SEQUENCE order_id_seq START 1;
CREATE SEQUENCE order_item_id_seq START 1;


CREATE TABLE client (
    client_id INTEGER PRIMARY KEY DEFAULT nextval('client_id_seq'),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    city VARCHAR(255) NOT NULL
);

CREATE TABLE product (
    product_id INTEGER PRIMARY KEY DEFAULT nextval('product_id_seq'),
    name VARCHAR(255) NOT NULL,
    price NUMERIC(10, 2) NOT NULL,
    category VARCHAR(100) NOT NULL
);

CREATE TABLE "order" (
    order_id INTEGER PRIMARY KEY DEFAULT nextval('order_id_seq'),
    client_id INTEGER REFERENCES client(client_id),
    status VARCHAR(50) NOT NULL
);

CREATE TABLE order_item (
    order_item_id INTEGER PRIMARY KEY DEFAULT nextval('order_item_id_seq'),
    order_id INTEGER REFERENCES "order"(order_id) ON DELETE CASCADE,
    product_id INTEGER REFERENCES product(product_id),
    quantity INTEGER NOT NULL
);
```

В нем создается несколько последовательностей для назначения идентификаторов каждой сущности(первичных ключей) и собственно сами команды создания таблиц.
Поля довольно стандратные в таблицах, каких либо ограничений мы тут не задавали, разве что значения должны быть не нулевыми. Так же обозначили внешние ключи для таблиц `order` и `order_item`.

Далее напишем скрипты генерации данных для созданных таблиц:

для клиентов

```sql
CREATE OR REPLACE FUNCTION generate_test_client()
RETURNS void AS $$
DECLARE
    client_count INTEGER := 1000000;
BEGIN

    FOR i IN 1..client_count LOOP
        INSERT INTO client (name, email, city)
        VALUES (
            'Клиент ' || i,
            'client' || i || '@example.com',
            CASE (i % 10)
                WHEN 0 THEN 'Новосибирск'
                WHEN 1 THEN 'Москва'
                WHEN 2 THEN 'Санкт-Петербург'
                WHEN 3 THEN 'Великий Новгород'
                WHEN 4 THEN 'Саратов'
                WHEN 5 THEN 'Томск'
                WHEN 6 THEN 'Омск'
                WHEN 7 THEN 'Владивосток'
                WHEN 8 THEN 'Екатеринбург'
                ELSE 'Великий Новгород'
            END
        );
    END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT generate_test_client();
```

для продуктов

```sql
CREATE OR REPLACE FUNCTION generate_test_product()
RETURNS void AS $$
DECLARE
    product_count INTEGER := 500000;
    i INTEGER :=1;
BEGIN
    FOR i IN 1..product_count LOOP
        INSERT INTO product (name, price, category)
        VALUES (
            'Продукт' || i,
            round(CAST(100 + random() * 4900 AS numeric), 2),
            CASE (i % 5)
                WHEN 0 THEN 'Электроника'
                WHEN 1 THEN 'Одежда'
                WHEN 2 THEN 'Книги'
                WHEN 3 THEN 'Бытовая техника'
                ELSE 'Спорт'
            END
       );
    END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT generate_test_product();
```

для заказов и позиций в них

```sql
CREATE OR REPLACE FUNCTION generate_test_orders()
RETURNS void AS $$
DECLARE
    order_count INTEGER := 100000;
    current_client_id BIGINT;
    current_product_id BIGINT;
    current_order_id BIGINT;
    items_per_order INTEGER;
    i INTEGER :=1;
BEGIN

    FOR i IN 1..order_count LOOP
        INSERT INTO "order" (client_id, status)
        VALUES (
            (SELECT client_id FROM client ORDER BY random() LIMIT 1),
            CASE (i % 5)
                WHEN 0 THEN 'Оплачен'
                WHEN 1 THEN 'В обработке'
                WHEN 2 THEN 'Отправлен'
                WHEN 3 THEN 'Доставлен'
                ELSE 'Отменен'
            END
       );
    END LOOP;

    FOR current_order_id IN
        SELECT order_id FROM "order" ORDER BY random()
    LOOP
        items_per_order := (random() * 5 + 1):: integer;

        FOR i IN 1..items_per_order LOOP
            INSERT INTO order_item (order_id, product_id, quantity)
            VALUES (
                current_order_id,
                    (SELECT product_id FROM product ORDER BY random() LIMIT 1),
                    (random() * 10 + 1):: integer
            );
        END LOOP;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT generate_test_orders();

```

По сути мы создаем функции для генерации которые будут доступны для переиспользования(хранимые процедуры) `generate_test_client`, `generate_test_product`, `generate_test_orders`.

После запуска данных функций, я потерпел некий фейл при генерации заказов и позиций в них. Если при генерации 100000, скрипт еще отрабатывал, то при генерации в 1 млн. записей, работа скрипта уходила в бесконечность.

После гугления и общения с "оракулами", я понял что написал довольно страшный скрипт. При разраборе, дело оказалось в том что цикл FOR создает милионы отдельных вставок а также рандомная выборка клиента и продукта заставляет сканировать всю таблицу `client` и `product`
что в перспективе создает триллионы операций.

По итогу сделал следующее:
- Заменил `ORDER BY random() LIMIT 1` на расчет `floor(random() * max_id + 1)`.
- Убрал циклы и добавил функцию `generate_series()` для создания строк в памяти, который потом будет встален единой пачкой.
- Заиспользовал `CROSS JOIN LATERAL` что позволяет для каждого заказа разножить строки, создав от 1 до 5 позиций в каждом.

Поправив все, получил следующий скрипт:

```sql
CREATE OR REPLACE FUNCTION generate_test_orders()
RETURNS void AS $$
DECLARE
    order_count INTEGER := 1000000;
    max_client_id INTEGER;
    max_product_id INTEGER;
BEGIN
    SELECT max(client_id) INTO max_client_id FROM client;
    SELECT max(product_id) INTO max_product_id FROM product;
    
    INSERT INTO "order" (client_id, status)
    SELECT
        floor(random() * max_client_id + 1)::integer,
        CASE (s.id % 5)
            WHEN 0 THEN 'Оплачен'
            WHEN 1 THEN 'В обработке'
            WHEN 2 THEN 'Отправлен'
            WHEN 3 THEN 'Доставлен'
            ELSE 'Отменен'
        END
    FROM generate_series(1, order_count) AS s(id);

    INSERT INTO order_item (order_id, product_id, quantity)
    SELECT
        o.order_id,
        floor(random() * max_product_id + 1)::integer,
        floor(random() * 10 + 1)::integer
    FROM (
        SELECT order_id
        FROM "order"
    ) o

    CROSS JOIN LATERAL generate_series(1, floor(random() * 5 + 1)::integer);

END;
$$ LANGUAGE plpgsql;

SELECT generate_test_orders();
```

После этого скрипт заработал и не вешал большел БД. В результате генерации получились следующие результаты:


| Таблица  | Количество  | Время выполнеения  |
|----------|-------------|--------------------|
| client   | 1000000     | 12 сек             | 
| product  | 500000      | 4 сек              |
| order    | 1000000     | 44 сек             |

