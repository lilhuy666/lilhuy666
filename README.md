import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import json
import os
import re
import hashlib
import shutil
from datetime import datetime

# ─── PIL import (подавляем серый цвет — импортируем один раз наверху) ──────────
try:
    from PIL import Image, ImageTk, ImageDraw
except ImportError:
    import subprocess, sys
    subprocess.check_call([sys.executable, "-m", "pip", "install", "Pillow"])
    from PIL import Image, ImageTk, ImageDraw

# ─── Data storage ──────────────────────────────────────────────────────────────
DATA_FILE = os.path.join(os.path.expanduser("~"), ".fuel_calc_data.json")
PHOTO_DIR = os.path.join(os.path.expanduser("~"), ".fuel_calc_photos")
os.makedirs(PHOTO_DIR, exist_ok=True)

def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {"users": {}, "current_user": None}

def save_data(data):
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def hash_pw(pw):
    return hashlib.sha256(pw.encode()).hexdigest()

def is_valid_email(email):
    return re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", email) is not None

# ─── Currencies ────────────────────────────────────────────────────────────────
CURRENCIES = {
    "₽ RUB": "₽",
    "$ USD": "$",
    "€ EUR": "€",
    "£ GBP": "£",
    "₴ UAH": "₴",
    "₸ KZT": "₸",
}

# ─── Theme ─────────────────────────────────────────────────────────────────────
THEMES = {
    "dark": {
        "bg":        "#0d1117",
        "bg2":       "#161b22",
        "bg3":       "#21262d",
        "fg":        "#e6edf3",
        "fg2":       "#8b949e",
        "accent":    "#58a6ff",
        "green":     "#3fb950",
        "yellow":    "#d29922",
        "red":       "#f85149",
        "border":    "#30363d",
        "btn":       "#238636",
        "btn_hover": "#2ea043",
        "btn2":      "#21262d",
        "btn2_hover":"#30363d",
        "input_bg":  "#0d1117",
    },
    "light": {
        "bg":        "#f0f4f8",
        "bg2":       "#ffffff",
        "bg3":       "#e2e8f0",
        "fg":        "#1a202c",
        "fg2":       "#718096",
        "accent":    "#3182ce",
        "green":     "#38a169",
        "yellow":    "#d69e2e",
        "red":       "#e53e3e",
        "border":    "#cbd5e0",
        "btn":       "#3182ce",
        "btn_hover": "#2b6cb0",
        "btn2":      "#e2e8f0",
        "btn2_hover":"#cbd5e0",
        "input_bg":  "#ffffff",
    }
}

current_theme = "dark"

def T():
    return THEMES[current_theme]

# ─── Helper: placeholder image ─────────────────────────────────────────────────
def make_placeholder(w=300, h=180, text="Нет фото"):
    img = Image.new("RGB", (w, h), "#1c2333")
    draw = ImageDraw.Draw(img)
    draw.rectangle([0, 0, w-1, h-1], outline="#30363d", width=2)
    draw.text((w//2 - 30, h//2 - 8), text, fill="#8b949e")
    return img

def load_car_image(path, w=300, h=180):
    try:
        img = Image.open(path).convert("RGB")
        img.thumbnail((w, h), Image.LANCZOS)
        nw, nh = img.size
        left = (nw - w) // 2 if nw > w else 0
        top  = (nh - h) // 2 if nh > h else 0
        img = img.crop((left, top, left+min(nw,w), top+min(nh,h)))
        bg = Image.new("RGB", (w, h), "#1c2333")
        bg.paste(img, ((w - img.width)//2, (h - img.height)//2))
        return bg
    except Exception:
        return make_placeholder(w, h)

# ─── Main Application ──────────────────────────────────────────────────────────
class FuelApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Калькулятор расхода топлива")
        self.geometry("1100x720")
        self.minsize(900, 600)
        self.resizable(True, True)

        self.data = load_data()
        self.current_user = self.data.get("current_user")
        self._photo_refs = {}
        self._currency_symbol = "₽"

        self._build_ui()

    # ── Layout ──────────────────────────────────────────────────────────────────
    def _build_ui(self):
        t = T()
        self.configure(bg=t["bg"])

        # Header
        self.header = tk.Frame(self, bg=t["bg2"], pady=12)
        self.header.pack(fill="x", side="top")

        self.menu_btn = tk.Button(self.header, text="☰", font=("Courier", 16, "bold"),
                                  bg=t["bg3"], fg=t["fg"], bd=0, padx=10, pady=4,
                                  cursor="hand2", command=self._toggle_sidebar)
        self.menu_btn.pack(side="left", padx=16)

        tk.Label(self.header, text="Калькулятор расхода топлива",
                 font=("Georgia", 18, "bold"), bg=t["bg2"], fg=t["fg"]).pack(side="left", padx=8)

        tk.Label(self.header, text="Расчёт расхода топлива для вашего автомобиля",
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(side="left", padx=4)

        self.settings_btn = tk.Button(self.header, text="⚙", font=("Courier", 16),
                                       bg=t["bg3"], fg=t["fg"], bd=0, padx=10, pady=4,
                                       cursor="hand2", command=lambda: self._show_section("settings"))
        self.settings_btn.pack(side="right", padx=16)

        # Body
        self.body = tk.Frame(self, bg=t["bg"])
        self.body.pack(fill="both", expand=True)

        # Sidebar — скрыта при старте
        self.sidebar = tk.Frame(self.body, bg=t["bg2"], width=200)
        self.sidebar.pack_propagate(False)
        self._sidebar_visible = False   # ← сайдбар скрыт при запуске
        self._build_sidebar()
        # НЕ пакуем sidebar сразу

        # Content
        self.content = tk.Frame(self.body, bg=t["bg"])
        self.content.pack(side="left", fill="both", expand=True)

        # Sections
        self.sections = {}
        for name in ["calculator", "profile", "history", "about", "settings"]:
            frame = tk.Frame(self.content, bg=t["bg"])
            self.sections[name] = frame

        self._build_calculator()
        self._build_profile()
        self._build_history()
        self._build_about()
        self._build_settings()

        self._show_section("calculator")

    def _rebuild_ui(self):
        self.header.destroy()
        self.body.destroy()
        self._photo_refs.clear()
        self._build_ui()

    def _build_sidebar(self):
        t = T()
        for w in self.sidebar.winfo_children():
            w.destroy()
        items = [
            ("🧮  Калькулятор", "calculator"),
            ("👤  Профиль",     "profile"),
            ("📋  История",     "history"),
            ("ℹ️   О приложении","about"),
        ]
        tk.Frame(self.sidebar, bg=t["bg2"], height=20).pack()
        for label, key in items:
            btn = tk.Button(self.sidebar, text=label, font=("Courier", 12),
                            bg=t["bg2"], fg=t["fg"], bd=0, padx=20, pady=12,
                            anchor="w", cursor="hand2",
                            activebackground=t["bg3"], activeforeground=t["fg"],
                            command=lambda k=key: self._show_section(k))
            btn.pack(fill="x")
            btn.bind("<Enter>", lambda e, b=btn: b.configure(bg=T()["bg3"]))
            btn.bind("<Leave>", lambda e, b=btn: b.configure(bg=T()["bg2"]))

        # Разделитель без логина пользователя (убран по запросу)
        tk.Frame(self.sidebar, bg=t["border"], height=1).pack(fill="x", padx=16, pady=8)

    def _toggle_sidebar(self):
        if self._sidebar_visible:
            self.sidebar.pack_forget()
        else:
            self.sidebar.pack(side="left", fill="y", before=self.content)
        self._sidebar_visible = not self._sidebar_visible

    def _show_section(self, name):
        for k, f in self.sections.items():
            f.pack_forget()
        self.sections[name].pack(fill="both", expand=True)
        if name == "history":
            self._refresh_history()
        if name == "profile":
            self._refresh_profile()

    # ── Calculator ──────────────────────────────────────────────────────────────
    def _build_calculator(self):
        t = T()
        f = self.sections["calculator"]
        for w in f.winfo_children(): w.destroy()

        outer = tk.Frame(f, bg=t["bg"])
        outer.pack(fill="both", expand=True, padx=24, pady=24)

        top = tk.Frame(outer, bg=t["bg"])
        top.pack(fill="x")

        # Left: car image + selector
        left = tk.Frame(top, bg=t["bg2"], bd=0, relief="flat")
        left.pack(side="left", fill="y", padx=(0, 16))

        self.calc_car_frame = tk.Frame(left, bg=t["bg2"])
        self.calc_car_frame.pack()
        self.calc_car_label = tk.Label(self.calc_car_frame, bg=t["bg2"])
        self.calc_car_label.pack()
        self._set_calc_car_image(None)

        tk.Label(left, text="Автомобиль:", font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(pady=(8,0))
        self.calc_car_var = tk.StringVar(value="— Без авто —")
        self.calc_car_combo = ttk.Combobox(left, textvariable=self.calc_car_var,
                                            state="readonly", width=22, font=("Courier", 10))
        self.calc_car_combo.pack(padx=12, pady=(2, 8))
        self.calc_car_combo.bind("<<ComboboxSelected>>", self._on_car_selected)
        self._refresh_car_combo()

        # Quick stats
        stats_frame = tk.Frame(left, bg=t["bg3"])
        stats_frame.pack(fill="x", padx=8, pady=8, ipady=8)
        tk.Label(stats_frame, text="Быстрая статистика",
                 font=("Georgia", 12, "bold"), bg=t["bg3"], fg=t["fg"]).pack(anchor="w", padx=12, pady=(8,4))

        self.stat_labels = {}
        stats = [
            ("avg",  "💧  Средний расход", "— л/100 км", t["green"]),
            ("dist", "📏  Расстояние",      "— км",       t["accent"]),
            ("fuel", "⛽  Топлива",         "— л",        t["yellow"]),
            ("cost", "💳  Стоимость",       "—",          t["red"]),
        ]
        for key, label, default, color in stats:
            row = tk.Frame(stats_frame, bg=t["bg3"])
            row.pack(fill="x", padx=12, pady=3)
            tk.Label(row, text=label, font=("Courier", 10), bg=t["bg3"], fg=t["fg2"],
                     width=22, anchor="w").pack(side="left")
            lbl = tk.Label(row, text=default, font=("Courier", 10, "bold"),
                           bg=t["bg3"], fg=color)
            lbl.pack(side="right")
            self.stat_labels[key] = lbl

        # Right: calculator form
        right = tk.Frame(top, bg=t["bg2"])
        right.pack(side="left", fill="both", expand=True)

        tk.Label(right, text="Калькулятор", font=("Georgia", 15, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(16, 4))

        form = tk.Frame(right, bg=t["bg2"])
        form.pack(fill="x", padx=20)

        # Поля пустые по умолчанию
        self.dist_var  = tk.StringVar(value="")
        self.fuel_var  = tk.StringVar(value="")
        self.price_var = tk.StringVar(value="")

        # Расстояние
        tk.Label(form, text="Расстояние", font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(8,2))
        row_d = tk.Frame(form, bg=t["input_bg"], bd=1, relief="solid")
        row_d.pack(fill="x", pady=2)
        tk.Label(row_d, text="📏", font=("Courier", 13), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="left")
        tk.Entry(row_d, textvariable=self.dist_var, font=("Courier", 13),
                 bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                 bd=0, relief="flat").pack(side="left", fill="x", expand=True, pady=8)
        tk.Label(row_d, text="км", font=("Courier", 10), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="right")

        # Объём топлива
        tk.Label(form, text="Объём топлива", font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(8,2))
        row_f = tk.Frame(form, bg=t["input_bg"], bd=1, relief="solid")
        row_f.pack(fill="x", pady=2)
        tk.Label(row_f, text="⛽", font=("Courier", 13), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="left")
        tk.Entry(row_f, textvariable=self.fuel_var, font=("Courier", 13),
                 bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                 bd=0, relief="flat").pack(side="left", fill="x", expand=True, pady=8)
        tk.Label(row_f, text="л", font=("Courier", 10), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="right")

        # Цена топлива + выбор валюты
        tk.Label(form, text="Цена топлива за литр", font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(8,2))
        row_p = tk.Frame(form, bg=t["input_bg"], bd=1, relief="solid")
        row_p.pack(fill="x", pady=2)
        tk.Label(row_p, text="💰", font=("Courier", 13), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="left")
        tk.Entry(row_p, textvariable=self.price_var, font=("Courier", 13),
                 bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                 bd=0, relief="flat").pack(side="left", fill="x", expand=True, pady=8)

        self.currency_var = tk.StringVar(value="₽ RUB")
        currency_combo = ttk.Combobox(row_p, textvariable=self.currency_var,
                                       values=list(CURRENCIES.keys()),
                                       state="readonly", width=8, font=("Courier", 10))
        currency_combo.pack(side="right", padx=6, pady=4)
        currency_combo.bind("<<ComboboxSelected>>", self._on_currency_changed)

        # Кнопки
        btn_row = tk.Frame(right, bg=t["bg2"])
        btn_row.pack(fill="x", padx=20, pady=12)
        tk.Button(btn_row, text="🧮  Рассчитать",
                  font=("Georgia", 12, "bold"), bg=t["btn"], fg="white", bd=0, pady=10,
                  cursor="hand2", command=self._calculate).pack(side="left", fill="x", expand=True, padx=(0, 8))
        tk.Button(btn_row, text="🔄  Очистить",
                  font=("Courier", 11), bg=t["btn2"], fg=t["fg"], bd=0, pady=10,
                  cursor="hand2", command=self._clear_calc).pack(side="left", fill="x", expand=True)

        # Результат
        self.result_frame = tk.Frame(right, bg=t["bg3"])
        self.result_frame.pack(fill="x", padx=20, pady=(4, 16))

        res_left = tk.Frame(self.result_frame, bg=t["bg3"])
        res_left.pack(side="left", fill="x", expand=True, padx=16, pady=12)
        tk.Label(res_left, text="💧  Расход топлива", font=("Courier", 10), bg=t["bg3"], fg=t["fg2"]).pack(anchor="w")
        self.result_consumption_label = tk.Label(res_left, text="— л/100 км",
                                                  font=("Georgia", 18, "bold"), bg=t["bg3"], fg=t["green"])
        self.result_consumption_label.pack(anchor="w")

        res_right = tk.Frame(self.result_frame, bg=t["bg3"])
        res_right.pack(side="left", fill="x", expand=True, padx=16, pady=12)
        tk.Label(res_right, text="💳  Стоимость поездки", font=("Courier", 10), bg=t["bg3"], fg=t["fg2"]).pack(anchor="w")
        self.result_cost_label = tk.Label(res_right, text="—",
                                           font=("Georgia", 18, "bold"), bg=t["bg3"], fg=t["fg"])
        self.result_cost_label.pack(anchor="w")

    def _on_currency_changed(self, event=None):
        self._currency_symbol = CURRENCIES.get(self.currency_var.get(), "₽")

    def _set_calc_car_image(self, photo_path):
        t = T()
        if photo_path and os.path.exists(photo_path):
            img = load_car_image(photo_path, 300, 180)
        else:
            img = make_placeholder(300, 180)
        photo = ImageTk.PhotoImage(img)
        self._photo_refs["calc_car"] = photo
        self.calc_car_label.configure(image=photo, bg=t["bg2"])

    def _refresh_car_combo(self):
        cars = ["— Без авто —"]
        if self.current_user and self.current_user in self.data.get("users", {}):
            user = self.data["users"][self.current_user]
            cars += [c["name"] for c in user.get("cars", [])]
        self.calc_car_combo["values"] = cars
        self.calc_car_var.set(cars[0])

    def _on_car_selected(self, event=None):
        sel = self.calc_car_var.get()
        if sel == "— Без авто —" or not self.current_user:
            self._set_calc_car_image(None)
            return
        user = self.data["users"].get(self.current_user, {})
        for car in user.get("cars", []):
            if car["name"] == sel:
                self._set_calc_car_image(car.get("photo"))
                return
        self._set_calc_car_image(None)

    def _calculate(self):
        try:
            dist  = float(self.dist_var.get().replace(",", "."))
            fuel  = float(self.fuel_var.get().replace(",", "."))
            price = float(self.price_var.get().replace(",", "."))
        except ValueError:
            messagebox.showerror("Ошибка", "Введите корректные числа!")
            return
        if dist <= 0:
            messagebox.showerror("Ошибка", "Расстояние должно быть больше 0!")
            return

        sym = CURRENCIES.get(self.currency_var.get(), "₽")
        consumption = (fuel / dist) * 100
        cost = fuel * price

        self.result_consumption_label.configure(text=f"{consumption:.2f} л/100 км")
        self.result_cost_label.configure(text=f"{cost:,.2f} {sym}")

        self.stat_labels["avg"].configure(text=f"{consumption:.2f} л/100 км")
        self.stat_labels["dist"].configure(text=f"{dist:.0f} км")
        self.stat_labels["fuel"].configure(text=f"{fuel:.1f} л")
        self.stat_labels["cost"].configure(text=f"{cost:,.2f} {sym}")

        if self.current_user:
            entry = {
                "date":        datetime.now().strftime("%d.%m.%Y %H:%M"),
                "car":         self.calc_car_var.get(),
                "distance":    dist,
                "fuel":        fuel,
                "price":       price,
                "currency":    sym,
                "consumption": round(consumption, 2),
                "cost":        round(cost, 2),
            }
            user = self.data["users"][self.current_user]
            user.setdefault("history", []).insert(0, entry)
            save_data(self.data)

    def _clear_calc(self):
        self.dist_var.set("")
        self.fuel_var.set("")
        self.price_var.set("")
        self.result_consumption_label.configure(text="— л/100 км")
        self.result_cost_label.configure(text="—")
        defaults = {"avg": "— л/100 км", "dist": "— км", "fuel": "— л", "cost": "—"}
        for k, v in defaults.items():
            self.stat_labels[k].configure(text=v)

    # ── Profile ─────────────────────────────────────────────────────────────────
    def _build_profile(self):
        t = T()
        f = self.sections["profile"]
        for w in f.winfo_children(): w.destroy()

        canvas = tk.Canvas(f, bg=t["bg"], highlightthickness=0)
        scroll = ttk.Scrollbar(f, orient="vertical", command=canvas.yview)
        canvas.configure(yscrollcommand=scroll.set)
        scroll.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)

        inner = tk.Frame(canvas, bg=t["bg"])
        win = canvas.create_window((0, 0), window=inner, anchor="nw")
        inner.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.bind("<Configure>", lambda e: canvas.itemconfig(win, width=e.width))

        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=32, pady=24)

        if not self.current_user:
            self._build_login_form(pad)
        else:
            self._build_logged_in_profile(pad)

    def _build_login_form(self, parent):
        t = T()
        self._auth_mode = tk.StringVar(value="login")

        tab_frame = tk.Frame(parent, bg=t["bg"])
        tab_frame.pack(pady=(0, 16))

        def switch(mode):
            self._auth_mode.set(mode)
            build()

        tk.Button(tab_frame, text="Войти", font=("Georgia", 12, "bold"),
                  bg=t["btn"], fg="white", bd=0, padx=20, pady=6,
                  cursor="hand2", command=lambda: switch("login")).pack(side="left", padx=4)
        tk.Button(tab_frame, text="Регистрация", font=("Georgia", 12),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=20, pady=6,
                  cursor="hand2", command=lambda: switch("register")).pack(side="left", padx=4)

        self._auth_form_frame = tk.Frame(parent, bg=t["bg2"], padx=32, pady=24)
        self._auth_form_frame.pack(fill="x", ipadx=8, ipady=8)

        def build():
            for w in self._auth_form_frame.winfo_children(): w.destroy()
            mode = self._auth_mode.get()
            tk.Label(self._auth_form_frame,
                     text="Создать аккаунт" if mode == "register" else "Вход в аккаунт",
                     font=("Georgia", 15, "bold"), bg=t["bg2"], fg=t["fg"]).pack(pady=(0, 16))

            # Email (логин — только почта)
            tk.Label(self._auth_form_frame, text="Email",
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
            email_entry = tk.Entry(self._auth_form_frame, font=("Courier", 12),
                                   bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                                   bd=1, relief="solid", width=30)
            email_entry.pack(fill="x", pady=4, ipady=6)

            tk.Label(self._auth_form_frame, text="Пароль (мин. 6 символов)",
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
            pw_entry = tk.Entry(self._auth_form_frame, font=("Courier", 12), show="•",
                                bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                                bd=1, relief="solid", width=30)
            pw_entry.pack(fill="x", pady=4, ipady=6)

            pw2_entry = None
            if mode == "register":
                tk.Label(self._auth_form_frame, text="Повторите пароль",
                         font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
                pw2_entry = tk.Entry(self._auth_form_frame, font=("Courier", 12), show="•",
                                     bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                                     bd=1, relief="solid", width=30)
                pw2_entry.pack(fill="x", pady=4, ipady=6)

            status = tk.Label(self._auth_form_frame, text="", font=("Courier", 10),
                              bg=t["bg2"], fg=t["red"])
            status.pack(pady=4)

            def submit():
                email = email_entry.get().strip().lower()
                pw    = pw_entry.get()

                if not email or not pw:
                    status.configure(text="Заполните все поля")
                    return
                if not is_valid_email(email):
                    status.configure(text="Введите корректный email")
                    return
                if len(pw) < 6:
                    status.configure(text="Пароль должен содержать минимум 6 символов")
                    return

                if mode == "register":
                    pw2 = pw2_entry.get() if pw2_entry else ""
                    if pw != pw2:
                        status.configure(text="Пароли не совпадают")
                        return
                    if email in self.data.get("users", {}):
                        status.configure(text="Пользователь уже существует")
                        return
                    self.data.setdefault("users", {})[email] = {
                        "password": hash_pw(pw), "cars": [], "history": []
                    }
                    save_data(self.data)
                    self.current_user = email
                    self.data["current_user"] = email
                    save_data(self.data)
                else:
                    users = self.data.get("users", {})
                    if email not in users or users[email]["password"] != hash_pw(pw):
                        status.configure(text="Неверный email или пароль")
                        return
                    self.current_user = email
                    self.data["current_user"] = email
                    save_data(self.data)

                self._refresh_car_combo()
                self._refresh_profile()

            tk.Button(self._auth_form_frame,
                      text="Войти" if mode == "login" else "Зарегистрироваться",
                      font=("Georgia", 12, "bold"), bg=t["btn"], fg="white", bd=0,
                      pady=8, cursor="hand2", command=submit).pack(fill="x", pady=8)

        build()

    def _build_logged_in_profile(self, parent):
        t = T()

        # Шапка
        hrow = tk.Frame(parent, bg=t["bg"])
        hrow.pack(fill="x", pady=(0, 16))
        tk.Label(hrow, text=f"👤  {self.current_user}", font=("Georgia", 14, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(side="left")

        btn_frame = tk.Frame(hrow, bg=t["bg"])
        btn_frame.pack(side="right")
        tk.Button(btn_frame, text="Выйти", font=("Courier", 10),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=12, pady=4,
                  cursor="hand2", command=self._logout).pack(side="left", padx=(0, 8))
        tk.Button(btn_frame, text="🗑 Удалить аккаунт", font=("Courier", 10),
                  bg=t["red"], fg="white", bd=0, padx=12, pady=4,
                  cursor="hand2", command=self._delete_account).pack(side="left")

        # Смена пароля (дважды)
        pw_frame = tk.Frame(parent, bg=t["bg2"])
        pw_frame.pack(fill="x", pady=8, ipady=4)
        tk.Label(pw_frame, text="🔒  Сменить пароль", font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=16, pady=(12, 4))

        tk.Label(pw_frame, text="Новый пароль (мин. 6 символов)",
                 font=("Courier", 9), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=16)
        new_pw = tk.Entry(pw_frame, font=("Courier", 11), show="•",
                          bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                          bd=1, relief="solid")
        new_pw.pack(fill="x", padx=16, ipady=5, pady=(2, 4))

        tk.Label(pw_frame, text="Повторите новый пароль",
                 font=("Courier", 9), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=16)
        new_pw2 = tk.Entry(pw_frame, font=("Courier", 11), show="•",
                           bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                           bd=1, relief="solid")
        new_pw2.pack(fill="x", padx=16, ipady=5, pady=(2, 4))

        pw_status = tk.Label(pw_frame, text="", font=("Courier", 9), bg=t["bg2"], fg=t["green"])
        pw_status.pack(anchor="w", padx=16)

        def change_pw():
            np1 = new_pw.get()
            np2 = new_pw2.get()
            if len(np1) < 6:
                pw_status.configure(text="Пароль должен содержать минимум 6 символов", fg=T()["red"])
                return
            if np1 != np2:
                pw_status.configure(text="Пароли не совпадают", fg=T()["red"])
                return
            self.data["users"][self.current_user]["password"] = hash_pw(np1)
            save_data(self.data)
            pw_status.configure(text="✓ Пароль изменён", fg=T()["green"])
            new_pw.delete(0, "end")
            new_pw2.delete(0, "end")

        tk.Button(pw_frame, text="Сохранить пароль", font=("Courier", 10),
                  bg=t["btn"], fg="white", bd=0, padx=10, pady=5,
                  cursor="hand2", command=change_pw).pack(anchor="w", padx=16, pady=(0, 12))

        # Автомобили
        tk.Frame(parent, bg=t["border"], height=1).pack(fill="x", pady=16)
        cars_header = tk.Frame(parent, bg=t["bg"])
        cars_header.pack(fill="x")
        tk.Label(cars_header, text="🚗  Мои автомобили", font=("Georgia", 14, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(side="left")
        tk.Button(cars_header, text="+ Добавить авто", font=("Courier", 10),
                  bg=t["btn"], fg="white", bd=0, padx=12, pady=4,
                  cursor="hand2", command=lambda: self._add_car_dialog(parent)).pack(side="right")

        self.cars_container = tk.Frame(parent, bg=t["bg"])
        self.cars_container.pack(fill="x", pady=8)
        self._render_cars()

    def _render_cars(self):
        t = T()
        for w in self.cars_container.winfo_children(): w.destroy()
        if not self.current_user:
            return
        cars = self.data["users"][self.current_user].get("cars", [])
        if not cars:
            tk.Label(self.cars_container, text="Автомобили не добавлены",
                     font=("Courier", 11), bg=t["bg"], fg=t["fg2"]).pack(pady=16)
            return
        row = tk.Frame(self.cars_container, bg=t["bg"])
        row.pack(fill="x")
        for i, car in enumerate(cars):
            card = tk.Frame(row, bg=t["bg2"], bd=0, relief="flat", padx=8, pady=8)
            card.pack(side="left", padx=8, pady=4)
            ph = car.get("photo")
            img = load_car_image(ph, 140, 90) if ph and os.path.exists(ph) else make_placeholder(140, 90, "Нет фото")
            photo = ImageTk.PhotoImage(img)
            self._photo_refs[f"car_{i}"] = photo
            tk.Label(card, image=photo, bg=t["bg2"]).pack()
            tk.Label(card, text=car["name"], font=("Courier", 10, "bold"),
                     bg=t["bg2"], fg=t["fg"]).pack(pady=2)
            tk.Button(card, text="🗑 Удалить", font=("Courier", 9),
                      bg=t["red"], fg="white", bd=0, padx=8, pady=3,
                      cursor="hand2", command=lambda idx=i: self._delete_car(idx)).pack()

    def _add_car_dialog(self, parent):
        t = T()
        dlg = tk.Toplevel(self)
        dlg.title("Добавить автомобиль")
        dlg.configure(bg=t["bg2"])
        dlg.geometry("360x300")
        dlg.resizable(False, False)

        tk.Label(dlg, text="Название автомобиля", font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=20, pady=(16,2))
        name_var = tk.StringVar()
        tk.Entry(dlg, textvariable=name_var, font=("Courier", 12),
                 bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                 bd=1, relief="solid").pack(fill="x", padx=20, ipady=6)

        photo_path_var = tk.StringVar(value="")
        photo_label = tk.Label(dlg, text="Фото не выбрано", font=("Courier", 9),
                               bg=t["bg2"], fg=t["fg2"])
        photo_label.pack(pady=4)

        def choose_photo():
            path = filedialog.askopenfilename(
                filetypes=[("Изображения", "*.jpg *.jpeg *.png *.bmp *.webp")])
            if path:
                ext = os.path.splitext(path)[1]
                dest = os.path.join(PHOTO_DIR, f"car_{datetime.now().strftime('%Y%m%d%H%M%S')}{ext}")
                shutil.copy2(path, dest)
                photo_path_var.set(dest)
                photo_label.configure(text=os.path.basename(path))

        tk.Button(dlg, text="📷  Выбрать фото", font=("Courier", 10),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=12, pady=6,
                  cursor="hand2", command=choose_photo).pack(pady=8)

        status = tk.Label(dlg, text="", font=("Courier", 9), bg=t["bg2"], fg=t["red"])
        status.pack()

        def save_car():
            name = name_var.get().strip()
            if not name:
                status.configure(text="Введите название")
                return
            car = {"name": name, "photo": photo_path_var.get() or None}
            self.data["users"][self.current_user].setdefault("cars", []).append(car)
            save_data(self.data)
            self._render_cars()
            self._refresh_car_combo()
            dlg.destroy()

        tk.Button(dlg, text="💾  Сохранить", font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  cursor="hand2", command=save_car).pack(fill="x", padx=20, pady=12)

    def _delete_car(self, idx):
        if messagebox.askyesno("Удалить?", "Удалить этот автомобиль?"):
            del self.data["users"][self.current_user]["cars"][idx]
            save_data(self.data)
            self._render_cars()
            self._refresh_car_combo()

    def _logout(self):
        self.current_user = None
        self.data["current_user"] = None
        save_data(self.data)
        self._refresh_car_combo()
        self._refresh_profile()

    def _delete_account(self):
        if messagebox.askyesno("Удалить аккаунт?",
                               "Вы уверены? Все данные (история, автомобили) будут удалены безвозвратно."):
            # Удаляем фото
            for car in self.data["users"].get(self.current_user, {}).get("cars", []):
                if car.get("photo") and os.path.exists(car["photo"]):
                    try:
                        os.remove(car["photo"])
                    except Exception:
                        pass
            del self.data["users"][self.current_user]
            self.current_user = None
            self.data["current_user"] = None
            save_data(self.data)
            self._refresh_car_combo()
            self._refresh_profile()

    def _refresh_profile(self):
        f = self.sections["profile"]
        for w in f.winfo_children(): w.destroy()
        self._build_profile()

    # ── History ─────────────────────────────────────────────────────────────────
    def _build_history(self):
        t = T()
        f = self.sections["history"]
        for w in f.winfo_children(): w.destroy()

        tk.Label(f, text="📋  История расчётов", font=("Georgia", 16, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", padx=24, pady=(20, 4))

        if not self.current_user:
            tk.Label(f, text="Войдите в аккаунт для просмотра истории.",
                     font=("Courier", 12), bg=t["bg"], fg=t["fg2"]).pack(pady=40)
            return

        hdr = tk.Frame(f, bg=t["bg"])
        hdr.pack(fill="x", padx=24)
        tk.Button(hdr, text="🗑  Очистить всё", font=("Courier", 9),
                  bg=t["red"], fg="white", bd=0, padx=8, pady=4,
                  cursor="hand2", command=self._clear_all_history).pack(side="right")

        self.history_frame = tk.Frame(f, bg=t["bg"])
        self.history_frame.pack(fill="both", expand=True, padx=24, pady=8)

        canvas = tk.Canvas(self.history_frame, bg=t["bg"], highlightthickness=0)
        scroll = ttk.Scrollbar(self.history_frame, orient="vertical", command=canvas.yview)
        canvas.configure(yscrollcommand=scroll.set)
        scroll.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)

        self._hist_inner = tk.Frame(canvas, bg=t["bg"])
        win = canvas.create_window((0, 0), window=self._hist_inner, anchor="nw")
        self._hist_inner.bind("<Configure>",
                               lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.bind("<Configure>", lambda e: canvas.itemconfig(win, width=e.width))

        self._render_history_entries()

    def _render_history_entries(self):
        t = T()
        for w in self._hist_inner.winfo_children(): w.destroy()
        if not self.current_user:
            return
        history = self.data["users"][self.current_user].get("history", [])
        if not history:
            tk.Label(self._hist_inner, text="История пуста", font=("Courier", 12),
                     bg=t["bg"], fg=t["fg2"]).pack(pady=40)
            return

        for i, entry in enumerate(history):
            card = tk.Frame(self._hist_inner, bg=t["bg2"], pady=8)
            card.pack(fill="x", pady=6, ipadx=8)

            # Левая часть: фото машины (большая)
            ph = None
            car_name = entry.get("car", "—")
            if self.current_user and car_name != "— Без авто —":
                for car in self.data["users"][self.current_user].get("cars", []):
                    if car["name"] == car_name and car.get("photo"):
                        ph = car["photo"]
                        break

            img = load_car_image(ph, 200, 120) if ph and os.path.exists(ph) else make_placeholder(200, 120, "Нет фото")
            photo = ImageTk.PhotoImage(img)
            self._photo_refs[f"hist_car_{i}"] = photo
            tk.Label(card, image=photo, bg=t["bg2"]).pack(side="left", padx=12)

            # Правая часть: инфо
            info = tk.Frame(card, bg=t["bg2"])
            info.pack(side="left", fill="x", expand=True, padx=8)

            sym = entry.get("currency", "₽")
            tk.Label(info, text=f"📅 {entry['date']}",
                     font=("Courier", 10, "bold"), bg=t["bg2"], fg=t["fg"]).pack(anchor="w")
            tk.Label(info, text=f"🚗 {car_name}",
                     font=("Courier", 10), bg=t["bg2"], fg=t["accent"]).pack(anchor="w")
            tk.Label(info,
                     text=f"📏 {entry['distance']} км   ⛽ {entry['fuel']} л   💰 {entry['price']} {sym}/л",
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=2)
            tk.Label(info,
                     text=f"💧 {entry['consumption']} л/100км   💳 {entry['cost']:,.2f} {sym}",
                     font=("Courier", 11, "bold"), bg=t["bg2"], fg=t["green"]).pack(anchor="w")

            # Кнопки
            btn_col = tk.Frame(card, bg=t["bg2"])
            btn_col.pack(side="right", padx=8, pady=4)

            copy_text = (
                f"Дата: {entry['date']}\n"
                f"Авто: {car_name}\n"
                f"Расстояние: {entry['distance']} км\n"
                f"Топливо: {entry['fuel']} л\n"
                f"Цена: {entry['price']} {sym}/л\n"
                f"Расход: {entry['consumption']} л/100 км\n"
                f"Стоимость: {entry['cost']:,.2f} {sym}"
            )

            def copy_entry(text=copy_text):
                self.clipboard_clear()
                self.clipboard_append(text)
                messagebox.showinfo("Скопировано", "Расчёт скопирован в буфер обмена!")

            tk.Button(btn_col, text="📋 Копировать", font=("Courier", 9),
                      bg=t["btn2"], fg=t["fg"], bd=0, padx=8, pady=4,
                      cursor="hand2", command=copy_entry).pack(pady=(0, 6))
            tk.Button(btn_col, text="✕ Удалить", font=("Courier", 9),
                      bg=t["red"], fg="white", bd=0, padx=8, pady=4,
                      cursor="hand2", command=lambda idx=i: self._delete_history_entry(idx)).pack()

    def _delete_history_entry(self, idx):
        del self.data["users"][self.current_user]["history"][idx]
        save_data(self.data)
        self._render_history_entries()

    def _clear_all_history(self):
        if messagebox.askyesno("Очистить?", "Удалить всю историю?"):
            self.data["users"][self.current_user]["history"] = []
            save_data(self.data)
            self._render_history_entries()

    def _refresh_history(self):
        f = self.sections["history"]
        for w in f.winfo_children(): w.destroy()
        self._build_history()

    # ── About ───────────────────────────────────────────────────────────────────
    def _build_about(self):
        t = T()
        f = self.sections["about"]
        for w in f.winfo_children(): w.destroy()

        canvas = tk.Canvas(f, bg=t["bg"], highlightthickness=0)
        scroll = ttk.Scrollbar(f, orient="vertical", command=canvas.yview)
        canvas.configure(yscrollcommand=scroll.set)
        scroll.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)

        inner = tk.Frame(canvas, bg=t["bg"])
        win = canvas.create_window((0, 0), window=inner, anchor="nw")
        inner.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.bind("<Configure>", lambda e: canvas.itemconfig(win, width=e.width))

        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=48, pady=32)

        hero = tk.Frame(pad, bg=t["bg2"], pady=32)
        hero.pack(fill="x", pady=(0, 24))
        tk.Label(hero, text="⛽", font=("Courier", 48), bg=t["bg2"], fg=t["accent"]).pack()
        tk.Label(hero, text="Калькулятор расхода топлива",
                 font=("Georgia", 22, "bold"), bg=t["bg2"], fg=t["fg"]).pack()
        tk.Label(hero, text="Версия 2.0  •  2024",
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(pady=4)

        sections_info = [
            ("🎯  О приложении",
             "Это профессиональное приложение для расчёта расхода топлива вашего автомобиля. "
             "Создано для водителей, которые хотят следить за эффективностью транспортного средства "
             "и планировать затраты на топливо."),
            ("✨  Возможности",
             "• Точный расчёт расхода топлива в л/100 км\n"
             "• Расчёт стоимости поездки с выбором валюты (₽, $, €, £, ₴, ₸)\n"
             "• Хранение нескольких автомобилей с фотографиями\n"
             "• Полная история всех расчётов с возможностью копирования\n"
             "• Личный профиль с регистрацией по email\n"
             "• Светлая и тёмная темы оформления"),
            ("🚗  Как использовать",
             "1. Введите пройденное расстояние в километрах\n"
             "2. Укажите количество использованного топлива в литрах\n"
             "3. Введите цену топлива за литр и выберите валюту\n"
             "4. Нажмите «Рассчитать» для получения результата\n\n"
             "Для сохранения истории расчётов — войдите в аккаунт."),
            ("🔒  Безопасность",
             "Регистрация только по email. Пароли зашифрованы (SHA-256). "
             "Все данные хранятся локально на устройстве. Никакие данные не передаются на серверы."),
            ("📊  Интерпретация результатов",
             "🟢  До 8 л/100 км  — экономичный расход\n"
             "🟡  8–12 л/100 км  — средний расход\n"
             "🔴  Более 12 л/100 км — высокий расход"),
        ]

        for title, body in sections_info:
            sec = tk.Frame(pad, bg=t["bg2"])
            sec.pack(fill="x", pady=8, ipadx=16, ipady=12)
            tk.Label(sec, text=title, font=("Georgia", 13, "bold"),
                     bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=16, pady=(8, 2))
            tk.Label(sec, text=body, font=("Courier", 10), bg=t["bg2"],
                     fg=t["fg2"], justify="left", wraplength=700,
                     anchor="w").pack(anchor="w", padx=16, pady=(2, 8))

        tk.Label(pad, text="© 2024 Калькулятор расхода топлива. Все права защищены.",
                 font=("Courier", 9), bg=t["bg"], fg=t["fg2"]).pack(pady=16)

    # ── Settings ────────────────────────────────────────────────────────────────
    def _build_settings(self):
        t = T()
        f = self.sections["settings"]
        for w in f.winfo_children(): w.destroy()

        pad = tk.Frame(f, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=48, pady=32)

        tk.Label(pad, text="⚙  Настройки", font=("Georgia", 18, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0, 24))

        theme_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        theme_card.pack(fill="x", pady=8)
        tk.Label(theme_card, text="🎨  Тема оформления", font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(0, 4))
        tk.Label(theme_card, text="Выберите цветовую схему приложения",
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=20)

        btn_row = tk.Frame(theme_card, bg=t["bg2"])
        btn_row.pack(padx=20, pady=12, anchor="w")

        def apply_theme(name):
            global current_theme
            current_theme = name
            self._rebuild_ui()
            self._show_section("settings")

        tk.Button(btn_row, text="🌙  Тёмная",
                  font=("Georgia", 11, "bold"), bg="#1f2937", fg="#e5e7eb",
                  bd=0, padx=20, pady=8, cursor="hand2",
                  command=lambda: apply_theme("dark")).pack(side="left", padx=(0, 12))
        tk.Button(btn_row, text="☀️  Светлая",
                  font=("Georgia", 11, "bold"), bg="#e2e8f0", fg="#1a202c",
                  bd=0, padx=20, pady=8, cursor="hand2",
                  command=lambda: apply_theme("light")).pack(side="left")

        tk.Label(theme_card,
                 text=f"Текущая тема: {'Тёмная 🌙' if current_theme == 'dark' else 'Светлая ☀️'}",
                 font=("Courier", 10), bg=t["bg2"], fg=t["accent"]).pack(anchor="w", padx=20, pady=4)


# ─── Entry point ───────────────────────────────────────────────────────────────
if __name__ == "__main__":
    app = FuelApp()
    app.mainloop()
