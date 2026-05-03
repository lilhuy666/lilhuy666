def show_history():
    clear()

    tk.Label(content, text="История расчётов",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=20)

    # 🔒 Проверка авторизации
    if not current_user:
        tk.Label(content, text="Вы не авторизованы",
                 bg=BG, fg=SUB,
                 font=("Arial", 14)).pack()
        return

    user = data["users"].get(current_user)

    # 🔒 Проверка пользователя
    if not user:
        tk.Label(content, text="Ошибка пользователя",
                 bg=BG, fg=SUB,
                 font=("Arial", 14)).pack()
        return

    history = user.get("history", [])

    # 🔒 Проверка истории
    if not history:
        tk.Label(content, text="История пуста",
                 bg=BG, fg=SUB,
                 font=("Arial", 14)).pack()
        return

    # 📦 Основной блок
    frame = tk.Frame(content, bg=CARD)
    frame.pack(fill="both", expand=True, padx=20, pady=10)

    listbox = tk.Listbox(frame, font=("Arial", 11))
    listbox.pack(fill="both", expand=True, padx=10, pady=10)

    # 📄 Заполнение
    for record in history:
        text = (
            f"{record.get('date', '')} | "
            f"{record.get('mileage', '')} км | "
            f"{record.get('fuel', '')} л | "
            f"{record.get('price', '')} руб | "
            f"{record.get('result', '')}"
        )
        listbox.insert(tk.END, text)

    # 🗑 Очистка истории (внутри функции!)
    def clear_history():
        if messagebox.askyesno("Очистка", "Удалить всю историю?"):
            user["history"] = []
            save_data()
            show_history()

    tk.Button(content,
              text="Очистить историю",
              command=clear_history,
              bg=DANGER,
              fg="white").pack(pady=10)
