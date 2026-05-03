def show_history():
    clear()

    # ... твой код ...

    for record in history:
        text = (
            f"{record.get('date', '')} | "
            f"{record.get('mileage', '')} км | "
            f"{record.get('fuel', '')} л | "
            f"{record.get('price', '')} руб | "
            f"{record.get('result', '')}"
        )
        listbox.insert(tk.END, text)

    # 🗑 Очистка истории (ВАЖНО: с отступом!)
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
