#!/usr/bin/env python3
"""
🚗 Калькулятор расхода топлива транспортных средств
Красивый интерфейс на Tkinter с анимацией и современным дизайном
"""

import tkinter as tk
from tkinter import ttk, messagebox
import math
import json
import os
from datetime import datetime


# ══════════════════════════════════════════════════════════════════
#  ЦВЕТОВАЯ ПАЛИТРА — «БОРТОВОЙ КОМПЬЮТЕР»
# ══════════════════════════════════════════════════════════════════
COLORS = {
    "bg_dark":      "#0A0E1A",
    "bg_panel":     "#111827",
    "bg_card":      "#1A2035",
    "bg_input":     "#0F1629",
    "accent":       "#00D4FF",
    "accent2":      "#FF6B35",
    "accent3":      "#4CAF50",
    "accent_warn":  "#FFB800",
    "text_primary": "#E8F4FD",
    "text_secondary":"#8BA3BD",
    "text_dim":     "#4A6080",
    "border":       "#1E3A5F",
    "border_bright":"#00D4FF",
    "red":          "#FF4444",
    "green":        "#00FF88",
    "gauge_bg":     "#0D1520",
}

FONTS = {
    "mono_xl":   ("Courier New", 28, "bold"),
    "mono_lg":   ("Courier New", 18, "bold"),
    "mono_md":   ("Courier New", 13, "bold"),
    "mono_sm":   ("Courier New", 10),
    "sans_xl":   ("Arial", 22, "bold"),
    "sans_lg":   ("Arial", 15, "bold"),
    "sans_md":   ("Arial", 12),
    "sans_sm":   ("Arial", 10),
    "sans_xs":   ("Arial", 9),
}

VEHICLES = {
    "🚗 Легковой автомобиль":    {"norm": 8.5,  "tank": 55,  "icon": "🚗"},
    "🚙 Внедорожник (SUV)":      {"norm": 12.0, "tank": 75,  "icon": "🚙"},
    "🚐 Минивэн":                {"norm": 11.0, "tank": 70,  "icon": "🚐"},
    "🚌 Автобус":                {"norm": 25.0, "tank": 200, "icon": "🚌"},
    "🚛 Грузовик":               {"norm": 30.0, "tank": 300, "icon": "🚛"},
    "🏍️ Мотоцикл":              {"norm": 5.0,  "tank": 18,  "icon": "🏍️"},
    "🚜 Трактор":                {"norm": 15.0, "tank": 120, "icon": "🚜"},
    "⛵ Лодка":                  {"norm": 20.0, "tank": 100, "icon": "⛵"},
}

FUEL_TYPES = {
    "АИ-92":    60,
    "АИ-95":    65,
    "АИ-98":    75,
    "Дизель":   58,
    "Газ (LPG)":40,
    "Электро":  0,
}

HISTORY_FILE = os.path.join(os.path.expanduser("~"), ".fuel_calc_history.json")


# ══════════════════════════════════════════════════════════════════
#  ВСПОМОГАТЕЛЬНЫЕ КОМПОНЕНТЫ
# ══════════════════════════════════════════════════════════════════

class AnimatedValue:
    """Плавная анимация числового значения"""
    def __init__(self, widget, var, duration=600):
        self.widget = widget
        self.var = var
        self.duration = duration
        self._job = None

    def animate(self, start, end, steps=30, fmt="{:.1f}"):
        if self._job:
            try: self.widget.after_cancel(self._job)
            except: pass
        delta = (end - start) / steps
        self._step(start, end, delta, steps, fmt)

    def _step(self, current, end, delta, remaining, fmt):
        if remaining <= 0:
            self.var.set(fmt.format(end))
            return
        current += delta
        self.var.set(fmt.format(current))
        delay = self.duration // 30
        self._job = self.widget.after(delay, self._step,
                                      current, end, delta, remaining-1, fmt)


class GaugeWidget(tk.Canvas):
    """Аналоговый спидометр-индикатор"""
    def __init__(self, parent, size=180, **kwargs):
        super().__init__(parent, width=size, height=size+10,
                         bg=COLORS["bg_card"], highlightthickness=0, **kwargs)
        self.size = size
        self.cx = size // 2
        self.cy = size // 2 + 5
        self.r = size // 2 - 12
        self._value = 0
        self._anim_job = None
        self._draw_base()

    def _draw_base(self):
        s, cx, cy, r = self.size, self.cx, self.cy, self.r
        # Внешнее кольцо
        self.create_oval(cx-r-4, cy-r-4, cx+r+4, cy+r+4,
                         outline=COLORS["border"], width=2, fill=COLORS["gauge_bg"])
        self.create_oval(cx-r, cy-r, cx+r, cy+r,
                         outline=COLORS["border_bright"], width=1,
                         fill=COLORS["gauge_bg"])
        # Засечки
        for i in range(11):
            angle = math.radians(-220 + i * 26)
            r_outer = r - 2
            r_inner = r - (12 if i % 2 == 0 else 7)
            x1 = cx + r_outer * math.cos(angle)
            y1 = cy + r_outer * math.sin(angle)
            x2 = cx + r_inner * math.cos(angle)
            y2 = cy + r_inner * math.sin(angle)
            color = COLORS["accent"] if i % 2 == 0 else COLORS["border"]
            self.create_line(x1, y1, x2, y2, fill=color, width=2 if i % 2 == 0 else 1)
        # Центральная точка
        self.create_oval(cx-5, cy-5, cx+5, cy+5,
                         fill=COLORS["accent"], outline=COLORS["bg_dark"])
        # Дуга заливки (создаём тэги для обновления)
        self.arc_id = self.create_arc(cx-r+8, cy-r+8, cx+r-8, cy+r-8,
                                      start=220, extent=0,
                                      outline="", fill=COLORS["accent"],
                                      style=tk.PIESLICE)
        # Скрываем центр дуги
        self.create_oval(cx-r+22, cy-r+22, cx+r-22, cy+r-22,
                         fill=COLORS["gauge_bg"], outline="")
        # Стрелка (поверх всего)
        self.needle_id = self.create_line(cx, cy,
                                           cx + (r-20)*math.cos(math.radians(-220)),
                                           cy + (r-20)*math.sin(math.radians(-220)),
                                           fill=COLORS["accent2"], width=3,
                                           capstyle=tk.ROUND)
        self.create_oval(cx-4, cy-4, cx+4, cy+4,
                         fill=COLORS["accent2"], outline="")

    def set_value(self, value, max_val=30, animate=True):
        if animate:
            self._animate_to(self._value, value, max_val)
        else:
            self._draw_needle(value, max_val)
        self._value = value

    def _animate_to(self, start, end, max_val, steps=25):
        if self._anim_job:
            try: self.after_cancel(self._anim_job)
            except: pass
        delta = (end - start) / steps
        self.__anim_step(start, end, delta, steps, max_val)

    def __anim_step(self, current, end, delta, remaining, max_val):
        if remaining <= 0:
            self._draw_needle(end, max_val)
            return
        current += delta
        self._draw_needle(current, max_val)
        self._anim_job = self.after(20, self.__anim_step,
                                     current, end, delta, remaining-1, max_val)

    def _draw_needle(self, value, max_val):
        frac = max(0, min(1, value / max_val)) if max_val else 0
        angle = math.radians(-220 + frac * 260)
        cx, cy, r = self.cx, self.cy, self.r
        nx = cx + (r-20) * math.cos(angle)
        ny = cy + (r-20) * math.sin(angle)
        self.coords(self.needle_id, cx, cy, nx, ny)
        # Обновляем дугу
        extent = frac * 260
        color = (COLORS["accent3"] if frac < 0.5
                 else COLORS["accent_warn"] if frac < 0.8
                 else COLORS["red"])
        self.itemconfig(self.arc_id,
                        start=220-extent, extent=extent,
                        fill=color)
        self.itemconfig(self.needle_id, fill=color)


# ══════════════════════════════════════════════════════════════════
#  ГЛАВНОЕ ПРИЛОЖЕНИЕ
# ══════════════════════════════════════════════════════════════════

class FuelCalculatorApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("⛽  FUEL CALCULATOR — Расход топлива")
        self.geometry("1000x720")
        self.minsize(920, 660)
        self.configure(bg=COLORS["bg_dark"])
        self.resizable(True, True)

        # Данные
        self.history = self._load_history()

        # Переменные
        self.var_vehicle  = tk.StringVar(value=list(VEHICLES.keys())[0])
        self.var_fuel     = tk.StringVar(value=list(FUEL_TYPES.keys())[0])
        self.var_distance = tk.StringVar(value="100")
        self.var_rate     = tk.StringVar(value="8.5")
        self.var_price    = tk.StringVar(value="60")
        self.var_load     = tk.DoubleVar(value=0)      # доп. нагрузка %

        # Результаты
        self.res_liters   = tk.StringVar(value="—")
        self.res_cost     = tk.StringVar(value="—")
        self.res_per100   = tk.StringVar(value="—")
        self.res_range    = tk.StringVar(value="—")

        self._build_ui()
        self._on_vehicle_change()
        self._pulse_accent()   # живая анимация рамки

    # ── UI ────────────────────────────────────────────────────────
    def _build_ui(self):
        # Шапка
        self._build_header()
        # Основной контент
        main = tk.Frame(self, bg=COLORS["bg_dark"])
        main.pack(fill="both", expand=True, padx=16, pady=(0, 10))
        main.columnconfigure(0, weight=1)
        main.columnconfigure(1, weight=1)
        main.rowconfigure(0, weight=1)

        left = tk.Frame(main, bg=COLORS["bg_dark"])
        left.grid(row=0, column=0, sticky="nsew", padx=(0, 8))
        right = tk.Frame(main, bg=COLORS["bg_dark"])
        right.grid(row=0, column=1, sticky="nsew", padx=(8, 0))

        self._build_inputs(left)
        self._build_results(right)
        self._build_history_panel(right)

    def _build_header(self):
        hdr = tk.Frame(self, bg=COLORS["bg_panel"], height=70)
        hdr.pack(fill="x")
        hdr.pack_propagate(False)

        # Логотип
        logo_frame = tk.Frame(hdr, bg=COLORS["bg_panel"])
        logo_frame.pack(side="left", padx=20, pady=10)
        tk.Label(logo_frame, text="⛽", font=("Arial", 28),
                 bg=COLORS["bg_panel"], fg=COLORS["accent"]).pack(side="left")
        title_f = tk.Frame(logo_frame, bg=COLORS["bg_panel"])
        title_f.pack(side="left", padx=10)
        tk.Label(title_f, text="FUEL CALCULATOR",
                 font=FONTS["mono_lg"], bg=COLORS["bg_panel"],
                 fg=COLORS["accent"]).pack(anchor="w")
        tk.Label(title_f, text="Калькулятор расхода топлива",
                 font=FONTS["sans_sm"], bg=COLORS["bg_panel"],
                 fg=COLORS["text_secondary"]).pack(anchor="w")

        # Часы
        self.lbl_time = tk.Label(hdr, font=FONTS["mono_md"],
                                  bg=COLORS["bg_panel"], fg=COLORS["text_secondary"])
        self.lbl_time.pack(side="right", padx=20)
        self._tick()

        # Разделитель
        sep = tk.Frame(self, bg=COLORS["border_bright"], height=2)
        sep.pack(fill="x")

    def _build_inputs(self, parent):
        parent.columnconfigure(0, weight=1)

        # ── Выбор ТС ──
        card = self._card(parent, "🚗  ТРАНСПОРТНОЕ СРЕДСТВО")
        card.pack(fill="x", pady=(0, 8))

        # Сетка кнопок ТС
        btn_frame = tk.Frame(card, bg=COLORS["bg_card"])
        btn_frame.pack(fill="x", padx=12, pady=(0, 12))
        self.vehicle_btns = {}
        vehicles = list(VEHICLES.keys())
        for i, name in enumerate(vehicles):
            icon = VEHICLES[name]["icon"]
            short = name.split(" ", 1)[1][:12]
            btn = tk.Button(btn_frame, text=f"{icon}\n{short}",
                            font=FONTS["sans_xs"], width=7,
                            bg=COLORS["bg_input"], fg=COLORS["text_secondary"],
                            activebackground=COLORS["accent"],
                            activeforeground=COLORS["bg_dark"],
                            relief="flat", bd=0, cursor="hand2",
                            command=lambda n=name: self._select_vehicle(n))
            btn.grid(row=i//4, column=i%4, padx=3, pady=3, sticky="ew")
            btn_frame.columnconfigure(i%4, weight=1)
            self.vehicle_btns[name] = btn

        # ── Параметры ──
        card2 = self._card(parent, "⚙️  ПАРАМЕТРЫ РАСЧЁТА")
        card2.pack(fill="x", pady=(0, 8))

        fields = [
            ("📏 Расстояние (км):",      self.var_distance, "км"),
            ("💧 Расход (л/100 км):",    self.var_rate,     "л/100"),
            ("💰 Цена топлива (₽/л):",   self.var_price,    "₽/л"),
        ]
        for label, var, hint in fields:
            self._input_row(card2, label, var, hint)

        # Нагрузка слайдером
        load_f = tk.Frame(card2, bg=COLORS["bg_card"])
        load_f.pack(fill="x", padx=12, pady=(0, 8))
        tk.Label(load_f, text="🏋️ Доп. нагрузка / пробки:",
                 font=FONTS["sans_sm"], bg=COLORS["bg_card"],
                 fg=COLORS["text_secondary"]).pack(anchor="w")
        slider_row = tk.Frame(load_f, bg=COLORS["bg_card"])
        slider_row.pack(fill="x", pady=4)
        self.slider = ttk.Scale(slider_row, from_=0, to=50,
                                 variable=self.var_load, orient="horizontal",
                                 command=self._on_slider)
        self.slider.pack(side="left", fill="x", expand=True)
        self.lbl_load = tk.Label(slider_row, text="0%",
                                  width=5, font=FONTS["mono_sm"],
                                  bg=COLORS["bg_card"], fg=COLORS["accent"])
        self.lbl_load.pack(side="left", padx=8)

        # Вид топлива
        fuel_f = tk.Frame(card2, bg=COLORS["bg_card"])
        fuel_f.pack(fill="x", padx=12, pady=(0, 12))
        tk.Label(fuel_f, text="⛽ Вид топлива:",
                 font=FONTS["sans_sm"], bg=COLORS["bg_card"],
                 fg=COLORS["text_secondary"]).pack(anchor="w")
        fuel_row = tk.Frame(fuel_f, bg=COLORS["bg_card"])
        fuel_row.pack(fill="x", pady=4)
        self.fuel_btns = {}
        for i, (name, price) in enumerate(FUEL_TYPES.items()):
            btn = tk.Button(fuel_row, text=name,
                            font=FONTS["sans_xs"],
                            bg=COLORS["bg_input"], fg=COLORS["text_secondary"],
                            relief="flat", bd=0, cursor="hand2",
                            command=lambda n=name, p=price: self._select_fuel(n, p))
            btn.pack(side="left", padx=2)
            self.fuel_btns[name] = btn
        self._highlight_fuel(list(FUEL_TYPES.keys())[0])

        # ── Кнопка расчёта ──
        self.calc_btn = tk.Button(parent,
                                   text="▶  РАССЧИТАТЬ",
                                   font=FONTS["mono_md"],
                                   bg=COLORS["accent"], fg=COLORS["bg_dark"],
                                   activebackground=COLORS["accent2"],
                                   activeforeground=COLORS["bg_dark"],
                                   relief="flat", bd=0, cursor="hand2",
                                   height=2, command=self._calculate)
        self.calc_btn.pack(fill="x", pady=(4, 0))
        self.calc_btn.bind("<Enter>", lambda e: self.calc_btn.config(bg=COLORS["accent2"]))
        self.calc_btn.bind("<Leave>", lambda e: self.calc_btn.config(bg=COLORS["accent"]))

    def _build_results(self, parent):
        # Карточка результатов
        card = self._card(parent, "📊  РЕЗУЛЬТАТЫ")
        card.pack(fill="x", pady=(0, 8))

        # Датчик (канвас)
        gauge_row = tk.Frame(card, bg=COLORS["bg_card"])
        gauge_row.pack(fill="x", padx=12, pady=8)

        gauge_col = tk.Frame(gauge_row, bg=COLORS["bg_card"])
        gauge_col.pack(side="left")
        tk.Label(gauge_col, text="РАСХОД  л/100км",
                 font=FONTS["sans_xs"], bg=COLORS["bg_card"],
                 fg=COLORS["text_dim"]).pack()
        self.gauge = GaugeWidget(gauge_col, size=150)
        self.gauge.pack()
        self.gauge_lbl = tk.Label(gauge_col, textvariable=self.res_per100,
                                   font=FONTS["mono_lg"],
                                   bg=COLORS["bg_card"], fg=COLORS["accent"])
        self.gauge_lbl.pack()

        # Цифры
        nums = tk.Frame(gauge_row, bg=COLORS["bg_card"])
        nums.pack(side="left", fill="both", expand=True, padx=16)

        self._result_block(nums, "💧 Топлива потребуется",
                            self.res_liters, "литров", COLORS["accent"])
        self._result_block(nums, "💰 Стоимость поездки",
                            self.res_cost, "рублей", COLORS["accent2"])
        self._result_block(nums, "🛣️ Запас хода (полный бак)",
                            self.res_range, "км", COLORS["accent3"])

    def _build_history_panel(self, parent):
        card = self._card(parent, "📋  ИСТОРИЯ РАСЧЁТОВ")
        card.pack(fill="both", expand=True)

        hdr_row = tk.Frame(card, bg=COLORS["bg_card"])
        hdr_row.pack(fill="x", padx=12, pady=(0, 6))
        tk.Button(hdr_row, text="🗑 Очистить",
                  font=FONTS["sans_xs"],
                  bg=COLORS["bg_input"], fg=COLORS["text_dim"],
                  relief="flat", bd=0, cursor="hand2",
                  command=self._clear_history).pack(side="right")

        # Список
        list_frame = tk.Frame(card, bg=COLORS["bg_card"])
        list_frame.pack(fill="both", expand=True, padx=12, pady=(0, 12))

        scrollbar = ttk.Scrollbar(list_frame)
        scrollbar.pack(side="right", fill="y")

        self.history_lb = tk.Listbox(list_frame,
                                      bg=COLORS["bg_input"],
                                      fg=COLORS["text_secondary"],
                                      font=FONTS["mono_sm"],
                                      selectbackground=COLORS["accent"],
                                      selectforeground=COLORS["bg_dark"],
                                      relief="flat", bd=0,
                                      yscrollcommand=scrollbar.set,
                                      height=6)
        self.history_lb.pack(fill="both", expand=True)
        scrollbar.config(command=self.history_lb.yview)
        self._refresh_history_list()

    # ── Вспомогательные строители ─────────────────────────────────
    def _card(self, parent, title):
        frame = tk.Frame(parent, bg=COLORS["bg_card"],
                          highlightbackground=COLORS["border"],
                          highlightthickness=1)
        tk.Label(frame, text=title,
                 font=FONTS["sans_xs"], bg=COLORS["bg_card"],
                 fg=COLORS["text_dim"]).pack(anchor="w", padx=12, pady=(8, 4))
        tk.Frame(frame, bg=COLORS["border"], height=1).pack(fill="x", padx=12)
        return frame

    def _input_row(self, parent, label, var, hint):
        row = tk.Frame(parent, bg=COLORS["bg_card"])
        row.pack(fill="x", padx=12, pady=3)
        tk.Label(row, text=label, font=FONTS["sans_sm"], width=22, anchor="w",
                 bg=COLORS["bg_card"], fg=COLORS["text_secondary"]).pack(side="left")
        entry = tk.Entry(row, textvariable=var, font=FONTS["mono_sm"],
                         bg=COLORS["bg_input"], fg=COLORS["accent"],
                         insertbackground=COLORS["accent"],
                         relief="flat", bd=4, width=10)
        entry.pack(side="left", padx=(0, 6))
        tk.Label(row, text=hint, font=FONTS["sans_xs"],
                 bg=COLORS["bg_card"], fg=COLORS["text_dim"]).pack(side="left")

    def _result_block(self, parent, title, var, unit, color):
        f = tk.Frame(parent, bg=COLORS["bg_card"])
        f.pack(fill="x", pady=6)
        tk.Label(f, text=title, font=FONTS["sans_xs"],
                 bg=COLORS["bg_card"], fg=COLORS["text_dim"]).pack(anchor="w")
        row = tk.Frame(f, bg=COLORS["bg_card"])
        row.pack(anchor="w")
        tk.Label(row, textvariable=var, font=FONTS["mono_lg"],
                 bg=COLORS["bg_card"], fg=color).pack(side="left")
        tk.Label(row, text=f"  {unit}", font=FONTS["sans_sm"],
                 bg=COLORS["bg_card"], fg=COLORS["text_secondary"]).pack(side="left")

    # ── Логика ────────────────────────────────────────────────────
    def _select_vehicle(self, name):
        self.var_vehicle.set(name)
        self._on_vehicle_change()

    def _on_vehicle_change(self):
        name = self.var_vehicle.get()
        data = VEHICLES[name]
        self.var_rate.set(str(data["norm"]))
        # Подсветка активной кнопки
        for n, btn in self.vehicle_btns.items():
            if n == name:
                btn.config(bg=COLORS["accent"], fg=COLORS["bg_dark"],
                           font=FONTS["sans_xs"])
            else:
                btn.config(bg=COLORS["bg_input"], fg=COLORS["text_secondary"],
                           font=FONTS["sans_xs"])

    def _select_fuel(self, name, price):
        self.var_fuel.set(name)
        if price > 0:
            self.var_price.set(str(price))
        self._highlight_fuel(name)

    def _highlight_fuel(self, active):
        for n, btn in self.fuel_btns.items():
            if n == active:
                btn.config(bg=COLORS["accent2"], fg=COLORS["bg_dark"])
            else:
                btn.config(bg=COLORS["bg_input"], fg=COLORS["text_secondary"])

    def _on_slider(self, val):
        v = int(float(val))
        self.lbl_load.config(text=f"+{v}%")

    def _calculate(self):
        try:
            distance = float(self.var_distance.get())
            rate     = float(self.var_rate.get())
            price    = float(self.var_price.get())
            load_pct = self.var_load.get()

            if distance <= 0 or rate <= 0:
                messagebox.showerror("Ошибка",
                                     "Расстояние и расход должны быть > 0")
                return

            # Расчёт
            adj_rate = rate * (1 + load_pct / 100)
            liters   = (distance / 100) * adj_rate
            cost     = liters * price
            vehicle  = self.var_vehicle.get()
            tank     = VEHICLES[vehicle]["tank"]
            full_range = (tank / adj_rate) * 100 if adj_rate else 0

            # Обновление результатов
            self.res_liters.set(f"{liters:.1f}")
            self.res_cost.set(f"{cost:,.0f}")
            self.res_per100.set(f"{adj_rate:.1f}")
            self.res_range.set(f"{full_range:.0f}")

            # Анимация датчика
            self.gauge.set_value(adj_rate, max_val=40)

            # Мигание кнопки
            self._flash_btn()

            # Сохранение в историю
            entry = {
                "time":     datetime.now().strftime("%d.%m %H:%M"),
                "vehicle":  vehicle.split(" ", 1)[1][:10],
                "dist":     distance,
                "liters":   liters,
                "cost":     cost,
                "rate":     adj_rate,
            }
            self.history.insert(0, entry)
            self.history = self.history[:50]
            self._save_history()
            self._refresh_history_list()

        except ValueError:
            messagebox.showerror("Ошибка ввода",
                                  "Проверьте введённые данные — "
                                  "все поля должны содержать числа.")

    def _flash_btn(self):
        colors = [COLORS["accent3"], COLORS["accent"], COLORS["accent3"]]
        for i, c in enumerate(colors):
            self.after(i * 120, lambda col=c: self.calc_btn.config(bg=col))
        self.after(len(colors)*120, lambda: self.calc_btn.config(bg=COLORS["accent"]))

    # ── История ───────────────────────────────────────────────────
    def _refresh_history_list(self):
        self.history_lb.delete(0, "end")
        for h in self.history:
            line = (f"{h['time']}  {h['vehicle']:<12}"
                    f"  {h['dist']:.0f}км"
                    f"  {h['liters']:.1f}л"
                    f"  {h['cost']:,.0f}₽")
            self.history_lb.insert("end", line)

    def _clear_history(self):
        self.history = []
        self._save_history()
        self._refresh_history_list()

    def _load_history(self):
        try:
            with open(HISTORY_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception:
            return []

    def _save_history(self):
        try:
            with open(HISTORY_FILE, "w", encoding="utf-8") as f:
                json.dump(self.history, f, ensure_ascii=False)
        except Exception:
            pass

    # ── Анимации интерфейса ───────────────────────────────────────
    def _tick(self):
        now = datetime.now().strftime("⏱  %H:%M:%S   %d.%m.%Y")
        self.lbl_time.config(text=now)
        self.after(1000, self._tick)

    def _pulse_accent(self):
        """Пульсирующая граница кнопки расчёта"""
        colors = [COLORS["accent"], "#007A99", COLORS["accent"]]
        idx = getattr(self, "_pulse_idx", 0)
        self.calc_btn.config(highlightbackground=colors[idx % len(colors)])
        self._pulse_idx = idx + 1
        self.after(800, self._pulse_accent)


# ══════════════════════════════════════════════════════════════════
#  ЗАПУСК
# ══════════════════════════════════════════════════════════════════

def main():
    # Настройка стилей ttk
    app = FuelCalculatorApp()
    style = ttk.Style(app)
    style.theme_use("clam")
    style.configure("TScale",
                     background=COLORS["bg_card"],
                     troughcolor=COLORS["bg_input"],
                     sliderthickness=18)
    style.configure("TScrollbar",
                     background=COLORS["bg_input"],
                     troughcolor=COLORS["bg_card"],
                     borderwidth=0)
    app.mainloop()


if __name__ == "__main__":
    main()
