        email.pack(fill="x", pady=8)

        # Поле пароля
        tk.Label(c, text="Пароль", bg=CARD, fg=SUB, font=("Segoe UI", 11)).pack(anchor="w", pady=(0, 5))
        password = tk.Entry(c, show="*", font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
        password.pack(fill="x", pady=8)

        def login():
            global current_user
            e = email.get().strip()
            p = password.get().strip()

            if e in data["users"] and data["users"][e]["password"] == p:
                current_user = e
                update_user()
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные")

        def register():
            e = email.get().strip()
            p = password.get().strip()

            if not e or not p:
                return messagebox.showerror("Ошибка", "Заполните все поля")

            if e in data["users"]:
                return messagebox.showerror("Ошибка", "Аккаунт с таким email уже существует")

            data["users"][e] = {
                "password": p,
                "name": "",
                "car": "",
                "history": []
            }
            save_data()
            messagebox.showinfo("Успех", "Аккаунт создан!")

        # Кнопки входа и регистрации
        login_btn = create_modern_button(c, "Войти", command=login, bg=ACCENT)
        login_btn.pack(fill="x", pady=10)

        register_btn = create_modern_button(c, "Регистрация", command=register, bg=ACCENT2)
        register_btn.pack(fill="x")
        return

    # Если пользователь авторизован — показываем профиль
    user = data["users"][current_user]
    c = card()

    tk.Label(c, text="Профиль пользователя",
             bg=CARD, fg=HIGHLIGHT,
             font=("Segoe UI", 22, "bold")).pack(pady=20)

    # Имя пользователя
    name_frame = tk.Frame(c, bg=CARD)
    name_frame.pack(fill="x", pady=15)

    tk.Label(name_frame, text="Имя:", bg=CARD, fg=SUB, font=("Segoe UI", 12)).pack(anchor="w")
    name_entry = tk.Entry(name_frame, font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
    name_entry.insert(0, user.get("name", ""))
    name_entry.pack(side="left", fill="x", expand=True, padx=(0, 10))

    def save_name():
        user["name"] = name_entry.get()
        save_data()
        messagebox.showinfo("Успех", "Имя сохранено!")

    save_name_btn = create_modern_button(name_frame, "Сохранить", command=save_name, bg=ACCENT, padx=10)
    save_name_btn.pack(side="right")

    # Автомобиль
    car_frame = tk.Frame(c, bg=CARD)
    car_frame.pack(fill="x", pady=15)

    tk.Label(car_frame, text="Автомобиль:", bg=CARD, fg=SUB, font=("Segoe UI", 12)).pack(anchor="w")
    car_entry = tk.Entry(car_frame, font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
    car_entry.insert(0, user.get("car", ""))
    car_entry.pack(side="left", fill="x", expand=True, padx=(0, 10))

    def save_car():
        user["car"] = car_entry.get()
        save_data()
        messagebox.showinfo("Успех", "Данные об автомобиле сохранены!")

    save_car_btn = create_modern_button(car_frame, "Сохранить", command=save_car, bg=ACCENT, padx=10)
    save_car_btn.pack(side="right")

    # Смена пароля
    pass_frame = tk.Frame(c, bg=CARD)
    pass_frame.pack(fill="x", pady=15)

    tk.Label(pass_frame, text="Новый пароль:", bg=CARD, fg=SUB, font=("Segoe UI", 12)).pack(anchor="w")
    pass1 = tk.Entry(pass_frame, show="*", font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
    pass1.pack(fill="x", pady=5)

    tk.Label(pass_frame, text="Повтор пароля:", bg=CARD, fg=SUB, font=("Segoe UI", 12)).pack(anchor="w")
    pass2 = tk.Entry(pass_frame, show="*", font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
    pass2.pack(side="left", fill="x", expand=True, padx=(0, 10))

    def save_password():
        p1 = pass1.get()
        p2 = pass2.get()

        if not p1 or not p2:
            return messagebox.showerror("Ошибка", "Введите пароль")

        if p1 != p2:
            return messagebox.showerror("Ошибка", "Пароли не совпадают")

        user["password"] = p1
        save_data()
        messagebox.showinfo("Успех", "Пароль изменён!")

    save_pass_btn = create_modern_button(pass_frame, "Сохранить", command=save_password, bg=ACCENT, padx=10)
    save_pass_btn.pack(side="right")

    # Кнопка выхода
    logout_btn = create_modern_button(c, "Выйти из аккаунта", command=logout, bg=DANGER)
    logout_btn.pack(fill="x", pady=20)

def logout():
    global current_user
    current_user = None
    update_user()
    show_profile()

# ===================== CALCULATOR =====================
def show_calc():
    clear()

    title_frame = tk.Frame(content, bg=BG)
    title_frame.pack(pady=20)

    tk.Label(
        title_frame,
        text="🧮 Калькулятор расхода топлива",
        bg=BG,
        fg=HIGHLIGHT,
        font=("Segoe UI", 24, "bold")
    ).pack()

    mode = tk.StringVar(value="1")

    switch = tk.Frame(content, bg=BG)
    switch.pack(pady=15)

    def create_switch_btn(text, value, active=False):
        color = ACCENT if active else PANEL
        btn = tk.Button(
            switch,
            text=text,
            bg=color,
            fg=TEXT,
            font=("Segoe UI", 11),
            bd=0,
            padx=25,
            pady=8,
            relief="flat",
            cursor="hand2"
        )
        btn.pack(side="left", padx=5)
        return btn

    btn1 = create_switch_btn("Стоимость поездки", "1", True)
    btn2 = create_switch_btn("Средний расход", "2")

    def switch_mode(value):
        mode.set(value)
        btn1.config(bg=ACCENT if value == "1" else PANEL)
        btn2.config(bg=ACCENT if value == "2" else PANEL)
        build()

    btn1.config(command=lambda: switch_mode("1"))
    btn2.config(command=lambda: switch_mode("2"))

    form = tk.Frame(content, bg=CARD, padx=40, pady=40
