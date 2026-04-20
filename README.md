import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json, os

# ===================== MODERN STYLE =====================
BG = "#0f172a"  # Тёмно‑синий фон
PANEL = "#1e293b"  # Панель меню
CARD = "#334155"  # Карточки контента
ACCENT = "#3b82f6"  # Основной акцент (синий)
ACCENT_HOVER = "#2563eb"  # Акцент при наведении
ACCENT2 = "#10b981"  # Вторичный акцент (зелёный)
ACCENT2_HOVER = "#059669"  # Зелёный при наведении
TEXT = "#f1f5f9"  # Основной текст
SUB = "#94a3b8"  # Второстепенный текст
DANGER = "#ef4444"  # Ошибки (красный)
DANGER_HOVER = "#dc2626"  # Красный при наведении
BORDER = "#64748b"  # Границы
HIGHLIGHT = "#fcd34d"  # Подсветка важных элементов
SHADOW = "#020617"  # Тень

# Анимационные параметры
ANIMATION_SPEED = 100  # мс

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
            if "users" not in data:
                data = {"users": {}}
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
root.resizable(True, True)

main = tk.Frame(root, bg=BG)
main.pack(fill="both", expand=True)

# ===================== MODERN HEADER =====================
header = tk.Frame(main, bg=PANEL, height=80)
header.pack(fill="x")
header.pack_propagate(False)

# Логотип с иконкой
logo_frame = tk.Frame(header, bg=PANEL)
logo_frame.pack(side="left", padx=20)

tk.Label(
    logo_frame,
    text="🚗",
    bg=PANEL,
    fg=HIGHLIGHT,
    font=("Segoe UI", 24)
).pack(side="left")

tk.Label(
    logo_frame,
    text="CalculatCar",
    bg=PANEL,
    fg=TEXT,
    font=("Segoe UI", 18, "bold")
).pack(side="left", padx=10)

# Меню (улучшенное)
menu_window = None

def toggle_menu():
    global menu_window

    if menu_window and menu_window.winfo_exists():
        menu_window.destroy()
        return

    x = root.winfo_x() + 20
    y = root.winfo_y() + 80

    menu_window = tk.Toplevel(root)
    menu_window.overrideredirect(True)
    menu_window.configure(bg=CARD)
    menu_window.geometry(f"240x280+{x}+{y}")

    # Тень для меню
    menu_window.attributes('-alpha', 0.95)

    def close(e=None):
        if menu_window:
            menu_window.destroy()

    menu_window.bind("<FocusOut>", close)

    def nav(text, cmd, color=ACCENT):
        btn = create_modern_button(
            menu_window,
            text=text,
            command=lambda: [cmd(), close()],
            bg=color,
            hover_bg=ACCENT_HOVER if color == ACCENT else ACCENT2_HOVER
        )
        btn.pack(fill="x", padx=15, pady=5)

    nav("🧮 Калькулятор", show_calc, ACCENT)
    nav("👤 Профиль", show_profile, ACCENT2)
    nav("📊 История", show_history, ACCENT)
    nav("�� Настройки", show_settings, ACCENT)
    nav("ℹ️ О программе", show_about, ACCENT)

    menu_window.focus_force()

# Функция создания стильной кнопки
def create_modern_button(parent, text, command=None, bg=ACCENT, fg=TEXT, hover_bg=None, hover_fg=None):
    """Создаёт стильную анимированную кнопку"""
    if hover_bg is None:
        hover_bg = ACCENT_HOVER if bg == ACCENT else ACCENT2_HOVER
    if hover_fg is None:
        hover_fg = TEXT

    btn = tk.Button(
        parent,
        text=text,
        bg=bg,
        fg=fg,
        bd=0,
        font=("Segoe UI", 11, "bold"),
        cursor="hand2",
        padx=20,
        pady=10,
        relief="flat"
    )

    # Эффекты наведения
    def on_enter(e):
        btn.config(bg=hover_bg, fg=hover_fg)

    def on_leave(e):
        btn.config(bg=bg, fg=fg)

    btn.bind("<Enter>", on_enter)
    btn.bind("<Leave>", on_leave)

    if command:
        btn.config(command=command)
    return btn

menu_btn = create_modern_button(
    header,
    text="☰",
    command=toggle_menu,
    bg=ACCENT,
    hover_bg=ACCENT_HOVER,
    padx=15
)
menu_btn.pack(side="right", padx=20)

user_label = tk.Label(header, text="",
                      bg=PANEL, fg=SUB,
                      font=("Segoe UI", 10))
user_label.pack(side="right", padx=15)

# ===================== CONTENT =====================
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True, padx=10, pady=10)

def clear():
    for w in content.winfo_children():
        w.destroy()

def card():
    """Создаёт стильную карточку с тенью"""
    f = tk.Frame(
        content,
        bg=CARD,
        padx=40,
        pady=40,
        highlightbackground=BORDER,
        highlightthickness=1,
        relief="raised"
    )
    f.pack(pady=30, padx=20)
    # Имитация тени
    shadow = tk.Frame(f, bg=SHADOW, height=2)
    shadow.pack(side="bottom", fill="x")
    return f

# ===================== PROFILE =====================
def show_profile():
    clear()

    if not current_user:
        c = card()

        tk.Label(c, text="Вход / Регистрация",
                 bg=CARD, fg=HIGHLIGHT,
                 font=("Segoe UI", 22, "bold")).pack(pady=20)

        # Поле email
        tk.Label(c, text="Email", bg=CARD, fg=SUB, font=("Segoe UI", 11)).pack(anchor="w", pady=(0, 5))
        email = tk.Entry(c, font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
        email.pack(
