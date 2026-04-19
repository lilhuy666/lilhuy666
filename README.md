# ===================== MENU =====================
def nav(text, cmd):
    tk.Button(sidebar,
              text=text,
              bg=PANEL,
              fg=TEXT,
              bd=0,
              anchor="w",
              padx=20,
              pady=14,
              font=("Arial", 12),
              activebackground=CARD,
              activeforeground=ACCENT,
              command=cmd).pack(fill="x")

nav("Калькулятор", show_calc)
nav("Профиль", show_profile)
nav("История", show_history)
nav("Настройки", show_settings)
nav("О программе", show_about)

# ===================== USER LABEL UPDATE =====================
def update_user():
    user_label.config(text=current_user if current_user else "")

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
