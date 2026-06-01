-- ============================================
-- СОЗДАНИЕ БАЗЫ ДАННЫХ
-- ============================================
CREATE DATABASE IF NOT EXISTS mydb;
USE mydb;

-- ============================================
-- УДАЛЕНИЕ СТАРЫХ ТАБЛИЦ (если нужно пересоздать)
-- ============================================
-- DROP TABLE IF EXISTS orders;
-- DROP TABLE IF EXISTS users;

-- ============================================
-- ТАБЛИЦА ПОЛЬЗОВАТЕЛЕЙ
-- ============================================
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(100) UNIQUE,
    age INT,
    city VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- ТАБЛИЦА ЗАКАЗОВ
-- ============================================
CREATE TABLE IF NOT EXISTS orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    product VARCHAR(100) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- ИНДЕКСЫ ДЛЯ УСКОРЕНИЯ ПОИСКА
-- ============================================
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_city ON users(city);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_date ON orders(order_date);

-- ============================================
-- ТЕСТОВЫЕ ДАННЫЕ - ПОЛЬЗОВАТЕЛИ
-- ============================================
INSERT INTO users (name, phone, email, age, city) VALUES
('Иван Иванов', '+7-999-123-45-67', 'ivan@mail.ru', 25, 'Москва'),
('Мария Петрова', '+7-999-765-43-21', 'maria@mail.ru', 30, 'Санкт-Петербург'),
('Алексей Сидоров', '+7-999-555-33-22', 'alex@mail.ru', 35, 'Казань'),
('Елена Смирнова', '+7-999-111-22-33', 'elena@mail.ru', 28, 'Новосибирск'),
('Дмитрий Козлов', '+7-999-444-55-66', 'dmitry@mail.ru', 32, 'Екатеринбург');

-- ============================================
-- ТЕСТОВЫЕ ДАННЫЕ - ЗАКАЗЫ
-- ============================================
INSERT INTO orders (user_id, product, price) VALUES
(1, 'Ноутбук', 55000.00),
(1, 'Мышь', 2500.00),
(2, 'Монитор', 15000.00),
(3, 'Клавиатура', 5000.00),
(4, 'Наушники', 8000.00);

-- ============================================
-- ПРОВЕРКА ДАННЫХ
-- ============================================
-- Просмотр пользователей
SELECT * FROM users;

-- Просмотр заказов с именами пользователей
SELECT 
    u.name AS 'Пользователь',
    o.product AS 'Товар',
    o.price AS 'Цена',
    o.order_date AS 'Дата заказа'
FROM users u
JOIN orders o ON u.id = o.user_id
ORDER BY o.order_date DESC;

-- Статистика
SELECT 
    COUNT(*) AS 'Всего пользователей',
    AVG(age) AS 'Средний возраст',
    COUNT(DISTINCT city) AS 'Количество городов'
FROM users;

-- Сумма всех заказов
SELECT 
    COUNT(*) AS 'Всего заказов',
    SUM(price) AS 'Общая сумма',
    AVG(price) AS 'Средний чек'
FROM orders;

-- ============================================
-- ПОЛЕЗНЫЕ ЗАПРОСЫ ДЛЯ ТЕСТИРОВАНИЯ
-- ============================================

-- 1. Добавление нового пользователя
/*
INSERT INTO users (name, phone, email, age, city) 
VALUES ('Тестовый Пользователь', '+7-000-000-00-00', 'test@mail.ru', 20, 'Тестовый Город');
*/

-- 2. Обновление пользователя
/*
UPDATE users 
SET phone = '+7-999-999-99-99', age = 26 
WHERE id = 1;
*/

-- 3. Удаление пользователя
/*
DELETE FROM users WHERE id = 5;
*/

-- 4. Поиск пользователей по городу
/*
SELECT * FROM users WHERE city = 'Москва';
*/

-- 5. Пользователи старше 30 лет
/*
SELECT * FROM users WHERE age > 30 ORDER BY age DESC;
*/

-- 6. Заказы дороже 10000 руб
/*
SELECT u.name, o.product, o.price 
FROM users u 
JOIN orders o ON u.id = o.user_id 
WHERE o.price > 10000 
ORDER BY o.price DESC;
*/

-- 7. Количество заказов по пользователям
/*
SELECT 
    u.name,
    COUNT(o.id) AS 'Количество заказов',
    SUM(o.price) AS 'Общая сумма'
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name
ORDER BY COUNT(o.id) DESC;
*/
