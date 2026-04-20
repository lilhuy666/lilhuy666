File "C:\Users\Кирилл\PycharmProjects\PythonProject5\main.py", line 393
    form = tk.Frame(content, bg=CARD, padx=40, pady=40
                   ^
SyntaxError: '(' was never closed


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
        email.pack(fill="x", pady=8)

        # Поле пароля
        tk.Label(c, text="Пароль", bg=CARD, fg=SUB, font=("Segoe UI", 11)).pack(anchor="w", pady=(0, 5))
        password = tk.Entry(c, show="*", font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
        password.pack(fill="x", pady=8)

        def login():
            global current_user
            e = email.get().strip()
            p = password.get().strip()

            if e in data["users"] and data["users"][e]["password"] == p:
                current_user = e
                update_user()
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные")

        def register():
            e = email.get().strip()
            p = password.get().strip()

            if not e or not p:
                return messagebox.showerror("Ошибка", "Заполните все поля")

            if e in data["users"]:
                return messagebox.showerror("Ошибка", "Аккаунт с таким email уже существует")

            data["users"][e] = {
                "password": p,
                "name": "",
                "car": "",
                "history": []
            }
            save_data()
            messagebox.showinfo("Успех", "Аккаунт создан!")

        # Кнопки входа и регистрации
        login_btn = create_modern_button(c, "Войти", command=login, bg=ACCENT)
        login_btn.pack(fill="x", pady=10)

        register_btn = create_modern_button(c, "Регистрация", command=register, bg=ACCENT2)
        register_btn.pack(fill="x")
        return

        # Если пользователь авторизован — показываем профиль
    user = data["users"][current_user]
    c = card()

    tk.Label(c, text="Профиль пользователя",
             bg=CARD, fg=HIGHLIGHT,
             font=("Segoe UI", 22, "bold")).pack(pady=20)

    # Имя пользователя
    name_frame = tk.Frame(c, bg=CARD)
    name_frame.pack(fill="x", pady=15)

    tk.Label(name_frame, text="Имя:", bg=CARD, fg=SUB, font=("Segoe UI", 12)).pack(anchor="w")
    name_entry = tk.Entry(name_frame, font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
    name_entry.insert(0, user.get("name", ""))
    name_entry.pack(side="left", fill="x", expand=True, padx=(0, 10))

    def save_name():
        user["name"] = name_entry.get()
        save_data()
        messagebox.showinfo("Успех", "Имя сохранено!")

    save_name_btn = create_modern_button(name_frame, "Сохранить", command=save_name, bg=ACCENT, padx=10)
    save_name_btn.pack(side="right")

    # Автомобиль
    car_frame = tk.Frame(c, bg=CARD)
    car_frame.pack(fill="x", pady=15)

    tk.Label(car_frame, text="Автомобиль:", bg=CARD, fg=SUB, font=("Segoe UI", 12)).pack(anchor="w")
    car_entry = tk.Entry(car_frame, font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
    car_entry.insert(0, user.get("car", ""))
    car_entry.pack(side="left", fill="x", expand=True, padx=(0, 10))

    def save_car():
        user["car"] = car_entry.get()
        save_data()
        messagebox.showinfo("Успех", "Данные об автомобиле сохранены!")

    save_car_btn = create_modern_button(car_frame, "Сохранить", command=save_car, bg=ACCENT, padx=10)
    save_car_btn.pack(side="right")

    # Смена пароля
    pass_frame = tk.Frame(c, bg=CARD)
    pass_frame.pack(fill="x", pady=15)

    tk.Label(pass_frame, text="Новый пароль:", bg=CARD, fg=SUB, font=("Segoe UI", 12)).pack(anchor="w")
    pass1 = tk.Entry(pass_frame, show="*", font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
    pass1.pack(fill="x", pady=5)

    tk.Label(pass_frame, text="Повтор пароля:", bg=CARD, fg=SUB, font=("Segoe UI", 12)).pack(anchor="w")
    pass2 = tk.Entry(pass_frame, show="*", font=("Segoe UI", 14), bd=1, relief="flat", bg=PANEL, fg=TEXT)
    pass2.pack(side="left", fill="x", expand=True, padx=(0, 10))

    def save_password():
        p1 = pass1.get()
        p2 = pass2.get()

        if not p1 or not p2:
            return messagebox.showerror("Ошибка", "Введите пароль")

        if p1 != p2:
            return messagebox.showerror("Ошибка", "Пароли не совпадают")

        user["password"] = p1
        save_data()
        messagebox.showinfo("Успех", "Пароль изменён!")

    save_pass_btn = create_modern_button(pass_frame, "Сохранить", command=save_password, bg=ACCENT, padx=10)
    save_pass_btn.pack(side="right")

    # Кнопка выхода
    logout_btn = create_modern_button(c, "Выйти из аккаунта", command=logout, bg=DANGER)
    logout_btn.pack(fill="x", pady=20)


def logout():
    global current_user
    current_user = None
    update_user()
    show_profile()


# ===================== CALCULATOR =====================
def show_calc():
    clear()

    title_frame = tk.Frame(content, bg=BG)
    title_frame.pack(pady=20)

    tk.Label(
        title_frame,
        text="🧮 Калькулятор расхода топлива",
        bg=BG,
        fg=HIGHLIGHT,
        font=("Segoe UI", 24, "bold")
    ).pack()

    mode = tk.StringVar(value="1")

    switch = tk.Frame(content, bg=BG)
    switch.pack(pady=15)

    def create_switch_btn(text, value, active=False):
        color = ACCENT if active else PANEL
        btn = tk.Button(
            switch,
            text=text,
            bg=color,
            fg=TEXT,
            font=("Segoe UI", 11),
            bd=0,
            padx=25,
            pady=8,
            relief="flat",
            cursor="hand2"
        )
        btn.pack(side="left", padx=5)
        return btn

    btn1 = create_switch_btn("Стоимость поездки", "1", True)
    btn2 = create_switch_btn("Средний расход", "2")

    def switch_mode(value):
        mode.set(value)
        btn1.config(bg=ACCENT if value == "1" else PANEL)
        btn2.config(bg=ACCENT if value == "2" else PANEL)
        build()

    btn1.config(command=lambda: switch_mode("1"))
    btn2.config(command=lambda: switch_mode("2"))

    form = tk.Frame(content, bg=CARD, padx=40, pady=40
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


