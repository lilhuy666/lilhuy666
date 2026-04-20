            form.pack(pady=20)

    entries = {}

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(t, placeholder=""):
            label_frame = tk.Frame(form, bg=CARD)
            label_frame.pack(fill="x", pady=10)

            tk.Label(
                label_frame,
                text=t,
                bg=CARD,
                fg=SUB,
                font=("Segoe UI", 12)
            ).pack(anchor="w")

            e = tk.Entry(
                label_frame,
                font=("Segoe UI", 14),
                bd=1,
                relief="flat",
                bg=PANEL,
                fg=TEXT
            )
            if placeholder:
                e.insert(0, placeholder)
            e.pack(side="left", fill="x", expand=True, padx=(0, 10))
            entries[t] = e
            return e

        if mode.get() == "1":
            add("Топливо (л)")
            add("Расстояние (км)")
            add("Цена за литр (₽)")
        else:
            add("Средний расход (л/100 км)")
            add("Расстояние (км)")
            add("Цена за литр (₽)")

    def calc():
        try:
            if mode.get() == "1":
                f = float(entries["Топливо (л)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр (₽)"].get())
                cons = (f / d) * 100
                cost = f * p
            else:
                cons = float(entries["Средний расход (л/100 км)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр (₽)"].get())
                cost = cons * d / 100 * p

            result.config(text=f"Расход: {cons:.1f} л/100 км | Стоимость: {cost:.2f} ₽")

            if current_user:
                h = data["users"][current_user]["history"]
                h.append({
                    "date": datetime.now().strftime("%d.%m %H:%M"),
                    "result": result.cget("text")
                })
                data["users"][current_user]["history"] = h[-50:]
                save_data()

        except ValueError:
            messagebox.showerror("Ошибка", "Проверьте корректность введённых данных")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Произошла ошибка: {str(e)}")

    # Поле результата
    result = tk.Label(
        content,
        text="—",
        bg=BG,
        fg=HIGHLIGHT,
        font=("Segoe UI", 18, "bold")
    )
    result.pack(pady=20)

    calc_btn = create_modern_button(
        content,
        "Рассчитать",
        command=calc,
        bg=ACCENT,
        padx=30,
        pady=12,
        font=("Segoe UI", 14, "bold")
    )
    calc_btn.pack(pady=15)

    build()

# ===================== HISTORY =====================
def show_history():
    clear()

    title_frame = tk.Frame(content, bg=BG)
    title_frame.pack(pady=20)

    tk.Label(
        title_frame,
        text="📊 История расчётов",
        bg=BG,
        fg=HIGHLIGHT,
        font=("Segoe UI", 24, "bold")
    ).pack()

    if not current_user:
        tk.Label(
            content,
            text="Войдите в аккаунт для просмотра истории",
            bg=BG,
            fg=SUB,
            font=("Segoe UI", 16)
        ).pack(pady=40)
        return

    history_data = data["users"][current_user]["history"]
    if not history_data:
        tk.Label(
            content,
            text="История расчётов пуста",
            bg=BG,
            fg=SUB,
            font=("Segoe UI", 14)
        ).pack(pady=30)
        return

    # Прокрутка для истории
    canvas = tk.Canvas(content, bg=BG, highlightthickness=0)
    scrollbar = tk.Scrollbar(content, orient="vertical", command=canvas.yview)
    scrollable_frame = tk.Frame(canvas, bg=BG)

    scrollable_frame.bind(
        "<Configure>",
        lambda e: canvas.configure(scrollregion=canvas.bbox("all"))
    )

    canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
    canvas.configure(yscrollcommand=scrollbar.set)

    canvas.pack(side="left", fill="both", expand=True)
    scrollbar.pack(side="right", fill="y")

    for h in history_data[::-1]:
        item_frame = tk.Frame(scrollable_frame, bg=CARD, padx=20, pady=15)
        item_frame.pack(fill="x", padx=40, pady=5)

        tk.Label(
            item_frame,
            text=h['date'],
            bg=CARD,
            fg=SUB,
            font=("Segoe UI", 10)
        ).pack(anchor="w")

        tk.Label(
            item_frame,
            text=h['result'],
            bg=CARD,
            fg=TEXT,
            font=("Segoe UI", 12, "bold")
        ).pack(anchor="w", pady=(5, 0))

# ===================== SETTINGS =====================
def show_settings():
    clear()

    c = card()

    tk.Label(
        c,
        text="�� Настройки",
        bg=CARD,
        fg=HIGHLIGHT,
        font=("Segoe UI", 22, "bold")
    ).pack(pady=20)

    # Тема
    theme_frame = tk.Frame(c, bg=CARD)
    theme_frame.pack(fill="x", pady=15)

    tk.Label(theme_frame, text="Тема:", bg=CARD, fg=SUB, font=("Segoe UI", 12)).pack(anchor="w")
    theme_var = tk.StringVar(value="Тёмная")
    tk.OptionMenu(theme_frame, theme_var, "Тёмная", "Светлая").pack(fill="x", pady=5)

    # Валюта
    currency_frame = tk.Frame(c, bg=CARD)
    currency_frame.pack(fill="x", pady=15)

    tk.Label(currency_frame, text="Валюта:", bg=CARD, fg=SUB, font=("Segoe UI", 12)).pack(anchor="w")
    currency_var = tk.StringVar(value="₽ Рубль")
    tk.OptionMenu(currency_frame, currency_var, "₽ Рубль", "€ Евро", "$ Доллар").pack(fill="x", pady=5)

    save_settings_btn = create_modern_button(c, "Сохранить настройки", bg=ACCENT)
    save_settings_btn.pack(fill="x", pady=20)

# ===================== ABOUT =====================
def show_about():
    clear()

    c = card()

    tk.Label(
        c,
        text="ℹ️ О программе",
        bg=CARD,
        fg=HIGHLIGHT,
        font=("Segoe UI", 22, "bold")
    ).pack(pady=20)

    about_text = """CalculatCar — калькулятор расхода топлива

Версия: 1.0
Разработчик: [Ваше имя]

Программа позволяет:
• Рассчитывать стоимость поездки
• Определять средний расход топлива
• Вести историю расчётов
• Хранить данные профиля

© 2024 Calcul
