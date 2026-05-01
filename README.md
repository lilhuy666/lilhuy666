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







    Exception in Tkinter callback
Traceback (most recent call last):
  File "C:\Users\Кирилл\AppData\Local\Programs\Python\Python312\Lib\tkinter\__init__.py", line 1968, in __call__
    return self.func(*args)
           ^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 126, in <lambda>
    command=lambda: [cmd(), close()]
                     ^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 774, in show_history
    f"{record['consumption']:.2f} л/100км",
       ~~~~~~^^^^^^^^^^^^^^^
KeyError: 'consumption'
Exception in Tkinter callback
Traceback (most recent call last):
  File "C:\Users\Кирилл\AppData\Local\Programs\Python\Python312\Lib\tkinter\__init__.py", line 1968, in __call__
    return self.func(*args)
           ^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 126, in <lambda>
    command=lambda: [cmd(), close()]
                     ^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 294, in show_profile
    create_profile_body(frame, user)
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 333, in create_profile_body
    right = create_right_column(grid, user)
            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 459, in create_right_column
    refresh_history()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 456, in refresh_history
    if filter_text.lower() in item.lower():
                              ^^^^^^^^^^
AttributeError: 'dict' object has no attribute 'lower'

