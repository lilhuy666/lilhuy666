    # ===================== PROFILE =====================
def update_user():
        """Обновляет отображение статуса пользователя в интерфейсе"""
        if current_user:
            user_label.config(text=f"Пользователь: {current_user}")
        else:
            user_label.config(text="Гость")


def show_auth():
    clear()  # Очищаем текущее содержимое экрана

    # Создаём основной фрейм для формы авторизации
    auth_frame = tk.Frame(content, bg=AUTH_BG, padx=40, pady=40)
    auth_frame.pack(fill="both", expand=True)

    # Заголовок формы
    tk.Label(
        auth_frame,
        text="Авторизация",
        bg=AUTH_BG,
        fg=ACCENT,
        font=("Arial", 18, "bold")
    ).pack(pady=(0, 30))

    # Поле для ввода логина
    tk.Label(auth_frame, text="Логин:", bg=AUTH_BG, fg=TEXT, font=("Arial", 12)).pack(anchor="w", pady=(10, 5))
    username_entry = tk.Entry(auth_frame, font=("Arial", 12), width=30)
    username_entry.pack(fill="x", pady=5)

    # Поле для ввода пароля
    tk.Label(auth_frame, text="Пароль:", bg=AUTH_BG, fg=TEXT, font=("Arial", 12)).pack(anchor="w", pady=(15, 5))
    password_entry = tk.Entry(auth_frame, show="*", font=("Arial", 12), width=30)
    password_entry.pack(fill="x", pady=5)

    # Функция обработки входа
    def login():
        username = username_entry.get().strip()
        password = password_entry.get()

        # Проверка на пустые поля
        if not username or not password:
            messagebox.showerror("Ошибка", "Заполните все поля")
            return

        # Проверка существования пользователя
        if username not in data["users"]:
            messagebox.showerror("Ошибка", "Пользователь не найден")
            return

        user = data["users"][username]

        # Проверка пароля
        if verify_password(user["password"], password):
            global current_user
            current_user = username
            update_user()
            show_profile()  # Переходим к профилю после успешного входа
        else:
            messagebox.showerror("Ошибка", "Неверный пароль")

    # Кнопка входа
    login_btn = tk.Button(
        auth_frame,
        text="Войти",
        command=login,
        bg=ACCENT,
        fg="white",
        font=("Arial", 12, "bold"),
        height=2
    )
    login_btn.pack(pady=20, fill="x")

    # Кнопка регистрации (переход к форме регистрации)
    register_btn = tk.Button(
        auth_frame,
        text="Зарегистрироваться",
        command=show_registration,  # Предполагаем, что функция show_registration существует
        bg=ACCENT2,
        fg="white",
        font=("Arial", 11),
        height=1
    )
    register_btn.pack(pady=10, fill="x")

    # Фокусируем на поле логина при открытии формы
    username_entry.focus_set()

def show_registration():
    clear()  # Очищаем текущее содержимое экрана

    # Создаём основной фрейм для формы регистрации
    reg_frame = tk.Frame(content, bg=AUTH_BG, padx=40, pady=40)
    reg_frame.pack(fill="both", expand=True)

    # Заголовок формы
    tk.Label(
        reg_frame,
        text="Регистрация",
        bg=AUTH_BG,
        fg=ACCENT,
        font=("Arial", 18, "bold")
    ).pack(pady=(0, 30))

    # Поле для ввода логина
    tk.Label(reg_frame, text="Логин:", bg=AUTH_BG, fg=TEXT, font=("Arial", 12)).pack(anchor="w", pady=(10, 5))
    username_entry = tk.Entry(reg_frame, font=("Arial", 12), width=30)
    username_entry.pack(fill="x", pady=5)


    # Поле для ввода пароля
    tk.Label(reg_frame, text="Пароль:", bg=AUTH_BG, fg=TEXT, font=("Arial", 12)).pack(anchor="w", pady=(15, 5))
    password_entry = tk.Entry(reg_frame, show="*", font=("Arial", 12), width=30)
    password_entry.pack(fill="x", pady=5)

    # Поле для подтверждения пароля
    tk.Label(reg_frame, text="Подтвердите пароль:", bg=AUTH_BG, fg=TEXT, font=("Arial", 12)).pack(anchor="w", pady=(15, 5))
    confirm_password_entry = tk.Entry(reg_frame, show="*", font=("Arial", 12), width=30)
    confirm_password_entry.pack(fill="x", pady=5)

    def register():
        username = username_entry.get().strip()
        password = password_entry.get()
        confirm_password = confirm_password_entry.get()

        # Проверка на пустые поля
        if not username or not password or not confirm_password:
            messagebox.showerror("Ошибка", "Заполните все поля")
            return

        # Проверка совпадения паролей
        if password != confirm_password:
            messagebox.showerror("Ошибка", "Пароли не совпадают")
            return

        # Проверка длины пароля
        if len(password) < 6:
            messagebox.showerror("Ошибка", "Пароль должен содержать минимум 6 символов")
            return

        # Проверка существования пользователя
        if username in data["users"]:
            messagebox.showerror("Ошибка", "Пользователь с таким логином уже существует")
            return

        # Создание нового пользователя
        data["users"][username] = {
            "password": hash_password(password),
            "name": username,  # по умолчанию имя = логин
            "registration_date": datetime.now().strftime("%d.%m.%Y"),
            "notifications": True,
            "history": [],
            "cars": []
        }

        save_data()  # Сохраняем данные

        messagebox.showinfo("Успех", "Аккаунт создан! Теперь войдите в систему.")
        show_auth()  # Возвращаемся к форме авторизации

    # Кнопка регистрации
    register_btn = tk.Button(
        reg_frame,
        text="Зарегистрироваться",
        command=register,
        bg=ACCENT2,
        fg="white",
        font=("Arial", 12, "bold"),
        height=2
    )
    register_btn.pack(pady=20, fill="x")

    # Кнопка «Назад» к авторизации
    back_btn = tk.Button(
        reg_frame,
        text="← Назад к авторизации",
        command=show_auth,
        bg=DANGER,
        fg="white",
        font=("Arial", 11),
        height=1
    )
    back_btn.pack(pady=10, fill="x")

    # Фокусируем на поле логина при открытии формы
    username_entry.focus_set()


def show_profile():
    if not current_user:
        return show_auth()

    clear()
    user = data["users"][current_user]

    frame = tk.Frame(content, bg=PROFILE_BG, padx=20, pady=20)
    frame.pack(fill="both", expand=True)

    create_header(frame)
    create_profile_body(frame, user)

def show_cars():
    clear()  # Очищаем текущее содержимое экрана

    # Создаём основной фрейм для раздела автомобилей
    cars_frame = tk.Frame(content, bg=CARD, padx=30, pady=30)
    cars_frame.pack(fill="both", expand=True)


    # Заголовок раздела
    tk.Label(
        cars_frame,
        text="Мои автомобили",
        bg=CARD,
        fg=ACCENT,
        font=("Arial", 18, "bold")
    ).pack(pady=(0, 20))

    # Получаем данные текущего пользователя
    user = data["users"][current_user]

    # Список автомобилей
    cars_list = tk.Listbox(cars_frame, height=8, font=("Arial", 11))
    cars_list.pack(fill="x", pady=(0, 15))

    # Обновляем список автомобилей
    def refresh_cars():
        cars_list.delete(0, tk.END)
        for car in user.get("cars", []):
            cars_list.insert(tk.END, f"{car['model']} ({car['year']})")

    refresh_cars()

    # Кнопки управления
    button_frame = tk.Frame(cars_frame, bg=CARD)
    button_frame.pack(fill="x")

    # Кнопка добавления автомобиля
    add_btn = tk.Button(
        button_frame,
        text="Добавить автомобиль",
        command=lambda: car_form(),
        bg=ACCENT,
        fg="white",
        font=("Arial", 11)
    )
    add_btn.pack(side="left", padx=(0, 10))

    # Кнопка редактирования
    edit_btn = tk.Button(
        button_frame,
        text="Редактировать",
        command=edit_car,
        bg=ACCENT2,
        fg="white",
        font=("Arial", 11)
    )
    edit_btn.pack(side="left", padx=(0, 10))

    # Кнопка удаления
    delete_btn = tk.Button(
        button_frame,
        text="Удалить",
        command=delete_car,
        bg=DANGER,
        fg="white",
        font=("Arial", 11)
    )
    delete_btn.pack(side="left")

    # Кнопка «Назад» в главное меню
    back_btn = tk.Button(
        cars_frame,
        text="← Назад в меню",
        command=show_main_menu,
        bg=PANEL,
        fg=TEXT,
        font=("Arial", 10)
    )
    back_btn.pack(pady=20)

    # Форма добавления/редактирования автомобиля
    def car_form(edit_index=None):
        # Создаём новое окно для формы
        form_window = tk.Toplevel()
        form_window.title("Автомобиль" if edit_index is None else "Редактировать автомобиль")
        form_window.geometry("300x200")
        form_window.resizable(False, False)

        # Поля формы
        tk.Label(form_window, text="Модель:").grid(row=0, column=0, sticky="w", pady=5)
        model_entry = tk.Entry(form_window)
        model_entry.grid(row=0, column=1, pady=5, padx=(10, 0))

        tk.Label(form_window, text="Год:").grid(row=1, column=0, sticky="w", pady=5)
        year_entry = tk.Entry(form_window)
        year_entry.grid(row=1, column=1, pady=5, padx=(10, 0))

        if edit_index is not None:  # Если редактируем существующий автомобиль
            car = user["cars"][edit_index]
            model_entry.insert(0, car["model"])
            year_entry.insert(0, car["year"])

        def save():
            model = model_entry.get().strip()
            year = year_entry.get().strip()

            # Валидация
            if not model or not year:
                messagebox.showerror("Ошибка", "Заполните все поля")
                return
            if not year.isdigit():
                messagebox.showerror("Ошибка", "Год должен быть числом")
                return

            year_num = int(year)
            if year_num < 1900 or year_num > 2030:
                messagebox.showerror("Ошибка", "Введите корректный год (1900–2030)")
                return

            car_data = {"model": model, "year": year_num}

            if edit_index is not None:
                user["cars"][edit_index] = car_data
            else:
                user["cars"].append(car_data)

            save_data()
            refresh_cars()
            form_window.destroy()

        # Кнопки формы
        tk.Button(form_window, text="Сохранить", command=save, bg=ACCENT, fg="white").grid(row=2, column=0, pady=10)
        tk.Button(form_window, text="Отмена", command=form_window.destroy, bg=PANEL).grid(row=2, column=1, pady=10)

def edit_car():
        sel = cars_list.curselection()
        if not sel:
            messagebox.showwarning("Предупреждение", "Выберите автомобиль для редактирования")
            return
        car_form(sel[0])

def delete_car():
        sel = cars_list.curselection()
        if not sel:
            messagebox.showwarning("Предупреждение", "Выберите автомобиль для удаления")
            return

        response = messagebox.askyesno("Подтверждение", "Удалить выбранный автомобиль?")
        if response:
            del user["cars"][sel[0]]
            save_data()
            refresh_cars()
            
def show_orders():
    clear()  # Очищаем текущее содержимое экрана

    # Создаём основной фрейм для раздела заказов
    orders_frame = tk.Frame(content, bg=CARD, padx=30, pady=30)
    orders_frame.pack(fill="both", expand=True)

    # Заголовок раздела
    tk.Label(
        orders_frame,
        text="Мои заказы",
        bg=CARD,
        fg=ACCENT,
        font=("Arial", 18, "bold")
    ).pack(pady=(0, 20))

    # Получаем данные текущего пользователя
    user = data["users"][current_user]

    # Список заказов (используем Listbox для простоты)
    orders_list = tk.Listbox(orders_frame, height=10, font=("Arial", 11))
    orders_list.pack(fill="x", pady=(0, 15))

    # Обновляем список заказов
    def refresh_orders():
        orders_list.delete(0, tk.END)
        orders = user.get("orders", [])
        if not orders:
            orders_list.insert(tk.END, "У вас пока нет заказов")
        else:
            for i, order in enumerate(orders, 1):
                status = order.get("status", "В обработке")
                car_model = order.get("car_model", "Не указан")
                date = order.get("date", "Дата не указана")
                orders_list.insert(
                    tk.END,
            f"{i}. {car_model} — {status} ({date})"
        )

    refresh_orders()

    # Кнопки управления
    button_frame = tk.Frame(orders_frame, bg=CARD)
    button_frame.pack(fill="x")

    # Кнопка создания нового заказа
    new_order_btn = tk.Button(
        button_frame,
        text="Новый заказ",
        command=create_order,
        bg=ACCENT,
        fg="white",
        font=("Arial", 11)
    )
    new_order_btn.pack(side="left", padx=(0, 10))

    # Кнопка просмотра деталей заказа
    view_btn = tk.Button(
        button_frame,
        text="Подробнее",
        command=view_order_details,
        bg=ACCENT2,
        fg="white",
        font=("Arial", 11)
    )
    view_btn.pack(side="left", padx=(0, 10))

    # Кнопка отмены заказа
    cancel_btn = tk.Button(
        button_frame,
        text="Отменить заказ",
        command=cancel_order,
        bg=DANGER,
        fg="white",
        font=("Arial", 11)
    )
    cancel_btn.pack(side="left")

    # Кнопка «Назад» в главное меню
    back_btn = tk.Button(
        orders_frame,
        text="← Назад в меню",
        command=show_main_menu,
        bg=PANEL,
        fg=TEXT,
        font=("Arial", 10)
    )
    back_btn.pack(pady=20)

    # Функции для работы с заказами
    def create_order():
        # Создаём окно для создания заказа
        order_window = tk.Toplevel()
        order_window.title("Новый заказ")
        order_window.geometry("400x300")
        order_window.resizable(False, False)

        # Выбор автомобиля
        tk.Label(order_window, text="Выберите автомобиль:").grid(row=0, column=0, sticky="w", pady=5)
        car_var = tk.StringVar()
        car_choices = [f"{car['model']} ({car['year']})" for car in user.get("cars", [])]
        if car_choices:
            car_menu = ttk.Combobox(order_window, textvariable=car_var, values=car_choices)
            car_menu.grid(row=0, column=1, pady=5, padx=(10, 0))
        else:
            tk.Label(order_window, text="Нет доступных автомобилей", fg="red").grid(row=0, column=1)

        # Тип услуги
        tk.Label(order_window, text="Тип услуги:").grid(row=1, column=0, sticky="w", pady=5)
        service_var = tk.StringVar(value="Техническое обслуживание")
        services = ["Техническое обслуживание", "Ремонт", "Шиномонтаж", "Диагностика"]
        service_menu = ttk.Combobox(order_window, textvariable=service_var, values=services)
        service_menu.grid(row=1, column=1, pady=5, padx=(10, 0))

        def save_order():
            car_choice = car_var.get()
            service = service_var.get()

            if not car_choice or not service:
                messagebox.showerror("Ошибка", "Заполните все поля")
                return

            # Создаём новый заказ
            new_order = {
                "car_model": car_choice,
                "service": service,
                "status": "В обработке",
                "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                "order_id": len(user.get("orders", [])) + 1
            }

            if "orders" not in user:
                user["orders"] = []
            user["orders"].append(new_order)
            save_data()
            refresh_orders()
            order_window.destroy()
            messagebox.showinfo("Успех", "Заказ создан!")

        # Кнопки формы
        tk.Button(order_window, text="Создать заказ", command=save_order, bg=ACCENT, fg="white").grid(row=2, column=0, pady=15)
        tk.Button(order_window, text="Отмена", command=order_window.destroy, bg=PANEL).grid(row=2, column=1, pady=15)


    def view_order_details():
        sel = orders_list.curselection()
        if not sel:
            messagebox.showwarning("Предупреждение", "Выберите заказ для просмотра")
            return

        order = user["orders"][sel[0]]
        details = f"""
Детали заказа #{order.get('order_id', 'N/A')}

Автомобиль: {order.get('car_model', 'Не указан')}
Услуга: {order.get('service', 'Не указана')}
Статус: {order.get('status', 'В обработке')}
Дата создания: {order.get('date', 'Не указана')}
"""
        messagebox.showinfo("Детали заказа", details)


    def cancel_order():
        sel = orders_list.curselection()
        if not sel:
            messagebox.showwarning("Предупреждение", "Выберите заказ для отмены")
            return

        response = messagebox.askyesno("Подтверждение", "Отменить выбранный заказ?")
        if response:
            del user["orders"][sel[0]]
            save_data()
            refresh_orders()

# ===================== HEADER =====================
def show_main_menu():
    clear()  # Очищаем текущее содержимое экрана

    # Создаём основной фрейм для главного меню
    menu_frame = tk.Frame(content, bg=MAIN_BG, padx=40, pady=40)
    menu_frame.pack(fill="both", expand=True)


    # Заголовок главного меню
    tk.Label(
        menu_frame,
        text="Главное меню",
        bg=MAIN_BG,
        fg=ACCENT,
        font=("Arial", 18, "bold")
    ).pack(pady=(0, 40))

    # Кнопка перехода к профилю
    profile_btn = tk.Button(
        menu_frame,
        text="Мой профиль",
        command=show_profile,
        bg=ACCENT,
        fg="white",
        font=("Arial", 14, "bold"),
        height=2
    )
    profile_btn.pack(fill="x", pady=10)

    # Кнопка для работы с заказами
    orders_btn = tk.Button(
        menu_frame,
        text="Мои заказы",
        command=show_orders,  # Предполагаем, что функция show_orders существует
        bg=ACCENT2,
        fg="white",
        font=("Arial", 14),
        height=2
    )
    orders_btn.pack(fill="x", pady=10)

    # Кнопка для управления автомобилями
    cars_btn = tk.Button(
        menu_frame,
        text="Мои автомобили",
        command=show_cars,  # Предполагаем, что функция show_cars существует
        bg=ACCENT2,
        fg="white",
        font=("Arial", 14),
        height=2
    )
    cars_btn.pack(fill="x", pady=10)

    # Кнопка настроек
    settings_btn = tk.Button(
        menu_frame,
        text="Настройки",
        command=show_settings,  # Предполагаем, что функция show_settings существует
        bg=ACCENT2,
        fg="white",
        font=("Arial", 14),
        height=2
    )
    settings_btn.pack(fill="x", pady=10)





def create_header(parent):
    header = tk.Frame(parent, bg=PANEL, height=60)
    header.pack(fill="x", pady=(0, 15))
    header.pack_propagate(False)

    tk.Label(header, text="Профиль пользователя",
             bg=PANEL, fg=ACCENT,
             font=("Arial", 16, "bold")).pack(pady=10)

    tk.Button(header, text="← Назад",
              bg=DANGER, fg="white",
              command=lambda: (clear(), show_main_menu()),
              font=("Arial", 10)).place(x=10, y=15)


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

    tk.Label(frame, text=current_user,
             bg=CARD, fg=TEXT,
             font=("Arial", 12, "bold")).pack()

    tk.Label(frame, text="Имя:",
             bg=CARD, fg=SUB).pack(anchor="w", pady=(10, 0))

    name_entry = tk.Entry(frame, textvariable=name_var)
    name_entry.pack(fill="x")

    def save_name():
        user["name"] = name_var.get()
        save_data()
        messagebox.showinfo("OK", "Имя обновлено")

    tk.Button(frame, text="Сохранить имя",
              command=save_name,
              bg=ACCENT, fg="white").pack(fill="x", pady=5)

    # Смена пароля
    tk.Label(frame, text="Смена пароля",
             bg=CARD, fg=TEXT).pack(pady=(15, 5))

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

    tk.Button(frame, text="Изменить пароль",
              command=change_password,
              bg=ACCENT2).pack(fill="x", pady=5)

    tk.Label(frame,
             text=f"Регистрация: {user.get('registration_date','-')}",
             bg=CARD, fg=SUB).pack(pady=10)

    return frame


# ===================== RIGHT =====================
def create_right_column(parent, user):
    frame = tk.Frame(parent, bg=CARD, padx=15, pady=15)

    # ---------- УВЕДОМЛЕНИЯ ----------
    notify_var = tk.BooleanVar(value=user.get("notifications", True))

    tk.Checkbutton(frame,
                   text="Уведомления",
                   variable=notify_var,
                   bg=CARD, fg=TEXT).pack(anchor="w")

    def save_notify():
        user["notifications"] = notify_var.get()
        save_data()

    tk.Button(frame, text="Сохранить настройки",
              command=save_notify).pack(fill="x", pady=5)

    # ---------- ИСТОРИЯ ----------
    tk.Label(frame, text="История заказов",
             bg=CARD, fg=TEXT,
             font=("Arial", 11, "bold")).pack(anchor="w", pady=(10, 5))

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

    search_entry.bind("<KeyRelease>",
                      lambda e: refresh_history(search_entry.get()))

    def clear_history():
        if messagebox.askyesno("Очистка", "Удалить всю историю?"):
            user["history"] = []
            save_data()
            refresh_history()

    tk.Button(frame, text="Очистить историю",
              command=clear_history,
              bg=DANGER, fg="white").pack(fill="x", pady=5)

    # ---------- АВТО ----------
    tk.Label(frame, text="Автомобили",
             bg=CARD, fg=TEXT,
             font=("Arial", 11, "bold")).pack(anchor="w", pady=(10, 5))

    cars_list = tk.Listbox(frame)
    cars_list.pack(fill="x")

    def refresh_cars():
        cars_list.delete(0, tk.END)
        for c in sorted(user.get("cars", []), key=lambda x: x["year"]):
            cars_list.insert(tk.END, f"{c['make']} {c['model']} ({c['year']})")

    refresh_cars()

    def car_form(edit_index=None):
        win = tk.Toplevel(frame)
        win.geometry("300x250")

        makes = ["Toyota", "BMW", "Audi", "Ford"]
        make_var = tk.StringVar(value=makes[0])

        model = tk.Entry(win)
        year = tk.Entry(win)

        ttk.Combobox(win, textvariable=make_var,
                     values=makes).pack(fill="x", pady=5)
        model.pack(fill="x", pady=5)
        year.pack(fill="x", pady=5)

        if edit_index is not None:
            car = user["cars"][edit_index]
            make_var.set(car["make"])
            model.insert(0, car["model"])
            year.insert(0, car["year"])

        def save():
            if not year.get().isdigit():
                return

            car_data = {
                "make": make_var.get(),
                "model": model.get(),
                "year": year.get()
            }

            if edit_index is None:
                user["cars"].append(car_data)
            else:
                user["cars"][edit_index] = car_data

            save_data()
            refresh_cars()
            win.destroy()

        tk.Button(win, text="Сохранить",
                  command=save,
                  bg=ACCENT).pack(pady=10)

    def delete_car():
        sel = cars_list.curselection()
        if not sel:
            return
        if messagebox.askyesno("Удаление", "Удалить авто?"):
            user["cars"].pop(sel[0])
            save_data()
            refresh_cars()

    def edit_car():
        sel = cars_list.curselection()
        if sel:
            car_form(sel[0])

    tk.Button(frame, text="Добавить",
              command=lambda: car_form(),
              bg=ACCENT2).pack(fill="x", pady=2)

    tk.Button(frame, text="Редактировать",
              command=edit_car).pack(fill="x", pady=2)

    tk.Button(frame, text="Удалить",
              command=delete_car,
              bg=DANGER, fg="white").pack(fill="x", pady=2)

    # ---------- АККАУНТ ----------
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

    tk.Button(frame, text="Выйти",
              command=logout,
              bg=DANGER, fg="white").pack(fill="x", pady=10)

    tk.Button(frame, text="Удалить аккаунт",
              command=delete_account,
              bg="#992222", fg="white").pack(fill="x")

    return frame
