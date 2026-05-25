from flask import Flask, render_template, request, redirect, url_for, session
import psycopg2
from psycopg2.extras import RealDictCursor

app = Flask(__name__)
app.secret_key = 'exam2026'

# ПОДСТАВЬТЕ СВОИ ДАННЫЕ
DB = {
    'host': 'localhost',
    'database': 'exam_db',
    'user': 'postgres',
    'password': '1234'
}

def get_db():
    return psycopg2.connect(**DB)

# СОЗДАНИЕ БД (запускается 1 раз)
def init():
    conn = get_db()
    cur = conn.cursor()
    cur.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id SERIAL PRIMARY KEY,
            login TEXT UNIQUE, password TEXT, role TEXT, full_name TEXT
        );
        CREATE TABLE IF NOT EXISTS products (
            id SERIAL PRIMARY KEY,
            name TEXT, price REAL, quantity INTEGER, discount REAL, image TEXT
        );
        DELETE FROM users;
        INSERT INTO users (login, password, role, full_name) VALUES 
            ('admin', '1', 'admin', 'Админ'),
            ('manager', '1', 'manager', 'Менеджер'),
            ('client', '1', 'client', 'Клиент');
    ''')
    conn.commit()
    conn.close()
init()

@app.route('/', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        conn = get_db()
        cur = conn.cursor(cursor_factory=RealDictCursor)
        cur.execute('SELECT * FROM users WHERE login=%s AND password=%s', 
                   (request.form['login'], request.form['password']))
        user = cur.fetchone()
        conn.close()
        if user:
            session['user_id'] = user['id']
            session['role'] = user['role']
            session['name'] = user['full_name']
            return redirect('/products')
        return 'Ошибка входа'
    return '''
    <form method="POST">
        <input name="login" placeholder="Логин"><br>
        <input name="password" type="password" placeholder="Пароль"><br>
        <button>Войти</button>
    </form>
    <p>Тестовые: admin/1, manager/1, client/1</p>
    <a href="/products">Войти как гость</a>
    '''

@app.route('/products')
def products():
    role = session.get('role', 'guest')
    
    conn = get_db()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute('SELECT * FROM products')
    items = cur.fetchall()
    conn.close()
    
    # Обработка скидок и подсветка
    for p in items:
        p['final_price'] = round(p['price'] * (1 - p['discount']/100), 2)
        p['old_price'] = p['price']
        p['class'] = ''
        if p['discount'] > 15:
            p['class'] = 'discount-high'
        if p['quantity'] == 0:
            p['class'] = 'out-of-stock'
    
    html = f'''
    <style>
        .discount-high {{ background: #2E8B57; color: white; }}
        .out-of-stock {{ background: #add8e6; }}
        .old-price {{ text-decoration: line-through; color: red; }}
        table {{ border-collapse: collapse; width: 100%; }}
        th, td {{ border: 1px solid #ddd; padding: 8px; text-align: left; }}
        .btn {{ padding: 5px 10px; text-decoration: none; border-radius: 3px; }}
        .edit {{ background: orange; color: black; }}
        .delete {{ background: red; color: white; }}
        .add {{ background: green; color: white; display: inline-block; margin: 10px 0; }}
        .logout {{ float: right; }}
    </style>
    <div>
        <strong>{session.get('name', 'Гость')} ({role})</strong>
        <a href="/logout" class="logout">Выйти</a>
    </div>
    <h2>Товары</h2>
    '''
    
    if role == 'admin':
        html += '<a href="/add" class="add">+ Добавить товар</a>'
    
    html += '<table><tr><th>Название</th><th>Цена</th><th>Кол-во</th><th>Скидка</th>'
    if role in ['admin', 'manager']:
        html += '<th>Действия</th>'
    html += '</tr>'
    
    for p in items:
        price_html = f'<span class="old-price">{p["old_price"]}₽</span> {p["final_price"]}₽' if p['discount'] > 0 else f'{p["price"]}₽'
        html += f'<tr class="{p["class"]}"><td>{p["name"]}</td><td>{price_html}</td><td>{p["quantity"]}</td><td>{p["discount"]}%</td>'
        
        if role in ['admin', 'manager']:
            actions = ''
            if role == 'admin':
                actions = f'<a href="/edit/{p["id"]}" class="btn edit">Ред</a> <a href="/delete/{p["id"]}" class="btn delete" onclick="return confirm(\'Удалить?\')">Уд</a>'
            html += f'<td>{actions}</td>'
        html += '</tr>'
    
    html += '</table>'
    
    # Поиск/фильтр (для менеджера и админа)
    if role in ['admin', 'manager']:
        html += '''
        <script>
            function filter() {
                let search = document.getElementById('search').value;
                window.location.href = '/products?search=' + encodeURIComponent(search);
            }
        </script>
        <input id="search" placeholder="Поиск..." onkeyup="filter()">
        '''
    
    return html

@app.route('/add', methods=['GET', 'POST'])
def add():
    if session.get('role') != 'admin':
        return redirect('/products')
    
    if request.method == 'POST':
        conn = get_db()
        cur = conn.cursor()
        cur.execute('INSERT INTO products (name, price, quantity, discount) VALUES (%s,%s,%s,%s)',
                   (request.form['name'], float(request.form['price']), 
                    int(request.form['quantity']), float(request.form['discount'])))
        conn.commit()
        conn.close()
        return redirect('/products')
    
    return '''
    <form method="POST">
        <input name="name" placeholder="Название" required><br>
        <input name="price" type="number" step="0.01" placeholder="Цена" required><br>
        <input name="quantity" type="number" placeholder="Количество" required><br>
        <input name="discount" type="number" step="0.1" placeholder="Скидка %"><br>
        <button>Сохранить</button>
    </form>
    <a href="/products">Назад</a>
    '''

@app.route('/edit/<int:id>', methods=['GET', 'POST'])
def edit(id):
    if session.get('role') != 'admin':
        return redirect('/products')
    
    conn = get_db()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    
    if request.method == 'POST':
        cur.execute('UPDATE products SET name=%s, price=%s, quantity=%s, discount=%s WHERE id=%s',
                   (request.form['name'], float(request.form['price']),
                    int(request.form['quantity']), float(request.form['discount']), id))
        conn.commit()
        conn.close()
        return redirect('/products')
    
    cur.execute('SELECT * FROM products WHERE id=%s', (id,))
    p = cur.fetchone()
    conn.close()
    
    return f'''
    <form method="POST">
        <input name="name" value="{p["name"]}" required><br>
        <input name="price" type="number" step="0.01" value="{p["price"]}" required><br>
        <input name="quantity" type="number" value="{p["quantity"]}" required><br>
        <input name="discount" type="number" step="0.1" value="{p["discount"]}"><br>
        <button>Сохранить</button>
    </form>
    <a href="/products">Назад</a>
    '''

@app.route('/delete/<int:id>')
def delete(id):
    if session.get('role') != 'admin':
        return redirect('/products')
    
    conn = get_db()
    cur = conn.cursor()
    cur.execute('DELETE FROM products WHERE id=%s', (id,))
    conn.commit()
    conn.close()
    return redirect('/products')

@app.route('/logout')
def logout():
    session.clear()
    return redirect('/')

if __name__ == '__main__':
    app.run(debug=True)
