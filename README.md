def show_history():
    clear()

    tk.Label(content, text="История расчётов",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=20)

    if not current_user:
        tk.Label(content, text="Вы не авторизованы", bg=BG, fg=SUB,
                 font=("Arial", 14)).pack()
        return

    history = data["users"][current_user].get("history", [])

    if not history:
        tk.Label(content, text="История пуста", bg=BG, fg=SUB,
                 font=("Arial", 14)).pack()
        return

    frame = tk.Frame(content, bg=CARD)
    frame.pack(fill="both", expand=True, padx=20, pady=10)

    # 🔍 Поиск
    search_var = tk.StringVar()
    search_entry = tk.Entry(frame, textvariable=search_var)
    search_entry.pack(fill="x", pady=5)

    listbox = tk.Listbox(frame, font=("Arial", 11))
    listbox.pack(fill="both", expand=True, pady=10)

    def format_record(r):
        if "Расход" in r.get("mode", ""):
            return f"{r['date']} | {r['mileage']}км | {r['fuel']}л → {r['result']}"
        else:
            return f"{r['date']} | {r['mileage']}км | {r['fuel']}л × {r['price']} → {r['result']}"

    def refresh():
        listbox.delete(0, tk.END)
        query = search_var.get().lower()

        for r in history:
            text = format_record(r)
            if query in text.lower():
                listbox.insert(tk.END, text)

    refresh()
    search_var.trace_add("write", lambda *args: refresh())

    # 🔁 Повтор расчёта
    def reuse():
        sel = listbox.curselection()
        if not sel:
            return

        record = history[sel[0]]

        show_calc()

        # после отрисовки — вставляем значения
        for widget in content.winfo_children():
            for child in widget.winfo_children():
                if isinstance(child, tk.Entry):
                    pass  # можно доработать при желании

        messagebox.showinfo("Подсказка", "Данные можно ввести вручную (автовставку можно допилить)")

    # 🗑 Очистка истории
    def clear_history():
        if messagebox.askyesno("Очистка", "Удалить всю историю?"):
            data["users"][current_user]["history"] = []
            save_data()
            refresh()

    btns = tk.Frame(frame, bg=CARD)
    btns.pack(fill="x")

    tk.Button(btns, text="Повторить расчёт", command=reuse,
              bg=ACCENT2).pack(side="left", padx=5)

    tk.Button(btns, text="Очистить историю", command=clear_history,
              bg=DANGER, fg="white").pack(side="right", padx=5)
