def show_history():
    clear()

    if not current_user:
        tk.Label(content, text="Сначала войдите в аккаунт",
                 bg=BG, fg=SUB,
                 font=("Arial", 14)).pack(pady=20)
        return
