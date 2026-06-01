import tkinter as tk
from tkinter import ttk, messagebox
import mysql.connector
from werkzeug.security import generate_password_hash, check_password_hash
import configparser
import os

class ShopApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Магазин")
        self.root.geometry("1200x700")
        
        # Загрузка конфигурации
        self.config = self.load_config()
        
        # Подключение к MySQL
        try:
            self.conn = mysql.connector.connect(
                host=self.config.get('DATABASE', 'host', fallback='localhost'),
                database=self.config.get('DATABASE', 'database', fallback='shop'),
                user=self.config.get('DATABASE', 'user', fallback='root'),
                password=self.config.get('DATABASE', 'password', fallback=''),
                charset='utf8mb4',
                autocommit=False
            )
            print("Подключение к MySQL успешно установлено")
        except mysql.connector.Error as err:
            messagebox.showerror("Ошибка", f"Ошибка подключения к базе данных: {err}")
            root.destroy()
            return
            
        self.current_user = self.current_role = self.current_user_name = None
        self.init_db()
        self.setup_ui()
        self.show_auth()

    def load_config(self):
        """Загрузка конфигурации из файла"""
        config = configparser.ConfigParser()
        config_file = 'config.ini'
        
        if os.path.exists(config_file):
            config.read(config_file)
        else:
            # Создаем файл конфигурации по умолчанию
            config['DATABASE'] = {
                'host': 'localhost',
                'database': 'shop',
                'user': 'root',
                'password': ''
            }
            with open(config_file, 'w') as f:
                config.write(f)
            print(f"Создан файл конфигурации {config_file}")
        
        return config

    def init_db(self):
        """Инициализация базы данных"""
        c = self.conn.cursor()
        
        try:
            # Проверка существования таблиц
            c.execute("SELECT EXISTS(SELECT 1 FROM information_schema.tables WHERE table_name='u' AND table_schema=%s)", 
                     (self.config.get('DATABASE', 'database', fallback='shop'),))
            
            if not c.fetchone()[0]:
                print("Создание таблиц...")
                
                # Создание таблиц
                create_queries = [
                    """CREATE TABLE u (
                        id INT AUTO_INCREMENT PRIMARY KEY,
                        n VARCHAR(100),
                        e VARCHAR(100) UNIQUE,
                        p TEXT,
                        ph VARCHAR(20) DEFAULT '',
                        adr TEXT DEFAULT '',
                        r VARCHAR(20) DEFAULT 'client'
                    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4""",
                    
                    """CREATE TABLE p (
                        id INT AUTO_INCREMENT PRIMARY KEY,
                        n TEXT,
                        pr DECIMAL(10,2),
                        old_pr DECIMAL(10,2) DEFAULT 0,
                        d TEXT,
                        cat VARCHAR(50) DEFAULT 'other',
                        st INT DEFAULT 0,
                        manufacturer VARCHAR(100) DEFAULT '',
                        supplier VARCHAR(100) DEFAULT '',
                        unit VARCHAR(20) DEFAULT 'шт.'
                    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4""",
                    
                    """CREATE TABLE cr (
                        id INT AUTO_INCREMENT PRIMARY KEY,
                        pid INT,
                        q INT,
                        uid INT,
                        FOREIGN KEY (pid) REFERENCES p(id) ON DELETE CASCADE,
                        FOREIGN KEY (uid) REFERENCES u(id) ON DELETE CASCADE
                    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4""",
                    
                    """CREATE TABLE o (
                        id INT AUTO_INCREMENT PRIMARY KEY,
                        uid INT,
                        dt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                        adr TEXT,
                        tot DECIMAL(10,2),
                        st TEXT DEFAULT 'Ожидает',
                        pay TEXT DEFAULT 'Наличные',
                        FOREIGN KEY (uid) REFERENCES u(id) ON DELETE CASCADE
                    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4""",
                    
                    """CREATE TABLE oi (
                        id INT AUTO_INCREMENT PRIMARY KEY,
                        oid INT,
                        pid INT,
                        q INT,
                        pr DECIMAL(10,2),
                        FOREIGN KEY (oid) REFERENCES o(id) ON DELETE CASCADE,
                        FOREIGN KEY (pid) REFERENCES p(id) ON DELETE CASCADE
                    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4"""
                ]
                
                for query in create_queries:
                    c.execute(query)
                
                # Добавление начальных данных
                products_data = [
                    ('Ноутбук', 49999, 59999, 'Мощный ноутбук', 'Электроника', 50, 'HP', 'ООО Техно', 'шт.'),
                    ('Телефон', 89999, 99999, 'Флагманский телефон', 'Электроника', 100, 'Samsung', 'ООО Мобайл', 'шт.'),
                    ('Наушники', 11999, 14999, 'Беспроводные наушники', 'Электроника', 200, 'Sony', 'ООО Аудио', 'шт.'),
                    ('Планшет', 44999, 49999, 'Легкий планшет', 'Электроника', 75, 'Apple', 'ООО Техно', 'шт.'),
                    ('Часы', 23999, 29999, 'Умные часы', 'Гаджеты', 150, 'Xiaomi', 'ООО Гаджеты', 'шт.'),
                    ('Колонка', 7999, 9999, 'BT колонка', 'Аудио', 0, 'JBL', 'ООО Аудио', 'шт.')
                ]
                
                insert_product = """INSERT INTO p(n,pr,old_pr,d,cat,st,manufacturer,supplier,unit) 
                                   VALUES(%s,%s,%s,%s,%s,%s,%s,%s,%s)"""
                for data in products_data:
                    c.execute(insert_product, data)
                
                # Добавление пользователей
                users_data = [
                    ('Админ', 'admin@shop.ru', generate_password_hash('admin123'), 'admin'),
                    ('Менеджер', 'manager@shop.ru', generate_password_hash('manager123'), 'manager'),
                    ('Клиент', 'client@shop.ru', generate_password_hash('client123'), 'client')
                ]
                
                insert_user = "INSERT INTO u(n,e,p,r) VALUES(%s,%s,%s,%s)"
                for data in users_data:
                    c.execute(insert_user, data)
                
                self.conn.commit()
                print("База данных инициализирована")
            else:
                # Проверка и добавление колонок если их нет
                columns_to_check = [
                    ("manufacturer", "VARCHAR(100) DEFAULT ''"),
                    ("supplier", "VARCHAR(100) DEFAULT ''"),
                    ("unit", "VARCHAR(20) DEFAULT 'шт.'")
                ]
                
                for col, col_type in columns_to_check:
                    try:
                        c.execute(f"ALTER TABLE p ADD COLUMN {col} {col_type}")
                        self.conn.commit()
                    except mysql.connector.Error:
                        self.conn.rollback()
                
        except mysql.connector.Error as err:
            print(f"Ошибка при инициализации БД: {err}")
            self.conn.rollback()
        finally:
            c.close()

    def setup_ui(self):
        """Настройка пользовательского интерфейса"""
        self.nav_frame = tk.Frame(self.root, bg='#2c3e50', height=50)
        self.nav_frame.pack(fill=tk.X)
        
        tk.Label(self.nav_frame, text="Магазин", bg='#2c3e50', fg='white', 
                font=('Arial', 16, 'bold')).pack(side=tk.LEFT, padx=10)
        
        self.user_info = tk.Frame(self.nav_frame, bg='#2c3e50')
        self.user_info.pack(side=tk.LEFT, padx=20)
        
        self.user_name_lbl = tk.Label(self.user_info, text="", bg='#2c3e50', 
                                     fg='#f39c12', font=('Arial', 12, 'bold'))
        self.user_name_lbl.pack(side=tk.LEFT, padx=5)
        
        self.user_role_lbl = tk.Label(self.user_info, text="", bg='#2c3e50', 
                                     fg='#95a5a6', font=('Arial', 10))
        self.user_role_lbl.pack(side=tk.LEFT, padx=5)
        
        self.nav_btns = tk.Frame(self.nav_frame, bg='#2c3e50')
        self.nav_btns.pack(side=tk.RIGHT, padx=10)
        
        self.content = tk.Frame(self.root, bg='#f5f5f5')
        self.content.pack(fill=tk.BOTH, expand=True)

    def update_user_info(self):
        """Обновление информации о пользователе"""
        self.user_name_lbl.config(text=self.current_user_name or "Гость")
        roles = {'admin': 'Администратор', 'manager': 'Менеджер', 'client': 'Клиент'}
        self.user_role_lbl.config(
            text=f"({roles.get(self.current_role, 'Клиент')})" if self.current_user else "(Не авторизован)"
        )

    def clear_content(self):
        """Очистка контента"""
        for w in self.content.winfo_children():
            w.destroy()

    def update_nav_buttons(self, buttons):
        """Обновление навигационных кнопок"""
        for w in self.nav_btns.winfo_children():
            w.destroy()
        for text, command in buttons:
            tk.Button(self.nav_btns, text=text, command=command, 
                     bg='#34495e', fg='white', bd=0, padx=10, pady=5, 
                     font=('Arial', 10)).pack(side=tk.LEFT, padx=2)

    def disc(self, pr, op):
        """Расчет скидки"""
        return (op-pr)/op*100 if op>0 and op>pr else 0

    def show_auth(self):
        """Отображение страницы авторизации"""
        self.clear_content()
        self.current_user = self.current_role = self.current_user_name = None
        self.update_user_info()
        
        for w in self.nav_btns.winfo_children():
            w.destroy()
        
        f = tk.Frame(self.content, bg='#f5f5f5')
        f.pack(expand=True)
        
        nb = ttk.Notebook(f)
        nb.pack(pady=20)
        
        # Вкладка входа
        lf = tk.Frame(nb, bg='white')
        nb.add(lf, text='Вход')
        
        tk.Label(lf, text="Email:").pack(pady=5)
        le = tk.Entry(lf, width=30)
        le.pack(pady=5)
        
        tk.Label(lf, text="Пароль:").pack(pady=5)
        lp = tk.Entry(lf, width=30, show='*')
        lp.pack(pady=5)
        
        tk.Button(lf, text="Войти", bg='#3498db', fg='white', 
                 command=lambda: self.login(le.get(), lp.get())).pack(pady=10)
        
        # Вкладка регистрации
        rf = tk.Frame(nb, bg='white')
        nb.add(rf, text='Регистрация')
        
        tk.Label(rf, text="Имя:").pack(pady=5)
        rn = tk.Entry(rf, width=30)
        rn.pack(pady=5)
        
        tk.Label(rf, text="Email:").pack(pady=5)
        re = tk.Entry(rf, width=30)
        re.pack(pady=5)
        
        tk.Label(rf, text="Пароль:").pack(pady=5)
        rp = tk.Entry(rf, width=30, show='*')
        rp.pack(pady=5)
        
        tk.Button(rf, text="Регистрация", bg='#27ae60', fg='white', 
                 command=lambda: self.register(rn.get(), re.get(), rp.get())).pack(pady=10)
        
        tk.Button(f, text="Продолжить как гость", bg='#95a5a6', fg='white', 
                 command=self.show_catalog).pack(pady=10)

    def login(self, e, p):
        """Авторизация пользователя"""
        if not e or not p:
            messagebox.showerror("Ошибка", "Заполните все поля")
            return
        
        try:
            c = self.conn.cursor()
            c.execute("SELECT id, p, r, n FROM u WHERE e=%s", (e,))
            u = c.fetchone()
            c.close()
            
            if u and check_password_hash(u[1], p):
                self.current_user, self.current_role, self.current_user_name = u[0], u[2], u[3]
                self.update_user_info()
                self.show_catalog()
            else:
                messagebox.showerror("Ошибка", "Неверный email или пароль")
        except mysql.connector.Error:
            self.conn.rollback()

    def register(self, n, e, p):
        """Регистрация нового пользователя"""
        if not e or not p:
            messagebox.showerror("Ошибка", "Заполните поля")
            return
        
        if not n:
            n = e.split('@')[0]
        
        try:
            c = self.conn.cursor()
            c.execute("SELECT id FROM u WHERE e=%s", (e,))
            if c.fetchone():
                messagebox.showerror("Ошибка", "Email занят")
                c.close()
                return
            
            c.execute("INSERT INTO u(n,e,p,r) VALUES(%s,%s,%s,'client')", 
                     (n, e, generate_password_hash(p)))
            self.conn.commit()
            
            c.execute("SELECT id, r, n FROM u WHERE e=%s", (e,))
            u = c.fetchone()
            c.close()
            
            self.current_user, self.current_role, self.current_user_name = u[0], u[1], u[2]
            self.update_user_info()
            self.show_catalog()
        except mysql.connector.Error:
            self.conn.rollback()

    def show_catalog(self):
        """Отображение каталога товаров"""
        self.clear_content()
        
        b = [("Товары", self.show_catalog)]
        if self.current_role == 'client':
            b += [("Корзина", self.show_cart), ("Заказы", self.show_orders)]
        elif self.current_role in ['manager', 'admin']:
            b += [("Заказы", self.show_orders)]
        
        if self.current_user:
            b += [("Профиль", self.show_profile), ("Выход", self.logout)]
        else:
            b += [("Войти", self.show_auth)]
        
        self.update_nav_buttons(b)
        
        tf = tk.Frame(self.content, bg='#f5f5f5')
        tf.pack(fill=tk.X, pady=10)
        
        tk.Label(tf, text="Каталог товаров", font=('Arial', 20, 'bold'), 
                bg='#f5f5f5').pack(side=tk.LEFT, padx=10)
        
        if self.current_role == 'admin':
            tk.Button(tf, text="+ Добавить", bg='#3498db', fg='white', 
                     command=self.show_add).pack(side=tk.RIGHT, padx=10)
        
        if self.current_role in ['manager', 'admin']:
            ff = tk.Frame(self.content, bg='white', bd=1, relief=tk.SOLID)
            ff.pack(fill=tk.X, padx=10, pady=5)
            
            tk.Label(ff, text="Поиск:", bg='white').pack(side=tk.LEFT, padx=5)
            se = tk.Entry(ff, width=20)
            se.pack(side=tk.LEFT, padx=5)
            
            tk.Label(ff, text="Мин:", bg='white').pack(side=tk.LEFT, padx=5)
            mn = tk.Entry(ff, width=10)
            mn.pack(side=tk.LEFT, padx=5)
            
            tk.Label(ff, text="Макс:", bg='white').pack(side=tk.LEFT, padx=5)
            mx = tk.Entry(ff, width=10)
            mx.pack(side=tk.LEFT, padx=5)
            
            tk.Button(ff, text="Фильтр", bg='#3498db', fg='white', 
                     command=lambda: self.load_products(se.get(), mn.get(), mx.get())).pack(side=tk.LEFT, padx=10)
        
        self.pf = tk.Frame(self.content, bg='#f5f5f5')
        self.pf.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        self.load_products()

    def load_products(self, s='', mn='', mx=''):
        """Загрузка и отображение товаров"""
        for w in self.pf.winfo_children():
            w.destroy()
        
        try:
            q = "SELECT id, n, pr, old_pr, d, cat, st, manufacturer, supplier, unit FROM p WHERE 1=1"
            pr = []
            
            if self.current_role in ['manager', 'admin']:
                if s:
                    q += " AND (LOWER(n) LIKE %s OR LOWER(d) LIKE %s OR LOWER(manufacturer) LIKE %s OR LOWER(supplier) LIKE %s)"
                    pr.extend([f'%{s.lower()}%']*4)
                if mn:
                    try:
                        q += " AND pr>=%s"
                        pr.append(float(mn))
                    except ValueError:
                        pass
                if mx:
                    try:
                        q += " AND pr<=%s"
                        pr.append(float(mx))
                    except ValueError:
                        pass
            
            q += " ORDER BY n ASC"
            
            c = self.conn.cursor()
            c.execute(q, pr)
            ps = c.fetchall()
            c.close()
            
            if not ps:
                tk.Label(self.pf, text="Нет товаров", font=('Arial', 14), 
                        bg='#f5f5f5').pack(expand=True)
                return
            
            cv = tk.Canvas(self.pf, bg='#f5f5f5')
            sb = tk.Scrollbar(self.pf, orient="vertical", command=cv.yview)
            sf = tk.Frame(cv, bg='#f5f5f5')
            
            sf.bind("<Configure>", lambda e: cv.configure(scrollregion=cv.bbox("all")))
            cv.create_window((0,0), window=sf, anchor="nw")
            cv.configure(yscrollcommand=sb.set)
            
            r, cl = 0, 0
            for p in ps:
                pid, n, prc, op, d, cat, st, manufacturer, supplier, unit = p
                disc = self.disc(prc, op)
                
                bg = '#87CEEB' if st == 0 else '#2E8B57' if disc > 15 else 'white'
                
                cd = tk.Frame(sf, bg=bg, bd=2, relief=tk.RAISED)
                cd.grid(row=r, column=cl, padx=10, pady=10, sticky="nsew")
                
                if disc > 0:
                    color = '#e74c3c' if disc > 15 else '#f39c12'
                    tk.Label(tk.Frame(cd, bg=color), 
                           text=f"Скидка {disc:.0f}%", font=('Arial', 10, 'bold'), 
                           fg='white', bg=color).pack(fill=tk.X, pady=2)
                
                if st == 0:
                    tk.Label(tk.Frame(cd, bg='#4682B4'), 
                           text="Нет в наличии", font=('Arial', 10, 'bold'), 
                           fg='white', bg='#4682B4').pack(fill=tk.X, pady=2)
                
                tk.Label(cd, text=n, font=('Arial', 14, 'bold'), bg=bg).pack(pady=5)
                
                if self.current_role in ['manager', 'admin']:
                    tk.Label(cd, text=f"{cat} | {st} {unit}", 
                           bg='#f39c12', fg='white', font=('Arial', 9)).pack(pady=2)
                    tk.Label(cd, text=f"Производитель: {manufacturer}", 
                           bg=bg, font=('Arial', 9)).pack(pady=2)
                    tk.Label(cd, text=f"Поставщик: {supplier}", 
                           bg=bg, font=('Arial', 9)).pack(pady=2)
                
                pf = tk.Frame(cd, bg=bg)
                pf.pack(pady=5)
                
                if disc > 0 and op > 0:
                    tk.Label(pf, text=f"{int(op):,} руб", 
                           font=('Arial', 12, 'overstrike'), fg='red', bg=bg).pack()
                    tk.Label(pf, text=f"{int(prc):,} руб", 
                           font=('Arial', 16, 'bold'), fg='black', bg=bg).pack()
                else:
                    tk.Label(pf, text=f"{int(prc):,} руб", 
                           font=('Arial', 16, 'bold'), fg='black', bg=bg).pack()
                
                tk.Label(cd, text=d, bg=bg, wraplength=200).pack(pady=5)
                
                bf = tk.Frame(cd, bg=bg)
                bf.pack(pady=5)
                
                if self.current_user:
                    if self.current_role == 'client' and st > 0:
                        tk.Button(bf, text="В корзину", bg='#27ae60', fg='white', 
                                 command=lambda pid=pid: self.add_cart(pid)).pack(side=tk.LEFT, padx=2)
                    elif self.current_role == 'client' and st == 0:
                        tk.Label(bf, text="Нет в наличии", bg='#e74c3c', fg='white', 
                                font=('Arial', 9)).pack(side=tk.LEFT, padx=2)
                    
                    if self.current_role == 'admin':
                        tk.Button(bf, text="Изменить", bg='#3498db', fg='white', 
                                 command=lambda pid=pid: self.show_edit(pid)).pack(side=tk.LEFT, padx=2)
                        tk.Button(bf, text="Удалить", bg='#e74c3c', fg='white', 
                                 command=lambda pid=pid: self.del_product(pid)).pack(side=tk.LEFT, padx=2)
                elif st > 0:
                    tk.Button(bf, text="Войти для покупки", bg='#3498db', fg='white', 
                             command=self.show_auth).pack(side=tk.LEFT, padx=2)
                else:
                    tk.Label(bf, text="Нет в наличии", bg='#e74c3c', fg='white', 
                            font=('Arial', 9)).pack(side=tk.LEFT, padx=2)
                
                cl += 1
                if cl >= 3:
                    cl = 0
                    r += 1
            
            cv.pack(side="left", fill="both", expand=True)
            sb.pack(side="right", fill="y")
            
        except mysql.connector.Error as err:
            print(f"Ошибка при загрузке товаров: {err}")
            self.conn.rollback()

    def add_cart(self, pid):
        """Добавление товара в корзину"""
        if not self.current_user:
            messagebox.showwarning("Предупреждение", "Необходимо авторизоваться")
            self.show_auth()
            return
        
        try:
            c = self.conn.cursor()
            c.execute("SELECT st FROM p WHERE id=%s", (pid,))
            result = c.fetchone()
            
            if result[0] <= 0:
                messagebox.showerror("Ошибка", "Товара нет в наличии")
                c.close()
                return
            
            c.execute("SELECT id FROM cr WHERE pid=%s AND uid=%s", (pid, self.current_user))
            if c.fetchone():
                c.execute("UPDATE cr SET q=q+1 WHERE pid=%s AND uid=%s", (pid, self.current_user))
            else:
                c.execute("INSERT INTO cr(pid,q,uid) VALUES(%s,1,%s)", (pid, self.current_user))
            
            self.conn.commit()
            c.close()
            messagebox.showinfo("Успех", "Товар добавлен в корзину")
        except mysql.connector.Error:
            self.conn.rollback()

    def show_cart(self):
        """Отображение корзины"""
        if not self.current_user:
            self.show_auth()
            return
        
        self.clear_content()
        self.update_nav_buttons([
            ("Товары", self.show_catalog), 
            ("Корзина", self.show_cart), 
            ("Заказы", self.show_orders), 
            ("Профиль", self.show_profile), 
            ("Выход", self.logout)
        ])
        
        try:
            c = self.conn.cursor()
            c.execute("""
                SELECT p.n, p.pr, p.old_pr, c.q, p.pr*c.q, c.id 
                FROM cr c 
                JOIN p ON c.pid=p.id 
                WHERE c.uid=%s
            """, (self.current_user,))
            items = c.fetchall()
            c.close()
            
            tk.Label(self.content, text="Корзина", font=('Arial', 20, 'bold'), 
                    bg='#f5f5f5').pack(pady=10)
            
            if not items:
                tk.Label(self.content, text="Корзина пуста", font=('Arial', 14), 
                        bg='#f5f5f5').pack(pady=20)
                return
            
            tf = tk.Frame(self.content, bg='white')
            tf.pack(fill=tk.X, padx=10, pady=5)
            
            headers = ["Товар", "Цена", "Кол-во", "Сумма", ""]
            for i, h in enumerate(headers):
                tk.Label(tf, text=h, font=('Arial', 10, 'bold'), bg='#f8f9fa', 
                        bd=1, relief=tk.SOLID, width=15).grid(row=0, column=i, sticky="nsew")
            
            total = 0
            for i, item in enumerate(items, 1):
                n, pr, op, q, sm, cid = item
                total += sm
                disc = self.disc(pr, op)
                bg = '#2E8B57' if disc > 15 else 'white'
                
                tk.Label(tf, text=n, bg=bg, bd=1, relief=tk.SOLID).grid(
                    row=i, column=0, sticky="nsew")
                
                pt = f"{int(pr):,} руб" + (f"\n(-{disc:.0f}%)" if disc > 0 else "")
                tk.Label(tf, text=pt, bg=bg, bd=1, relief=tk.SOLID).grid(
                    row=i, column=1, sticky="nsew")
                
                tk.Label(tf, text=str(q), bg=bg, bd=1, relief=tk.SOLID).grid(
                    row=i, column=2, sticky="nsew")
                
                tk.Label(tf, text=f"{int(sm):,} руб", bg=bg, bd=1, relief=tk.SOLID).grid(
                    row=i, column=3, sticky="nsew")
                
                tk.Button(tf, text="X", bg='#e74c3c', fg='white', 
                         command=lambda cid=cid: self.del_cart(cid)).grid(
                    row=i, column=4, sticky="nsew")
            
            ttf = tk.Frame(self.content, bg='white', bd=1, relief=tk.SOLID)
            ttf.pack(fill=tk.X, padx=10, pady=10)
            
            tk.Label(ttf, text=f"Итого: {int(total):,} руб", 
                    font=('Arial', 14, 'bold'), bg='white').pack(pady=10)
            
            tk.Button(ttf, text="Оформить заказ", bg='#f39c12', fg='white', 
                     command=self.show_order_form).pack(pady=10)
            
        except mysql.connector.Error:
            self.conn.rollback()

    def del_cart(self, cid):
        """Удаление товара из корзины"""
        if not self.current_user:
            return
        
        try:
            c = self.conn.cursor()
            c.execute("DELETE FROM cr WHERE id=%s AND uid=%s", (cid, self.current_user))
            self.conn.commit()
            c.close()
            self.show_cart()
        except mysql.connector.Error:
            self.conn.rollback()

    def show_order_form(self):
        """Форма оформления заказа"""
        if not self.current_user:
            self.show_auth()
            return
        
        try:
            c = self.conn.cursor()
            c.execute("SELECT COALESCE(SUM(p.pr*c.q),0) FROM cr c JOIN p ON c.pid=p.id WHERE c.uid=%s", 
                     (self.current_user,))
            total = c.fetchone()[0]
            c.close()
            
            if total == 0:
                messagebox.showwarning("Предупреждение", "Корзина пуста")
                return
            
            w = tk.Toplevel(self.root)
            w.title("Оформление заказа")
            w.geometry("400x300")
            
            tk.Label(w, text=f"Сумма: {int(total):,} руб", 
                    font=('Arial', 14, 'bold'), fg='#e74c3c').pack(pady=10)
            
            tk.Label(w, text="Адрес:").pack(pady=5)
            ae = tk.Entry(w, width=40)
            ae.pack(pady=5)
            
            tk.Label(w, text="Оплата:").pack(pady=5)
            pv = tk.StringVar(value="Наличные")
            ttk.Combobox(w, textvariable=pv, values=["Наличные", "Карта"], 
                        state="readonly").pack(pady=5)
            
            tk.Button(w, text="Заказать", bg='#f39c12', fg='white', 
                     command=lambda: self.create_order(ae.get(), pv.get(), w)).pack(pady=20)
            
        except mysql.connector.Error:
            self.conn.rollback()

    def create_order(self, adr, pay, w):
        """Создание заказа"""
        if not self.current_user:
            return
        
        if not adr:
            messagebox.showerror("Ошибка", "Введите адрес")
            return
        
        try:
            c = self.conn.cursor()
            
            # Получаем общую сумму
            c.execute("SELECT COALESCE(SUM(p.pr*c.q),0) FROM cr c JOIN p ON c.pid=p.id WHERE c.uid=%s", 
                     (self.current_user,))
            total = c.fetchone()[0]
            
            # Создаем заказ
            c.execute("INSERT INTO o(uid,adr,tot,pay) VALUES(%s,%s,%s,%s)", 
                     (self.current_user, adr, total, pay))
            oid = c.lastrowid
            
            # Добавляем товары в заказ
            c.execute("""
                INSERT INTO oi(oid,pid,q,pr) 
                SELECT %s,c.pid,c.q,p.pr 
                FROM cr c 
                JOIN p ON c.pid=p.id 
                WHERE c.uid=%s
            """, (oid, self.current_user))
            
            # Очищаем корзину
            c.execute("DELETE FROM cr WHERE uid=%s", (self.current_user,))
            
            self.conn.commit()
            c.close()
            
            w.destroy()
            messagebox.showinfo("Успех", "Заказ оформлен!")
            self.show_orders()
            
        except mysql.connector.Error as err:
            print(f"Ошибка при создании заказа: {err}")
            self.conn.rollback()

    def show_orders(self):
        """Отображение заказов"""
        if not self.current_user:
            self.show_auth()
            return
        
        self.clear_content()
        
        if self.current_role == 'client':
            nav = [("Товары", self.show_catalog), ("Корзина", self.show_cart), 
                  ("Заказы", self.show_orders), ("Профиль", self.show_profile), 
                  ("Выход", self.logout)]
        else:
            nav = [("Товары", self.show_catalog), ("Заказы", self.show_orders), 
                  ("Профиль", self.show_profile), ("Выход", self.logout)]
        
        self.update_nav_buttons(nav)
        
        tk.Label(self.content, text="Заказы", font=('Arial', 20, 'bold'), 
                bg='#f5f5f5').pack(pady=10)
        
        try:
            c = self.conn.cursor()
            
            if self.current_role in ['manager', 'admin']:
                c.execute("""
                    SELECT o.id, o.dt, o.adr, o.tot, o.st, o.pay, u.n, u.e 
                    FROM o 
                    JOIN u ON o.uid=u.id 
                    ORDER BY o.dt DESC 
                    LIMIT 50
                """)
            else:
                c.execute("""
                    SELECT id, dt, adr, tot, st, pay 
                    FROM o 
                    WHERE uid=%s 
                    ORDER BY dt DESC 
                    LIMIT 20
                """, (self.current_user,))
            
            orders = c.fetchall()
            c.close()
            
            if not orders:
                tk.Label(self.content, text="Нет заказов", font=('Arial', 14), 
                        bg='#f5f5f5').pack(pady=20)
                return
            
            cv = tk.Canvas(self.content, bg='#f5f5f5')
            sb = tk.Scrollbar(self.content, orient="vertical", command=cv.yview)
            sf = tk.Frame(cv, bg='#f5f5f5')
            
            sf.bind("<Configure>", lambda e: cv.configure(scrollregion=cv.bbox("all")))
            cv.create_window((0,0), window=sf, anchor="nw")
            cv.configure(yscrollcommand=sb.set)
            
            sc = {
                'Доставлен': '#27ae60',
                'В обработке': '#3498db',
                'Отменен': '#e74c3c'
            }
            
            for o in orders:
                if self.current_role in ['manager', 'admin']:
                    oid, dt, adr, tot, st, pay, un, ue = o
                else:
                    oid, dt, adr, tot, st, pay = o
                    un = ue = None
                
                cd = tk.Frame(sf, bg='white', bd=1, relief=tk.SOLID)
                cd.pack(fill=tk.X, padx=10, pady=5)
                
                tk.Label(cd, text=f"Заказ #{oid}", font=('Arial', 12, 'bold'), 
                        bg='white').pack(anchor=tk.W, padx=10, pady=5)
                
                tk.Label(cd, text=st, bg=sc.get(st, '#f39c12'), fg='white', 
                        font=('Arial', 9)).pack(anchor=tk.W, padx=10)
                
                if dt:
                    tk.Label(cd, text=f"Дата: {dt.strftime('%d.%m.%Y %H:%M')}", 
                            bg='white').pack(anchor=tk.W, padx=10)
                
                if self.current_role in ['manager', 'admin']:
                    tk.Label(cd, text=f"Клиент: {un} ({ue})", 
                            bg='white').pack(anchor=tk.W, padx=10)
                
                tk.Label(cd, text=f"Адрес: {adr}", bg='white').pack(anchor=tk.W, padx=10)
                
                tk.Label(cd, text=f"Сумма: {int(tot):,} руб", 
                        font=('Arial', 10, 'bold'), bg='white').pack(anchor=tk.W, padx=10)
                
                tk.Label(cd, text=f"Оплата: {pay}", bg='white').pack(anchor=tk.W, padx=10)
                
                if self.current_role in ['manager', 'admin']:
                    sv = tk.StringVar(value=st)
                    ttk.Combobox(cd, textvariable=sv, 
                               values=["Ожидает", "Оплачен", "В обработке", 
                                      "Отправлен", "Доставлен", "Отменен"], 
                               state="readonly").pack(anchor=tk.W, padx=10, pady=5)
                    
                    tk.Button(cd, text="Обновить", bg='#3498db', fg='white', 
                             command=lambda oid=oid, sv=sv: self.upd_order(oid, sv.get())
                             ).pack(anchor=tk.W, padx=10, pady=5)
            
            cv.pack(side="left", fill="both", expand=True)
            sb.pack(side="right", fill="y")
            
        except mysql.connector.Error:
            self.conn.rollback()

    def upd_order(self, oid, st):
        """Обновление статуса заказа"""
        try:
            c = self.conn.cursor()
            c.execute("UPDATE o SET st=%s WHERE id=%s", (st, oid))
            self.conn.commit()
            c.close()
            self.show_orders()
        except mysql.connector.Error:
            self.conn.rollback()

    def show_profile(self):
        """Отображение профиля"""
        if not self.current_user:
            self.show_auth()
            return
        
        self.clear_content()
        
        if self.current_role == 'client':
            nav = [("Товары", self.show_catalog), ("Корзина", self.show_cart), 
                  ("Заказы", self.show_orders), ("Профиль", self.show_profile), 
                  ("Выход", self.logout)]
        else:
            nav = [("Товары", self.show_catalog), ("Заказы", self.show_orders), 
                  ("Профиль", self.show_profile), ("Выход", self.logout)]
        
        self.update_nav_buttons(nav)
        
        try:
            c = self.conn.cursor()
            c.execute("SELECT n, e, ph, adr, r FROM u WHERE id=%s", (self.current_user,))
            u = c.fetchone()
            c.close()
            
            if not u:
                self.logout()
                return
            
            tk.Label(self.content, text="Профиль", font=('Arial', 20, 'bold'), 
                    bg='#f5f5f5').pack(pady=10)
            
            ff = tk.Frame(self.content, bg='white', bd=1, relief=tk.SOLID)
            ff.pack(padx=20, pady=10, fill=tk.X)
            
            es = {}
            profile_fields = [
                ("Имя:", u[0] or "", False),
                ("Email:", u[1] or "", True),
                ("Роль:", u[4], True),
                ("Телефон:", u[2] or "", False),
                ("Адрес:", u[3] or "", False)
            ]
            
            for label, value, disabled in profile_fields:
                rf = tk.Frame(ff, bg='white')
                rf.pack(fill=tk.X, padx=10, pady=5)
                
                tk.Label(rf, text=label, bg='white', width=10).pack(side=tk.LEFT)
                
                if disabled:
                    tk.Label(rf, text=value, bg='white').pack(side=tk.LEFT)
                else:
                    e = tk.Entry(rf, width=40)
                    e.insert(0, value)
                    e.pack(side=tk.LEFT)
                    es[label] = e
            
            tk.Button(ff, text="Сохранить", bg='#3498db', fg='white', 
                     command=lambda: self.save_profile(
                         es["Имя:"].get(), 
                         es["Телефон:"].get(), 
                         es["Адрес:"].get()
                     )).pack(pady=10)
            
        except mysql.connector.Error:
            self.conn.rollback()

    def save_profile(self, n, ph, adr):
        """Сохранение профиля"""
        try:
            c = self.conn.cursor()
            c.execute("UPDATE u SET n=%s, ph=%s, adr=%s WHERE id=%s", 
                     (n, ph, adr, self.current_user))
            self.conn.commit()
            c.close()
            
            self.current_user_name = n
            self.update_user_info()
            messagebox.showinfo("Успех", "Профиль обновлен")
        except mysql.connector.Error:
            self.conn.rollback()

    def show_add(self):
        """Форма добавления товара"""
        if not self.current_user or self.current_role != 'admin':
            return
        
        self.clear_content()
        self.update_nav_buttons([
            ("Товары", self.show_catalog), 
            ("Заказы", self.show_orders), 
            ("Профиль", self.show_profile), 
            ("Выход", self.logout)
        ])
        
        tk.Label(self.content, text="Добавить товар", font=('Arial', 20, 'bold'), 
                bg='#f5f5f5').pack(pady=10)
        
        ff = tk.Frame(self.content, bg='white', bd=1, relief=tk.SOLID)
        ff.pack(padx=20, pady=10, fill=tk.X)
        
        es = {}
        fields = [
            ("Название:", ""),
            ("Новая цена:", ""),
            ("Старая цена:", "0"),
            ("Описание:", ""),
            ("Категория:", ""),
            ("На складе:", ""),
            ("Производитель:", ""),
            ("Поставщик:", ""),
            ("Ед. измерения:", "шт.")
        ]
        
        for label, value in fields:
            rf = tk.Frame(ff, bg='white')
            rf.pack(fill=tk.X, padx=10, pady=5)
            
            tk.Label(rf, text=label, bg='white', width=12).pack(side=tk.LEFT)
            e = tk.Entry(rf, width=40)
            e.insert(0, value)
            e.pack(side=tk.LEFT)
            es[label] = e
        
        tk.Button(ff, text="Добавить", bg='#3498db', fg='white', 
                 command=lambda: self.add_product(
                     es["Название:"].get(), 
                     es["Новая цена:"].get(), 
                     es["Старая цена:"].get(), 
                     es["Описание:"].get(), 
                     es["Категория:"].get(), 
                     es["На складе:"].get(), 
                     es["Производитель:"].get(), 
                     es["Поставщик:"].get(), 
                     es["Ед. измерения:"].get()
                 )).pack(pady=10)

    def add_product(self, n, pr, op, d, cat, st, manufacturer, supplier, unit):
        """Добавление товара"""
        if not all([n, pr, d, cat, st]):
            messagebox.showerror("Ошибка", "Заполните все поля")
            return
        
        try:
            pr, op, st = float(pr), float(op), int(st)
        except ValueError:
            messagebox.showerror("Ошибка", "Некорректные числа")
            return
        
        manufacturer = manufacturer or ''
        supplier = supplier or ''
        unit = unit or 'шт.'
        
        try:
            c = self.conn.cursor()
            c.execute("""
                INSERT INTO p(n,pr,old_pr,d,cat,st,manufacturer,supplier,unit) 
                VALUES(%s,%s,%s,%s,%s,%s,%s,%s,%s)
            """, (n, pr, op, d, cat, st, manufacturer, supplier, unit))
            self.conn.commit()
            c.close()
            
            messagebox.showinfo("Успех", "Товар добавлен")
            self.show_catalog()
        except mysql.connector.Error as err:
            print(f"Ошибка при добавлении товара: {err}")
            self.conn.rollback()

    def show_edit(self, pid):
        """Форма редактирования товара"""
        if not self.current_user or self.current_role != 'admin':
            return
        
        try:
            c = self.conn.cursor()
            c.execute("""
                SELECT n, pr, old_pr, d, cat, st, manufacturer, supplier, unit 
                FROM p 
                WHERE id=%s
            """, (pid,))
            p = c.fetchone()
            c.close()
            
            if not p:
                return
            
            self.clear_content()
            self.update_nav_buttons([
                ("Товары", self.show_catalog), 
                ("Заказы", self.show_orders), 
                ("Профиль", self.show_profile), 
                ("Выход", self.logout)
            ])
            
            tk.Label(self.content, text="Изменить товар", font=('Arial', 20, 'bold'), 
                    bg='#f5f5f5').pack(pady=10)
            
            ff = tk.Frame(self.content, bg='white', bd=1, relief=tk.SOLID)
            ff.pack(padx=20, pady=10, fill=tk.X)
            
            es = {}
            fields = [
                ("Название:", p[0]),
                ("Новая цена:", str(p[1])),
                ("Старая цена:", str(p[2])),
                ("Описание:", p[3]),
                ("Категория:", p[4]),
                ("На складе:", str(p[5])),
                ("Производитель:", p[6] or ""),
                ("Поставщик:", p[7] or ""),
                ("Ед. измерения:", p[8] or "шт.")
            ]
            
            for label, value in fields:
                rf = tk.Frame(ff, bg='white')
                rf.pack(fill=tk.X, padx=10, pady=5)
                
                tk.Label(rf, text=label, bg='white', width=12).pack(side=tk.LEFT)
                e = tk.Entry(rf, width=40)
                e.insert(0, value)
                e.pack(side=tk.LEFT)
                es[label] = e
            
            tk.Button(ff, text="Сохранить", bg='#3498db', fg='white', 
                     command=lambda: self.edit_product(
                         pid,
                         es["Название:"].get(), 
                         es["Новая цена:"].get(), 
                         es["Старая цена:"].get(), 
                         es["Описание:"].get(), 
                         es["Категория:"].get(), 
                         es["На складе:"].get(), 
                         es["Производитель:"].get(), 
                         es["Поставщик:"].get(), 
                         es["Ед. измерения:"].get()
                     )).pack(pady=10)
            
        except mysql.connector.Error:
            self.conn.rollback()

    def edit_product(self, pid, n, pr, op, d, cat, st, manufacturer, supplier, unit):
        """Редактирование товара"""
        if not all([n, pr, d, cat, st]):
            messagebox.showerror("Ошибка", "Заполните все поля")
            return
        
        try:
            pr, op, st = float(pr), float(op), int(st)
        except ValueError:
            messagebox.showerror("Ошибка", "Некорректные числа")
            return
        
        manufacturer = manufacturer or ''
        supplier = supplier or ''
        unit = unit or 'шт.'
        
        try:
            c = self.conn.cursor()
            c.execute("""
                UPDATE p 
                SET n=%s, pr=%s, old_pr=%s, d=%s, cat=%s, st=%s, 
                    manufacturer=%s, supplier=%s, unit=%s 
                WHERE id=%s
            """, (n, pr, op, d, cat, st, manufacturer, supplier, unit, pid))
            self.conn.commit()
            c.close()
            
            messagebox.showinfo("Успех", "Товар обновлен")
            self.show_catalog()
        except mysql.connector.Error as err:
            print(f"Ошибка при редактировании товара: {err}")
            self.conn.rollback()

    def del_product(self, pid):
        """Удаление товара"""
        if messagebox.askyesno("Подтверждение", "Удалить товар?"):
            try:
                c = self.conn.cursor()
                c.execute("DELETE FROM p WHERE id=%s", (pid,))
                self.conn.commit()
                c.close()
                self.show_catalog()
            except mysql.connector.Error:
                self.conn.rollback()

    def logout(self):
        """Выход из системы"""
        self.current_user = self.current_role = self.current_user_name = None
        self.update_user_info()
        self.show_auth()

    def __del__(self):
        """Деструктор"""
        if hasattr(self, 'conn'):
            self.conn.close()


if __name__ == "__main__":
    root = tk.Tk()
    app = ShopApp(root)
    root.mainloop()
