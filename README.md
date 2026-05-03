import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime
import json, os
import hashlib
import secrets

# ===================== STYLE =====================
MAIN_BG = "#f8f9fa"  # фон главного меню
AUTH_BG = "#f8f9fa"   # фон формы авторизации
BG = "#0b1220"
PANEL = "#111a2e"
CARD = "#162238"
ACCENT = "#4f8cff"
ACCENT2 = "#22c55e"
TEXT = "#e5e7eb"
SUB = "#94a3b8"
DANGER = "#ef4444"
PROFILE_BG = "#162238"  # единый фон с основной темой
AVATAR_BG = "#253145"


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
def update_user():
        """Обновляет отображение статуса пользователя в интерфейсе"""
        if current_user:
            user_label.config(text=f"Пользователь: {current_user}")
        else:
            user_label.config(text="Гость")


def show_auth():
    clear()

    frame = tk.Frame(content, bg=AUTH_BG, padx=30, pady=30)
    frame.pack(fill="both", expand=True)

    # Заголовок
    tk.Label(
        frame,
        text="Авторизация",
        bg=AUTH_BG,
        fg=ACCENT,
        font=("Arial", 18, "bold")
    ).pack(pady=(0, 20))

    # Поле для логина
    tk.Label(frame, text="Логин:", bg=AUTH_BG, fg=TEXT).pack(anchor="w")
    login_entry = tk.Entry(frame, font=("Arial", 12), width=25)
    login_entry.pack(pady=(0, 15), fill="x")

    # Поле для пароля
    tk.Label(frame, text="Пароль:", bg=AUTH_BG, fg=TEXT).pack(anchor="w")
    password_entry = tk.Entry(frame, show="*", font=("Arial", 12), width=25)
    password_entry.pack(pady=(0, 20), fill="x")

    # Кнопка входа
    def login():
        login = login_entry.get().strip()
        password = password_entry.get()

        if not login or not password:
            messagebox.showerror("Ошибка", "Заполните все поля")
            return

        user_data = data["users"].get(login)
        if user_data and verify_password(user_data["password"], password):
            global current_user
            current_user = login
            update_user()
            show_profile()
        else:
            messagebox.showerror("Ошибка", "Неверный логин или пароль")

    tk.Button(
        frame,
        text="Войти",
        command=login,
        bg=ACCENT,
        fg="white",
        font=("Arial", 12, "bold"),
        height=2
    ).pack(fill="x", pady=(0, 10))

    # Кнопка регистрации
    def register():
        login = login_entry.get().strip()
        password = password_entry.get()

        if not login or not password:
            messagebox.showerror("Ошибка", "Заполните все поля")
            return

        if len(password) < 6:
            messagebox.showerror("Ошибка", "Пароль должен быть не менее 6 символов")
            return

        if login in data["users"]:
            messagebox.showerror("Ошибка", "Пользователь с таким логином уже существует")
            return

        # Создание нового пользователя
        data["users"][login] = {
            "name": login,
            "password": hash_password(password),
            "registration_date": datetime.now().strftime("%d.%m.%Y"),
            "notifications": True,
            "history": [],
            "cars": []
        }
        save_data()

        global current_user
        current_user = login
        update_user()
        show_profile()

    tk.Button(
        frame,
        text="Зарегистрироваться",
        command=register,
        bg=ACCENT2,
        fg="white",
        font=("Arial", 11),
        height=1
    ).pack(fill="x")

    # Подвал с информацией
    tk.Label(
        frame,
        text="Используйте существующие учётные данные\nили создайте новую",
        bg=AUTH_BG,
        fg=SUB,
        font=("Arial", 9),
        justify="center"
    ).pack(pady=(20, 0))



def show_profile():
    if not current_user:
        return show_auth()

    clear()
    user = data["users"].get(current_user)

    if not user:
        return show_auth()

    frame = tk.Frame(content, bg=PROFILE_BG, padx=20, pady=20)
    frame.pack(fill="both", expand=True)

    create_header(frame)
    create_profile_body(frame, user)


# ===================== HEADER =====================

def show_main_menu():
    pass


def create_header(parent):
    header = tk.Frame(parent, bg=PANEL, height=60)
    header.pack(fill="x", pady=(0, 15))
    header.pack_propagate(False)

    tk.Label(
        header,
        text="Профиль пользователя",
        bg=PANEL,
        fg=ACCENT,
        font=("Arial", 16, "bold")
    ).pack(pady=10)

    tk.Button(
        header,
        text="← Назад",
        bg=DANGER,
        fg="white",
        command=lambda: (clear(), show_main_menu()),
        font=("Arial", 10)
    ).place(x=10, y=15)


# ===================== BODY =====================

def create_profile_body(parent, user):
    grid = tk.Frame(parent, bg=PROFILE_BG)
    grid.pack(fill="both", expand=True)

    left = create_left_column(grid, user)
    right = create_right_column(grid, user)

    left.grid(row=0, column=0, padx=(0, 20), sticky="n")
    right.grid(row=0, column=1, sticky="nsew")

    grid.columnconfigure(1, weight=1)


# ===================== LEFT =====================

def create_left_column(parent, user):
    frame = tk.Frame(parent, bg=CARD, padx=20, pady=20)

    name_var = tk.StringVar(value=user.get("name", ""))

    tk.Label(
        frame,
        text=current_user,
        bg=CARD,
        fg=TEXT,
        font=("Arial", 12, "bold")
    ).pack()

    tk.Label(frame, text="Имя:", bg=CARD, fg=SUB).pack(anchor="w", pady=(10, 0))

    name_entry = tk.Entry(frame, textvariable=name_var)
    name_entry.pack(fill="x")

    def save_name():
        user["name"] = name_var.get().strip()
        save_data()
        messagebox.showinfo("OK", "Имя обновлено")

    tk.Button(
        frame,
        text="Сохранить имя",
        command=save_name,
        bg=ACCENT,
        fg="white"
    ).pack(fill="x", pady=5)

    # -------- СМЕНА ПАРОЛЯ --------

    tk.Label(frame, text="Смена пароля", bg=CARD, fg=TEXT).pack(pady=(15, 5))

    old_p = tk.Entry(frame, show="*")
    new_p = tk.Entry(frame, show="*")

    old_p.pack(fill="x", pady=2)
    new_p.pack(fill="x", pady=2)

    def change_password():
        if not verify_password(user["password"], old_p.get()):
            return messagebox.showerror("Ошибка", "Старый пароль неверный")

        if len(new_p.get()) < 6:
            return messagebox.showerror("Ошибка", "Пароль слишком короткий")

        user["password"] = hash_password(new_p.get())
        save_data()
        messagebox.showinfo("OK", "Пароль изменён")

    tk.Button(
        frame,
        text="Изменить пароль",
        command=change_password,
        bg=ACCENT2
    ).pack(fill="x", pady=5)

    tk.Label(
        frame,
        text=f"Регистрация: {user.get('registration_date', '-')}",
        bg=CARD,
        fg=SUB
    ).pack(pady=10)

    return frame


# ===================== RIGHT =====================

def create_right_column(parent, user):
    frame = tk.Frame(parent, bg=CARD, padx=15, pady=15)

    # -------- УВЕДОМЛЕНИЯ --------

    notify_var = tk.BooleanVar(value=user.get("notifications", True))

    tk.Checkbutton(
        frame,
        text="Уведомления",
        variable=notify_var,
        bg=CARD,
        fg=TEXT
    ).pack(anchor="w")

    def save_notify():
        user["notifications"] = notify_var.get()
        save_data()
        messagebox.showinfo("OK", "Настройки сохранены")

    tk.Button(
        frame,
        text="Сохранить настройки",
        command=save_notify
    ).pack(fill="x", pady=5)

    # -------- ИСТОРИЯ --------

    tk.Label(
        frame,
        text="История заказов",
        bg=CARD,
        fg=TEXT,
        font=("Arial", 11, "bold")
    ).pack(anchor="w", pady=(10, 5))

    history_list = tk.Listbox(frame, height=6)
    history_list.pack(fill="x")

    def refresh_history(filter_text=""):
        history_list.delete(0, tk.END)
        for item in user.get("history", []):
            if filter_text.lower() in item.lower():
                history_list.insert(tk.END, item)

    refresh_history()

    search_entry = tk.Entry(frame)
    search_entry.pack(fill="x", pady=3)

    search_entry.bind(
        "<KeyRelease>",
        lambda e: refresh_history(search_entry.get())
    )

    def clear_history():
        if messagebox.askyesno("Очистка", "Удалить всю историю?"):
            user["history"] = []
            save_data()
            refresh_history()

    tk.Button(
        frame,
        text="Очистить историю",
        command=clear_history,
        bg=DANGER,
        fg="white"
    ).pack(fill="x", pady=5)

    # -------- АВТОМОБИЛИ --------

    tk.Label(
        frame,
        text="Автомобили",
        bg=CARD,
        fg=TEXT,
        font=("Arial", 11, "bold")
    ).pack(anchor="w", pady=(10, 5))

    cars_list = tk.Listbox(frame)
    cars_list.pack(fill="x")

    # гарантируем структуру
    if "cars" not in user:
        user["cars"] = []

    def refresh_cars():
        cars_list.delete(0, tk.END)
        for c in sorted(user["cars"], key=lambda x: int(x["year"])):
            cars_list.insert(tk.END, f"{c['make']} {c['model']} ({c['year']})")

    refresh_cars()

    def car_form(edit_index=None):
        win = tk.Toplevel(frame)
        win.geometry("300x250")

        makes = ["Toyota", "BMW", "Audi", "Ford"]
        make_var = tk.StringVar(value=makes[0])

        model_entry = tk.Entry(win)
        year_entry = tk.Entry(win)

        ttk.Combobox(win, textvariable=make_var, values=makes).pack(fill="x", pady=5)
        model_entry.pack(fill="x", pady=5)
        year_entry.pack(fill="x", pady=5)

        if edit_index is not None:
            car = user["cars"][edit_index]
            make_var.set(car["make"])
            model_entry.insert(0, car["model"])
            year_entry.insert(0, car["year"])

        def save():
            model = model_entry.get().strip()
            year = year_entry.get().strip()

            if not model or not year.isdigit():
                return messagebox.showerror("Ошибка", "Некорректные данные")

            car_data = {
                "make": make_var.get(),
                "model": model,
                "year": int(year)
            }

            if edit_index is None:
                user["cars"].append(car_data)
            else:
                user["cars"][edit_index] = car_data

            save_data()
            refresh_cars()
            win.destroy()

        tk.Button(win, text="Сохранить", command=save, bg=ACCENT).pack(pady=10)

    def edit_car():
        sel = cars_list.curselection()
        if not sel:
            return messagebox.showwarning("Ошибка", "Выберите авто")
        car_form(sel[0])

    def delete_car():
        sel = cars_list.curselection()
        if not sel:
            return

        if messagebox.askyesno("Удаление", "Удалить авто?"):
            user["cars"].pop(sel[0])
            save_data()
            refresh_cars()

    tk.Button(frame, text="Добавить", command=lambda: car_form(), bg=ACCENT2).pack(fill="x", pady=2)
    tk.Button(frame, text="Редактировать", command=edit_car).pack(fill="x", pady=2)
    tk.Button(frame, text="Удалить", command=delete_car, bg=DANGER, fg="white").pack(fill="x", pady=2)

    # -------- АККАУНТ --------

    def logout():
        global current_user
        current_user = None
        update_user()
        show_auth()

    def delete_account():
        if messagebox.askyesno("Удаление", "Удалить аккаунт навсегда?"):
            del data["users"][current_user]
            save_data()
            logout()

    tk.Button(frame, text="Выйти", command=logout, bg=DANGER, fg="white").pack(fill="x", pady=10)
    tk.Button(frame, text="Удалить аккаунт", command=delete_account, bg="#992222", fg="white").pack(fill="x")

    return frame
    # ===================== CALCULATOR =====================
def show_calc():
    clear()

    # Заголовок по центру
    tk.Label(content, text="Калькулятор расхода топлива",
              bg=BG, fg=TEXT,
              font=("Arial", 20, "bold")).pack(pady=20)

    # Основной фрейм калькулятора
    calc_frame = tk.Frame(content, bg=CARD, padx=40, pady=40)
    calc_frame.pack(fill="both", expand=True, padx=50, pady=30)

    # Настраиваем сетку для адаптивности
    for i in range(2):
        calc_frame.columnconfigure(i, weight=1)
    calc_frame.rowconfigure(6, weight=1)  # последняя строка растягивается

    # Переменная для выбора режима расчёта
    calc_mode = tk.StringVar(value="consumption")  # по умолчанию — расход

    # Кнопки для выбора режима (вместо радиокнопок)
    mode_frame = tk.Frame(calc_frame, bg=CARD)
    mode_frame.grid(row=0, column=0, columnspan=2, pady=(0, 20), sticky="ew")

    # Кнопка «Рассчитать средний расход на 100 км»
    consumption_btn = tk.Button(
        mode_frame,
        text="Рассчитать средний расход на 100 км",
        bg=ACCENT,
        fg="white",
        font=("Arial", 14),
        command=lambda: calc_mode.set("consumption"),
        padx=20,
        pady=10
    )
    consumption_btn.pack(side="left", fill="x", expand=True, padx=(0, 10))

    # Кнопка «Рассчитать стоимость поездки»
    cost_btn = tk.Button(
        mode_frame,
        text="Рассчитать стоимость поездки",
        bg=ACCENT,
        fg="white",
        font=("Arial", 14),
        command=lambda: calc_mode.set("cost"),
        padx=20,
        pady=10
    )
    cost_btn.pack(side="left", fill="x", expand=True, padx=(10, 0))

    # Метка пробега/расстояния — делаем её динамической
    mileage_label = tk.Label(calc_frame, text="Расстояние (км):", bg=CARD, fg=TEXT, font=("Arial", 14))
    mileage_label.grid(row=1, column=0, sticky="e", pady=15, padx=(0, 15))

    # Поле ввода пробега
    mileage_entry = tk.Entry(calc_frame, width=25, font=("Arial", 14), bg=PANEL, fg=TEXT)
    mileage_entry.grid(row=1, column=1, pady=15, sticky="w")

    # Метка израсходованного топлива — делаем её динамической
    fuel_label = tk.Label(calc_frame, text="Израсходовано топлива (л):", bg=CARD, fg=TEXT, font=("Arial", 14))
    fuel_label.grid(row=2, column=0, sticky="e", pady=15, padx=(0, 15))


    # Поле ввода израсходованного топлива
    fuel_entry = tk.Entry(calc_frame, width=25, font=("Arial", 14), bg=PANEL, fg=TEXT)
    fuel_entry.grid(row=2, column=1, pady=15, sticky="w")

    # Цена за литр
    tk.Label(calc_frame, text="Цена за литр (руб):", bg=CARD, fg=TEXT, font=("Arial", 14)).grid(
        row=3, column=0, sticky="e", pady=15, padx=(0, 15))
    price_entry = tk.Entry(calc_frame, width=25, font=("Arial", 14), bg=PANEL, fg=TEXT)
    price_entry.grid(row=3, column=1, pady=15, sticky="w")


    def update_labels():
        """Обновляет текст меток в зависимости от выбранного режима"""
        if calc_mode.get() == "cost":
            mileage_label.config(text="Расстояние (км):")
            fuel_label.config(text="Расход топлива (л):")
        else:
            mileage_label.config(text="Расстояние (км):")
            fuel_label.config(text="Израсходовано топлива (л):")

    # Привязываем обновление меток к изменению режима
    calc_mode.trace_add("write", lambda *args: update_labels())

    def calculate():
        try:
            mileage = float(mileage_entry.get().strip())
            fuel = float(fuel_entry.get().strip())
            price = float(price_entry.get().strip())

            if mileage <= 0 or fuel <= 0 or price <= 0:
                messagebox.showerror("Ошибка", "Все значения должны быть положительными числами больше нуля")
                return

            mode = calc_mode.get()
            result_text = ""

            if mode == "consumption":
                # Расход на 100 км
                consumption = (fuel / mileage) * 100
                result_text = f"Расход топлива: {consumption:.2f} л/100км"
            else:  # mode == "cost"
                # Общая стоимость
                total_cost = fuel * price
                result_text = f"Общая стоимость поездки: {total_cost:.2f} руб"

            result_label.config(text=result_text)

            # Сохраняем в историю, если пользователь авторизован
            if current_user:
                record = {
                    "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                    "mileage": mileage,
                    "fuel": fuel,
                    "price": price,
                    "mode": "Расход на 100 км" if mode == "consumption" else "Стоимость поездки",
                    "result": result_text
                }
                data["users"][current_user]["history"].append(record)
                save_data()

        except ValueError:
            messagebox.showerror(
                "Ошибка",
                "Введите корректные числовые значения во всех полях"
            )
        except Exception as e:
            messagebox.showerror("Ошибка", f"Произошла непредвиденная ошибка: {str(e)}")

    # Кнопка расчёта
    calc_btn = tk.Button(calc_frame, text="Рассчитать", bg=ACCENT, fg="white",
                        font=("Arial", 16, "bold"), command=calculate,
                        padx=30, pady=12)
    calc_btn.grid(row=4, column=0, columnspan=2, pady=25)

    # Фрейм для результата
    result_frame = tk.Frame(calc_frame, bg=CARD)
    result_frame.grid(row=5, column=0, columnspan=2, pady=20, sticky="ew")
    result_frame.columnconfigure(0, weight=1)

    # Метка результата
    result_label = tk.Label(result_frame, text="", bg=CARD, fg=ACCENT2,
                           font=("Arial", 16, "bold"))
    result_label.pack(padx=20, pady=15)

    # Дополнительная информация
    info_label = tk.Label(calc_frame,
                      text="Выберите режим расчёта и введите данные",
                      bg=CARD, fg=SUB, font=("Arial", 12))
    info_label.grid(row=6, column=0, columnspan=2, pady=(10, 0))


    # ===================== HISTORY =====================
def show_history():
        clear()

        tk.Label(content, text="История расчётов",
                 bg=BG, fg=TEXT,
                 font=("Arial", 20, "bold")).pack(pady=20)

        if not current_user or not data["users"][current_user].get("history"):
            tk.Label(content, text="История пуста", bg=BG, fg=SUB,
                     font=("Arial", 14)).pack()
            return

        history_frame = tk.Frame(content, bg=CARD)
        history_frame.pack(fill="both", expand=True, padx=20, pady=10)

        # Заголовки таблицы
        headers = ["Дата", "Пробег", "Топливо", "Цена", "Расход", "Стоимость"]
        for col, header in enumerate(headers):
            tk.Label(history_frame, text=header, bg=PANEL, fg=TEXT,
                     font=("Arial", 10, "bold"), padx=10).grid(row=0, column=col, sticky="ew")

        # Данные истории
        history = data["users"][current_user]["history"]
        for row, record in enumerate(history, start=1):
            values = [
                record["date"],
                f"{record['mileage']} км",
                f"{record['fuel']} л",
                f"{record['price']} руб",
                f"{record['consumption']:.2f} л/100км",
                f"{record['total_cost']:.2f} руб"
            ]
            for col, value in enumerate(values):
                tk.Label(history_frame, text=value, bg=CARD, fg=TEXT,
                         font=("Arial", 9)).grid(row=row, column=col, sticky="ew", padx=5, pady=5)

            # Кнопка для повторного использования расчёта
            def reuse(rec, mileage_entry, fuel_entry, price_entry):
                clear()
                show_calc()
                mileage_entry.insert(0, str(rec["mileage"]))
                fuel_entry.insert(0, str(rec["fuel"]))
                price_entry.insert(0, str(rec["price"]))

            tk.Button(history_frame, text="↻", bg=ACCENT2, fg="black",
                      command=reuse, font=("Arial", 8)).grid(row=row, column=len(values), padx=5)

        # Настройка растяжения колонок
        for col in range(len(headers)):
            history_frame.columnconfigure(col, weight=1)

    # ===================== SETTINGS =====================
def show_settings():
        clear()

        tk.Label(content, text="Настройки приложения",
                 bg=BG, fg=TEXT,
                 font=("Arial", 20, "bold")).pack(pady=20)

        settings_frame = tk.Frame(content, bg=CARD, padx=30, pady=30)
        settings_frame.pack(fill="both", expand=True, padx=20, pady=10)

        # Тема оформления
        theme_frame = tk.LabelFrame(settings_frame, text="Тема оформления", bg=CARD, fg=TEXT, padx=15, pady=15)
        theme_frame.pack(fill="x", pady=(0, 20))

        theme_var = tk.StringVar(value="Тёмная")
        themes = ["Тёмная", "Светлая", "Синяя"]

        for i, theme in enumerate(themes):
            tk.Radiobutton(theme_frame,
                           text=theme,
                           variable=theme_var,
                           value=theme,
                           bg=CARD,
                           fg=TEXT,
                           selectcolor=ACCENT,
                           activebackground=CARD,
                           font=("Arial", 10)
                           ).pack(anchor="w", pady=5)

        def apply_theme():
            theme = theme_var.get()

            # Сохраняем выбранную тему в данные пользователя
            if current_user:
                data["users"][current_user]["theme"] = theme
                save_data()

            # Применяем тему к интерфейсу
            if theme == "Тёмная":
                apply_dark_theme()
            elif theme == "Светлая":
                apply_light_theme()
            elif theme == "Синяя":
                apply_blue_theme()

            messagebox.showinfo("Тема", f"Тема '{theme}' успешно применена")

        apply_theme_btn = tk.Button(theme_frame, text="Применить тему", bg=ACCENT, fg="white",
                                    command=apply_theme, font=("Arial", 9), padx=10, pady=4)
        apply_theme_btn.pack(pady=10)

        # Настройки уведомлений
        notifications_settings = tk.LabelFrame(settings_frame, text="Уведомления", bg=CARD, fg=TEXT, padx=15, pady=15)
        notifications_settings.pack(fill="x", pady=(0, 20))

        notify_options = [
            ("О новых версиях приложения", True),
            ("О специальных предложениях", True),
            ("Напоминания о ТО", False),
            ("Новости автоиндустрии", False)
        ]
        notify_vars = {}

        for text, default in notify_options:
            var = tk.BooleanVar(value=default)
            notify_vars[text] = var
            tk.Checkbutton(notifications_settings,
                           text=text,
                           variable=var,
                           bg=CARD,
                           fg=TEXT,
                           selectcolor=ACCENT,
                           activebackground=CARD
                           ).pack(anchor="w", pady=3)

        def save_notifications():
            selected = [text for text, var in notify_vars.items() if var.get()]
            if current_user:
                data["users"][current_user]["notification_settings"] = selected
                save_data()
            messagebox.showinfo("Настройки сохранены",
                                f"Вы подписаны на: {', '.join(selected) if selected else 'ничего'}")

        save_notify_btn = tk.Button(notifications_settings, text="Сохранить настройки", bg=ACCENT2, fg="black",
                                    command=save_notifications, font=("Arial", 9), padx=10, pady=4)
        save_notify_btn.pack(pady=10)

        # Резервное копирование
        backup_frame = tk.LabelFrame(settings_frame, text="Резервное копирование", bg=CARD, fg=TEXT, padx=15, pady=15)
        backup_frame.pack(fill="x")

        tk.Label(backup_frame, text=f"Файл данных: {DATA_FILE}", bg=CARD, fg=SUB).pack(anchor="w")
        tk.Label(backup_frame, text=f"Бэкап: {BACKUP_FILE}", bg=CARD, fg=SUB).pack(anchor="w", pady=(0, 10))

        def create_manual_backup():
            create_backup()
            last_backup = datetime.fromtimestamp(os.path.getmtime(BACKUP_FILE)).strftime("%d.%m.%Y %H:%M")
            backup_status.config(text=f"Последний бэкап: {last_backup}")
            messagebox.showinfo("Бэкап", "Ручной бэкап создан успешно")

        backup_btn = tk.Button(backup_frame, text="Создать бэкап сейчас", bg=ACCENT, fg="white",
                               command=create_manual_backup, font=("Arial", 9), padx=10, pady=4)
        backup_btn.pack(anchor="w")

        last_backup = "Не создавался"
        if os.path.exists(BACKUP_FILE):
            last_backup = datetime.fromtimestamp(os.path.getmtime(BACKUP_FILE)).strftime("%d.%m.%Y %H:%M")
        backup_status = tk.Label(backup_frame, text=f"Последний бэкап: {last_backup}", bg=CARD, fg=SUB)
        backup_status.pack(anchor="w", pady=(10, 0))

    # Функции смены темы6:
def apply_dark_theme():
        global BG, PANEL, CARD, ACCENT, ACCENT2, TEXT, SUB, DANGER, PROFILE_BG, AVATAR_BG
        BG = "#0b1220"
        PANEL = "#111a2e"
        CARD = "#162238"
        ACCENT = "#4f8cff"
        ACCENT2 = "#22c55e"
        TEXT = "#e5e7eb"
        SUB = "#94a3b8"
        DANGER = "#ef4444"
        PROFILE_BG = "#162238"
        AVATAR_BG = "#253145"
        refresh_ui()

def apply_light_theme():
        global BG, PANEL, CARD, ACCENT, ACCENT2, TEXT, SUB, DANGER, PROFILE_BG, AVATAR_BG
        BG = "#f8fafc"
        PANEL = "#e2e8f0"
        CARD = "#ffffff"
        ACCENT = "#3b82f6"
        ACCENT2 = "#10b981"
        TEXT = "#1e293b"
        SUB = "#64748b"
        DANGER = "#dc2626"
        PROFILE_BG = "#ffffff"
        AVATAR_BG = "#e2e8f0"
        refresh_ui()

def apply_blue_theme():
        global BG, PANEL, CARD, ACCENT, ACCENT2, TEXT, SUB, DANGER, PROFILE_BG, AVATAR_BG
        BG = "#0c1b33"
        PANEL = "#1a2b4d"
        CARD = "#2a3f66"
        ACCENT = "#60a5fa"
        ACCENT2 = "#818cf8"
        TEXT = "#dbeafe"
        SUB = "#93c5fd"
        DANGER = "#fca5a5"
        PROFILE_BG = "#2a3f66"
        AVATAR_BG = "#384e77"
        refresh_ui()

def refresh_ui():
        """Обновляет цвета всех элементов интерфейса"""
        root.configure(bg=BG)
        header.configure(bg=BG)
        user_label.configure(bg=BG, fg=TEXT)
        content.configure(bg=BG)

        # Обновляем все фреймы и метки в content
        for widget in content.winfo_children():
            if isinstance(widget, tk.Frame):
                widget.configure(bg=CARD)
            elif isinstance(widget, tk.Label):
                if widget.cget("text") in ["Калькулятор расхода топлива", "История расчётов", "Настройки приложения",
                                           "О программе"]:
                    widget.configure(bg=BG, fg=ACCENT)
                else:
                    widget.configure(bg=CARD, fg=TEXT)

    # ===================== ABOUT =====================
def show_about():
        clear()

        about_frame = tk.Frame(content, bg=CARD, padx=40, pady=40)
        about_frame.pack(fill="both", expand=True, padx=20, pady=20)

        tk.Label(about_frame, text="CalculatCar",
                 bg=CARD, fg=ACCENT,
                 font=("Arial", 24, "bold")).pack(pady=(0, 10))
        tk.Label(about_frame, text="Версия 1.0.0",
                 bg=CARD, fg=SUB,
                 font=("Arial", 12)).pack(pady=(0, 20))

        features = [
            "Калькулятор расхода топлива",
            "История расчётов с возможностью повторного использования",
            "Персональный профиль с данными автомобиля",
            "Система уведомлений",
            "Резервное копирование данных"
        ]

        tk.Label(about_frame, text="Возможности:",
                 bg=CARD, fg=TEXT,
                 font=("Arial", 14, "bold")).pack(anchor="w", pady=(0, 5))

        for feature in features:
            tk.Label(about_frame, text=f"• {feature}",
                     bg=CARD, fg=SUB,
                     font=("Arial", 11)).pack(anchor="w", pady=2)

        tk.Label(about_frame, text="© 2024 CalculatCar. Все права защищены.",
                 bg=CARD, fg=SUB,
                 font=("Arial", 10)).pack(side="bottom", pady=(30, 0))

    # ===================== INITIALIZATION =====================
    # Загружаем данные при старте
load_data()
update_user()

    # Показываем калькулятор по умолчанию
show_calc()

    # Запускаем главный цикл
root.mainloop()
