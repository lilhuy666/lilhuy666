```python
# ===================== PROFILE =====================

def show_profile():
    clear()

    if not current_user:
        return show_auth()

    user = data["users"][current_user]

    root = tk.Frame(content, bg=PROFILE_BG, padx=20, pady=20)
    root.pack(fill="both", expand=True)

    create_header(root)
    create_profile_body(root, user)


# ---------------- HEADER ----------------
def create_header(parent):
    header = tk.Frame(parent, bg=PANEL, height=60)
    header.pack(fill="x", pady=(0, 20))
    header.pack_propagate(False)

    tk.Label(header,
             text="Профиль",
             bg=PANEL,
             fg=ACCENT,
             font=("Arial", 16, "bold")).pack(pady=10)

    tk.Button(header,
              text="← Назад",
              bg=DANGER,
              fg="white",
              command=lambda: (clear(), show_main_menu()),
              font=("Arial", 10),
              width=10).place(x=10, y=15)


# ---------------- BODY ----------------
def create_profile_body(parent, user):
    container = tk.Frame(parent, bg=PROFILE_BG)
    container.pack(fill="both", expand=True)

    container.columnconfigure(0, weight=1)
    container.columnconfigure(1, weight=2)

    left = create_left_column(container, user)
    right = create_right_column(container, user)

    left.grid(row=0, column=0, sticky="nsew", padx=(0, 10))
    right.grid(row=0, column=1, sticky="nsew", padx=(10, 0))


# ---------------- LEFT ----------------
def create_left_column(parent, user):
    frame = tk.Frame(parent, bg=CARD, highlightthickness=1, highlightbackground=PANEL)

    inner = tk.Frame(frame, bg=CARD, padx=20, pady=20)
    inner.pack(fill="both", expand=True)

    tk.Label(inner,
             text=current_user,
             font=("Arial", 14, "bold"),
             bg=CARD, fg=TEXT).pack(anchor="w")

    tk.Label(inner,
             text=f"Дата регистрации: {user.get('registration_date', '-')}",
             bg=CARD, fg=SUB).pack(anchor="w", pady=(5, 15))

    tk.Frame(inner, height=1, bg=PANEL).pack(fill="x", pady=10)

    cars = user.get("cars", [])
    history = user.get("history", [])

    tk.Label(inner, text="Статистика",
             font=("Arial", 11, "bold"),
             bg=CARD, fg=ACCENT).pack(anchor="w")

    tk.Label(inner, text=f"Авто: {len(cars)}",
             bg=CARD, fg=SUB).pack(anchor="w")

    tk.Label(inner, text=f"Заказы: {len(history)}",
             bg=CARD, fg=SUB).pack(anchor="w")

    if history:
        tk.Label(inner,
                 text=f"Последний: {history[-1]}",
                 bg=CARD, fg=SUB).pack(anchor="w", pady=(5, 0))

    return frame


# ---------------- RIGHT ----------------
def create_right_column(parent, user):
    frame = tk.Frame(parent, bg=PROFILE_BG)

    def card(title):
        c = tk.Frame(frame, bg=CARD, highlightthickness=1, highlightbackground=PANEL)
        inner = tk.Frame(c, bg=CARD, padx=15, pady=15)
        inner.pack(fill="both", expand=True)

        tk.Label(inner,
                 text=title,
                 font=("Arial", 11, "bold"),
                 bg=CARD, fg=ACCENT).pack(anchor="w", pady=(0, 10))

        return c, inner

    # ===== УВЕДОМЛЕНИЯ =====
    c1, inner1 = card("Уведомления")

    var = tk.BooleanVar(value=user.get("notifications", True))

    tk.Checkbutton(inner1,
                   text="Получать уведомления",
                   variable=var,
                   bg=CARD, fg=TEXT,
                   selectcolor=CARD).pack(anchor="w")

    def save_notify():
        user["notifications"] = var.get()
        save_data()
        messagebox.showinfo("Успех", "Сохранено")

    tk.Button(inner1,
              text="Сохранить",
              bg=ACCENT,
              fg="white",
              command=save_notify).pack(anchor="e", pady=10)

    c1.pack(fill="x", pady=(0, 10))


    # ===== АВТО =====
    c2, inner2 = card("Мои автомобили")

    listbox = tk.Listbox(inner2, height=5)
    listbox.pack(fill="x")

    for car in user.get("cars", []):
        listbox.insert(tk.END, f"{car['make']} {car['model']} ({car['year']})")

    def add_car():
        win = tk.Toplevel()
        win.title("Добавить авто")
        win.geometry("300x250")

        tk.Label(win, text="Марка").pack()
        make = ttk.Combobox(win, values=["BMW", "Audi", "Toyota", "Mercedes"])
        make.pack()

        tk.Label(win, text="Модель").pack()
        model = tk.Entry(win)
        model.pack()

        tk.Label(win, text="Год").pack()
        year = tk.Entry(win)
        year.pack()

        def save():
            if not model.get() or not year.get().isdigit():
                return

            car = {
                "make": make.get(),
                "model": model.get(),
                "year": year.get()
            }

            user["cars"].append(car)
            save_data()
            listbox.insert(tk.END, f"{car['make']} {car['model']} ({car['year']})")
            win.destroy()

        tk.Button(win, text="Добавить", command=save).pack(pady=10)

    def delete_car():
        sel = listbox.curselection()
        if not sel:
            return
        index = sel[0]
        listbox.delete(index)
        user["cars"].pop(index)
        save_data()

    tk.Button(inner2, text="Добавить", command=add_car).pack(fill="x", pady=5)
    tk.Button(inner2, text="Удалить", bg=DANGER, fg="white", command=delete_car).pack(fill="x")

    c2.pack(fill="x", pady=(0, 10))


    # ===== ДЕЙСТВИЯ =====
    c3, inner3 = card("Аккаунт")

    def logout():
        global current_user
        current_user = None
        update_user()
        show_auth()

    def delete_account():
        if messagebox.askyesno("Удалить?", "Точно удалить аккаунт?"):
            del data["users"][current_user]
            save_data()
            logout()

    tk.Button(inner3,
              text="Выйти",
              bg=DANGER,
              fg="white",
              command=logout).pack(fill="x", pady=5)

    tk.Button(inner3,
              text="Удалить аккаунт",
              command=delete_account).pack(fill="x")

    c3.pack(fill="x")

    return frame
```
