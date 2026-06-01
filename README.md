CREATE DATABASE shoe_store;
USE shoe_store;

-- Таблица пользователей
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role ENUM('client', 'manager', 'admin') NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица категорий обуви
CREATE TABLE categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT
);

-- Таблица товаров (обувь)
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    category_id INT,
    brand VARCHAR(50),
    size VARCHAR(10),
    color VARCHAR(30),
    stock_quantity INT DEFAULT 0,
    image_url VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL
);

-- Таблица заказов
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    total_amount DECIMAL(10, 2) NOT NULL,
    shipping_address TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Таблица деталей заказа
CREATE TABLE order_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    price_per_unit DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);

-- Таблица корзины
CREATE TABLE cart (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT DEFAULT 1,
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    UNIQUE KEY unique_cart_item (user_id, product_id)
);

-- Вставка тестовых данных
INSERT INTO users (username, password, role, email, full_name) VALUES
('admin', 'admin123', 'admin', 'admin@shoestore.com', 'Администратор'),
('manager', 'manager123', 'manager', 'manager@shoestore.com', 'Менеджер Иванов'),
('client1', 'client123', 'client', 'client1@email.com', 'Петров Петр'),
('client2', 'client123', 'client', 'client2@email.com', 'Сидорова Анна');

INSERT INTO categories (name, description) VALUES
('Мужская обувь', 'Обувь для мужчин'),
('Женская обувь', 'Обувь для женщин'),
('Спортивная обувь', 'Кроссовки и спортивная обувь'),
('Детская обувь', 'Обувь для детей');

INSERT INTO products (name, description, price, category_id, brand, size, color, stock_quantity) VALUES
('Классические туфли', 'Элегантные мужские туфли', 5999.99, 1, 'ECCO', '42', 'Черный', 50),
('Кроссовки Air', 'Спортивные кроссовки', 8999.99, 3, 'Nike', '43', 'Белый', 30),
('Босоножки летние', 'Женские босоножки', 3499.99, 2, 'Mascotte', '38', 'Бежевый', 25),
('Детские сандалии', 'Удобные детские сандалии', 1999.99, 4, 'Капика', '30', 'Синий', 40),
('Зимние ботинки', 'Теплые зимние ботинки', 7999.99, 1, 'Timberland', '44', 'Коричневый', 20);
