import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime
import json, os
import hashlib
import secrets

import mode
from fontTools.ttLib.tables.grUtils import entries

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
BACKUP_FILE = "data.json.backup"

# ===================== DATA =====================
data = {"users": {}}
current_user = None

def hash_password(password, salt=None):
    """Хеширование пароля с солью"""
    if salt is None:
        salt = secrets.token_hex(16)
    hashed = hashlib.pbkdf2_hmac('sha256', password.encode(), salt.encode(), 100000)
    return f"{salt}:{hashed.hex()}"

def verify_password(stored_password, provided_password):
    """Проверка пароля"""
    salt, _ = stored_password.split(':')
    return hash_password(provided_password, salt) == stored_password

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
    # Создаём бэкап при загрузке
    create_backup()

def create_backup():
    """Создание бэкапа данных"""
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f_in:
            with open(BACKUP_FILE, "w", encoding="utf-8") as f_out:
                f_out.write(f_in.read())
    except:
        pass

def save_data():
    try:
        # Создаём бэкап перед сохранением
        create_backup()
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

            if e in data["users"] and verify_password(data["users"][e]["password"], p):
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
        make = make_var.get()
        model = model_var.get()
        year_text = year_var.get()

        if year_text:
            try:
                year = int(year_text)
                if not (1900 <= year <= datetime.now().year):
                    messagebox.showerror("Ошибка", "Год должен быть от 1900 до текущего года")
                    return
            except ValueError:
                messagebox.showerror("Ошибка", "Год должен быть числом")
                return
        else:
            year = ""

        user["car_make"] = make
        user["car_model"] = model
        user["car_year"] = str(year) if year else ""
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
            stored_password = user["password"]
            if not verify_password(stored_password, old_pass.get()):
                messagebox.showerror("Ошибка", "Неверный старый пароль")
                return

            new_pass_1 = new_pass1.get()
            new_pass_2 = new_pass2.get()

            if not new_pass_1 or not new_pass_2:
                messagebox.showerror("Ошибка", "Поля с новым паролем не могут быть пустыми")
                return

            if new_pass_1 != new_pass_2:
                messagebox.showerror("Ошибка", "Пароли не совпадают")
                return

            user["password"] = hash_password(new_pass_1)
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
        global current_user
        if messagebox.askyesno("Подтверждение", "Вы уверены, что хотите удалить аккаунт?"):
            del data["users"][current_user]
            save_data()
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


def update_user():
    """Обновление информации о пользователе в заголовке"""
    if current_user:
        user_label.config(text=f"Пользователь: {current_user}")
    else:
        user_label.config(text="")


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
                try:
                    f = float(entries["Топливо (л)"].get())
                    d = float(entries["Расстояние (км)"].get())
                    p = float(entries["Цена за литр"].get())
                except ValueError:
                    messagebox.showerror("Ошибка", "Все поля должны содержать числа")
                    return

                if d <= 0 or p <= 0 or f <= 0:
                    messagebox.showerror("Ошибка", "Все значения должны быть положительными")
                    return

                cons = (f / d) * 100
                cost = f * p
            else:
                try:
                    cons = float(entries["Средний расход (л/100км)"].get())
                    d = float(entries["Расстояние (км)"].get())
                    p = float(entries["Цена за литр"].get())
                except ValueError:
                    messagebox.showerror("Ошибка", "Все поля должны содержать числа")
                    return

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

    tk.Label(content, text="История расчётов",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    if not current_user:
        tk.Label(content, text="Войдите в аккаунт для просмотра истории",
                 bg=BG, fg=SUB,
                 font=("Arial", 14)).pack(pady=20)
        return

    user_history = data["users"][current_user]["history"]

    if not user_history:
        tk.Label(content, text="История пуста",
                 bg=BG, fg=SUB,
                 font=("Arial", 14)).pack(pady=20)
        return

    history_frame = tk.Frame(content, bg=CARD)
    history_frame.pack(fill="both", expand=True, padx=20, pady=10)

    # Заголовки таблицы
    headers = ["Дата", "Расстояние (км)", "Расход (л/100км)", "Стоимость (₽)", "Действия"]
    for col, header in enumerate(headers):
        tk.Label(history_frame, text=header, bg=PANEL, fg=TEXT,
                 font=("Arial", 10, "bold"), padx=5, pady=5).grid(row=0, column=col, sticky="ew")

    # Данные истории
    for row, item in enumerate(user_history, start=1):
        tk.Label(history_frame, text=item["date"], bg=CARD, fg=TEXT).grid(row=row, column=0, padx=5, pady=2, sticky="w")
        tk.Label(history_frame, text=str(item["distance"]), bg=CARD, fg=TEXT).grid(row=row, column=1, padx=5, pady=2)
        tk.Label(history_frame, text=f"{item['consumption']:.1f}", bg=CARD, fg=TEXT).grid(row=row, column=2, padx=5,
                                                                                          pady=2)
        tk.Label(history_frame, text=f"{item['total_cost']:.2f}", bg=CARD, fg=TEXT).grid(row=row, column=3, padx=5,
                                                                                         pady=2)

        # Кнопка для повторного использования данных
        def create_use_callback(item_data):
            def callback():
                open_calculation(item_data)

            return callback

        use_btn = tk.Button(history_frame, text="↻", bg=ACCENT, fg="white",
                            command=create_use_callback(item), width=3)
        use_btn.grid(row=row, column=4, padx=5, pady=2)

        # Кнопка удаления записи
        def create_delete_callback(row_idx):
            def callback():
                del user_history[row_idx - 1]
                save_data()
                show_history()

            return callback

        delete_btn = tk.Button(history_frame, text="×", bg=DANGER, fg="white",
                               command=create_delete_callback(row), width=3)
        delete_btn.grid(row=row, column=5, padx=5, pady=2)

    # Настройка растяжения колонок
    for col in range(len(headers)):
        history_frame.grid_columnconfigure(col, weight=1)


def open_calculation(item_data):
    """Открытие калькулятора с заполнением полей из истории"""
    show_calc()
    if item_data['fuel'] is not None:
        mode.set("1")
        entries["Топливо (л)"].delete(0, tk.END)
        entries["Топливо (л)"].insert(0, str(item_data['fuel']))
        entries["Средний расход (л/100км)"].delete(0, tk.END)
    else:
        mode.set("2")
        entries["Средний расход (л/100км)"].delete(0, tk.END)
        entries["Средний расход (л/100км)"].insert(0, str(item_data['consumption']))
        entries["Топливо (л)"].delete(0, tk.END)

    entries["Расстояние (км)"].delete(0, tk.END)
    entries["Расстояние (км)"].insert(0, str(item_data['distance']))
    entries["Цена за литр"].delete(0, tk.END)
    entries["Цена за литр"].insert(0, str(item_data['price_per_liter']))


# ===================== SETTINGS =====================
def show_settings():
    clear()
    tk.Label(content, text="Настройки",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    settings_frame = tk.Frame(content, bg=CARD, padx=40, pady=40)
    settings_frame.pack(pady=20)

    # Опция резервного копирования
    backup_btn = tk.Button(settings_frame,
                           text="Создать бэкап данных",
                           bg=ACCENT2, fg="black",
                           command=create_backup,
                           font=("Arial", 12))
    backup_btn.pack(pady=10, fill="x")

    # Информация о версии
    version_label = tk.Label(settings_frame,
                             text="CalculatCar v1.0\n© 2024",
                             bg=CARD, fg=SUB,
                             font=("Arial", 10))
    version_label.pack(pady=20)


# ===================== ABOUT =====================
def show_about():
    clear()
    tk.Label(content, text="О программе",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    about_text = """CalculatCar — приложение для расчёта расхода топлива и стоимости поездок.

    Основные функции:
    • Расчёт расхода топлива на 100 км
    • Расчёт стоимости поездки
    • История расчётов
    • Личный профиль пользователя
    • Сохранение данных между сеансами


    Версия: 1.0
    Разработчик: [Ваше имя]"""

    text_widget = tk.Text(content, wrap="word", bg=CARD, fg=TEXT, font=("Arial", 11), height=15)
    text_widget.pack(padx=20, pady=10, fill="both", expand=True)
    text_widget.insert("1.0", about_text)
    text_widget.config(state="disabled")


# Инициализация приложения
load_data()
update_user()
show_calc()  # По умолчанию показываем калькулятор

# Запуск главного цикла
root.mainloop()
