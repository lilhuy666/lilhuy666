import tkinter as tk
from tkinter import messagebox
from datetime import datetime

# ======================
# 🎨 Цвета
# ======================
BG = "#eef2f7"
BLUE = "#2f6fed"
DARK = "#1e293b"
CARD = "#ffffff"
MUTED = "#94a3b8"

# ======================
# 📦 Данные
# ======================
history_data = []
users = {}
current_user = None

# ======================
# 🖥️ Окно
# ======================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

# ======================
# 🔷 HEADER
# ======================
header = tk.Frame(root, bg=BLUE, height=70)
header.pack(fill="x")

tk.Label(header, text="☰", bg=BLUE, fg="white",
         font=("Arial", 20)).pack(side="left", padx=20)

tk.Label(header, text="CalculatCar", bg=BLUE, fg="white",
         font=("Arial", 22, "bold")).pack(pady=15)

# ======================
# 📦 КОНТЕЙНЕР
# ======================
container = tk.Frame(root, bg=BG)
container.pack(fill="both", expand=True)

# ======================
# 📚 SIDEBAR
# ======================
sidebar = tk.Frame(container, bg="#f8fafc", width=220)
sidebar.pack(side="left", fill="y")

main = tk.Frame(container, bg=BG)
main.pack(side="left", fill="both", expand=True, padx=20, pady=20)

def clear_main():
    for w in main.winfo_children():
        w.destroy()

# ======================
# 🧮 КАЛЬКУЛЯТОР
# ======================
def show_calc():
    clear_main()

    tk.Label(main, text="Калькулятор расхода топлива",
             bg=BG, fg=DARK, font=("Arial", 20, "bold")).pack(anchor="w", pady=10)

    content = tk.Frame(main, bg=BG)
    content.pack(fill="both", expand=True)

    # -------- ЛЕВО --------
    left = tk.Frame(content, bg=CARD, padx=20, pady=20)
    left.pack(side="left", padx=10, fill="y")

    def field(label):
        tk.Label(left, text=label, bg=CARD).pack(anchor="w")
        e = tk.Entry(left, width=25)
        e.pack(pady=5)
        return e

    distance = field("Дистанция (км):")
    fuel = field("Кол-во топлива (л):")
    price = field("Цена топлива (₽/л):")
    needed_distance = field("Планируемая дистанция (км):")

    # -------- ЦЕНТР --------
    center = tk.Frame(content, bg=CARD, padx=20, pady=20)
    center.pack(side="left", padx=10, fill="y")

    tk.Label(center, text="Результаты",
             bg=CARD, font=("Arial", 14, "bold")).pack(anchor="w")

    tk.Label(center, text="Расход топлива",
             bg=CARD, fg=MUTED).pack(anchor="w", pady=(10, 0))

    result_consumption = tk.Label(center, text="0.0 л/100 км",
                                 bg=CARD, font=("Arial", 18, "bold"))
    result_consumption.pack(anchor="w")

    tk.Label(center, text="Стоимость поездки",
             bg=CARD, fg=MUTED).pack(anchor="w", pady=(15, 0))

    result_cost = tk.Label(center, text="0 ₽",
                           bg=CARD, font=("Arial", 18, "bold"))
    result_cost.pack(anchor="w")

    tk.Label(center, text="Потребуется топлива",
             bg=CARD, fg=MUTED).pack(anchor="w", pady=(15, 0))

    result_needed = tk.Label(center, text="0 л",
                            bg=CARD, font=("Arial", 18, "bold"))
    result_needed.pack(anchor="w")

    # -------- ИСТОРИЯ --------
    history_frame = tk.Frame(main, bg=CARD, padx=20, pady=20)
    history_frame.pack(fill="x", pady=20)

    tk.Label(history_frame, text="Последние расчёты",
             bg=CARD, font=("Arial", 12, "bold")).pack(anchor="w")

    def update_history_view():
        for w in history_frame.winfo_children()[1:]:
            w.destroy()

        for item in history_data[-5:][::-1]:
            tk.Label(history_frame,
                     text=f"{item['cons']} — {item['cost']} ({item['time']})",
                     bg=CARD).pack(anchor="w")

    # -------- ЛОГИКА --------
    def calculate():
        try:
            d = float(distance.get())
            f = float(fuel.get())
            p = float(price.get())
            nd = float(needed_distance.get() or 0)

            cons = f / d * 100
            cost = f * p

            result_consumption.config(text=f"{cons:.1f} л/100 км")
            result_cost.config(text=f"{cost:.0f} ₽")

            if nd > 0:
                needed = cons / 100 * nd
                result_needed.config(text=f"{needed:.1f} л")
            else:
                result_needed.config(text="0 л")

            record = {
                "cons": f"{cons:.1f} л/100 км",
                "cost": f"{cost:.0f} ₽",
                "time": datetime.now().strftime("%H:%M")
            }

            history_data.append(record)
            update_history_view()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    def reset():
        distance.delete(0, tk.END)
        fuel.delete(0, tk.END)
        price.delete(0, tk.END)
        needed_distance.delete(0, tk.END)

        result_consumption.config(text="0.0 л/100 км")
        result_cost.config(text="0 ₽")
        result_needed.config(text="0 л")

    tk.Button(left, text="Рассчитать",
              bg=BLUE, fg="white",
              command=calculate).pack(fill="x", pady=10)

    tk.Button(left, text="Сброс",
              command=reset).pack(fill="x")

    update_history_view()

# ======================
# 🕒 ИСТОРИЯ
# ======================
def show_history():
    clear_main()

    tk.Label(main, text="История",
             bg=BG, font=("Arial", 20, "bold")).pack()

    for item in history_data[::-1]:
        tk.Label(main,
                 text=f"{item['cons']} — {item['cost']} ({item['time']})",
                 bg=BG).pack(anchor="w")

# ======================
# 👤 ПРОФИЛЬ
# ======================
def show_profile():
    clear_main()

    global current_user

    if current_user is None:
        frame = tk.Frame(main, bg=CARD, padx=40, pady=40)
        frame.pack(expand=True, fill="both")

        tk.Label(frame, text="Вход / Регистрация",
                 font=("Arial", 20, "bold"), bg=CARD).pack(pady=10)

        tk.Label(frame, text="Логин", bg=CARD).pack(anchor="w")
        login_entry = tk.Entry(frame)
        login_entry.pack(fill="x", pady=5)

        tk.Label(frame, text="Пароль", bg=CARD).pack(anchor="w")
        password_entry = tk.Entry(frame, show="*")
        password_entry.pack(fill="x", pady=5)

        def login():
            global current_user
            login = login_entry.get()
            password = password_entry.get()

            if login in users and users[login]["password"] == password:
                current_user = login
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные")

        def register():
            login = login_entry.get()
            password = password_entry.get()

            if login in users:
                messagebox.showerror("Ошибка", "Пользователь уже есть")
                return

            users[login] = {
                "password": password,
                "name": "Гость",
                "car": "Не указано"
            }

            messagebox.showinfo("Успех", "Аккаунт создан!")

        tk.Button(frame, text="Войти", bg=BLUE, fg="white",
                  command=login).pack(fill="x", pady=5)

        tk.Button(frame, text="Регистрация",
                  command=register).pack(fill="x")

    else:
        data = users[current_user]

        tk.Label(main, text="Профиль",
                 bg=BG, font=("Arial", 20, "bold")).pack(pady=10)

        name_entry = tk.Entry(main)
        name_entry.insert(0, data["name"])
        name_entry.pack(pady=5)

        car_entry = tk.Entry(main)
        car_entry.insert(0, data["car"])
        car_entry.pack(pady=5)

        def save():
            data["name"] = name_entry.get()
            data["car"] = car_entry.get()
            messagebox.showinfo("Сохранено", "Данные обновлены")

        def change_password():
            win = tk.Toplevel(root)
            win.title("Смена пароля")

            tk.Label(win, text="Новый пароль").pack()
            new_pass = tk.Entry(win, show="*")
            new_pass.pack()

            def save_pass():
                data["password"] = new_pass.get()
                win.destroy()

            tk.Button(win, text="Сохранить", command=save_pass).pack()

        def logout():
            global current_user
            current_user = None
            show_profile()

        tk.Button(main, text="Сохранить", command=save).pack(pady=5)
        tk.Button(main, text="Сменить пароль", command=change_password).pack(pady=5)
        tk.Button(main, text="Выйти", command=logout).pack(pady=10)

# ======================
# ℹ️ О НАС
# ======================
def show_about():
    clear_main()

    tk.Label(main, text="О программе",
             bg=BG, font=("Arial", 20, "bold")).pack(pady=20)

    tk.Label(main,
             text="CalculatCar — приложение для расчёта топлива и стоимости поездки 🚗",
             bg=BG).pack()

# ======================
# 📚 КНОПКИ
# ======================
def btn(text, cmd):
    return tk.Button(sidebar, text=text,
                     command=cmd,
                     anchor="w",
                     bg="#f8fafc",
                     relief="flat",
                     padx=20,
                     pady=12)

btn("Профиль", show_profile).pack(fill="x")
btn("Калькулятор", show_calc).pack(fill="x")
btn("История", show_history).pack(fill="x")
btn("О нас", show_about).pack(fill="x")

# ======================
# ▶️ СТАРТ
# ======================
show_calc()
root.mainloop()
