from flask import Flask, render_template_string, request, redirect, session
import psycopg2  # <-- ИСПРАВЛЕНО: psycopg2, а не pyscopg2
import random, string

app = Flask(__name__)
app.secret_key = 'secret'

DB = {
    'dbname': 'shop',
    'user': 'postgres',
    'password': '12345',
    'host': 'localhost',
    'port': 5432
}

def db(): 
    return psycopg2.connect(**DB)  # <-- ИСПРАВЛЕНО: psycopg2 и **DB

HTML = '''
<!DOCTYPE html><html><head><title>Магазин</title>
<style>
body{font-family:Arial;max-width:800px;margin:0 auto;padding:20px}
.product{border:1px solid #ddd;padding:10px;margin:10px 0;border-radius:5px}
.product img{width:200px;height:150px;object-fit:cover}
.btn{padding:8px 15px;border:none;border-radius:3px;cursor:pointer;color:white;text-decoration:none}
.btn-add{background:#007bff}.btn-del{background:#dc3545}.btn-pay{background:#28a745}.btn-order{background:#ffc107;color:black}
.cart-item{background:#f9f9f9;padding:10px;margin:5px 0;border-radius:5px;display:flex;justify-content:space-between}
</style></head><body>
<h1>🛒 Интернет-магазин</h1>
{% if paid %}<div style="background:#d4edda;padding:15px;border-radius:5px"><h2>✅ Заказ #{{order_id}} оплачен!</h2><p>Чек отправлен на email</p><a href="/">В магазин</a></div>
{% elif pay_page %}
<h2>Оплата заказа #{{order_id}}</h2><p>Сумма: {{total}} ₽</p>
<form method="POST" action="/pay">
<input type="hidden" name="order_id" value="{{order_id}}">
<input name="card" placeholder="Номер карты" required style="padding:10px;width:100%;margin:10px 0">
<input name="cvv" placeholder="CVV" required style="padding:10px;width:100px;margin:10px 0">
<button class="btn btn-pay">Оплатить {{total}} ₽</button>
</form>
{% else %}
<h2>Товары</h2>
{% for p in products %}<div class="product">
<img src="{{p[4]}}" alt="{{p[1]}}" onerror="this.src='https://via.placeholder.com/200x150'">
<h3>{{p[1]}}</h3><p>Цена: {{p[2]}} ₽ | В наличии: {{p[3]}} шт.</p>
{% if p[3]>0 %}<form method="POST" action="/add"><input type="hidden" name="id" value="{{p[0]}}">
<input name="qty" value="1" min="1" max="{{p[3]}}" style="width:50px">
<button class="btn btn-add">В корзину</button></form>{% else %}<p style="color:red">Нет в наличии</p>{% endif %}
</div>{% endfor %}

{% if cart %}<h2>Корзина</h2>
{% for item in cart %}<div class="cart-item">
<span>{{item[1]}} x{{item[2]}} = {{item[2]*item[3]}} ₽</span>
<a href="/del/{{loop.index0}}" class="btn btn-del">Удалить</a>
</div>{% endfor %}
<h3>Итого: {{total}} ₽</h3>
<form method="POST" action="/order">
<input name="name" placeholder="Ваше имя" required style="padding:10px;margin:10px 0">
<button class="btn btn-order">Оформить заказ</button></form>{% endif %}
{% endif %}</body></html>'''

@app.route('/')
def index():
    conn = db()
    cur = conn.cursor()
    cur.execute("SELECT * FROM products WHERE quantity>0")
    products = cur.fetchall()
    cur.close()
    conn.close()
    cart = session.get('cart', [])
    total = sum(i[2]*i[3] for i in cart)
    return render_template_string(HTML, products=products, cart=cart, total=total, pay_page=False, paid=False)

@app.route('/add', methods=['POST'])
def add():
    pid = int(request.form['id'])
    qty = int(request.form['qty'])
    conn = db()
    cur = conn.cursor()
    cur.execute("SELECT id, name, price FROM products WHERE id=%s", (pid,))
    p = cur.fetchone()
    cur.close()
    conn.close()
    if p:
        cart = session.get('cart', [])
        for i in cart:
            if i[0]==p[0]:
                i[2]+=qty
                session['cart']=cart
                return redirect('/')
        cart.append([p[0], p[1], qty, p[2]])
        session['cart'] = cart
    return redirect('/')

@app.route('/del/<int:idx>')
def delete(idx):
    cart = session.get('cart', [])
    if idx < len(cart):
        cart.pop(idx)
        session['cart']=cart
    return redirect('/')

@app.route('/order', methods=['POST'])
def order():
    cart = session.get('cart', [])
    if not cart:
        return redirect('/')
    total = sum(i[2]*i[3] for i in cart)
    name = request.form['name']
    conn = db()
    cur = conn.cursor()
    cur.execute("INSERT INTO orders (customer_name, total) VALUES (%s,%s) RETURNING id", (name, total))
    oid = cur.fetchone()[0]
    for i in cart:
        cur.execute("UPDATE products SET quantity=quantity-%s WHERE id=%s", (i[2], i[0]))
    conn.commit()
    cur.close()
    conn.close()
    session['cart'] = []
    return redirect(f'/pay/{oid}')

@app.route('/pay/<int:oid>')
def pay_page(oid):
    conn = db()
    cur = conn.cursor()
    cur.execute("SELECT * FROM orders WHERE id=%s", (oid,))
    order = cur.fetchone()
    cur.close()
    conn.close()
    if order:
        return render_template_string(HTML, pay_page=True, order_id=oid, total=order[2], paid=False)
    return redirect('/')

@app.route('/pay', methods=['POST'])
def pay():
    oid = request.form['order_id']
    conn = db()
    cur = conn.cursor()
    cur.execute("UPDATE orders SET paid=TRUE WHERE id=%s", (oid,))
    conn.commit()
    cur.execute("SELECT total FROM orders WHERE id=%s", (oid,))
    total = cur.fetchone()[0]
    cur.close()
    conn.close()
    return render_template_string(HTML, paid=True, order_id=oid, total=total)

if __name__ == '__main__':
    app.run(debug=True)
