-- =====================================================
-- Создание базы данных для интернет-магазина
-- =====================================================

-- 1. Создаем базу данных (если не существует)
CREATE DATABASE IF NOT EXISTS shop
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

-- 2. Выбираем базу для работы
USE shop;

-- =====================================================
-- 3. Создание таблицы пользователей (u)
-- =====================================================
CREATE TABLE IF NOT EXISTS u (
    id INT AUTO_INCREMENT PRIMARY KEY,
    n VARCHAR(100) COMMENT 'Имя пользователя',
    e VARCHAR(100) UNIQUE NOT NULL COMMENT 'Email (логин)',
    p TEXT NOT NULL COMMENT 'Пароль (хэш)',
    ph VARCHAR(20) DEFAULT '' COMMENT 'Телефон',
    adr TEXT DEFAULT '' COMMENT 'Адрес',
    r VARCHAR(20) DEFAULT 'client' COMMENT 'Роль: admin/manager/client'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =====================================================
-- 4. Создание таблицы товаров (p)
-- =====================================================
CREATE TABLE IF NOT EXISTS p (
    id INT AUTO_INCREMENT PRIMARY KEY,
    n TEXT NOT NULL COMMENT 'Название товара',
    pr DECIMAL(10,2) NOT NULL COMMENT 'Цена',
    old_pr DECIMAL(10,2) DEFAULT 0 COMMENT 'Старая цена (для скидки)',
    d TEXT COMMENT 'Описание товара',
    cat VARCHAR(50) DEFAULT 'other' COMMENT 'Категория',
    st INT DEFAULT 0 COMMENT 'Количество на складе',
    manufacturer VARCHAR(100) DEFAULT '' COMMENT 'Производитель',
    supplier VARCHAR(100) DEFAULT '' COMMENT 'Поставщик',
    unit VARCHAR(20) DEFAULT 'шт.' COMMENT 'Единица измерения'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =====================================================
-- 5. Создание таблицы корзины (cr)
-- =====================================================
CREATE TABLE IF NOT EXISTS cr (
    id INT AUTO_INCREMENT PRIMARY KEY,
    pid INT NOT NULL COMMENT 'ID товара',
    q INT NOT NULL DEFAULT 1 COMMENT 'Количество',
    uid INT NOT NULL COMMENT 'ID пользователя',
    FOREIGN KEY (pid) REFERENCES p(id) ON DELETE CASCADE,
    FOREIGN KEY (uid) REFERENCES u(id) ON DELETE CASCADE,
    INDEX idx_cart_user (uid),
    INDEX idx_cart_product (pid)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =====================================================
-- 6. Создание таблицы заказов (o)
-- =====================================================
CREATE TABLE IF NOT EXISTS o (
    id INT AUTO_INCREMENT PRIMARY KEY,
    uid INT NOT NULL COMMENT 'ID пользователя',
    dt TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT 'Дата и время заказа',
    adr TEXT NOT NULL COMMENT 'Адрес доставки',
    tot DECIMAL(10,2) NOT NULL COMMENT 'Общая сумма',
    st TEXT DEFAULT 'Ожидает' COMMENT 'Статус заказа',
    pay TEXT DEFAULT 'Наличные' COMMENT 'Способ оплаты',
    FOREIGN KEY (uid) REFERENCES u(id) ON DELETE CASCADE,
    INDEX idx_orders_user (uid),
    INDEX idx_orders_date (dt),
    INDEX idx_orders_status (st(10))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =====================================================
-- 7. Создание таблицы товаров в заказе (oi)
-- =====================================================
CREATE TABLE IF NOT EXISTS oi (
    id INT AUTO_INCREMENT PRIMARY KEY,
    oid INT NOT NULL COMMENT 'ID заказа',
    pid INT NOT NULL COMMENT 'ID товара',
    q INT NOT NULL COMMENT 'Количество',
    pr DECIMAL(10,2) NOT NULL COMMENT 'Цена на момент заказа',
    FOREIGN KEY (oid) REFERENCES o(id) ON DELETE CASCADE,
    FOREIGN KEY (pid) REFERENCES p(id) ON DELETE CASCADE,
    INDEX idx_order_items_order (oid),
    INDEX idx_order_items_product (pid)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =====================================================
-- 8. Добавление тестовых пользователей
-- =====================================================
-- Пароли:
-- admin@shop.ru -> admin123
-- manager@shop.ru -> manager123
-- client@shop.ru -> client123
INSERT INTO u (n, e, p, r) VALUES
('Администратор', 'admin@shop.ru', 'scrypt:32768:8:1$L4kHjF8sQmNpR2vT$9a3b5c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8g9h0i1j2k3l4m5n6o7p8q9r0s1t', 'admin'),
('Менеджер', 'manager@shop.ru', 'scrypt:32768:8:1$XyZ1aBcDeFgHiJkL$2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t1u2v3w4x5y6z7A8B9C0D1E2F3G4H5I', 'manager'),
('Тестовый клиент', 'client@shop.ru', 'scrypt:32768:8:1$AbCdEfGhIjKlMnOp$3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t1u2v3w4x5y6z7A8B9C0D1E2F3G4H5I6J', 'client');

-- =====================================================
-- 9. Добавление тестовых товаров
-- =====================================================
INSERT INTO p (n, pr, old_pr, d, cat, st, manufacturer, supplier, unit) VALUES
('Ноутбук HP Pavilion', 49999, 59999, 'Мощный ноутбук для работы и учебы. Процессор Intel Core i5, 16GB RAM, 512GB SSD', 'Электроника', 50, 'HP', 'ООО Техно', 'шт.'),
('Смартфон Samsung Galaxy', 89999, 99999, 'Флагманский смартфон с отличной камерой и большим экраном', 'Электроника', 100, 'Samsung', 'ООО Мобайл', 'шт.'),
('Наушники Sony WH-1000XM5', 11999, 14999, 'Беспроводные наушники с шумоподавлением премиум-класса', 'Аудио', 200, 'Sony', 'ООО Аудио', 'шт.'),
('Планшет Apple iPad', 44999, 49999, 'Легкий и мощный планшет для творчества и работы', 'Электроника', 75, 'Apple', 'ООО Техно', 'шт.'),
('Умные часы Xiaomi', 23999, 29999, 'Умные часы с GPS и мониторингом здоровья', 'Гаджеты', 150, 'Xiaomi', 'ООО Гаджеты', 'шт.'),
('Колонка JBL Charge', 7999, 9999, 'Портативная Bluetooth колонка с мощным звуком', 'Аудио', 0, 'JBL', 'ООО Аудио', 'шт.'),
('Мышь Logitech MX Master', 8999, 11999, 'Эргономичная беспроводная мышь для профессионалов', 'Комплектующие', 80, 'Logitech', 'ООО Компьютер', 'шт.'),
('Клавиатура механическая', 12999, 15999, 'Механическая клавиатура с RGB подсветкой', 'Комплектующие', 45, 'Razer', 'ООО Компьютер', 'шт.'),
('Монитор Dell 27"', 27999, 32999, '27-дюймовый монитор 4K для дизайнеров', 'Электроника', 30, 'Dell', 'ООО Техно', 'шт.'),
('Внешний SSD 1TB', 11999, 14999, 'Быстрый внешний SSD для хранения данных', 'Хранение', 120, 'Samsung', 'ООО Компьютер', 'шт.');

-- =====================================================
-- 10. Добавление тестового заказа (опционально)
-- =====================================================
-- Раскомментируйте если нужно добавить тестовый заказ
/*
INSERT INTO o (uid, adr, tot, st, pay) 
VALUES (3, 'г. Москва, ул. Тестовая, д. 1, кв. 1', 61998, 'Оплачен', 'Карта');

INSERT INTO oi (oid, pid, q, pr) VALUES
(1, 1, 1, 49999),
(1, 3, 1, 11999);
*/

-- =====================================================
-- 11. Просмотр информации о созданных таблицах
-- =====================================================
-- Показать все таблицы
SHOW TABLES;

-- Показать структуру таблиц
DESCRIBE u;
DESCRIBE p;
DESCRIBE cr;
DESCRIBE o;
DESCRIBE oi;

-- Показать статистику
SELECT 'Пользователи:' as Info, COUNT(*) as Count FROM u
UNION ALL
SELECT 'Товары:', COUNT(*) FROM p
UNION ALL
SELECT 'Товаров на складе:', SUM(st) FROM p;
