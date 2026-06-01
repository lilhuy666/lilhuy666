import mysql.connector
from mysql.connector import Error
from datetime import datetime
import hashlib
import sys

# Конфигурация базы данных
DB_CONFIG = {
    'host': 'localhost',
    'user': 'root',
    'password': 'your_password',  # ЗАМЕНИТЕ НА ВАШ ПАРОЛЬ
    'database': 'shoe_store',
    'charset': 'utf8mb4'
}

class Database:
    """Класс для работы с базой данных"""
    
    def __init__(self):
        self.connection = None
        self.connect()
    
    def connect(self):
        """Подключение к базе данных"""
        try:
            self.connection = mysql.connector.connect(**DB_CONFIG)
            if self.connection.is_connected():
                print("✓ Успешное подключение к базе данных")
        except Error as e:
            print(f"Ошибка подключения к MySQL: {e}")
            sys.exit(1)
    
    def execute_query(self, query, params=None):
        """Выполнение запроса с возвратом результатов"""
        cursor = self.connection.cursor(dictionary=True)
        try:
            if params:
                cursor.execute(query, params)
            else:
                cursor.execute(query)
            
            if query.strip().upper().startswith(('INSERT', 'UPDATE', 'DELETE')):
                self.connection.commit()
                return cursor.lastrowid
            else:
                return cursor.fetchall()
        except Error as e:
            print(f"Ошибка выполнения запроса: {e}")
            return None
        finally:
            cursor.close()
    
    def close(self):
        """Закрытие соединения"""
        if self.connection and self.connection.is_connected():
            self.connection.close()
            print("Соединение с базой данных закрыто")


class User:
    """Класс пользователя"""
    
    def __init__(self, db):
        self.db = db
        self.current_user = None
    
    def login(self, username, password):
        """Авторизация пользователя"""
        query = "SELECT * FROM users WHERE username = %s AND password = %s"
        users = self.db.execute_query(query, (username, password))
        
        if users:
            self.current_user = users[0]
            return True
        return False
    
    def register(self, username, password, email, full_name):
        """Регистрация нового клиента"""
        query = """
        INSERT INTO users (username, password, role, email, full_name) 
        VALUES (%s, %s, 'client', %s, %s)
        """
        try:
            self.db.execute_query(query, (username, password, email, full_name))
            return True
        except:
            return False
    
    def get_role(self):
        """Получение роли текущего пользователя"""
        return self.current_user['role'] if self.current_user else None
    
    def get_username(self):
        """Получение имени пользователя"""
        return self.current_user['username'] if self.current_user else None


class ProductManager:
    """Класс для управления товарами (менеджер/админ)"""
    
    def __init__(self, db):
        self.db = db
    
    def add_product(self, name, description, price, category_id, brand, size, color, stock):
        """Добавление нового товара"""
        query = """
        INSERT INTO products (name, description, price, category_id, brand, size, color, stock_quantity)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
        """
        self.db.execute_query(query, (name, description, price, category_id, brand, size, color, stock))
        print("✓ Товар успешно добавлен")
    
    def update_product(self, product_id, name, description, price, category_id, brand, size, color, stock):
        """Обновление товара"""
        query = """
        UPDATE products 
        SET name=%s, description=%s, price=%s, category_id=%s, brand=%s, size=%s, color=%s, stock_quantity=%s
        WHERE id=%s
        """
        self.db.execute_query(query, (name, description, price, category_id, brand, size, color, stock, product_id))
        print("✓ Товар успешно обновлен")
    
    def delete_product(self, product_id):
        """Удаление товара"""
        query = "DELETE FROM products WHERE id = %s"
        self.db.execute_query(query, (product_id,))
        print("✓ Товар успешно удален")
    
    def get_all_products(self):
        """Получение всех товаров"""
        query = """
        SELECT p.*, c.name as category_name 
        FROM products p 
        LEFT JOIN categories c ON p.category_id = c.id
        """
        return self.db.execute_query(query)
    
    def get_products_by_category(self, category_id):
        """Получение товаров по категории"""
        query = "SELECT * FROM products WHERE category_id = %s"
        return self.db.execute_query(query, (category_id,))


class CartManager:
    """Класс для управления корзиной (клиент)"""
    
    def __init__(self, db, user_id):
        self.db = db
        self.user_id = user_id
    
    def add_to_cart(self, product_id, quantity=1):
        """Добавление товара в корзину"""
        query = """
        INSERT INTO cart (user_id, product_id, quantity)
        VALUES (%s, %s, %s)
        ON DUPLICATE KEY UPDATE quantity = quantity + %s
        """
        self.db.execute_query(query, (self.user_id, product_id, quantity, quantity))
        print("✓ Товар добавлен в корзину")
    
    def remove_from_cart(self, product_id):
        """Удаление товара из корзины"""
        query = "DELETE FROM cart WHERE user_id = %s AND product_id = %s"
        self.db.execute_query(query, (self.user_id, product_id))
        print("✓ Товар удален из корзины")
    
    def view_cart(self):
        """Просмотр корзины"""
        query = """
        SELECT c.*, p.name, p.price, p.brand, (p.price * c.quantity) as total_price
        FROM cart c
        JOIN products p ON c.product_id = p.id
        WHERE c.user_id = %s
        """
        items = self.db.execute_query(query, (self.user_id,))
        
        if not items:
            print("Корзина пуста")
            return
        
        total_sum = 0
        print("\n=== ВАША КОРЗИНА ===")
        for item in items:
            print(f"ID: {item['product_id']} | {item['name']} | {item['brand']} | "
                  f"Цена: {item['price']:.2f} руб. | Кол-во: {item['quantity']} | "
                  f"Сумма: {item['total_price']:.2f} руб.")
            total_sum += item['total_price']
        print(f"\nОБЩАЯ СУММА: {total_sum:.2f} руб.")
    
    def checkout(self, shipping_address):
        """Оформление заказа"""
        # Получаем товары из корзины
        cart_query = """
        SELECT c.product_id, c.quantity, p.price, p.stock_quantity
        FROM cart c
        JOIN products p ON c.product_id = p.id
        WHERE c.user_id = %s
        """
        cart_items = self.db.execute_query(cart_query, (self.user_id,))
        
        if not cart_items:
            print("Корзина пуста!")
            return
        
        # Проверяем наличие товаров
        for item in cart_items:
            if item['quantity'] > item['stock_quantity']:
                print(f"Недостаточно товара ID {item['product_id']} на складе!")
                return
        
        # Создаем заказ
        total_amount = sum(item['price'] * item['quantity'] for item in cart_items)
        order_query = """
        INSERT INTO orders (user_id, total_amount, shipping_address)
        VALUES (%s, %s, %s)
        """
        order_id = self.db.execute_query(order_query, (self.user_id, total_amount, shipping_address))
        
        # Добавляем товары в заказ
        for item in cart_items:
            item_query = """
            INSERT INTO order_items (order_id, product_id, quantity, price_per_unit)
            VALUES (%s, %s, %s, %s)
            """
            self.db.execute_query(item_query, (order_id, item['product_id'], item['quantity'], item['price']))
            
            # Обновляем количество на складе
            update_query = "UPDATE products SET stock_quantity = stock_quantity - %s WHERE id = %s"
            self.db.execute_query(update_query, (item['quantity'], item['product_id']))
        
        # Очищаем корзину
        clear_query = "DELETE FROM cart WHERE user_id = %s"
        self.db.execute_query(clear_query, (self.user_id,))
        
        print(f"✓ Заказ #{order_id} успешно оформлен!")
        print(f"Сумма заказа: {total_amount:.2f} руб.")


class OrderManager:
    """Класс для управления заказами"""
    
    def __init__(self, db):
        self.db = db
    
    def get_user_orders(self, user_id):
        """Просмотр заказов пользователя"""
        query = """
        SELECT o.*, 
               GROUP_CONCAT(CONCAT(p.name, ' (', oi.quantity, ' шт.)') SEPARATOR ', ') as products
        FROM orders o
        JOIN order_items oi ON o.id = oi.order_id
        JOIN products p ON oi.product_id = p.id
        WHERE o.user_id = %s
        GROUP BY o.id
        ORDER BY o.order_date DESC
        """
        orders = self.db.execute_query(query, (user_id,))
        
        if not orders:
            print("У вас пока нет заказов")
            return
        
        print("\n=== ВАШИ ЗАКАЗЫ ===")
        for order in orders:
            print(f"\nЗаказ #{order['id']} | Дата: {order['order_date']} | Статус: {order['status']}")
            print(f"Товары: {order['products']}")
            print(f"Сумма: {order['total_amount']:.2f} руб.")
    
    def get_all_orders(self):
        """Просмотр всех заказов (для менеджера/админа)"""
        query = """
        SELECT o.*, u.full_name, u.email,
               GROUP_CONCAT(CONCAT(p.name, ' (', oi.quantity, ' шт.)') SEPARATOR ', ') as products
        FROM orders o
        JOIN users u ON o.user_id = u.id
        JOIN order_items oi ON o.id = oi.order_id
        JOIN products p ON oi.product_id = p.id
        GROUP BY o.id
        ORDER BY o.order_date DESC
        """
        return self.db.execute_query(query)
    
    def update_order_status(self, order_id, status):
        """Обновление статуса заказа"""
        valid_statuses = ['pending', 'processing', 'shipped', 'delivered', 'cancelled']
        if status not in valid_statuses:
            print("Недопустимый статус!")
            return
        
        query = "UPDATE orders SET status = %s WHERE id = %s"
        self.db.execute_query(query, (status, order_id))
        print(f"✓ Статус заказа #{order_id} обновлен на '{status}'")


class UserManager:
    """Класс для управления пользователями (админ)"""
    
    def __init__(self, db):
        self.db = db
    
    def get_all_users(self):
        """Получение списка всех пользователей"""
        query = "SELECT id, username, role, email, full_name, created_at FROM users"
        return self.db.execute_query(query)
    
    def change_user_role(self, user_id, new_role):
        """Изменение роли пользователя"""
        query = "UPDATE users SET role = %s WHERE id = %s"
        self.db.execute_query(query, (new_role, user_id))
        print(f"✓ Роль пользователя изменена на '{new_role}'")
    
    def delete_user(self, user_id):
        """Удаление пользователя"""
        query = "DELETE FROM users WHERE id = %s"
        self.db.execute_query(query, (user_id,))
        print("✓ Пользователь удален")


class ShoeStoreApp:
    """Основной класс приложения магазина обуви"""
    
    def __init__(self):
        self.db = Database()
        self.user = User(self.db)
        self.running = True
    
    def display_menu(self):
        """Отображение меню в зависимости от роли"""
        role = self.user.get_role()
        
        print(f"\n=== МАГАЗИН ОБУВИ ===")
        print(f"Пользователь: {self.user.get_username()} ({role})")
        
        if role == 'admin':
            self.admin_menu()
        elif role == 'manager':
            self.manager_menu()
        elif role == 'client':
            self.client_menu()
    
    def admin_menu(self):
        """Меню администратора"""
        product_manager = ProductManager(self.db)
        order_manager = OrderManager(self.db)
        user_manager = UserManager(self.db)
        
        print("\n1. Просмотр всех товаров")
        print("2. Добавить товар")
        print("3. Редактировать товар")
        print("4. Удалить товар")
        print("5. Просмотр всех заказов")
        print("6. Изменить статус заказа")
        print("7. Просмотр пользователей")
        print("8. Изменить роль пользователя")
        print("9. Удалить пользователя")
        print("0. Выйти")
        
        choice = input("\nВыберите действие: ")
        
        if choice == '1':
            products = product_manager.get_all_products()
            self.display_products(products)
        elif choice == '2':
            self.add_product_interface(product_manager)
        elif choice == '3':
            self.update_product_interface(product_manager)
        elif choice == '4':
            product_id = int(input("ID товара для удаления: "))
            product_manager.delete_product(product_id)
        elif choice == '5':
            orders = order_manager.get_all_orders()
            self.display_all_orders(orders)
        elif choice == '6':
            order_id = int(input("ID заказа: "))
            status = input("Новый статус (pending/processing/shipped/delivered/cancelled): ")
            order_manager.update_order_status(order_id, status)
        elif choice == '7':
            users = user_manager.get_all_users()
            self.display_users(users)
        elif choice == '8':
            user_id = int(input("ID пользователя: "))
            new_role = input("Новая роль (client/manager/admin): ")
            user_manager.change_user_role(user_id, new_role)
        elif choice == '9':
            user_id = int(input("ID пользователя для удаления: "))
            user_manager.delete_user(user_id)
        elif choice == '0':
            self.logout()
    
    def manager_menu(self):
        """Меню менеджера"""
        product_manager = ProductManager(self.db)
        order_manager = OrderManager(self.db)
        
        print("\n1. Просмотр всех товаров")
        print("2. Добавить товар")
        print("3. Редактировать товар")
        print("4. Просмотр всех заказов")
        print("5. Изменить статус заказа")
        print("0. Выйти")
        
        choice = input("\nВыберите действие: ")
        
        if choice == '1':
            products = product_manager.get_all_products()
            self.display_products(products)
        elif choice == '2':
            self.add_product_interface(product_manager)
        elif choice == '3':
            self.update_product_interface(product_manager)
        elif choice == '4':
            orders = order_manager.get_all_orders()
            self.display_all_orders(orders)
        elif choice == '5':
            order_id = int(input("ID заказа: "))
            status = input("Новый статус (pending/processing/shipped/delivered/cancelled): ")
            order_manager.update_order_status(order_id, status)
        elif choice == '0':
            self.logout()
    
    def client_menu(self):
        """Меню клиента"""
        cart_manager = CartManager(self.db, self.user.current_user['id'])
        order_manager = OrderManager(self.db)
        product_manager = ProductManager(self.db)
        
        print("\n1. Просмотр каталога")
        print("2. Просмотр корзины")
        print("3. Добавить в корзину")
        print("4. Удалить из корзины")
        print("5. Оформить заказ")
        print("6. Мои заказы")
        print("0. Выйти")
        
        choice = input("\nВыберите действие: ")
        
        if choice == '1':
            products = product_manager.get_all_products()
            self.display_products(products)
        elif choice == '2':
            cart_manager.view_cart()
        elif choice == '3':
            product_id = int(input("ID товара для добавления: "))
            quantity = int(input("Количество: "))
            cart_manager.add_to_cart(product_id, quantity)
        elif choice == '4':
            product_id = int(input("ID товара для удаления: "))
            cart_manager.remove_from_cart(product_id)
        elif choice == '5':
            address = input("Введите адрес доставки: ")
            cart_manager.checkout(address)
        elif choice == '6':
            order_manager.get_user_orders(self.user.current_user['id'])
        elif choice == '0':
            self.logout()
    
    def add_product_interface(self, product_manager):
        """Интерфейс добавления товара"""
        print("\n=== ДОБАВЛЕНИЕ ТОВАРА ===")
        name = input("Название: ")
        description = input("Описание: ")
        price = float(input("Цена: "))
        
        # Показываем категории
        categories = self.db.execute_query("SELECT * FROM categories")
        print("\nКатегории:")
        for cat in categories:
            print(f"ID: {cat['id']} - {cat['name']}")
        
        category_id = int(input("ID категории: "))
        brand = input("Бренд: ")
        size = input("Размер: ")
        color = input("Цвет: ")
        stock = int(input("Количество на складе: "))
        
        product_manager.add_product(name, description, price, category_id, brand, size, color, stock)
    
    def update_product_interface(self, product_manager):
        """Интерфейс обновления товара"""
        product_id = int(input("ID товара для редактирования: "))
        print("\nВведите новые данные:")
        name = input("Название: ")
        description = input("Описание: ")
        price = float(input("Цена: "))
        
        categories = self.db.execute_query("SELECT * FROM categories")
        print("\nКатегории:")
        for cat in categories:
            print(f"ID: {cat['id']} - {cat['name']}")
        
        category_id = int(input("ID категории: "))
        brand = input("Бренд: ")
        size = input("Размер: ")
        color = input("Цвет: ")
        stock = int(input("Количество на складе: "))
        
        product_manager.update_product(product_id, name, description, price, category_id, brand, size, color, stock)
    
    def display_products(self, products):
        """Отображение списка товаров"""
        if not products:
            print("Товары не найдены")
            return
        
        print("\n=== КАТАЛОГ ТОВАРОВ ===")
        for product in products:
            category_name = product.get('category_name', 'Нет категории')
            print(f"\nID: {product['id']} | {product['name']}")
            print(f"Категория: {category_name} | Бренд: {product['brand']}")
            print(f"Цена: {product['price']:.2f} руб. | Размер: {product['size']} | Цвет: {product['color']}")
            print(f"В наличии: {product['stock_quantity']} шт.")
            print(f"Описание: {product['description']}")
    
    def display_all_orders(self, orders):
        """Отображение всех заказов"""
        if not orders:
            print("Заказы не найдены")
            return
        
        print("\n=== ВСЕ ЗАКАЗЫ ===")
        for order in orders:
            print(f"\nЗаказ #{order['id']} | Клиент: {order['full_name']} | Дата: {order['order_date']}")
            print(f"Статус: {order['status']} | Сумма: {order['total_amount']:.2f} руб.")
            print(f"Товары: {order['products']}")
    
    def display_users(self, users):
        """Отображение списка пользователей"""
        if not users:
            print("Пользователи не найдены")
            return
        
        print("\n=== ПОЛЬЗОВАТЕЛИ ===")
        for user in users:
            print(f"ID: {user['id']} | {user['full_name']} | {user['username']} | "
                  f"Роль: {user['role']} | Email: {user['email']}")
    
    def login_screen(self):
        """Экран авторизации"""
        print("\n=== АВТОРИЗАЦИЯ ===")
        print("1. Войти")
        print("2. Зарегистрироваться")
        print("0. Выйти из приложения")
        
        choice = input("\nВыберите действие: ")
        
        if choice == '1':
            username = input("Логин: ")
            password = input("Пароль: ")
            
            if self.user.login(username, password):
                print(f"✓ Добро пожаловать, {self.user.get_username()}!")
                return True
            else:
                print("Неверный логин или пароль!")
                return False
        elif choice == '2':
            username = input("Логин: ")
            password = input("Пароль: ")
            email = input("Email: ")
            full_name = input("Полное имя: ")
            
            if self.user.register(username, password, email, full_name):
                print("✓ Регистрация успешна! Теперь войдите в систему.")
            else:
                print("Ошибка регистрации. Возможно, пользователь уже существует.")
            return False
        elif choice == '0':
            self.running = False
            return False
    
    def logout(self):
        """Выход из системы"""
        print(f"До свидания, {self.user.get_username()}!")
        self.user.current_user = None
    
    def run(self):
        """Запуск приложения"""
        print("\nДобро пожаловать в магазин обуви 'ShoeStore'!")
        
        while self.running:
            if not self.user.current_user:
                if not self.login_screen():
                    continue
            
            self.display_menu()
            
            if not self.running:
                break
    
    def close(self):
        """Завершение работы"""
        self.db.close()
        print("Приложение завершено")


# Точка входа
if __name__ == "__main__":
    app = ShoeStoreApp()
    try:
        app.run()
    except KeyboardInterrupt:
        print("\nПриложение прервано пользователем")
    finally:
        app.close()
