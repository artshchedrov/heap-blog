---
layout: post
title: "[P0001DP] 2. Генерация данных для БД (пробуем SQL и терпим фейл на большем объёме)"
categories: homelab
---

## \[P0001DP\] 2. Генерация данных для БД (пробуем SQL и терпим фейл на большем объёме)

Для генерации я написал несколько скриптов SQL, один для создания таблиц и несколько для генерации данных для них. 
В дальнейшем эти скрипты наврятли будут приметяться, так как для дальнейших экспериментов понадобиться постаянный поток случайных данных в БД.

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

В нем создается несколько последовательностей для назначения индетификаторов каждой сущности(первичных ключей) и собственно сами команды создания таблиц.
Поля довольно стандратные в таблицах, каких либо ограничений мы тут не задавали, разве что значения должны быть не нулевыми. Так же обозначили внешние ключи для таблиц `order` и `order_item`.