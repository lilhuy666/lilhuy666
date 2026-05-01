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
            return show_auth()

        user = data["users"][current_user]

        profile_frame = tk.Frame(content, bg=PROFILE_BG, padx=20, pady=20)
        profile_frame.pack(fill="both", expand=True)

        create_header(profile_frame)
        create_profile_body(profile_frame, user)

    # -------------------------
    # 🔐 Авторизация
    # -------------------------
def show_auth():
        c = card()

        tk.Label(c, text="Вход / Регистрация",
                 bg=CARD, fg=TEXT,
                 font=("Arial", 20, "bold")).pack(pady=10)

        # Поле email с подсказкой
        email_frame = tk.Frame(c, bg=CARD)
        email_frame.pack(fill="x", pady=8)
        tk.Label(email_frame, text="Email:", bg=CARD, fg=SUB, font=("Arial", 10)).pack(anchor="w")
        email = tk.Entry(email_frame, font=("Arial", 14))
        email.pack(fill="x")

        # Поле пароля с подсказкой
        password_frame = tk.Frame(c, bg=CARD)
        password_frame.pack(fill="x", pady=8)
        tk.Label(password_frame, text="Пароль:", bg=CARD, fg=SUB, font=("Arial", 10)).pack(anchor="w")
        password = tk.Entry(password_frame, show="*", font=("Arial", 14))
        password.pack(fill="x")

        def login():
            global current_user
            e, p = email.get().strip(), password.get().strip()

            if not e or not p:
                return messagebox.showerror("Ошибка", "Заполните все поля")

            if e in data["users"] and verify_password(data["users"][e]["password"], p):
                current_user = e
                update_user()
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные для входа")

        def register():
            e, p = email.get().strip(), password.get().strip()

            if not e or not p:
                return messagebox.showerror("Ошибка", "Заполните все поля")

            # Проверка формата email
            if "@" not in e or "." not in e:
                return messagebox.showerror("Ошибка", "Неверный формат email")

            if len(p) < 6:
                return messagebox.showerror("Ошибка", "Пароль должен содержать минимум 6 символов")

            if e in data["users"]:
                return messagebox.showerror("Ошибка", "Пользователь с таким email уже существует")

            data["users"][e] = {
                "password": hash_password(p),
                "cars": [],  # Список автомобилей
                "notifications": True,
                "history": [],
                "registration_date": datetime.now().strftime("%d.%m.%Y")
            }

            save_data()
            messagebox.showinfo("Успех", "Аккаунт успешно создан! Теперь войдите в систему")
            login()  # Автоматический вход после регистрации

        login_btn = tk.Button(c, text="Войти", bg=ACCENT, fg="white", command=login,
                              font=("Arial", 12, "bold"), height=2)
        login_btn.pack(fill="x", pady=(10, 5))

        register_btn = tk.Button(c, text="Регистрация", bg=ACCENT2, fg="black", command=register,
                                 font=("Arial", 12), height=2)
        register_btn.pack(fill="x", pady=(0, 10))

    # -------------------------
    # 🧱 Header
    # -------------------------
def show_main_menu():
        """Отображает главное меню приложения"""
        clear()  # Очищаем текущее содержимое интерфейса

        # Создаём основной фрейм для главного меню
        main_frame = tk.Frame(content, bg=BG, padx=20, pady=20)
        main_frame.pack(fill="both", expand=True)

        # Заголовок главного меню
        title_label = tk.Label(
            main_frame,
            text="Главное меню",
            bg=BG,
            fg=ACCENT,
            font=("Arial", 24, "bold")
        )
        title_label.pack(pady=(0, 30))

        # Список кнопок меню: текст и соответствующая функция
        menu_buttons = [
            ("Профиль", show_profile),
            ("История заказов", show_history),
            ("Настройки приложения", show_settings),
        ]

        # Создаём и размещаем кнопки
        for button_text, button_command in menu_buttons:
            button = tk.Button(
                main_frame,
                text=button_text,
                bg=ACCENT,
                fg="white",
                command=button_command,
                font=("Arial", 14),
                height=2,
                relief="flat",
                cursor="hand2"
            )
            # Добавляем эффект наведения (изменение цвета при наведении)
            button.bind(
                "<Enter>",
                lambda e, btn=button: btn.config(bg=ACCENT2)
            )
            button.bind(
                "<Leave>",
                lambda e, btn=button: btn.config(bg=ACCENT)
            )
            button.pack(fill="x", pady=8, ipady=5)

        # Дополнительная информация о пользователе (если авторизован)
        if current_user:
            user_info = tk.Label(
                main_frame,
                text=f"Добро пожаловать, {current_user}!",
                bg=BG,
                fg=SUB,
                font=("Arial", 10)
            )
            user_info.pack(pady=(20, 0))

def create_header(parent):
        header = tk.Frame(parent, bg=PANEL, height=60)
        header.pack(fill="x", pady=(0, 20))
        header.pack_propagate(False)

        title = tk.Label(header, text="Профиль пользователя",
                         bg=PANEL, fg=ACCENT,
                         font=("Arial", 16, "bold"))
        title.pack(pady=10)

        # Кнопка «Назад» в шапке — теперь вызывает существующую функцию
        back_btn = tk.Button(header, text="← Назад", bg=DANGER, fg="white",
                             command=lambda: (clear(), show_main_menu()),
                             font=("Arial", 10), width=8)
        back_btn.place(x=10, y=15)

def create_right_column(parent, user):
        """Создаёт правую колонку профиля с настройками и действиями"""
        frame = tk.Frame(parent, bg=CARD, padx=15, pady=15)

        # --- Блок уведомлений ---
        notify_frame = tk.LabelFrame(frame, text="Настройки уведомлений", bg=CARD, fg=TEXT,
                                     font=("Arial", 10, "bold"))
        notify_frame.pack(fill="x", pady=(0, 20))

        var = tk.BooleanVar(value=user.get("notifications", True))

        notify_cb = tk.Checkbutton(notify_frame, text="Получать уведомления",
                                   variable=var, bg=CARD, fg=TEXT,
                                   font=("Arial", 11))
        notify_cb.pack(anchor="w", padx=5, pady=5)

        def save_notify():
            user["notifications"] = var.get()
            save_data()
            messagebox.showinfo("Успех", "Настройки уведомлений сохранены")

        tk.Button(notify_frame, text="Применить", bg=ACCENT, fg="white",
                  command=save_notify, font=("Arial", 9)).pack(pady=5)

        # --- Конец блока уведомлений ---

        # --- Блок истории заказов ---
        history_frame = tk.LabelFrame(frame, text="История заказов", bg=CARD, fg=TEXT,
                                      font=("Arial", 10, "bold"))
        history_frame.pack(fill="x", pady=(0, 20))

        history = user.get("history", [])
        if history:
            history_label = tk.Label(history_frame, text=f"Всего заказов: {len(history)}",
                                     bg=CARD, fg=SUB, font=("Arial", 10))
            history_label.pack(anchor="w", padx=5, pady=2)

            last_order = history[-1] if history else "Нет данных"
            last_label = tk.Label(history_frame, text=f"Последний: {last_order}",
                                  bg=CARD, fg=SUB, font=("Arial", 9))
            last_label.pack(anchor="w", padx=5, pady=2)
        else:
            no_history = tk.Label(history_frame, text="История пуста",
                                  bg=CARD, fg=SUB, font=("Arial", 10))
            no_history.pack(padx=5, pady=10)
        # --- Конец блока истории заказов ---

        # --- Блок управления автомобилями ---
        cars_frame = tk.LabelFrame(frame, text="Мои автомобили", bg=CARD, fg=TEXT,
                                   font=("Arial", 10, "bold"))
        cars_frame.pack(fill="x", pady=(0, 20))

        cars_listbox = tk.Listbox(cars_frame, height=6, font=("Arial", 10))
        cars_listbox.pack(fill="x", pady=5)

        # Заполняем список автомобилей
        for car in user.get("cars", []):
            cars_listbox.insert(tk.END, f"{car['make']} {car['model']} ({car['year']})")

        def add_car():
            # Создаём диалоговое окно для добавления автомобиля
            add_window = tk.Toplevel(frame)
            add_window.title("Добавить автомобиль")
            add_window.geometry("300x600")
            add_window.resizable(False, False)

            # Марки автомобилей (можно расширить)
            makes = ["Toyota", "Honda", "BMW", "Mercedes", "Audi", "Volkswagen", "Ford", "Nissan"]

            tk.Label(add_window, text="Марка:", bg=CARD, fg=SUB).pack(anchor="w", padx=10, pady=(10, 5))
            make_var = tk.StringVar(value=makes[0])
            make_combo = ttk.Combobox(add_window, textvariable=make_var, values=makes, state="readonly")
            make_combo.pack(fill="x", padx=10, pady=2)

            tk.Label(add_window, text="Модель:", bg=CARD, fg=SUB).pack(anchor="w", padx=10, pady=(10, 5))
            model_entry = tk.Entry(add_window, font=("Arial", 10))
            model_entry.pack(fill="x", padx=10, pady=2)

            tk.Label(add_window, text="Год выпуска:", bg=CARD, fg=SUB).pack(anchor="w", padx=10, pady=(10, 5))
            year_entry = tk.Entry(add_window, font=("Arial", 10))
            year_entry.pack(fill="x", padx=10, pady=2)

            def save_car():
                make = make_var.get()
                model = model_entry.get().strip()
                year = year_entry.get().strip()

                # Валидация
                if not model:
                    messagebox.showerror("Ошибка", "Введите модель автомобиля")
                    return
                if not year.isdigit() or not (1900 <= int(year) <= 2026):
                    messagebox.showerror("Ошибка", "Год выпуска должен быть числом от 1900 до 2026")
                    return

                # Добавляем автомобиль в список пользователя
                user["cars"].append({
                    "make": make,
                    "model": model,
                    "year": year
                })
                save_data()

                # Обновляем список в интерфейсе
                cars_listbox.insert(tk.END, f"{make} {model} ({year})")
                messagebox.showinfo("Успех", "Автомобиль добавлен!")
                add_window.destroy()

            tk.Button(add_window, text="Добавить", bg=ACCENT, fg="white",
                      command=save_car, font=("Arial", 10)).pack(pady=15)

        def delete_car():
            selection = cars_listbox.curselection()
            if not selection:
                messagebox.showwarning("Предупреждение", "Выберите автомобиль для удаления")
                return

            index = selection[0]
            cars_listbox.delete(index)
            user["cars"].pop(index)
            save_data()
            messagebox.showinfo("Успех", "Автомобиль удалён!")

        add_car_btn = tk.Button(cars_frame, text="Добавить автомобиль", bg=ACCENT2, fg="black",
                                command=add_car, font=("Arial", 9))
        add_car_btn.pack(fill="x", pady=(0, 5))

        delete_car_btn = tk.Button(cars_frame, text="Удалить выбранный", bg=DANGER, fg="white",
                                   command=delete_car, font=("Arial", 9))
        delete_car_btn.pack(fill="x")
        # --- Конец блока управления автомобилями ---

        # --- Блок действий пользователя ---
        actions_frame = tk.LabelFrame(frame, text="Действия", bg=CARD, fg=TEXT,
                                      font=("Arial", 10, "bold"))
        actions_frame.pack(fill="x")

        def logout():
            global current_user
            current_user = None
            update_user()
            show_auth()
            messagebox.showinfo("Выход", "Вы вышли из аккаунта")

        logout_btn = tk.Button(actions_frame, text="Выйти из аккаунта",
                               bg=DANGER, fg="white", command=logout,
                               font=("Arial", 10, "bold"), height=2)
        logout_btn.pack(fill="x", pady=5)

        def delete_account():
            if messagebox.askyesno("Подтверждение",
                                   "Вы уверены, что хотите удалить аккаунт?\n"
                                   "Это действие нельзя отменить!"):
                del data["users"][current_user]
                save_data()
                logout()

        delete_btn = tk.Button(actions_frame, text="Удалить аккаунт",
                               bg="#992222", fg="white", command=delete_account,
                               font=("Arial", 9), height=1)
        delete_btn.pack(fill="x", pady=(0, 5))
        # --- Конец блока действий ---

        return frame

        # -------------------------
        # 🧩 Основное тело профиля
        # -------------------------
def create_profile_body(parent, user):
            grid = tk.Frame(parent, bg=PROFILE_BG)
            grid.pack(fill="both", expand=True, pady=(0, 20))

            left = create_left_column(grid, user)
            right = create_right_column(grid, user)

            left.grid(row=0, column=0, padx=(0, 20), sticky="n")
            right.grid(row=0, column=1, padx=(20, 0), sticky="nsew")

            grid.columnconfigure(1, weight=1)

        # -------------------------
        # 👤 Левая колонка
        # -------------------------
def create_left_column(parent, user):
            frame = tk.Frame(parent, bg=CARD, padx=20, pady=20)

            # Блок с базовой информацией пользователя
            info_frame = tk.Frame(frame, bg=CARD)
            info_frame.pack(pady=10)

            # Отображение email пользователя (как идентификатор)
            tk.Label(info_frame, text=current_user,
                     font=("Arial", 12, "bold"), bg=CARD, fg=TEXT).pack()


            # Количество автомобилей
            cars_count = len(user.get("cars", []))
            tk.Label(info_frame, text=f"Автомобилей в профиле: {cars_count}", fg=SUB, bg=CARD,
                     font=("Arial", 9)).pack(pady=(5, 0))

            # Разделитель
            separator = tk.Frame(frame, height=2, bg=PANEL)
            separator.pack(fill="x", pady=15)

            # Статистика по заказам
            history = user.get("history", [])
            tk.Label(frame, text="Статистика", font=("Arial", 11, "bold"),
                     bg=CARD, fg=ACCENT).pack(anchor="w")

            tk.Label(frame, text=f"Всего заказов: {len(history)}", fg=SUB, bg=CARD,
                     font=("Arial", 9)).pack(anchor="w", pady=(5, 2))

            if history:
                last_order = history[-1]
                tk.Label(frame, text=f"Последний заказ: {last_order}", fg=SUB, bg=CARD,
                         font=("Arial", 9)).pack(anchor="w", pady=(2, 0))
            else:
                tk.Label(frame, text="Нет выполненных заказов", fg=SUB, bg=CARD,
                         font=("Arial", 9)).pack(anchor="w", pady=(2, 0))

            return frame

        # -------------------------
        # 📝 Центральная колонка (убрана, так как больше не нужна)
        # -------------------------

        # -------------------------
        # 🧩 Основное тело профиля (обновлено без центральной колонки)
        # -------------------------
def create_profile_body(parent, user):
            grid = tk.Frame(parent, bg=PROFILE_BG)
            grid.pack(fill="both", expand=True, pady=(0, 20))

            left = create_left_column(grid, user)
            right = create_right_column(grid, user)

            # Размещаем колонки: левая — фиксированная ширина, правая — растягивается
            left.grid(row=0, column=0, padx=(0, 20), sticky="n")
            right.grid(row=0, column=1, padx=(20, 0), sticky="nsew")

            # Настраиваем растягивание правой колонки
            grid.columnconfigure(1, weight=1)

            return grid
