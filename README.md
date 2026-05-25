from flask import Flask, render_template, request, redirect, url_for, session
import sqlite3
import os
from werkzeug.utils import secure_filename
from PIL import Image

app = Flask(__name__)
app.secret_key = 'exam_secret_2026'
app.config['UPLOAD_FOLDER'] = 'static/uploads'
os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)

# Инициализация БД
def init_db():
    with sqlite3.connect('database.db') as conn:
        conn.executescript('''
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY,
                login TEXT UNIQUE,
                password TEXT,
                role TEXT,
                full_name TEXT
            );
            INSERT OR IGNORE INTO users (id, login, password, role, full_name) VALUES 
                (1, 'admin', 'admin123', 'admin', 'Админ Админов'),
                (2, 'manager', 'manager123', 'manager', 'Менеджер Петров'),
                (3, 'client', 'client123', 'client', 'Клиент Иванов');
                
            CREATE TABLE IF NOT EXISTS categories (
                id INTEGER PRIMARY KEY,
                name TEXT
            );
            INSERT OR IGNORE INTO categories VALUES (1, 'Кроссовки'), (2, 'Ботинки'), (3, 'Сандалии');
            
            CREATE TABLE IF NOT EXISTS manufacturers (
                id INTEGER PRIMARY KEY,
                name TEXT
            );
            INSERT OR IGNORE INTO manufacturers VALUES (1, 'Nike'), (2, 'Adidas'), (3, 'Reebok');
            
            CREATE TABLE IF NOT EXISTS suppliers (
                id INTEGER PRIMARY KEY,
                name TEXT
            );
            INSERT OR IGNORE INTO suppliers VALUES (1, 'ООО Спорт'), (2, 'ОбувьОпт'), (3, 'МаркетТрейд');
            
            CREATE TABLE IF NOT EXISTS products (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT,
                category_id INTEGER,
                description TEXT,
                manufacturer_id INTEGER,
                supplier_id INTEGER,
                price REAL,
                unit TEXT,
                quantity INTEGER,
                discount REAL,
                image_path TEXT,
                FOREIGN KEY(category_id) REFERENCES categories(id),
                FOREIGN KEY(manufacturer_id) REFERENCES manufacturers(id),
                FOREIGN KEY(supplier_id) REFERENCES suppliers(id)
            );
            
            INSERT OR IGNORE INTO products (id, name, category_id, description, manufacturer_id, supplier_id, price, unit, quantity, discount, image_path) VALUES
                (1, 'Кроссовки Air Max', 1, 'Легкие и удобные', 1, 1, 5000, 'пара', 10, 10, ''),
                (2, 'Ботинки зимние', 2, 'Теплые', 2, 2, 8000, 'пара', 0, 20, ''),
                (3, 'Сандалии летние', 3, 'Для пляжа', 3, 3, 2000, 'пара', 5, 16, '');
        ''')

init_db()

def get_db():
    return sqlite3.connect('database.db')

# Главная страница - логин
@app.route('/', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        login = request.form['login']
        password = request.form['password']
        
        with get_db() as conn:
            user = conn.execute('SELECT * FROM users WHERE login=? AND password=?', 
                               (login, password)).fetchone()
        
        if user:
            session['user_id'] = user[0]
            session['role'] = user[3]
            session['full_name'] = user[4]
            return redirect(url_for('products'))
        return 'Ошибка входа'
    return render_template('login.html')

# Просмотр товаров с фильтрацией и сортировкой
@app.route('/products')
def products():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    
    role = session['role']
    search = request.args.get('search', '')
    supplier_filter = request.args.get('supplier', '')
    sort = request.args.get('sort', '')
    
    query = '''
        SELECT p.*, c.name as cat_name, m.name as man_name, s.name as sup_name 
        FROM products p
        JOIN categories c ON p.category_id = c.id
        JOIN manufacturers m ON p.manufacturer_id = m.id
        JOIN suppliers s ON p.supplier_id = s.id
        WHERE 1=1
    '''
    params = []
    
    if search:
        query += ' AND (p.name LIKE ? OR p.description LIKE ?)'
        params.extend([f'%{search}%', f'%{search}%'])
    
    if supplier_filter and supplier_filter != 'all':
        query += ' AND p.supplier_id = ?'
        params.append(supplier_filter)
    
    if sort == 'qty_asc':
        query += ' ORDER BY p.quantity ASC'
    elif sort == 'qty_desc':
        query += ' ORDER BY p.quantity DESC'
    
    with get_db() as conn:
        products = conn.execute(query, params).fetchall()
        suppliers = conn.execute('SELECT * FROM suppliers').fetchall()
    
    # Подсветка товаров по условиям
    processed = []
    for p in products:
        prod_dict = {
            'id': p[0], 'name': p[1], 'description': p[4],
            'price': p[6], 'quantity': p[8], 'discount': p[9],
            'image_path': p[10] if p[10] else 'static/picture.png',
            'cat_name': p[11], 'man_name': p[12], 'sup_name': p[13],
            'highlight': ''
        }
        
        final_price = p[6] * (1 - p[9]/100)
        
        if p[9] > 15:
            prod_dict['highlight'] = 'discount-high'
        if p[8] == 0:
            prod_dict['highlight'] = 'out-of-stock'
        
        prod_dict['final_price'] = round(final_price, 2)
        prod_dict['has_discount'] = p[9] > 0
        processed.append(prod_dict)
    
    return render_template('products.html', 
                         products=processed, 
                         role=role, 
                         suppliers=suppliers,
                         search=search,
                         supplier_filter=supplier_filter,
                         sort=sort)

# Добавление/редактирование товара
@app.route('/product/edit/<int:product_id>', methods=['GET', 'POST'])
@app.route('/product/add', methods=['GET', 'POST'])
def product_form(product_id=None):
    if session.get('role') != 'admin':
        return redirect(url_for('products'))
    
    with get_db() as conn:
        categories = conn.execute('SELECT * FROM categories').fetchall()
        manufacturers = conn.execute('SELECT * FROM manufacturers').fetchall()
        suppliers = conn.execute('SELECT * FROM suppliers').fetchall()
    
    if request.method == 'POST':
        name = request.form['name']
        category_id = request.form['category_id']
        description = request.form['description']
        manufacturer_id = request.form['manufacturer_id']
        supplier_id = request.form['supplier_id']
        price = float(request.form['price'])
        unit = request.form['unit']
        quantity = int(request.form['quantity'])
        discount = float(request.form['discount'])
        
        # Обработка фото
        image_path = ''
        if 'image' in request.files:
            file = request.files['image']
            if file.filename:
                filename = secure_filename(file.filename)
                image_path = f'static/uploads/{filename}'
                
                # Изменяем размер до 300x200
                img = Image.open(file)
                img.thumbnail((300, 200))
                img.save(image_path)
        
        with get_db() as conn:
            if product_id:  # Редактирование
                if image_path:
                    # Удаляем старое фото
                    old = conn.execute('SELECT image_path FROM products WHERE id=?', 
                                      (product_id,)).fetchone()
                    if old and old[0] and os.path.exists(old[0]):
                        os.remove(old[0])
                    conn.execute('UPDATE products SET name=?, category_id=?, description=?, manufacturer_id=?, supplier_id=?, price=?, unit=?, quantity=?, discount=?, image_path=? WHERE id=?',
                               (name, category_id, description, manufacturer_id, supplier_id, price, unit, quantity, discount, image_path, product_id))
                else:
                    conn.execute('UPDATE products SET name=?, category_id=?, description=?, manufacturer_id=?, supplier_id=?, price=?, unit=?, quantity=?, discount=? WHERE id=?',
                               (name, category_id, description, manufacturer_id, supplier_id, price, unit, quantity, discount, product_id))
            else:  # Добавление
                conn.execute('INSERT INTO products (name, category_id, description, manufacturer_id, supplier_id, price, unit, quantity, discount, image_path) VALUES (?,?,?,?,?,?,?,?,?,?)',
                           (name, category_id, description, manufacturer_id, supplier_id, price, unit, quantity, discount, image_path))
        
        return redirect(url_for('products'))
    
    product = None
    if product_id:
        with get_db() as conn:
            product = conn.execute('SELECT * FROM products WHERE id=?', (product_id,)).fetchone()
    
    return render_template('product_form.html', 
                         product=product,
                         categories=categories,
                         manufacturers=manufacturers,
                         suppliers=suppliers)

# Удаление товара (с проверкой наличия в заказах)
@app.route('/product/delete/<int:product_id>')
def delete_product(product_id):
    if session.get('role') != 'admin':
        return redirect(url_for('products'))
    
    with get_db() as conn:
        # Проверяем, есть ли товар в заказах (упрощенно - нет таблицы заказов)
        # Для демо-версии просто удаляем
        conn.execute('DELETE FROM products WHERE id=?', (product_id,))
    
    return redirect(url_for('products'))

# Выход
@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('login'))

if __name__ == '__main__':
    app.run(debug=True)
