"""
==============================================================
  КАЛЬКУЛЯТОР РАСХОДА ТОПЛИВА ТРАНСПОРТНЫХ СРЕДСТВ
  Дипломная работа
==============================================================
"""

import tkinter as tk
from tkinter import ttk, messagebox, simpledialog
import json
import os
import hashlib
import re
from datetime import datetime

# ─────────────────────────────────────────────────────────────
#  КОНФИГУРАЦИЯ И ДАННЫЕ
# ─────────────────────────────────────────────────────────────

DATA_FILE = "users_data.json"

COLORS = {
    "bg":         "#0f1117",
    "panel":      "#1a1d27",
    "card":       "#232736",
    "accent":     "#4f8ef7",
    "accent2":    "#7c5cfc",
    "success":    "#3ecf8e",
    "danger":     "#f75f5f",
    "warning":    "#f7b731",
    "text":       "#e8eaf0",
    "subtext":    "#8a8fa8",
    "border":     "#2e3248",
    "input_bg":   "#161824",
    "hover":      "#2a2f45",
}

FONTS = {
    "title":    ("Segoe UI", 22, "bold"),
    "heading":  ("Segoe UI", 14, "bold"),
    "subhead":  ("Segoe UI", 11, "bold"),
    "body":     ("Segoe UI", 10),
    "small":    ("Segoe UI", 9),
    "mono":     ("Consolas", 10),
    "big":      ("Segoe UI", 32, "bold"),
}

VEHICLE_TYPES = {
    "Легковой автомобиль":       {"norm": 8.5,  "icon": "🚗"},
    "Внедорожник / SUV":         {"norm": 11.0, "icon": "🚙"},
    "Минивэн":                   {"norm": 10.5, "icon": "🚐"},
    "Грузовик (малый)":          {"norm": 14.0, "icon": "🚚"},
    "Грузовик (большой/фура)":   {"norm": 28.0, "icon": "🚛"},
    "Автобус":                   {"norm": 20.0, "icon": "🚌"},
    "Мотоцикл":                  {"norm": 5.0,  "icon": "🏍️"},
    "Спецтехника":               {"norm": 25.0, "icon": "🚜"},
}

FUEL_TYPES = ["АИ-92", "АИ-95", "АИ-98", "ДТ (Дизель)", "Газ (LPG)", "Газ (CNG)"]

ROAD_CONDITIONS = {
    "Трасса (ровная)":      1.0,
    "Смешанный режим":      1.15,
    "Городской цикл":       1.30,
    "Горная дорога":        1.40,
    "Бездорожье / грунт":   1.55,
}

SEASON_FACTORS = {
    "Лето":         1.0,
    "Весна / Осень": 1.05,
    "Зима":         1.18,
}

LOAD_FACTORS = {
    "Пустой (без груза)":   0.90,
    "Лёгкая загрузка":      1.0,
    "Средняя загрузка":     1.12,
    "Полная загрузка":      1.25,
    "Сверхнагрузка":        1.40,
}


# ─────────────────────────────────────────────────────────────
#  УТИЛИТЫ
# ─────────────────────────────────────────────────────────────

def hash_password(password: str) -> str:
    return hashlib.sha256(password.encode()).hexdigest()

def load_data() -> dict:
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {"users": {}}

def save_data(data: dict):
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def is_valid_email(email: str) -> bool:
    return bool(re.match(r"^[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}$", email))

def is_valid_password(password: str) -> bool:
    return len(password) == 6 and password.isdigit()


# ─────────────────────────────────────────────────────────────
#  ВИДЖЕТЫ-ПОМОЩНИКИ
# ─────────────────────────────────────────────────────────────

def styled_button(parent, text, command, color=None, width=20, **kwargs):
    color = color or COLORS["accent"]
    btn = tk.Button(
        parent, text=text, command=command,
        bg=color, fg=COLORS["text"],
        font=FONTS["subhead"],
        relief="flat", cursor="hand2",
        width=width, pady=8,
        activebackground=COLORS["hover"],
        activeforeground=COLORS["text"],
        **kwargs
    )
    btn.bind("<Enter>", lambda e: btn.config(bg=_lighten(color)))
    btn.bind("<Leave>", lambda e: btn.config(bg=color))
    return btn

def _lighten(hex_color: str) -> str:
    hex_color = hex_color.lstrip("#")
    r, g, b = (int(hex_color[i:i+2], 16) for i in (0, 2, 4))
    r, g, b = min(255, r + 30), min(255, g + 30), min(255, b + 30)
    return f"#{r:02x}{g:02x}{b:02x}"

def labeled_entry(parent, label_text, show=None, width=30):
    frame = tk.Frame(parent, bg=COLORS["panel"])
    tk.Label(frame, text=label_text, bg=COLORS["panel"],
             fg=COLORS["subtext"], font=FONTS["small"]).pack(anchor="w")
    entry = tk.Entry(frame, show=show, width=width,
                     bg=COLORS["input_bg"], fg=COLORS["text"],
                     insertbackground=COLORS["text"],
                     relief="flat", font=FONTS["body"],
                     highlightthickness=1,
                     highlightbackground=COLORS["border"],
                     highlightcolor=COLORS["accent"])
    entry.pack(fill="x", pady=(2, 0))
    return frame, entry

def section_label(parent, text):
    tk.Label(parent, text=text, bg=COLORS["bg"],
             fg=COLORS["subtext"], font=FONTS["small"]).pack(anchor="w", padx=6)

def card_frame(parent, **kwargs):
    return tk.Frame(parent, bg=COLORS["card"],
                    highlightthickness=1,
                    highlightbackground=COLORS["border"],
                    **kwargs)


# ─────────────────────────────────────────────────────────────
#  ГЛАВНОЕ ПРИЛОЖЕНИЕ
# ─────────────────────────────────────────────────────────────

class App(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Калькулятор расхода топлива")
        self.geometry("1000x680")
        self.minsize(900, 620)
        self.configure(bg=COLORS["bg"])
        self.resizable(True, True)

        self.data = load_data()
        self.current_user = None           # email текущего пользователя
        self.history = []                  # история расчётов сессии

        self._center_window()
        self.show_auth_screen()

    def _center_window(self):
        self.update_idletasks()
        w, h = 1000, 680
        x = (self.winfo_screenwidth() - w) // 2
        y = (self.winfo_screenheight() - h) // 2
        self.geometry(f"{w}x{h}+{x}+{y}")

    # ── очистка экрана ──────────────────────────────────────
    def clear(self):
        for w in self.winfo_children():
            w.destroy()

    # ══════════════════════════════════════════════════════════
    #  ЭКРАН АВТОРИЗАЦИИ / РЕГИСТРАЦИИ
    # ══════════════════════════════════════════════════════════

    def show_auth_screen(self, mode="login"):
        self.clear()
        self.title("Калькулятор расхода топлива — Вход")

        # Левая декоративная панель
        left = tk.Frame(self, bg=COLORS["accent2"], width=340)
        left.pack(side="left", fill="y")
        left.pack_propagate(False)

        tk.Label(left, text="⛽", font=("Segoe UI Emoji", 52),
                 bg=COLORS["accent2"], fg=COLORS["text"]).pack(pady=(80, 10))
        tk.Label(left, text="Калькулятор\nрасхода топлива",
                 font=FONTS["title"], bg=COLORS["accent2"],
                 fg=COLORS["text"], justify="center").pack()
        tk.Label(left, text="Дипломная работа",
                 font=FONTS["small"], bg=COLORS["accent2"],
                 fg="#d0c8ff").pack(pady=(6, 0))

        # Правая форма
        right = tk.Frame(self, bg=COLORS["panel"])
        right.pack(side="left", fill="both", expand=True)

        form = tk.Frame(right, bg=COLORS["panel"])
        form.place(relx=0.5, rely=0.5, anchor="center")

        if mode == "login":
            tk.Label(form, text="Войти в аккаунт", font=FONTS["heading"],
                     bg=COLORS["panel"], fg=COLORS["text"]).pack(pady=(0, 20))

            _, self.e_email = labeled_entry(form, "Электронная почта")
            self.e_email.master.pack(fill="x", pady=6)

            _, self.e_pass = labeled_entry(form, "Пароль (6 цифр)", show="●")
            self.e_pass.master.pack(fill="x", pady=6)

            styled_button(form, "🔐  Войти", self._do_login,
                          color=COLORS["accent"]).pack(fill="x", pady=(16, 6))
            styled_button(form, "📝  Регистрация",
                          lambda: self.show_auth_screen("register"),
                          color=COLORS["card"]).pack(fill="x")

        else:
            tk.Label(form, text="Создать аккаунт", font=FONTS["heading"],
                     bg=COLORS["panel"], fg=COLORS["text"]).pack(pady=(0, 20))

            _, self.e_email = labeled_entry(form, "Электронная почта")
            self.e_email.master.pack(fill="x", pady=6)

            _, self.e_pass = labeled_entry(form, "Пароль (ровно 6 цифр)", show="●")
            self.e_pass.master.pack(fill="x", pady=6)

            _, self.e_pass2 = labeled_entry(form, "Повторите пароль", show="●")
            self.e_pass2.master.pack(fill="x", pady=6)

            tk.Label(form, text="⚠  Пароль должен состоять из 6 цифр",
                     bg=COLORS["panel"], fg=COLORS["warning"],
                     font=FONTS["small"]).pack(anchor="w", pady=(0, 6))

            styled_button(form, "✅  Зарегистрироваться", self._do_register,
                          color=COLORS["success"]).pack(fill="x", pady=(10, 6))
            styled_button(form, "← Уже есть аккаунт",
                          lambda: self.show_auth_screen("login"),
                          color=COLORS["card"]).pack(fill="x")

    # ── Вход ──────────────────────────────────────────────────
    def _do_login(self):
        email = self.e_email.get().strip().lower()
        password = self.e_pass.get().strip()

        if not is_valid_email(email):
            messagebox.showerror("Ошибка", "Введите корректный e-mail.")
            return
        if not is_valid_password(password):
            messagebox.showerror("Ошибка", "Пароль — 6 цифр.")
            return

        users = self.data.get("users", {})
        if email not in users:
            messagebox.showerror("Ошибка", "Пользователь не найден.")
            return
        if users[email]["password"] != hash_password(password):
            messagebox.showerror("Ошибка", "Неверный пароль.")
            return

        self.current_user = email
        self.show_main_screen()

    # ── Регистрация ───────────────────────────────────────────
    def _do_register(self):
        email = self.e_email.get().strip().lower()
        password = self.e_pass.get().strip()
        password2 = self.e_pass2.get().strip()

        if not is_valid_email(email):
            messagebox.showerror("Ошибка", "Введите корректный e-mail.")
            return
        if not is_valid_password(password):
            messagebox.showerror("Ошибка", "Пароль должен состоять ровно из 6 цифр.")
            return
        if password != password2:
            messagebox.showerror("Ошибка", "Пароли не совпадают.")
            return
        if email in self.data.get("users", {}):
            messagebox.showerror("Ошибка", "Этот e-mail уже зарегистрирован.")
            return

        self.data.setdefault("users", {})[email] = {
            "password": hash_password(password),
            "created": datetime.now().strftime("%Y-%m-%d %H:%M"),
            "calculations": [],
            "settings": {
                "fuel_price": 55.0,
                "default_vehicle": "Легковой автомобиль",
                "default_fuel": "АИ-95",
                "currency": "₽",
            }
        }
        save_data(self.data)
        messagebox.showinfo("Успех", f"Аккаунт создан!\nДобро пожаловать, {email}")
        self.current_user = email
        self.show_main_screen()


    # ══════════════════════════════════════════════════════════
    #  ГЛАВНЫЙ ЭКРАН (sidebar + content)
    # ══════════════════════════════════════════════════════════

    def show_main_screen(self):
        self.clear()
        self.title("Калькулятор расхода топлива")

        # ── Sidebar ─────────────────────────────────────────
        sidebar = tk.Frame(self, bg=COLORS["panel"], width=210)
        sidebar.pack(side="left", fill="y")
        sidebar.pack_propagate(False)

        tk.Label(sidebar, text="⛽", font=("Segoe UI Emoji", 30),
                 bg=COLORS["panel"], fg=COLORS["accent"]).pack(pady=(24, 4))
        tk.Label(sidebar, text="ТопливоКалк",
                 font=FONTS["subhead"], bg=COLORS["panel"],
                 fg=COLORS["text"]).pack()
        tk.Label(sidebar, text=self.current_user,
                 font=FONTS["small"], bg=COLORS["panel"],
                 fg=COLORS["subtext"], wraplength=190).pack(pady=(2, 20))

        tk.Frame(sidebar, bg=COLORS["border"], height=1).pack(fill="x", padx=16)

        self.content = tk.Frame(self, bg=COLORS["bg"])
        self.content.pack(side="left", fill="both", expand=True)

        nav_items = [
            ("🧮  Калькулятор",   self.show_calculator),
            ("📋  История",       self.show_history),
            ("👤  Профиль",       self.show_profile),
            ("⚙️   Настройки",    self.show_settings),
        ]
        for label, cmd in nav_items:
            btn = tk.Button(
                sidebar, text=label, command=cmd,
                bg=COLORS["panel"], fg=COLORS["text"],
                font=FONTS["body"], relief="flat",
                cursor="hand2", anchor="w", padx=20, pady=10,
                activebackground=COLORS["hover"],
                activeforeground=COLORS["accent"],
            )
            btn.pack(fill="x")
            btn.bind("<Enter>", lambda e, b=btn: b.config(bg=COLORS["hover"]))
            btn.bind("<Leave>", lambda e, b=btn: b.config(bg=COLORS["panel"]))

        tk.Frame(sidebar, bg=COLORS["border"], height=1).pack(fill="x", padx=16, pady=10)

        logout_btn = tk.Button(
            sidebar, text="🚪  Выйти", command=self._logout,
            bg=COLORS["panel"], fg=COLORS["danger"],
            font=FONTS["body"], relief="flat", cursor="hand2",
            anchor="w", padx=20, pady=10,
            activebackground=COLORS["hover"],
        )
        logout_btn.pack(fill="x", side="bottom", pady=(0, 20))

        self.show_calculator()

    def _logout(self):
        self.current_user = None
        self.history = []
        self.show_auth_screen()

    def _clear_content(self):
        for w in self.content.winfo_children():
            w.destroy()

    # ══════════════════════════════════════════════════════════
    #  КАЛЬКУЛЯТОР
    # ══════════════════════════════════════════════════════════

    def show_calculator(self):
        self._clear_content()
        user = self.data["users"][self.current_user]
        settings = user.get("settings", {})

        # Заголовок
        hdr = tk.Frame(self.content, bg=COLORS["bg"])
        hdr.pack(fill="x", padx=28, pady=(22, 10))
        tk.Label(hdr, text="🧮  Калькулятор расхода топлива",
                 font=FONTS["heading"], bg=COLORS["bg"],
                 fg=COLORS["text"]).pack(side="left")

        # Прокручиваемая область
        canvas = tk.Canvas(self.content, bg=COLORS["bg"], highlightthickness=0)
        scrollbar = ttk.Scrollbar(self.content, orient="vertical",
                                  command=canvas.yview)
        scroll_frame = tk.Frame(canvas, bg=COLORS["bg"])

        scroll_frame.bind("<Configure>",
            lambda e: canvas.configure(scrollregion=canvas.bbox("all")))

        canvas.create_window((0, 0), window=scroll_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)

        canvas.pack(side="left", fill="both", expand=True, padx=(28, 0))
        scrollbar.pack(side="right", fill="y", padx=(0, 8))

        # ── Форма ────────────────────────────────────────────
        row1 = tk.Frame(scroll_frame, bg=COLORS["bg"])
        row1.pack(fill="x", pady=6)

        # Тип ТС
        col1 = card_frame(row1)
        col1.pack(side="left", fill="both", expand=True, padx=(0, 8))
        tk.Label(col1, text="Тип транспортного средства",
                 font=FONTS["subhead"], bg=COLORS["card"],
                 fg=COLORS["text"]).pack(anchor="w", padx=12, pady=(10, 4))
        self.var_vehicle = tk.StringVar(
            value=settings.get("default_vehicle", "Легковой автомобиль"))
        for vtype, info in VEHICLE_TYPES.items():
            rb = tk.Radiobutton(
                col1, text=f"{info['icon']}  {vtype}",
                variable=self.var_vehicle, value=vtype,
                bg=COLORS["card"], fg=COLORS["text"],
                selectcolor=COLORS["accent2"],
                activebackground=COLORS["card"],
                font=FONTS["body"], cursor="hand2",
                command=self._update_norm,
            )
            rb.pack(anchor="w", padx=16, pady=1)
        tk.Frame(col1, bg=COLORS["bg"], height=8).pack()

        # Маршрут и параметры
        col2 = tk.Frame(row1, bg=COLORS["bg"])
        col2.pack(side="left", fill="both", expand=True)

        # Расстояние
        dist_card = card_frame(col2)
        dist_card.pack(fill="x", pady=(0, 8))
        tk.Label(dist_card, text="Расстояние (км)",
                 font=FONTS["subhead"], bg=COLORS["card"],
                 fg=COLORS["text"]).pack(anchor="w", padx=12, pady=(10, 4))
        self.e_dist = tk.Entry(dist_card, width=14,
                               bg=COLORS["input_bg"], fg=COLORS["text"],
                               insertbackground=COLORS["text"],
                               relief="flat", font=FONTS["heading"],
                               highlightthickness=1,
                               highlightbackground=COLORS["border"],
                               highlightcolor=COLORS["accent"])
        self.e_dist.insert(0, "100")
        self.e_dist.pack(padx=12, pady=(0, 10), anchor="w")

        # Норма расхода
        norm_card = card_frame(col2)
        norm_card.pack(fill="x", pady=(0, 8))
        tk.Label(norm_card, text="Норма расхода (л/100км)",
                 font=FONTS["subhead"], bg=COLORS["card"],
                 fg=COLORS["text"]).pack(anchor="w", padx=12, pady=(10, 4))
        self.e_norm = tk.Entry(norm_card, width=14,
                               bg=COLORS["input_bg"], fg=COLORS["text"],
                               insertbackground=COLORS["text"],
                               relief="flat", font=FONTS["heading"],
                               highlightthickness=1,
                               highlightbackground=COLORS["border"],
                               highlightcolor=COLORS["accent"])
        self.e_norm.insert(0, str(VEHICLE_TYPES[self.var_vehicle.get()]["norm"]))
        self.e_norm.pack(padx=12, pady=(0, 10), anchor="w")

        # Цена топлива
        price_card = card_frame(col2)
        price_card.pack(fill="x")
        tk.Label(price_card, text=f"Цена топлива ({settings.get('currency','₽')}/л)",
                 font=FONTS["subhead"], bg=COLORS["card"],
                 fg=COLORS["text"]).pack(anchor="w", padx=12, pady=(10, 4))
        self.e_price = tk.Entry(price_card, width=14,
                                bg=COLORS["input_bg"], fg=COLORS["text"],
                                insertbackground=COLORS["text"],
                                relief="flat", font=FONTS["heading"],
                                highlightthickness=1,
                                highlightbackground=COLORS["border"],
                                highlightcolor=COLORS["accent"])
        self.e_price.insert(0, str(settings.get("fuel_price", 55.0)))
        self.e_price.pack(padx=12, pady=(0, 10), anchor="w")

        # ── Строка 2: Условия ────────────────────────────────
        row2 = tk.Frame(scroll_frame, bg=COLORS["bg"])
        row2.pack(fill="x", pady=6)

        def combo_card(parent, title, values, default):
            cf = card_frame(parent)
            cf.pack(side="left", fill="both", expand=True, padx=(0, 8))
            tk.Label(cf, text=title, font=FONTS["subhead"],
                     bg=COLORS["card"], fg=COLORS["text"]).pack(
                         anchor="w", padx=12, pady=(10, 4))
            var = tk.StringVar(value=default)
            combo = ttk.Combobox(cf, textvariable=var, values=values,
                                 state="readonly", width=22)
            combo.pack(padx=12, pady=(0, 10), anchor="w")
            return var

        self.var_road = combo_card(row2, "Тип дороги",
                                   list(ROAD_CONDITIONS.keys()),
                                   "Смешанный режим")
        self.var_season = combo_card(row2, "Сезон",
                                     list(SEASON_FACTORS.keys()), "Лето")
        self.var_load = combo_card(row2, "Загрузка",
                                   list(LOAD_FACTORS.keys()),
                                   "Лёгкая загрузка")

        # Тип топлива
        fuel_cf = card_frame(row2)
        fuel_cf.pack(side="left", fill="both", expand=True)
        tk.Label(fuel_cf, text="Тип топлива", font=FONTS["subhead"],
                 bg=COLORS["card"], fg=COLORS["text"]).pack(
                     anchor="w", padx=12, pady=(10, 4))
        self.var_fuel = tk.StringVar(
            value=settings.get("default_fuel", "АИ-95"))
        fuel_combo = ttk.Combobox(fuel_cf, textvariable=self.var_fuel,
                                  values=FUEL_TYPES, state="readonly", width=16)
        fuel_combo.pack(padx=12, pady=(0, 10), anchor="w")

        # ── Кнопка расчёта ───────────────────────────────────
        btn_frame = tk.Frame(scroll_frame, bg=COLORS["bg"])
        btn_frame.pack(pady=14)
        styled_button(btn_frame, "⚡  Рассчитать", self._calculate,
                      color=COLORS["accent"], width=28).pack()

        # ── Результат ────────────────────────────────────────
        self.result_frame = card_frame(scroll_frame)
        self.result_frame.pack(fill="x", pady=(0, 20))
        self.result_lbl = tk.Label(
            self.result_frame,
            text="Заполните параметры и нажмите «Рассчитать»",
            font=FONTS["body"], bg=COLORS["card"],
            fg=COLORS["subtext"], pady=18)
        self.result_lbl.pack()

    def _update_norm(self):
        vtype = self.var_vehicle.get()
        norm = VEHICLE_TYPES[vtype]["norm"]
        self.e_norm.delete(0, "end")
        self.e_norm.insert(0, str(norm))

    def _calculate(self):
        try:
            distance = float(self.e_dist.get().replace(",", "."))
            norm = float(self.e_norm.get().replace(",", "."))
            price = float(self.e_price.get().replace(",", "."))
        except ValueError:
            messagebox.showerror("Ошибка", "Введите числовые значения в поля расстояния, нормы и цены.")
            return

        if distance <= 0 or norm <= 0 or price <= 0:
            messagebox.showerror("Ошибка", "Значения должны быть больше нуля.")
            return

        road_k   = ROAD_CONDITIONS[self.var_road.get()]
        season_k = SEASON_FACTORS[self.var_season.get()]
        load_k   = LOAD_FACTORS[self.var_load.get()]

        base_fuel   = distance * norm / 100
        total_fuel  = base_fuel * road_k * season_k * load_k
        total_cost  = total_fuel * price

        currency = self.data["users"][self.current_user]["settings"].get("currency", "₽")
        vtype    = self.var_vehicle.get()
        icon     = VEHICLE_TYPES[vtype]["icon"]

        # Обновляем результат
        for w in self.result_frame.winfo_children():
            w.destroy()

        tk.Label(self.result_frame, text="📊  Результаты расчёта",
                 font=FONTS["subhead"], bg=COLORS["card"],
                 fg=COLORS["subtext"]).pack(pady=(12, 4))

        results_row = tk.Frame(self.result_frame, bg=COLORS["card"])
        results_row.pack(pady=10)

        def res_col(parent, label, value, color):
            col = tk.Frame(parent, bg=COLORS["card"])
            col.pack(side="left", padx=24)
            tk.Label(col, text=value, font=FONTS["big"],
                     bg=COLORS["card"], fg=color).pack()
            tk.Label(col, text=label, font=FONTS["small"],
                     bg=COLORS["card"], fg=COLORS["subtext"]).pack()

        res_col(results_row, "Топливо (л)", f"{total_fuel:.2f}", COLORS["accent"])
        res_col(results_row, f"Стоимость ({currency})", f"{total_cost:.0f}", COLORS["success"])
        res_col(results_row, "Расход (л/100км)", f"{total_fuel/distance*100:.2f}", COLORS["warning"])

        tk.Label(self.result_frame,
                 text=f"{icon}  {vtype}  ·  {distance:.0f} км  ·  {self.var_fuel.get()}  ·  {self.var_road.get()}",
                 font=FONTS["small"], bg=COLORS["card"],
                 fg=COLORS["subtext"]).pack(pady=(0, 12))

        # Сохраняем в историю
        record = {
            "date":         datetime.now().strftime("%d.%m.%Y %H:%M"),
            "vehicle":      vtype,
            "distance":     distance,
            "norm":         norm,
            "fuel_type":    self.var_fuel.get(),
            "road":         self.var_road.get(),
            "season":       self.var_season.get(),
            "load":         self.var_load.get(),
            "total_fuel":   round(total_fuel, 2),
            "total_cost":   round(total_cost, 2),
            "price":        price,
        }
        self.data["users"][self.current_user].setdefault("calculations", []).append(record)
        save_data(self.data)


    # ══════════════════════════════════════════════════════════
    #  ИСТОРИЯ
    # ══════════════════════════════════════════════════════════

    def show_history(self):
        self._clear_content()
        calcs = self.data["users"][self.current_user].get("calculations", [])
        currency = self.data["users"][self.current_user]["settings"].get("currency", "₽")

        hdr = tk.Frame(self.content, bg=COLORS["bg"])
        hdr.pack(fill="x", padx=28, pady=(22, 10))
        tk.Label(hdr, text="📋  История расчётов",
                 font=FONTS["heading"], bg=COLORS["bg"],
                 fg=COLORS["text"]).pack(side="left")
        tk.Label(hdr, text=f"{len(calcs)} записей",
                 font=FONTS["small"], bg=COLORS["bg"],
                 fg=COLORS["subtext"]).pack(side="left", padx=12)

        if calcs:
            styled_button(hdr, "🗑  Очистить историю",
                          self._clear_history, color=COLORS["danger"],
                          width=18).pack(side="right")

        if not calcs:
            tk.Label(self.content,
                     text="История расчётов пуста.\nВыполните расчёт в калькуляторе.",
                     font=FONTS["body"], bg=COLORS["bg"],
                     fg=COLORS["subtext"], justify="center").pack(expand=True)
            return

        # Таблица
        container = tk.Frame(self.content, bg=COLORS["bg"])
        container.pack(fill="both", expand=True, padx=28, pady=(0, 20))

        style = ttk.Style()
        style.theme_use("clam")
        style.configure("Custom.Treeview",
                         background=COLORS["card"],
                         foreground=COLORS["text"],
                         fieldbackground=COLORS["card"],
                         rowheight=30,
                         font=FONTS["body"],
                         borderwidth=0)
        style.configure("Custom.Treeview.Heading",
                         background=COLORS["panel"],
                         foreground=COLORS["accent"],
                         font=FONTS["small"])
        style.map("Custom.Treeview",
                  background=[("selected", COLORS["accent2"])])

        cols = ("Дата", "ТС", "Км", "Топливо (л)", f"Стоимость ({currency})", "Тип топлива")
        tree = ttk.Treeview(container, columns=cols, show="headings",
                            style="Custom.Treeview")

        widths = [120, 200, 70, 110, 130, 100]
        for col, w in zip(cols, widths):
            tree.heading(col, text=col)
            tree.column(col, width=w, anchor="center")

        for r in reversed(calcs):
            tree.insert("", "end", values=(
                r["date"], r["vehicle"], f"{r['distance']:.0f}",
                f"{r['total_fuel']:.2f}", f"{r['total_cost']:.0f}",
                r.get("fuel_type", "—")
            ))

        vsb = ttk.Scrollbar(container, orient="vertical", command=tree.yview)
        tree.configure(yscrollcommand=vsb.set)
        tree.pack(side="left", fill="both", expand=True)
        vsb.pack(side="right", fill="y")

    def _clear_history(self):
        if messagebox.askyesno("Подтверждение",
                                "Удалить всю историю расчётов?"):
            self.data["users"][self.current_user]["calculations"] = []
            save_data(self.data)
            self.show_history()


    # ══════════════════════════════════════════════════════════
    #  ПРОФИЛЬ
    # ══════════════════════════════════════════════════════════

    def show_profile(self):
        self._clear_content()
        user = self.data["users"][self.current_user]

        tk.Label(self.content, text="👤  Профиль пользователя",
                 font=FONTS["heading"], bg=COLORS["bg"],
                 fg=COLORS["text"]).pack(anchor="w", padx=28, pady=(22, 10))

        main = tk.Frame(self.content, bg=COLORS["bg"])
        main.pack(fill="both", padx=28, expand=True)

        # Карточка информации
        info_card = card_frame(main)
        info_card.pack(fill="x", pady=(0, 16))

        tk.Label(info_card, text="👤", font=("Segoe UI Emoji", 40),
                 bg=COLORS["card"]).pack(pady=(20, 8))
        tk.Label(info_card, text=self.current_user,
                 font=FONTS["heading"], bg=COLORS["card"],
                 fg=COLORS["text"]).pack()
        tk.Label(info_card,
                 text=f"Зарегистрирован: {user.get('created', '—')}",
                 font=FONTS["small"], bg=COLORS["card"],
                 fg=COLORS["subtext"]).pack(pady=(4, 0))

        calcs = user.get("calculations", [])
        tk.Label(info_card,
                 text=f"Всего расчётов: {len(calcs)}",
                 font=FONTS["small"], bg=COLORS["card"],
                 fg=COLORS["subtext"]).pack(pady=(2, 16))

        # Действия
        actions = card_frame(main)
        actions.pack(fill="x")
        tk.Label(actions, text="Управление аккаунтом",
                 font=FONTS["subhead"], bg=COLORS["card"],
                 fg=COLORS["subtext"]).pack(anchor="w", padx=16, pady=(12, 8))

        btn_row = tk.Frame(actions, bg=COLORS["card"])
        btn_row.pack(padx=16, pady=(0, 16))

        styled_button(btn_row, "🔑  Сменить пароль",
                      self._change_password,
                      color=COLORS["accent"], width=20).pack(side="left", padx=(0, 10))
        styled_button(btn_row, "🗑  Удалить аккаунт",
                      self._delete_account,
                      color=COLORS["danger"], width=20).pack(side="left")

    def _change_password(self):
        old = simpledialog.askstring("Смена пароля",
                                     "Введите текущий пароль (6 цифр):",
                                     show="*", parent=self)
        if old is None:
            return
        if not is_valid_password(old):
            messagebox.showerror("Ошибка", "Неверный формат пароля.")
            return
        if self.data["users"][self.current_user]["password"] != hash_password(old):
            messagebox.showerror("Ошибка", "Текущий пароль неверен.")
            return

        new = simpledialog.askstring("Смена пароля",
                                     "Введите новый пароль (6 цифр):",
                                     show="*", parent=self)
        if new is None:
            return
        if not is_valid_password(new):
            messagebox.showerror("Ошибка", "Пароль должен состоять ровно из 6 цифр.")
            return

        new2 = simpledialog.askstring("Смена пароля",
                                      "Повторите новый пароль:",
                                      show="*", parent=self)
        if new2 is None:
            return
        if new != new2:
            messagebox.showerror("Ошибка", "Пароли не совпадают.")
            return

        self.data["users"][self.current_user]["password"] = hash_password(new)
        save_data(self.data)
        messagebox.showinfo("Успех", "Пароль успешно изменён!")

    def _delete_account(self):
        confirm = messagebox.askyesno(
            "Удаление аккаунта",
            "Вы уверены? Все данные будут удалены безвозвратно!")
        if not confirm:
            return

        password = simpledialog.askstring(
            "Подтверждение", "Введите пароль для подтверждения:",
            show="*", parent=self)
        if password is None:
            return
        if self.data["users"][self.current_user]["password"] != hash_password(password):
            messagebox.showerror("Ошибка", "Неверный пароль.")
            return

        del self.data["users"][self.current_user]
        save_data(self.data)
        self.current_user = None
        messagebox.showinfo("Готово", "Аккаунт удалён.")
        self.show_auth_screen()


    # ══════════════════════════════════════════════════════════
    #  НАСТРОЙКИ
    # ══════════════════════════════════════════════════════════

    def show_settings(self):
        self._clear_content()
        user = self.data["users"][self.current_user]
        settings = user.get("settings", {})

        tk.Label(self.content, text="⚙️  Настройки",
                 font=FONTS["heading"], bg=COLORS["bg"],
                 fg=COLORS["text"]).pack(anchor="w", padx=28, pady=(22, 10))

        main = tk.Frame(self.content, bg=COLORS["bg"])
        main.pack(fill="both", padx=28, expand=True)

        # ── Карточка: Параметры по умолчанию ─────────────────
        c1 = card_frame(main)
        c1.pack(fill="x", pady=(0, 12))
        tk.Label(c1, text="Параметры по умолчанию",
                 font=FONTS["subhead"], bg=COLORS["card"],
                 fg=COLORS["text"]).pack(anchor="w", padx=16, pady=(12, 6))

        fields = tk.Frame(c1, bg=COLORS["card"])
        fields.pack(fill="x", padx=16, pady=(0, 14))

        # Цена топлива
        def setting_row(parent, label, widget_fn):
            row = tk.Frame(parent, bg=COLORS["card"])
            row.pack(fill="x", pady=5)
            tk.Label(row, text=label, width=28, anchor="w",
                     font=FONTS["body"], bg=COLORS["card"],
                     fg=COLORS["text"]).pack(side="left")
            w = widget_fn(row)
            w.pack(side="left")
            return w

        self.s_price = tk.Entry(fields, width=10,
                                bg=COLORS["input_bg"], fg=COLORS["text"],
                                insertbackground=COLORS["text"],
                                relief="flat", font=FONTS["body"],
                                highlightthickness=1,
                                highlightbackground=COLORS["border"],
                                highlightcolor=COLORS["accent"])
        self.s_price.insert(0, str(settings.get("fuel_price", 55.0)))
        setting_row(fields, "Цена топлива (руб/л) по умолч.:",
                    lambda p: self.s_price)

        self.s_vehicle = tk.StringVar(
            value=settings.get("default_vehicle", "Легковой автомобиль"))
        def mk_vcombo(parent):
            c = ttk.Combobox(parent, textvariable=self.s_vehicle,
                             values=list(VEHICLE_TYPES.keys()),
                             state="readonly", width=28)
            return c
        setting_row(fields, "ТС по умолчанию:", mk_vcombo)

        self.s_fuel = tk.StringVar(value=settings.get("default_fuel", "АИ-95"))
        def mk_fcombo(parent):
            c = ttk.Combobox(parent, textvariable=self.s_fuel,
                             values=FUEL_TYPES, state="readonly", width=28)
            return c
        setting_row(fields, "Тип топлива по умолч.:", mk_fcombo)

        # ── Карточка: Валюта ──────────────────────────────────
        c2 = card_frame(main)
        c2.pack(fill="x", pady=(0, 12))
        tk.Label(c2, text="Валюта",
                 font=FONTS["subhead"], bg=COLORS["card"],
                 fg=COLORS["text"]).pack(anchor="w", padx=16, pady=(12, 6))

        cur_frame = tk.Frame(c2, bg=COLORS["card"])
        cur_frame.pack(fill="x", padx=16, pady=(0, 12))
        self.s_currency = tk.StringVar(value=settings.get("currency", "₽"))
        for sym, name in [("₽", "Рубль"), ("$", "Доллар"),
                          ("€", "Евро"), ("₸", "Тенге")]:
            tk.Radiobutton(cur_frame, text=f"{sym}  {name}",
                           variable=self.s_currency, value=sym,
                           bg=COLORS["card"], fg=COLORS["text"],
                           selectcolor=COLORS["accent2"],
                           activebackground=COLORS["card"],
                           font=FONTS["body"], cursor="hand2").pack(
                               side="left", padx=14)

        # Кнопка сохранить
        btn_f = tk.Frame(main, bg=COLORS["bg"])
        btn_f.pack(pady=10)
        styled_button(btn_f, "💾  Сохранить настройки",
                      self._save_settings,
                      color=COLORS["success"], width=26).pack()

    def _save_settings(self):
        try:
            price = float(self.s_price.get().replace(",", "."))
            if price <= 0:
                raise ValueError
        except ValueError:
            messagebox.showerror("Ошибка", "Введите корректную цену топлива.")
            return

        self.data["users"][self.current_user]["settings"] = {
            "fuel_price":       price,
            "default_vehicle":  self.s_vehicle.get(),
            "default_fuel":     self.s_fuel.get(),
            "currency":         self.s_currency.get(),
        }
        save_data(self.data)
        messagebox.showinfo("Успех", "Настройки сохранены!")


# ─────────────────────────────────────────────────────────────
#  ЗАПУСК
# ─────────────────────────────────────────────────────────────

if __name__ == "__main__":
    app = App()
    app.mainloop()
