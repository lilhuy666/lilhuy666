import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime
import json, os
import hashlib
import secrets

# ===================== STYLE =====================
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

    # Основной контейнер профиля с единым стилем
    profile_frame = tk.Frame(content, bg=PROFILE_BG, padx=20, pady=20)
    profile_frame.pack(padx=20, pady=20, fill="both", expand=True)

    # Заголовок профиля
    title_frame = tk.Frame(profile_frame, bg=PANEL, height=60)
    title_frame.pack(fill="x", pady=(0, 20))
    title_frame.pack_propagate(False)
    tk.Label(title_frame, text="Профиль пользователя",
             bg=PANEL, fg=ACCENT,
             font=("Arial", 16, "bold")).pack(pady=10)

    # Основная сетка профиля
    main_grid = tk.Frame(profile_frame, bg=PROFILE_BG)
    main_grid.pack(fill="both", expand=True)

    # Левая колонка — аватар и основная информация
    left_col = tk.Frame(main_grid, bg=CARD, padx=20, pady=20)
    left_col.grid(row=0, column=0, padx=(0, 20), sticky="nsw")

    # Аватар с закруглёнными углами
    avatar_frame = tk.Frame(left_col, bg=AVATAR_BG, width=120, height=120)
    avatar_frame.pack(pady=(0, 15))
    avatar_frame.pack_propagate(False)

    avatar = tk.Canvas(avatar_frame, width=110, height=110, bg=AVATAR_BG, bd=0, highlightthickness=0)
    avatar.pack(pady=5, padx=5)
    # Рисуем закруглённый прямоугольник (имитация аватара)
    avatar.create_oval(5, 5, 105, 105, fill="#4f8cff", outline="", width=0)

    # Имя и фамилия
    full_name = f"{user.get('name', 'Имя')} {user.get('surname', 'Фамилия')}"
    name_label = tk.Label(
        left_col,
        text=full_name,
        font=("Arial", 14, "bold"),
        fg=TEXT,
        bg=CARD
    )
    name_label.pack(pady=(0, 5))

    email_label = tk.Label(
        left_col,
        text=current_user,
        fg=SUB,
        bg=CARD,
        font=("Arial", 10)
    )
    email_label.pack()

    # Кнопка редактирования
    edit_btn = tk.Button(
        left_col,
        text="Редактировать профиль",
        bg=ACCENT,
        fg="white",
        padx=15,
        pady=8,
        font=("Arial", 10),
        relief="flat"
    )
    edit_btn.pack(pady=15)

    # Информация об автомобиле
    car_info = tk.LabelFrame(left_col, text="Автомобиль", bg=CARD, fg=TEXT, padx=10, pady=10)
    car_info.pack(fill="x", pady=10)

    tk.Label(car_info, text="Марка:", font=("Arial", 9, "bold"), bg=CARD, fg=SUB).grid(
        row=0, column=0, sticky="w", pady=3)
    make_var = tk.StringVar(value=user.get("car_make", ""))
    make_dropdown = ttk.Combobox(car_info, textvariable=make_var, values=[
        "Toyota", "Honda", "BMW", "Mercedes", "Audi", "Volkswagen", "Ford", "Kia"
    ], state="readonly")
    make_dropdown.grid(row=0, column=1, padx=5, pady=3, sticky="ew")

    tk.Label(car_info, text="Модель:", font=("Arial", 9, "bold"), bg=CARD, fg=SUB).grid(
        row=1, column=0, sticky="w", pady=3)
    model_var = tk.StringVar(value=user.get("car_model", ""))
    model_dropdown = ttk.Combobox(car_info, textvariable=model_var, values=[
        "Camry", "RAV4", "Corolla", "Civic", "Accord", "X5", "E-Class"
    ], state="readonly")
    model_dropdown.grid(row=1, column=1, padx=5, pady=3, sticky="ew")

    tk.Label(car_info, text="Год:", font=("Arial", 9, "bold"), bg=CARD, fg=SUB).grid(
        row=2, column=0, sticky="w", pady=3)
    year_var = tk.StringVar(value=user.get("car_year", ""))
    year_entry = tk.Entry(car_info, textvariable=year_var, width=8, bg=PANEL, fg=TEXT)
    year_entry.grid(row=2, column=1, padx=5, pady=3, sticky="w")

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

    save_car_btn = tk.Button(car_info, text="Сохранить", bg=ACCENT2, fg="black",
                            command=save_car_info, font=("Arial", 9), padx=8, pady=4)
    save_car_btn.grid(row=3, column=0, columnspan=2, pady=8)

    # Центральная колонка — личная информация
    center_col = tk.LabelFrame(main_grid, text="Личная информация", bg=CARD, fg=TEXT, padx=15, pady=15)
    center_col.grid(row=0, column=1, padx=20, sticky="nsew", fill="both")

    fields = [
        ("Имя:", "name", user.get("name", "")),
        ("Фамилия:", "surname", user.get("surname", "")),
        ("E-mail:", "email", current_user),
        ("Телефон:", "phone", user.get("phone", ""))
    ]
    entries = {}

    for i, (label_text, field_name, default_value) in enumerate(fields):
        tk.Label(center_col, text=label_text, font=("Arial", 9, "bold"), bg=CARD, fg=SUB).grid(
            row=i, column=0, sticky="w", pady=5)
        entry = tk.Entry(center_col, width=25, bg=PANEL, fg=TEXT, font=("Arial", 10))
        entry.insert(0, default_value)
        entry.grid(row=i, column=1, padx=10, pady=5)
        entries[field_name] = entry


    def save_personal_info():
        user["name"] = entries["name"].get().strip()
        user["surname"] = entries["surname"].get().strip()
        user["phone"] = entries["phone"].get().strip()


        # Базовая валидация данных
        if not user["name"]:
            messagebox.showerror("Ошибка", "Имя не может быть пустым")
            return

        if not user["surname"]:
            messagebox.showerror("Ошибка", "Фамилия не может быть пустой")
            return

        # Проверка формата телефона (упрощённая — минимум 7 цифр)
        phone = user["phone"]
        if phone and len([c for c in phone if c.isdigit()]) < 7:
            messagebox.showerror("Ошибка", "Номер телефона должен содержать минимум 7 цифр")
            return

        save_data()
        messagebox.showinfo("Успех", "Личные данные успешно обновлены")

        # Обновляем отображение имени в профиле
        full_name = f"{user['name']} {user['surname']}"
        name_label.config(text=full_name)

    save_personal_btn = tk.Button(center_col, text="Сохранить изменения", bg=ACCENT, fg="white",
                              command=save_personal_info, font=("Arial", 10), padx=12, pady=6)
    save_personal_btn.grid(row=len(fields), column=0, columnspan=2, pady=10)

    # Правая колонка — настройки профиля и уведомления
    right_col = tk.Frame(main_grid, bg=CARD, padx=15, pady=15)
    right_col.grid(row=0, column=2, padx=(20, 0), sticky="nsew")

    # Настройки уведомлений
    notifications_frame = tk.LabelFrame(right_col, text="Уведомления", bg=CARD, fg=TEXT, padx=10, pady=10)
    notifications_frame.pack(fill="x", pady=(0, 15))

    notify_var = tk.BooleanVar(value=user.get("notifications", True))

    notify_check = tk.Checkbutton(notifications_frame,
                               text="Получать уведомления о событиях",
                               variable=notify_var,
                               bg=CARD,
                               fg=TEXT,
                               selectcolor=ACCENT,
                               activebackground=CARD)
    notify_check.pack(anchor="w")

    def toggle_notifications():
        user["notifications"] = notify_var.get()
        save_data()
        status = "включены" if user["notifications"] else "отключены"
        messagebox.showinfo("Статус", f"Уведомления {status}")

    tk.Button(notifications_frame, text="Применить", bg=ACCENT2, fg="black",
              command=toggle_notifications, font=("Arial", 9), padx=8, pady=4).pack(pady=5)

    # Настройки безопасности
    security_frame = tk.LabelFrame(right_col, text="Безопасность", bg=CARD, fg=TEXT, padx=10, pady=10)
    security_frame.pack(fill="x")

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

            new_pass_1 = new_pass1.get().strip()
            new_pass_2 = new_pass2.get().strip()

            if not new_pass_1 or not new_pass_2:
                messagebox.showerror("Ошибка", "Поля с новым паролем не могут быть пустыми")
                return

            if new_pass_1 != new_pass_2:
                messagebox.showerror("Ошибка", "Пароли не совпадают")
                return

            if len(new_pass_1) < 6:
                messagebox.showerror("Ошибка", "Пароль должен содержать минимум 6 символов")
                return

            user["password"] = hash_password(new_pass_1)
            save_data()
            messagebox.showinfo("Успех", "Пароль успешно изменён")
            change_pass_window.destroy()


        tk.Button(change_pass_window, text="Сохранить", bg=ACCENT, fg="white", command=save_new_password).pack(pady=10)


    change_pass_btn = tk.Button(security_frame,
        text="Сменить пароль",
        bg="#63b3ed",
        fg="white",
        padx=10,
        pady=5,
        command=change_password
    )
    change_pass_btn.pack(fill="x", pady=(0, 5))

    def delete_account():
        global current_user  # Объявляем глобальную переменную В САМОМ НАЧАЛЕ функции

        # Проверяем, авторизован ли пользователь
        if not current_user:
            messagebox.showwarning("Предупреждение", "Нет активного пользователя для удаления")
            return

        # Проверяем, существует ли пользователь в данных
        if current_user not in data["users"]:
            messagebox.showerror("Ошибка", "Пользователь не найден в базе данных")
            return

        # Запрашиваем подтверждение удаления
        if messagebox.askyesno("Подтверждение",
                               "Вы уверены, что хотите удалить аккаунт?\nЭто действие нельзя отменить!"):
            try:
                # Удаляем пользователя из данных
                del data["users"][current_user]
                save_data()  # Сохраняем изменения в файл

                # Сбрасываем текущий пользовательский сеанс
                current_user = None
                update_user()  # Обновляем отображение статуса пользователя в интерфейсе
                show_profile()  # Показываем экран профиля (вероятно, для гостя)
                messagebox.showinfo("Аккаунт удалён", "Ваш аккаунт успешно удалён")

            except KeyError:
                messagebox.showerror("Ошибка", "Не удалось удалить аккаунт — пользователь не найден")
            except Exception as e:
                messagebox.showerror("Ошибка", f"Произошла непредвиденная ошибка: {str(e)}")

        delete_btn = tk.Button(security_frame,
                               text="Удалить аккаунт",
                               bg=DANGER,
                               fg="white",
                               padx=10,
                               pady=5,
                               command=delete_account
                               )
        delete_btn.pack(fill="x")

        def logout():
            """Выходит из аккаунта текущего пользователя"""
            global current_user
            current_user = None
            update_user()  # Обновляем отображение статуса пользователя
            show_profile()  # Показываем экран профиля (возможно, для гостя)
            messagebox.showinfo("Выход", "Вы успешно вышли из аккаунта")

        # Кнопка выхода из аккаунта
        logout_btn = tk.Button(right_col,
                               text="Выйти из аккаунта",
                               bg=DANGER,
                               fg="white",
                               padx=15,
                               pady=8,
                               font=("Arial", 10),
                               command=logout)  # Теперь ссылка на функцию существует
        logout_btn.pack(fill="x", pady=20)

        # Настройка растяжения колонок для адаптивности
        main_grid.columnconfigure(0, weight=1)  # левая колонка
        main_grid.columnconfigure(1, weight=2)  # центральная колонка (шире)
        main_grid.columnconfigure(2, weight=1)  # правая колонка

        # Обновление отображения информации при изменении данных
        def refresh_profile():
            # Обновляем имя в профиле
            full_name = f"{user.get('name', 'Имя')} {user.get('surname', 'Фамилия')}"
            name_label.config(text=full_name)
            # Обновляем email
            email_label.config(text=current_user)

        # Привязываем обновление к событиям изменения данных
        make_dropdown.bind("<<ComboboxSelected>>", lambda e: save_car_info())
        model_dropdown.bind("<<ComboboxSelected>>", lambda e: save_car_info())

    # ===================== CALCULATOR =====================
def show_calc():
        clear()

        tk.Label(content, text="Калькулятор расхода топлива",
                 bg=BG, fg=TEXT,
                 font=("Arial", 20, "bold")).pack(pady=20)

        calc_frame = tk.Frame(content, bg=CARD, padx=30, pady=30)
        calc_frame.pack(fill="both", expand=True, padx=20, pady=10)

        # Поля ввода
        tk.Label(calc_frame, text="Пробег (км):", bg=CARD, fg=TEXT, font=("Arial", 12)).grid(row=0, column=0,
                                                                                             sticky="w", pady=10)
        mileage_entry = tk.Entry(calc_frame, width=20, font=("Arial", 12), bg=PANEL, fg=TEXT)
        mileage_entry.grid(row=0, column=1, padx=10, pady=10)

        tk.Label(calc_frame, text="Расход топлива (л):", bg=CARD, fg=TEXT, font=("Arial", 12)).grid(row=1, column=0,
                                                                                                    sticky="w", pady=10)
        fuel_entry = tk.Entry(calc_frame, width=20, font=("Arial", 12), bg=PANEL, fg=TEXT)
        fuel_entry.grid(row=1, column=1, padx=10, pady=10)

        tk.Label(calc_frame, text="Цена за литр (руб):", bg=CARD, fg=TEXT, font=("Arial", 12)).grid(row=2, column=0,
                                                                                                    sticky="w", pady=10)
        price_entry = tk.Entry(calc_frame, width=20, font=("Arial", 12), bg=PANEL, fg=TEXT)
        price_entry.grid(row=2, column=1, padx=10, pady=10)

        def calculate():
            try:
                mileage = float(mileage_entry.get())
                fuel = float(fuel_entry.get())
                price = float(price_entry.get())

                if mileage <= 0 or fuel <= 0 or price <= 0:
                    messagebox.showerror("Ошибка", "Все значения должны быть положительными")
                    return

                consumption = (fuel / mileage) * 100  # л/100км
                total_cost = fuel * price

                result_text = f"Расход: {consumption:.2f} л/100км\nСтоимость: {total_cost:.2f} руб"
                result_label.config(text=result_text)

                # Сохраняем в историю
                if current_user:
                    record = {
                        "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                        "mileage": mileage,
                        "fuel": fuel,
                        "price": price,
                        "consumption": consumption,
                        "total_cost": total_cost
                    }
                    data["users"][current_user]["history"].append(record)
                    save_data()
            except ValueError:
                messagebox.showerror("Ошибка", "Введите корректные числовые значения")

        calc_btn = tk.Button(calc_frame, text="Рассчитать", bg=ACCENT, fg="white",
                             font=("Arial", 12, "bold"), command=calculate, padx=20, pady=8)
        calc_btn.grid(row=3, column=0, columnspan=2, pady=20)

        result_label = tk.Label(calc_frame, text="", bg=CARD, fg=ACCENT2, font=("Arial", 14, "bold"))
        result_label.grid(row=4, column=0, columnspan=2)

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
