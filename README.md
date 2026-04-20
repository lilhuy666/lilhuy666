dropdown = tk.Frame(main, bg=PANEL)
dropdown.pack(fill="x")
dropdown.pack_forget()  # скрыто по умолчанию

menu_open = False




def toggle_menu():
    global menu_open
    if menu_open:
        dropdown.pack_forget()
    else:
        dropdown.pack(fill="x")
    menu_open = not menu_open






    def nav(text, cmd):
    tk.Button(dropdown,
              text=text,
              bg=PANEL,
              fg=TEXT,
              bd=0,
              anchor="w",
              padx=20,
              pady=12,
              font=("Arial", 12),
              activebackground=CARD,
              activeforeground=ACCENT,
              command=lambda: [cmd(), toggle_menu()]
              ).pack(fill="x")
