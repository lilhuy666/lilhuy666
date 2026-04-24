import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime
import json, os
import hashlib

# ===================== STYLE =====================
BG = "#0b1220"
PANEL = "#111a2e"
CARD = "#162238"
ACCENT = "#4f8cff"
ACCENT2 = "#22c55e"
TEXT = "#e5e7eb"
SUB = "#94a3b8"
DANGER = "#ef4444"
PROFILE_BG = "#f5f7fa"

DATA_FILE = "data.json"

# ===================== DATA =====================
data = {"users": {}}
current_user = None

def hash_password(password):
    """Хеширование пароля"""
    return hashlib.sha256(password.encode()).hexdigest()

def load_data():
    global data
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
            if "users" not in data:
                data = {"users": {}}
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось загрузить данные: {e}")
            data = {"users": {}}

def save_data():
    try:
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
    except Exception as e:
        messagebox.showerror("Ошибка", f"Не удалось сохранить данные: {e}")

# ===================== WINDOW =====================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

main = tk.Frame(root, bg=BG)
main.pack(fill="both", expand=True)

# ===================== HEADER =====================
header = tk.Frame(main, bg=ACCENT, height=70)
header.pack(fill="x")
header.pack_propagate(False)

menu_window = None

def toggle_menu():
    global menu_window

    if menu_window and menu_window.winfo_exists():
        menu_window.destroy()
        return

    x = root.winfo_x() + 20
    y = root.winfo_y() + 70

    menu_window = tk.Toplevel(root)
    menu_window.overrideredirect(True)
    menu_window.configure(bg=PANEL)
    menu_window.geometry(f"220x240+{x}+{y}")

    def close(e=None):
        if menu_window:
            menu_window.destroy()

    menu_window.bind("<FocusOut>", close)
    menu_window.bind("<Escape>", close)  # Закрытие по Esc

    def nav(text, cmd):
        tk.Button(menu_window,
                  text=text,
                  bg=PANEL,
                  fg=TEXT,
                  bd=0,
                  anchor="w",
                  padx=15,
                  pady=10,
                  font=("Arial", 11),
                  activebackground=CARD,
                  activeforeground=ACCENT,
                  command=lambda: [cmd(), close()]
                  ).pack(fill="x")

    nav("Калькулятор", show_calc)
    nav("Профиль", show_profile)
    nav("История", show_history)
    nav("Настройки", show_settings)
    nav("О программе", show_about)

    menu_window.focus_force()

tk.Button(header, text="☰", bg=ACCENT, fg="white",
          font=("Arial", 16, "bold"), bd=0,
          command=toggle_menu).pack(side="left", padx=15)

tk.Label(header, text="CalculatCar",
         bg=ACCENT, fg="white",
         font=("Arial", 20, "bold")).pack(side="left")

user_label = tk.Label(header, text="",
                      bg=ACCENT, fg="white",
                      font=("Arial", 11))
user_label.pack(side="right", padx=15)

# ===================== CONTENT =====================
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

def card():
    f = tk.Frame(content, bg=CARD, padx=40, pady=40)
    f.pack(pady=30)
    return f

# ===================== PROFILE =====================
def show_profile():
    clear()

    if not current_user:
        c = card()

        tk.Label(c, text="Вход / Регистрация",
                 bg=CARD, fg=TEXT,
                 font=("Arial", 20, "bold")).pack(pady=10)

        email = tk.Entry(c, font=("Arial", 14))
        password = tk.Entry(c, show="*", font=("Arial", 14))

        email.pack(fill="x", pady=8)
        password.pack(fill="x", pady=8)

        def login():
            global current_user
            e = email.get().strip()
            p = password.get().strip()

            if e in data["users"] and data["users"][e]["password"] == hash_password(p):
                current_user = e
                update_user()
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные")

        def register():
            e = email.get().strip()
            p = password.get().strip()

            if not e or not p:
                return messagebox.showerror("Ошибка", "Заполни поля")

            if e in data["users"]:
                return messagebox.showerror("Ошибка", "Уже существует")

            data["users"][e] = {
                "password": hash_password(p),
                "name": "",
                "surname": "",
                "phone": "",
                "car_make": "",
                "car_model": "",
                "car_year": "",
                "notifications": True,
                "history": []
            }
            save_data()
            messagebox.showinfo("OK", "Аккаунт создан")

        tk.Button(c, text="Войти", bg=ACCENT, fg="white",
                  command=login).pack(fill="x", pady=10)

        tk.Button(c, text="Регистрация", bg=ACCENT2, fg="black",
                  command=register).pack(fill="x")
        return

    user = data["users"][current_user]
    profile_frame = tk.Frame(content, bg=PROFILE_BG)
    profile_frame.pack(padx=20, pady=20, fill="both", expand=True)

    # Левая колонка — аватар и основная информация
    left_col = tk.Frame(profile_frame, bg=PROFILE_BG)
    left_col.pack(side="left", padx=(0, 20), fill="y")

    # Аватар
    avatar = tk.Canvas(left_col, width=120, height=120, bg="#4a5568", bd=0)
    avatar.create_oval(10, 10, 110, 110, fill="#6b7280")
    avatar.pack(pady=(0, 10))

    # Имя и фамилия
    full_name = f"{user.get('name', '')} {user.get('surname', '')}"
    tk.Label(
        left_col,
        text=full_name,
        font=("Arial", 16, "bold"),
        fg="#1e293b",
        bg=PROFILE_BG
    ).pack()

    tk.Label(
        left_col,
        text=current_user,
        fg="#6b7280",
        bg=PROFILE_BG
    ).pack()

    edit_btn = tk.Button(
        left_col,
        text="Редактировать",
        bg="#63b3ed",
        fg="white",
        padx=10,
        pady=5
    )
    edit_btn.pack(pady=10)

    # Информация об автомобиле
    car_info = tk.Frame(left_col, bg="#e2e8f0", padx=10, pady=10)
    car_info.pack(pady=10, ipadx=5, ipady=5)

        tk.Label(car_info, text="Марка автомобиля:", font=("Arial", 10, "bold")).grid(
        row=0, column=0, sticky="w")
    make_var = tk.StringVar(value=user.get("car_make", ""))
    make_dropdown = ttk.Combobox(car_info, textvariable=make_var, values=[
        "Toyota", "Honda", "BMW", "Mercedes", "Audi", "Volkswagen", "Ford", "Kia"
    ])
    make_dropdown.grid(row=0, column=1, padx=5, pady=5)

    tk.Label(car_info, text="Модель автомобиля:", font=("Arial", 10, "bold")).grid(
        row=1, column=0, sticky="w")
    model_var = tk.StringVar(value=user.get("car_model", ""))
    model_dropdown = ttk.Combobox(car_info, textvariable=model_var, values=[
        "Camry", "RAV4", "Corolla", "Civic", "Accord", "X5", "E-Class"
    ])
    model_dropdown.grid(row=1, column=1, padx=5, pady=5)

    tk.Label(car_info, text="Год выпуска:", font=("Arial", 10, "bold")).grid(
        row=2, column=0, sticky="w")
    year_var = tk.StringVar(value=user.get("car_year", ""))
    year_entry = tk.Entry(car_info, textvariable=year_var, width=10)
    year_entry.grid(row=2, column=1, padx=5, pady=5)

    def save_car_info():
        user["car_make"] = make_var.get()
        user["car_model"] = model_var.get()
        user["car_year"] = year_var.get()
        save_data()
        messagebox.showinfo("OK", "Данные об автомобиле сохранены")

    tk.Button(car_info, text="Сохранить", bg=ACCENT, fg="white",
              command=save_car_info).grid(row=3, column=0, columnspan=2, pady=10)

    # Центральная колонка — личная информация
    center_col = tk.Frame(profile_frame, bg=PROFILE_BG)
    center_col.pack(side="left", fill="both", expand=True)

    personal_info = tk.LabelFrame(center_col, text="Личная информация", padx=10, pady=10, bg=PROFILE_BG, fg=TEXT)
    personal_info.pack(fill="both", expand=True, pady=10)

    fields = [
        ("Имя:", "name", user.get("name", "")),
        ("Фамилия:", "surname", user.get("surname", "")),
        ("E-mail:", "email", current_user),
        ("Телефон:", "phone", user.get("phone", ""))
    ]
    entries = {}

    for i, (label_text, field_name, default_value) in enumerate(fields):
        tk.Label(personal_info, text=label_text, font=("Arial", 10, "bold"), bg=PROFILE_BG, fg=TEXT).grid(
            row=i, column=0, sticky="w", pady=5)
        entry = tk.Entry(personal_info, width=30)
        entry.insert(0, default_value)
        entry.grid(row=i, column=1, padx=10, pady=5)
        entries[field_name] = entry

    def save_personal_info():
        user["name"] = entries["name"].get()
        user["surname"] = entries["surname"].get()
        user["phone"] = entries["phone"].get()
        save_data()
        messagebox.showinfo("OK", "Личные данные обновлены")

    tk.Button(personal_info, text="Сохранить изменения", bg=ACCENT, fg="white",
              command=save_personal_info).grid(row=len(fields), column=0, columnspan=2, pady=10)

    # Правая колонка — настройки профиля и уведомления
    right_col = tk.Frame(profile_frame, bg=PROFILE_BG)
    right_col.pack(side="right", padx=(20, 0), fill="y")

    # Настройки профиля
    settings_frame = tk.LabelFrame(right_col, text="Настройки профиля", padx=10, pady=10, bg=PROFILE_BG, fg=TEXT)
    settings_frame.pack(fill="x", pady=(0, 10))

    def change_password():
        change_pass_window = tk.Toplevel(root)
        change_pass_window.title("Сменить пароль")
        change_pass_window.geometry("300x200")
        change_pass_window.configure(bg=BG)

        tk.Label(change_pass_window, text="Старый пароль:", bg=BG, fg=TEXT).pack(pady=5)
        old_pass = tk.Entry(change_pass_window, show="*", font=("Arial", 12))
        old_pass.pack(pady=5)

        tk.Label(change_pass_window, text="Новый пароль:", bg=BG, fg=TEXT).pack(pady=5)
        new_pass1 = tk.Entry(change_pass_window, show="*", font=("Arial", 12))
        new_pass1.pack(pady=5)

        tk.Label(change_pass_window, text="Повторите пароль:", bg=BG, fg=TEXT).pack(pady=5)
        new_pass2 = tk.Entry(change_pass_window, show="*", font=("Arial", 12))
        new_pass2.pack(pady=5)

        def save_new_password():
            if hash_password(old_pass.get()) != user["password"]:
                messagebox.showerror("Ошибка", "Неверный старый пароль")
                return
            if new_pass1.get() != new_pass2.get():
                messagebox.showerror("Ошибка", "Пароли не совпадают")
                return
            user["password"] = hash_password(new_pass1.get())
            save_data()
            messagebox.showinfo("OK", "Пароль изменён")
            change_pass_window.destroy()

        tk.Button(change_pass_window, text="Сохранить", bg=ACCENT, fg="white", command=save_new_password).pack(pady=10)

    change_pass_btn = tk.Button(
        settings_frame,
        text="Сменить пароль",
        bg="#63b3ed",
        fg="white",
        padx=10,
        pady=5,
        command=change_password
    )
    change_pass_btn.pack(fill="x", pady=(0, 5))

    def delete_account():
        if messagebox.askyesno("Подтверждение", "Вы уверены, что хотите удалить аккаунт?"):
            del data["users"][current_user]
            save_data()
            global current_user
            current_user = None
            update_user()
            show_profile()

    delete_acc_btn = tk.Button(
        settings_frame,
        text="Удалить аккаунт",
        bg="#ef4444",
        fg="white",
        padx=10,
        pady=5,
        command=delete_account
    )
    delete_acc_btn.pack(fill="x")

    # Уведомления
    notifications_frame = tk.LabelFrame(right_col, text="Уведомления", padx=10, pady=10, bg=PROFILE_BG, fg=TEXT)
    notifications_frame.pack(fill="x")

    notifications_var = tk.BooleanVar(value=user.get("notifications", True))

    def toggle_notifications():
        user["notifications"] = notifications_var.get()
        save_data()

    notifications_check = tk.Checkbutton(
        notifications_frame,
        text="Включено",
        variable=notifications_var,
        onvalue=True,
        offvalue=False,
        bg=PROFILE_BG,
                fg=TEXT,
        command=toggle_notifications
    )
    notifications_check.pack()

    # Кнопка выхода
    logout_btn = tk.Button(
        right_col,
        text="Выйти из аккаунта",
        bg=DANGER,
        fg="white",
        padx=10,
        pady=5,
        command=logout
    )
    logout_btn.pack(fill="x", pady=20)

def logout():
    global current_user
    current_user = None
    update_user()
    show_profile()

# ===================== CALCULATOR =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор расхода",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    mode = tk.StringVar(value="1")

    switch = tk.Frame(content, bg=BG)
    switch.pack(pady=10)

    tk.Radiobutton(switch, text="Рассчитать стоимость поездки",
                   variable=mode, value="1",
                   indicatoron=0, width=28,
                   bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT).pack(side="left", padx=5)

    tk.Radiobutton(switch, text="Средний расход на 100 км",
                   variable=mode, value="2",
                   indicatoron=0, width=28,
                   bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT).pack(side="left", padx=5)

    form = tk.Frame(content, bg=CARD, padx=40, pady=40)
    form.pack(pady=20)

    entries = {}

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(t):
            tk.Label(form, text=t, bg=CARD, fg=TEXT).pack(anchor="w")
            e = tk.Entry(form)
            e.pack(fill="x", pady=5)
            entries[t] = e

        if mode.get() == "1":
            add("Топливо (л)")
            add("Расстояние (км)")
            add("Цена за литр")
        else:
            add("Средний расход (л/100км)")
            add("Расстояние (км)")
            add("Цена за литр")

    def calc():
        try:
            if mode.get() == "1":
                f = float(entries["Топливо (л)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())

                if d <= 0 or p <= 0 or f <= 0:
                    messagebox.showerror("Ошибка", "Все значения должны быть положительными")
                    return

                cons = (f / d) * 100
                cost = f * p
            else:
                cons = float(entries["Средний расход (л/100км)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())

                if d <= 0 or p <= 0 or cons <= 0:
                    messagebox.showerror("Ошибка", "Все значения должны быть положительными")
                    return

                cost = cons * d / 100 * p

            result.config(text=f"{cons:.1f} л/100км | {cost:.2f} ₽")

            if current_user:
                data["users"][current_user]["history"].append({
                    "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                    "distance": d,
                    "fuel": f if mode.get() == "1" else None,
                    "price_per_liter": p,
            "consumption": cons,
            "total_cost": cost
                })
                data["users"][current_user]["history"] = data["users"][current_user]["history"][-50:]
                save_data()

        except ValueError:
            messagebox.showerror("Ошибка", "Проверь ввод — все поля должны содержать числа")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Произошла ошибка: {e}")

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=("Arial", 22, "bold"))
    result.pack()

    tk.Button(content, text="Рассчитать",
              bg=ACCENT, fg="white",
              font=("Arial", 14, "bold"),
              command=calc).pack(pady=10)

    mode.trace("w", lambda *a: build())
    build()

# ===================== HISTORY =====================
def show_history():
    clear()

    if not current_user:
        tk.Label(content, text="Авторизуйтесь для просмотра истории", bg=BG, fg=TEXT).pack(pady=20)
        return

    # Заголовок
    tk.Label(content, text="История расчётов", bg=BG, fg=TEXT, font=("Arial", 20, "bold")).pack(pady=10)
    tk.Label(content, text="Ваши последние расчёты", bg=BG, fg=SUB).pack(pady=5)

    # Кнопки «Фильтры» и «Очистить историю»
    btn_frame = tk.Frame(content, bg=BG)
    btn_frame.pack(fill="x", pady=10)

    tk.Button(btn_frame, text="Фильтры", bg=ACCENT, fg="white", padx=10).pack(side="right", padx=5)
    tk.Button(btn_frame, text="Очистить историю", bg=DANGER, fg="white", padx=10,
              command=lambda: clear_history()).pack(side="right", padx=5)

    # Таблица истории
    history_frame = tk.Frame(content, bg=CARD)
    history_frame.pack(fill="both", expand=True, padx=20, pady=10)

    # Шапка таблицы
    header = tk.Frame(history_frame, bg=CARD, height=30)
    header.pack(fill="x")

    tk.Label(header, text="Дата и время", bg=CARD, fg=TEXT, width=15).pack(side="left")
    tk.Label(header, text="Параметры расчёта", bg=CARD, fg=TEXT, width=20).pack(side="left")
    tk.Label(header, text="Результат", bg=CARD, fg=TEXT, width=15).pack(side="left")
    tk.Label(header, text="Действия", bg=CARD, fg=TEXT, width=10).pack(side="left")

    # Строки истории
    for index, item in enumerate(reversed(data["users"][current_user]["history"])):
        row = tk.Frame(history_frame, bg=CARD, height=50)
        row.pack(fill="x", pady=2)

        # Дата и время
        tk.Label(row, text=f"{item['date']}", bg=CARD, fg=TEXT, width=15).pack(side="left", padx=5)

        # Параметры расчёта
        params = tk.Frame(row, bg=CARD)
        params.pack(side="left", padx=10)

        if item['fuel'] is not None:
            tk.Label(params, text=f"Топливо: {item['fuel']} л", bg=CARD, fg=TEXT).pack(anchor="w")
        tk.Label(params, text=f"Расстояние: {item['distance']} км", bg=CARD, fg=TEXT).pack(anchor="w")
        tk.Label(params, text=f"Цена: {item['price_per_liter']} ₽/л", bg=CARD, fg=TEXT).pack(anchor="w")

                # Результат
        result_frame = tk.Frame(row, bg=ACCENT, padx=8, pady=4)
        result_frame.pack(side="left", padx=10)
        tk.Label(result_frame, text=f"{item['consumption']:.1f} л/100км",
                  bg=ACCENT, fg="white", font=("Arial", 9, "bold")).pack()
        tk.Label(result_frame, text=f"{item['total_cost']:.2f} ₽",
                  bg=ACCENT, fg="white", font=("Arial", 9, "bold")).pack()

        # Действия
        actions = tk.Frame(row, bg=CARD)
        actions.pack(side="left")

        def open_calculation(item_data):
            show_calc()
            # Заполняем поля калькулятора данными из истории
            if item_data['fuel'] is not None:
                mode.set("1")
                entries["Топливо (л)"].delete(0, tk.END)
                entries["Топливо (л)"].insert(0, str(item_data['fuel']))
            else:
                mode.set("2")
                entries["Средний расход (л/100км)"].delete(0, tk.END)
                entries["Средний расход (л/100км)"].insert(0, str(item_data['consumption']))

            entries["Расстояние (км)"].delete(0, tk.END)
            entries["Расстояние (км)"].insert(0, str(item_data['distance']))
            entries["Цена за литр"].delete(0, tk.END)
            entries["Цена за литр"].insert(0, str(item_data['price_per_liter']))

        tk.Button(actions, text="↻", bg=ACCENT2, fg="black",
                 command=lambda: open_calculation(item)).pack(side="left", padx=2)

        def delete_history_item(index):
            del data["users"][current_user]["history"][index]
            save_data()
            show_history()

        tk.Button(actions, text="❌", bg=DANGER, fg="white",
                 command=lambda idx=index: delete_history_item(idx)).pack(side="left", padx=2)

def clear_history():
    if messagebox.askyesno("Подтверждение", "Вы уверены, что хотите очистить всю историю?"):
        data["users"][current_user]["history"] = []
        save_data()
        show_history()

# ===================== SETTINGS =====================
def show_settings():
    clear()
    tk.Label(content, text="Настройки", bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=20)

    settings_frame = tk.Frame(content, bg=CARD, padx=40, pady=40)
    settings_frame.pack(pady=20)

    # Тема
    tk.Label(settings_frame, text="Тема:", bg=CARD, fg=TEXT,
             font=("Arial", 12, "bold")).pack(anchor="w", pady=(0, 10))
    theme_var = tk.StringVar(value="Тёмная")
    tk.OptionMenu(settings_frame, theme_var, "Тёмная", "Светлая").pack(anchor="w")

    # Язык
    tk.Label(settings_frame, text="Язык:", bg=CARD, fg=TEXT,
             font=("Arial", 12, "bold")).pack(anchor="w", pady=(20, 10))
    lang_var = tk.StringVar(value="Русский")
    tk.OptionMenu(settings_frame, lang_var, "Русский", "English").pack(anchor="w")

    def save_settings():
        messagebox.showinfo("OK", "Настройки сохранены")

    tk.Button(settings_frame, text="Сохранить настройки", bg=ACCENT, fg="white",
              command=save_settings).pack(pady=30)

# ===================== ABOUT =====================
def show_about():
    clear()
    about_frame = tk.Frame(content, bg=CARD, padx=40, pady=40)
    about_frame.pack(pady=30)

    tk.Label(about_frame, text="CalculatCar", bg=CARD, fg=ACCENT,
             font=("Arial", 24, "bold")).pack(pady=(0, 20))
    tk.Label(about_frame, text="Калькулятор расхода топлива", bg=CARD, fg=TEXT,
             font=("Arial", 14)).pack(pady=(0, 30))

    info_text = """
Версия: 1.0
Разработчик: CalculatCar Team
Год: 2024

Программа для расчёта расхода топлива и стоимости поездок.
Сохраняет историю расчётов и позволяет управлять профилем пользователя.

© Все права защищены.
    """
    tk.Label(about_frame, text=info_text, bg=CARD, fg=SUB,
             justify="left", font=("Arial", 11)).pack()

# ===================== UPDATE USER =====================
def update_user():
    if current_user:
        user_label.config(text=f"Пользователь: {current_user}")
    else:
        user_label.config(text="")

# ===================== START =====================
load_data()
update_user()
show_calc()  # По умолчанию показываем калькулятор

root.mainloop()
