import tkinter as tk
from tkinter import ttk, messagebox
import psycopg2
import bcrypt

# ===============================
# 🔌 Подключение БД
# ===============================
conn = psycopg2.connect(
    dbname="fuelcalc",
    user="postgres",
    password="1234",
    host="localhost",
    port="5432"
)
cursor = conn.cursor()

current_user = None
current_role = None

# ===============================
# 🚗 Транспорт
# ===============================
vehicles = {
    "Легковой автомобиль": {"city": 10, "highway": 6},
    "Кроссовер": {"city": 12, "highway": 8},
    "Грузовик": {"city": 30, "highway": 22},
    "Автобус": {"city": 35, "highway": 25},
}

FUEL_PRICE = 55

# ===============================
# 🖥️ UI
# ===============================
root = tk.Tk()
root.title("FuelCalc Pro")
root.geometry("1000x700")

# ===============================
# 🔐 Регистрация
# ===============================
def show_register():
    root_clear()

    frame = tk.Frame(root)
    frame.place(relx=0.5, rely=0.5, anchor="center")

    tk.Label(frame, text="Регистрация").pack()

    user_entry = tk.Entry(frame)
    user_entry.pack()

    pass_entry = tk.Entry(frame, show="*")
    pass_entry.pack()

    role_var = tk.StringVar(value="user")
    ttk.Combobox(frame, textvariable=role_var,
                 values=["user", "admin"]).pack()

    def register():
        username = user_entry.get()
        password = pass_entry.get()
        role = role_var.get()

        if not username or not password:
            messagebox.showerror("Ошибка", "Заполните поля")
            return

        hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt())

        try:
            cursor.execute(
                "INSERT INTO users (username, password, role) VALUES (%s, %s, %s)",
                (username, hashed.decode(), role)
            )
            conn.commit()
            messagebox.showinfo("Успех", "Пользователь создан")
            show_login()
        except:
            messagebox.showerror("Ошибка", "Пользователь уже существует")

    tk.Button(frame, text="Создать", command=register).pack()
    tk.Button(frame, text="Назад", command=show_login).pack()

# ===============================
# 🔐 Вход
# ===============================
def show_login():
    root_clear()

    frame = tk.Frame(root)
    frame.place(relx=0.5, rely=0.5, anchor="center")

    tk.Label(frame, text="Вход").pack()

    user_entry = tk.Entry(frame)
    user_entry.pack()

    pass_entry = tk.Entry(frame, show="*")
    pass_entry.pack()

    def login():
        global current_user, current_role

        username = user_entry.get()
        password = pass_entry.get()
        
        cursor.execute("SELECT password, role FROM users WHERE username=%s", (username,))
        result = cursor.fetchone()

        if result:
            db_pass, role = result
            if bcrypt.checkpw(password.encode(), db_pass.encode()):
                current_user = username
                current_role = role
                show_main()
                return

        messagebox.showerror("Ошибка", "Неверный логин или пароль")

    tk.Button(frame, text="Войти", command=login).pack()
    tk.Button(frame, text="Регистрация", command=show_register).pack()

# ===============================
# 🧮 Калькулятор
# ===============================
def build_calc():
    clear_main()

    tk.Label(main_frame, text=f"Пользователь: {current_user} ({current_role})").pack()

    combo = ttk.Combobox(main_frame, values=list(vehicles.keys()))
    combo.pack()

    dist = tk.Entry(main_frame)
    dist.pack()

    city = tk.Entry(main_frame)
    city.insert(0, "50")
    city.pack()

    highway = tk.Entry(main_frame)
    highway.insert(0, "50")
    highway.pack()

    result = tk.Label(main_frame, text="")
    result.pack()

    def calc():
        try:
            v = vehicles[combo.get()]
            d = float(dist.get())
            c = float(city.get())
            h = float(highway.get())

            if c + h != 100:
                raise ValueError

            avg = v["city"]*(c/100) + v["highway"]*(h/100)
            fuel = d/100*avg
            cost = fuel*FUEL_PRICE

            result.config(text=f"{fuel:.2f} л | {cost:.2f} ₽")

            # 💾 Сохраняем в БД
            cursor.execute("""
                INSERT INTO history
                (username, vehicle, distance, fuel, cost, city_percent, highway_percent)
                VALUES (%s,%s,%s,%s,%s,%s,%s)
            """, (current_user, combo.get(), d, fuel, cost, c, h))
            conn.commit()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    tk.Button(main_frame, text="Рассчитать", command=calc).pack()

# ===============================
# 📜 История
# ===============================
def show_history():
    clear_main()

    cursor.execute("SELECT * FROM history WHERE username=%s", (current_user,))
    rows = cursor.fetchall()

    for r in rows:
        text = f"{r[2]} | {r[3]} км | {r[5]:.2f} ₽"
        tk.Label(main_frame, text=text).pack(anchor='w')

# ===============================
# 👨‍💼 Админ панель
# ===============================
def admin_panel():
    clear_main()

    if current_role != "admin":
        tk.Label(main_frame, text="Нет доступа").pack()
        return

    cursor.execute("SELECT username, role FROM users")
    users = cursor.fetchall()

    for u in users:
        tk.Label(main_frame, text=f"{u[0]} ({u[1]})").pack(anchor='w')

# ===============================
# 🧭 Главное окно
# ===============================
def show_main():
    root_clear()

    global main_frame

    container = tk.Frame(root)
    container.pack(fill="both", expand=True)

    menu = tk.Frame(container, width=200)
    menu.pack(side="left", fill="y")

    main_frame = tk.Frame(container)
    main_frame.pack(side="right", expand=True, fill="both")

    tk.Button(menu, text="Калькулятор", command=build_calc).pack(pady=10)
    tk.Button(menu, text="История", command=show_history).pack(pady=10)
    tk.Button(menu, text="Админ", command=admin_panel).pack(pady=10)

    tk.Button(menu, text="Выход", command=show_login).pack(pady=20)

    build_calc()

# ===============================
# 🧹 Утилиты
# ===============================
def root_clear():
    for w in root.winfo_children():
        w.destroy()

def clear_main():
    for w in main_frame.winfo_children():
        w.destroy()

# ===============================
# ▶️ Старт
# ===============================
show_login()
root.mainloop()


python -m pip install psycopg2-binary bcrypt
