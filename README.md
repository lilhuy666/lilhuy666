import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json, os

# ===================== STYLE =====================
BG = "#0f172a"
PANEL = "#1e293b"
CARD = "#334155"
ACCENT = "#3b82f6"
ACCENT_HOVER = "#2563eb"
ACCENT2 = "#10b981"
ACCENT2_HOVER = "#059669"
TEXT = "#f1f5f9"
SUB = "#94a3b8"
DANGER = "#ef4444"
DANGER_HOVER = "#dc2626"
BORDER = "#64748b"
HIGHLIGHT = "#fcd34d"
SHADOW = "#020617"

DATA_FILE = "data.json"

# ===================== DATA =====================
data = {"users": {}}
current_user = None


def load_data():
    global data
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
        except:
            data = {"users": {}}


def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)


# ===================== WINDOW =====================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

main = tk.Frame(root, bg=BG)
main.pack(fill="both", expand=True)

# ===================== BUTTON =====================
def create_modern_button(parent, text, command=None,
                         bg=ACCENT, fg=TEXT,
                         hover_bg=None, hover_fg=None,
                         padx=20, pady=10, font=("Segoe UI", 11, "bold")):

    if hover_bg is None:
        hover_bg = ACCENT_HOVER if bg == ACCENT else ACCENT2_HOVER
    if hover_fg is None:
        hover_fg = TEXT

    btn = tk.Button(
        parent,
        text=text,
        bg=bg,
        fg=fg,
        font=font,
        bd=0,
        padx=padx,
        pady=pady,
        relief="flat",
        cursor="hand2"
    )

    def enter(e):
        btn.config(bg=hover_bg)

    def leave(e):
        btn.config(bg=bg)

    btn.bind("<Enter>", enter)
    btn.bind("<Leave>", leave)

    if command:
        btn.config(command=command)

    return btn


# ===================== HEADER =====================
header = tk.Frame(main, bg=PANEL, height=80)
header.pack(fill="x")
header.pack_propagate(False)

logo = tk.Label(header, text="🚗 CalculatCar",
                bg=PANEL, fg=TEXT,
                font=("Segoe UI", 18, "bold"))
logo.pack(side="left", padx=20)

user_label = tk.Label(header, text="", bg=PANEL, fg=SUB)
user_label.pack(side="right", padx=20)


menu_window = None


def toggle_menu():
    global menu_window

    if menu_window and menu_window.winfo_exists():
        menu_window.destroy()
        return

    menu_window = tk.Toplevel(root)
    menu_window.overrideredirect(True)
    menu_window.geometry("240x280+100+100")
    menu_window.configure(bg=CARD)

    def close(e=None):
        menu_window.destroy()

    menu_window.bind("<FocusOut>", close)

    def nav(t, c, col):
        btn = create_modern_button(menu_window, t, command=lambda: [c(), close()], bg=col)
        btn.pack(fill="x", padx=10, pady=5)

    nav("🧮 Калькулятор", show_calc, ACCENT)
    nav("👤 Профиль", show_profile, ACCENT2)
    nav("📊 История", show_history, ACCENT)
    nav("⚙️ Настройки", show_settings, ACCENT)
    nav("ℹ️ О программе", show_about, ACCENT)


menu_btn = create_modern_button(header, "☰", command=toggle_menu)
menu_btn.pack(side="right", padx=20)


# ===================== CONTENT =====================
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)


def clear():
    for w in content.winfo_children():
        w.destroy()


def card():
    f = tk.Frame(content, bg=CARD, padx=40, pady=40,
                 highlightbackground=BORDER, highlightthickness=1)
    f.pack(pady=20)
    return f


# ===================== PROFILE =====================
def show_profile():
    clear()

    c = card()

    if not current_user:
        tk.Label(c, text="Вход / Регистрация",
                 bg=CARD, fg=HIGHLIGHT,
                 font=("Segoe UI", 20, "bold")).pack()

        email = tk.Entry(c)
        email.pack(fill="x")

        password = tk.Entry(c, show="*")
        password.pack(fill="x")

        def login():
            global current_user
            e = email.get()
            p = password.get()

            if e in data["users"] and data["users"][e]["password"] == p:
                current_user = e
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные")

        def register():
            e = email.get()
            p = password.get()

            data["users"][e] = {"password": p, "history": []}
            save_data()
            messagebox.showinfo("OK", "Создано")

        create_modern_button(c, "Войти", command=login).pack(fill="x")
        create_modern_button(c, "Регистрация", command=register, bg=ACCENT2).pack(fill="x")
        return

    tk.Label(c, text=f"Профиль: {current_user}",
             bg=CARD, fg=TEXT, font=("Segoe UI", 16)).pack()


# ===================== CALCULATOR =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор",
             bg=BG, fg=HIGHLIGHT,
             font=("Segoe UI", 20, "bold")).pack(pady=10)

    form = tk.Frame(content, bg=CARD, padx=40, pady=40)
    form.pack(pady=20)

    entries = {}

    def add(label):
        tk.Label(form, text=label, bg=CARD, fg=SUB).pack()
        e = tk.Entry(form)
        e.pack(fill="x")
        entries[label] = e

    add("Топливо")
    add("Расстояние")
    add("Цена")

    def calc():
        try:
            f = float(entries["Топливо"].get())
            d = float(entries["Расстояние"].get())
            p = float(entries["Цена"].get())

            cost = f * p

            result.config(text=f"Стоимость: {cost:.2f}")
        except:
            messagebox.showerror("Ошибка", "Некорректные данные")

    result = tk.Label(content, text="—", bg=BG, fg=TEXT)
    result.pack()

    create_modern_button(content, "Рассчитать", command=calc).pack()


# ===================== HISTORY =====================
def show_history():
    clear()

    tk.Label(content, text="История",
             bg=BG, fg=HIGHLIGHT,
             font=("Segoe UI", 20)).pack()

    if not current_user:
        return

    for item in data["users"][current_user]["history"]:
        tk.Label(content, text=item, bg=BG, fg=TEXT).pack()


# ===================== SETTINGS =====================
def show_settings():
    clear()

    c = card()

    tk.Label(c, text="Настройки",
             bg=CARD, fg=HIGHLIGHT,
             font=("Segoe UI", 18)).pack()


# ===================== ABOUT =====================
def show_about():
    clear()

    c = card()

    tk.Label(c, text="О программе CalculatCar",
             bg=CARD, fg=TEXT,
             font=("Segoe UI", 18)).pack()


# ===================== START =====================
load_data()
show_profile()

root.mainloop()
