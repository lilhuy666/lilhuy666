#!/usr/bin/env python3
"""
⛽ FUEL CALCULATOR PRO — Калькулятор расхода топлива
Полный рефакторинг: бургер-меню, панель настроек, разделы, анимации
"""

import tkinter as tk
from tkinter import ttk, messagebox, font
import math, json, os, sys
from datetime import datetime


# ══════════════════════════════════════════════════════════════════
#  ТЕМЫ
# ══════════════════════════════════════════════════════════════════
THEMES = {
    "cyber": {
        "bg":           "#08091A",
        "bg2":          "#0D1128",
        "panel":        "#111830",
        "card":         "#141C35",
        "input_bg":     "#0A0F22",
        "accent":       "#00E5FF",
        "accent2":      "#FF4D6D",
        "accent3":      "#39FF14",
        "warn":         "#FFD600",
        "txt":          "#D8ECFF",
        "txt2":         "#5A7A99",
        "txt3":         "#2A4060",
        "border":       "#1A2B50",
        "border_hi":    "#00E5FF",
        "sidebar":      "#090E1F",
        "sidebar_hi":   "#00E5FF22",
        "name":         "Cyber Blue",
    },
    "neon_pink": {
        "bg":           "#0F0A14",
        "bg2":          "#160E1E",
        "panel":        "#1A1025",
        "card":         "#201530",
        "input_bg":     "#110A1A",
        "accent":       "#FF2D78",
        "accent2":      "#BC00FE",
        "accent3":      "#00F5A0",
        "warn":         "#FFB300",
        "txt":          "#F0D8FF",
        "txt2":         "#8A6AA0",
        "txt3":         "#4A3060",
        "border":       "#2A1A40",
        "border_hi":    "#FF2D78",
        "sidebar":      "#0D081A",
        "sidebar_hi":   "#FF2D7820",
        "name":         "Neon Pink",
    },
    "matrix": {
        "bg":           "#010A01",
        "bg2":          "#020F02",
        "panel":        "#031203",
        "card":         "#051505",
        "input_bg":     "#021002",
        "accent":       "#00FF41",
        "accent2":      "#39FF14",
        "accent3":      "#AAFFAA",
        "warn":         "#FFFF00",
        "txt":          "#BBFFCC",
        "txt2":         "#3A7A3A",
        "txt3":         "#1A4A1A",
        "border":       "#0A2A0A",
        "border_hi":    "#00FF41",
        "sidebar":      "#010801",
        "sidebar_hi":   "#00FF4120",
        "name":         "Matrix Green",
    },
    "amber": {
        "bg":           "#120A00",
        "bg2":          "#1A0F00",
        "panel":        "#221400",
        "card":         "#291A00",
        "input_bg":     "#180D00",
        "accent":       "#FF9500",
        "accent2":      "#FFD600",
        "accent3":      "#FF4500",
        "warn":         "#FF2200",
        "txt":          "#FFE8B0",
        "txt2":         "#A06020",
        "txt3":         "#603010",
        "border":       "#3A2000",
        "border_hi":    "#FF9500",
        "sidebar":      "#0E0800",
        "sidebar_hi":   "#FF950020",
        "name":         "Amber Retro",
    },
}

VEHICLES = {
    "🚗": ("Легковой",    8.5,  55),
    "🚙": ("Внедорожник", 12.0, 75),
    "🚐": ("Минивэн",     11.0, 70),
    "🚌": ("Автобус",     25.0, 200),
    "🚛": ("Грузовик",    30.0, 300),
    "🏍": ("Мотоцикл",    5.0,  18),
    "🚜": ("Трактор",     15.0, 120),
    "⛵": ("Лодка",       20.0, 100),
}

FUEL_DATA = {
    "АИ-92":    ("⛽", 60),
    "АИ-95":    ("⛽", 65),
    "АИ-98":    ("⛽", 75),
    "Дизель":   ("🛢", 58),
    "Газ LPG":  ("💨", 40),
    "Электро":  ("⚡", 0),
}

HISTORY_FILE = os.path.join(os.path.expanduser("~"), ".fuelcalc_pro.json")
SETTINGS_FILE = os.path.join(os.path.expanduser("~"), ".fuelcalc_settings.json")

SECTIONS = ["calculator", "profile", "history", "about"]
SECTION_LABELS = {
    "calculator": ("⚡", "Калькулятор"),
    "profile":    ("👤", "Профиль"),
    "history":    ("📋", "История"),
    "about":      ("ℹ",  "О программе"),
}


# ══════════════════════════════════════════════════════════════════
#  HELPERS
# ══════════════════════════════════════════════════════════════════

def load_json(path, default):
    try:
        with open(path, "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception:
        return default

def save_json(path, data):
    try:
        with open(path, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception:
        pass


# ══════════════════════════════════════════════════════════════════
#  GAUGE WIDGET
# ══════════════════════════════════════════════════════════════════

class Gauge(tk.Canvas):
    def __init__(self, parent, size=160, C=None, **kw):
        super().__init__(parent, width=size, height=size,
                         highlightthickness=0, **kw)
        self.size = size
        self.C = C or THEMES["cyber"]
        self.cx = self.cy = size // 2
        self.r = size // 2 - 14
        self._val = 0
        self._job = None
        self.configure(bg=self.C["card"])
        self._draw_base()

    def _draw_base(self):
        cx, cy, r = self.cx, self.cy, self.r
        C = self.C
        self.delete("all")
        # Outer glow ring
        for i in range(3, 0, -1):
            self.create_oval(cx-r-i*3, cy-r-i*3, cx+r+i*3, cy+r+i*3,
                             outline=C["accent"] + "18", width=1, fill="")
        self.create_oval(cx-r-2, cy-r-2, cx+r+2, cy+r+2,
                         outline=C["border_hi"], width=1, fill="")
        self.create_oval(cx-r, cy-r, cx+r, cy+r,
                         outline=C["border"], width=2, fill=C["bg"])
        # Ticks
        for i in range(21):
            a = math.radians(-210 + i * 21)
            ri = r - (13 if i % 2 == 0 else 7)
            x1 = cx + r * math.cos(a); y1 = cy + r * math.sin(a)
            x2 = cx + ri * math.cos(a); y2 = cy + ri * math.sin(a)
            col = C["accent"] if i % 2 == 0 else C["border"]
            self.create_line(x1,y1,x2,y2, fill=col, width=2 if i%2==0 else 1)
        # Arc fill
        self.arc_id = self.create_arc(cx-r+10, cy-r+10, cx+r-10, cy+r-10,
                                       start=210, extent=0,
                                       outline="", fill=C["accent"],
                                       style=tk.PIESLICE)
        # Inner mask
        self.create_oval(cx-r+26, cy-r+26, cx+r-26, cy+r-26,
                         fill=C["bg"], outline="")
        # Needle
        self.needle = self.create_line(cx, cy,
                                        cx + (r-18)*math.cos(math.radians(-210)),
                                        cy + (r-18)*math.sin(math.radians(-210)),
                                        fill=C["accent2"], width=3, capstyle=tk.ROUND)
        self.create_oval(cx-5, cy-5, cx+5, cy+5,
                         fill=C["accent2"], outline=C["bg"])

    def set_value(self, val, max_v=40):
        if self._job:
            try: self.after_cancel(self._job)
            except: pass
        self._anim(self._val, val, max_v, 20)
        self._val = val

    def _anim(self, cur, end, mv, n):
        if n <= 0:
            self._draw(end, mv); return
        cur += (end - cur) / n
        self._draw(cur, mv)
        self._job = self.after(18, self._anim, cur, end, mv, n-1)

    def _draw(self, val, mv):
        f = max(0.0, min(1.0, val / mv)) if mv else 0
        a = math.radians(-210 + f * 240)
        cx, cy, r = self.cx, self.cy, self.r
        nx = cx + (r-18)*math.cos(a)
        ny = cy + (r-18)*math.sin(a)
        self.coords(self.needle, cx, cy, nx, ny)
        ext = f * 240
        col = (self.C["accent3"] if f < 0.5
               else self.C["warn"] if f < 0.8
               else self.C["accent2"])
        self.itemconfig(self.arc_id, start=210-ext, extent=ext, fill=col)
        self.itemconfig(self.needle, fill=col)

    def apply_theme(self, C):
        self.C = C
        self.configure(bg=C["card"])
        self._draw_base()


# ══════════════════════════════════════════════════════════════════
#  MAIN APP
# ══════════════════════════════════════════════════════════════════

class FuelCalcPro(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("⛽ FUEL CALCULATOR PRO")
        self.geometry("1100x750")
        self.minsize(900, 640)
        self.resizable(True, True)

        # Load settings & history
        self._settings = load_json(SETTINGS_FILE, {
            "theme": "cyber",
            "currency": "₽",
            "distance_unit": "км",
            "compact_history": False,
            "auto_calc": False,
            "show_gauge": True,
            "notifications": True,
            "username": "Водитель",
            "vehicle_emoji": "🚗",
        })
        self.history = load_json(HISTORY_FILE, [])
        self.C = THEMES[self._settings.get("theme", "cyber")]

        # State
        self._sidebar_open = False
        self._settings_open = False
        self._section = tk.StringVar(value="calculator")

        # Calc vars
        self.v_vehicle  = tk.StringVar(value="🚗")
        self.v_fuel     = tk.StringVar(value="АИ-95")
        self.v_dist     = tk.StringVar(value="100")
        self.v_rate     = tk.StringVar(value="8.5")
        self.v_price    = tk.StringVar(value="65")
        self.v_load     = tk.DoubleVar(value=0)
        self.v_passes   = tk.IntVar(value=1)

        # Result vars
        self.r_liters   = tk.StringVar(value="—")
        self.r_cost     = tk.StringVar(value="—")
        self.r_per100   = tk.StringVar(value="—")
        self.r_range    = tk.StringVar(value="—")
        self.r_co2      = tk.StringVar(value="—")

        self._build()
        self._apply_theme()
        self._clock_tick()

    # ─────────────────────────────────────────────────────────────
    #  BUILD ROOT LAYOUT
    # ─────────────────────────────────────────────────────────────
    def _build(self):
        self.configure(bg=self.C["bg"])

        # ── TOP BAR ──────────────────────────────────────────────
        self.topbar = tk.Frame(self, height=56)
        self.topbar.pack(fill="x", side="top")
        self.topbar.pack_propagate(False)

        # Burger btn
        self.burger_btn = tk.Button(self.topbar, text="☰",
                                     font=("Courier New", 20, "bold"),
                                     relief="flat", bd=0, cursor="hand2",
                                     command=self._toggle_sidebar)
        self.burger_btn.pack(side="left", padx=16, pady=8)

        # Logo
        logo_f = tk.Frame(self.topbar)
        logo_f.pack(side="left")
        tk.Label(logo_f, text="⛽", font=("Arial", 22)).pack(side="left")
        lbl_frame = tk.Frame(logo_f)
        lbl_frame.pack(side="left", padx=6)
        self.lbl_title = tk.Label(lbl_frame, text="FUEL CALCULATOR PRO",
                                   font=("Courier New", 14, "bold"))
        self.lbl_title.pack(anchor="w")
        self.lbl_sub = tk.Label(lbl_frame, text="Профессиональный расчёт топлива",
                                 font=("Arial", 9))
        self.lbl_sub.pack(anchor="w")

        # Right controls
        right_f = tk.Frame(self.topbar)
        right_f.pack(side="right", padx=12)
        self.lbl_clock = tk.Label(right_f, font=("Courier New", 11))
        self.lbl_clock.pack(side="right", padx=16)

        # Settings button
        self.settings_btn = tk.Button(right_f, text="⚙  Настройки",
                                       font=("Arial", 10, "bold"),
                                       relief="flat", bd=0, cursor="hand2",
                                       padx=12, pady=4,
                                       command=self._toggle_settings)
        self.settings_btn.pack(side="right", padx=4)

        # Separator
        self.top_sep = tk.Frame(self, height=2)
        self.top_sep.pack(fill="x")

        # ── BODY ─────────────────────────────────────────────────
        self.body = tk.Frame(self)
        self.body.pack(fill="both", expand=True)

        # Sidebar overlay (hidden by default)
        self.sidebar = tk.Frame(self.body, width=240)
        self.sidebar.pack_propagate(False)
        # NOT packed yet — shown on toggle

        # Content area
        self.content_frame = tk.Frame(self.body)
        self.content_frame.pack(fill="both", expand=True, side="left")

        # Settings panel (right drawer, hidden by default)
        self.settings_panel = tk.Frame(self.body, width=280)
        self.settings_panel.pack_propagate(False)

        self._build_sidebar()
        self._build_settings_panel()
        self._show_section("calculator")

    # ─────────────────────────────────────────────────────────────
    #  SIDEBAR
    # ─────────────────────────────────────────────────────────────
    def _build_sidebar(self):
        C = self.C
        sb = self.sidebar

        # Header
        hdr = tk.Frame(sb, height=56)
        hdr.pack(fill="x")
        hdr.pack_propagate(False)
        tk.Label(hdr, text="МЕНЮ", font=("Courier New", 13, "bold")).pack(
            side="left", padx=20, pady=16)
        tk.Button(hdr, text="✕", font=("Arial", 14),
                  relief="flat", bd=0, cursor="hand2",
                  command=self._toggle_sidebar).pack(side="right", padx=12)

        tk.Frame(sb, height=1).pack(fill="x")

        # Nav items
        self.nav_btns = {}
        for sec in SECTIONS:
            icon, label = SECTION_LABELS[sec]
            f = tk.Frame(sb, cursor="hand2")
            f.pack(fill="x", padx=8, pady=2)
            ico_lbl = tk.Label(f, text=icon, font=("Arial", 16), width=3)
            ico_lbl.pack(side="left", pady=10, padx=(8, 4))
            lbl = tk.Label(f, text=label, font=("Arial", 12, "bold"), anchor="w")
            lbl.pack(side="left", fill="x", expand=True)
            for w in (f, ico_lbl, lbl):
                w.bind("<Button-1>", lambda e, s=sec: self._nav_click(s))
            self.nav_btns[sec] = (f, ico_lbl, lbl)

        # Bottom: version
        tk.Label(sb, text="v2.0.0  ©2025 FuelPro",
                 font=("Courier New", 8)).pack(side="bottom", pady=12)

    def _nav_click(self, sec):
        self._toggle_sidebar()
        self._show_section(sec)

    def _toggle_sidebar(self):
        if self._sidebar_open:
            self.sidebar.pack_forget()
            self._sidebar_open = False
        else:
            if self._settings_open:
                self._toggle_settings()
            self.sidebar.pack(fill="y", side="left", before=self.content_frame)
            self._sidebar_open = True
        self._refresh_sidebar_highlight()

    def _refresh_sidebar_highlight(self):
        cur = self._section.get()
        C = self.C
        for sec, (f, ico, lbl) in self.nav_btns.items():
            if sec == cur:
                f.configure(bg=C["sidebar_hi"])
                ico.configure(bg=C["sidebar_hi"], fg=C["accent"])
                lbl.configure(bg=C["sidebar_hi"], fg=C["accent"])
            else:
                f.configure(bg=C["sidebar"])
                ico.configure(bg=C["sidebar"], fg=C["txt2"])
                lbl.configure(bg=C["sidebar"], fg=C["txt2"])

    # ─────────────────────────────────────────────────────────────
    #  SETTINGS PANEL
    # ─────────────────────────────────────────────────────────────
    def _build_settings_panel(self):
        sp = self.settings_panel
        C = self.C

        # Header
        hdr = tk.Frame(sp, height=56)
        hdr.pack(fill="x")
        hdr.pack_propagate(False)
        tk.Label(hdr, text="⚙  НАСТРОЙКИ", font=("Courier New", 12, "bold")).pack(
            side="left", padx=16, pady=16)
        tk.Button(hdr, text="✕", font=("Arial", 14),
                  relief="flat", bd=0, cursor="hand2",
                  command=self._toggle_settings).pack(side="right", padx=12)

        tk.Frame(sp, height=1).pack(fill="x")

        scroll_canvas = tk.Canvas(sp, highlightthickness=0)
        scroll_canvas.pack(fill="both", expand=True)
        scr_inner = tk.Frame(scroll_canvas)
        scroll_canvas.create_window((0, 0), window=scr_inner, anchor="nw")
        scr_inner.bind("<Configure>", lambda e: scroll_canvas.configure(
            scrollregion=scroll_canvas.bbox("all")))
        self.settings_inner = scr_inner
        self._build_settings_content()

    def _build_settings_content(self):
        f = self.settings_inner
        for w in f.winfo_children():
            w.destroy()
        C = self.C
        f.configure(bg=C["panel"])

        def section_label(text):
            tk.Label(f, text=text, font=("Courier New", 9, "bold"),
                     bg=C["panel"], fg=C["txt2"], anchor="w").pack(
                fill="x", padx=16, pady=(14, 2))
            tk.Frame(f, height=1, bg=C["border"]).pack(fill="x", padx=16)

        def row(label, widget_fn):
            rf = tk.Frame(f, bg=C["panel"])
            rf.pack(fill="x", padx=16, pady=5)
            tk.Label(rf, text=label, font=("Arial", 10), bg=C["panel"],
                     fg=C["txt"], anchor="w", width=16).pack(side="left")
            widget_fn(rf)

        # ── Тема ──
        section_label("🎨  ВНЕШНИЙ ВИД")
        theme_var = tk.StringVar(value=self._settings.get("theme", "cyber"))
        def theme_row(rf):
            cb = ttk.Combobox(rf, textvariable=theme_var,
                              values=list(THEMES.keys()), width=14, state="readonly")
            cb.pack(side="left")
            tk.Button(rf, text="✓", font=("Arial", 10, "bold"),
                      bg=C["accent"], fg=C["bg"], relief="flat",
                      cursor="hand2", padx=6,
                      command=lambda: self._change_theme(theme_var.get())).pack(side="left", padx=4)
        row("Тема:", theme_row)

        # ── Валюта ──
        section_label("💰  ВАЛЮТА И ЕДИНИЦЫ")
        curr_var = tk.StringVar(value=self._settings.get("currency", "₽"))
        def curr_row(rf):
            for sym in ["₽", "$", "€", "£"]:
                btn = tk.Button(rf, text=sym, font=("Arial", 12, "bold"),
                                bg=C["accent"] if curr_var.get()==sym else C["input_bg"],
                                fg=C["bg"] if curr_var.get()==sym else C["txt"],
                                relief="flat", width=2, cursor="hand2")
                btn.pack(side="left", padx=2)
                def pick_curr(s=sym, b=btn):
                    curr_var.set(s)
                    self._settings["currency"] = s
                    self._save_settings()
                    self._build_settings_content()
                btn.config(command=pick_curr)
        row("Валюта:", curr_row)

        # Единица расстояния
        unit_var = tk.StringVar(value=self._settings.get("distance_unit", "км"))
        def unit_row(rf):
            for u in ["км", "мили"]:
                bg = C["accent"] if unit_var.get()==u else C["input_bg"]
                fg = C["bg"] if unit_var.get()==u else C["txt"]
                tk.Button(rf, text=u, font=("Arial", 10),
                          bg=bg, fg=fg, relief="flat", padx=8, cursor="hand2",
                          command=lambda x=u: [unit_var.set(x),
                                               self._settings.update({"distance_unit":x}),
                                               self._save_settings(),
                                               self._build_settings_content()]
                          ).pack(side="left", padx=2)
        row("Расстояние:", unit_row)

        # ── Поведение ──
        section_label("⚙️  ПОВЕДЕНИЕ")
        for key, label, default in [
            ("auto_calc",        "Авто-расчёт",       False),
            ("show_gauge",       "Показывать датчик", True),
            ("notifications",    "Уведомления",       True),
            ("compact_history",  "Компактная история",False),
        ]:
            self._settings.setdefault(key, default)
            var = tk.BooleanVar(value=self._settings[key])
            def mk_toggle(k=key, v=var):
                def toggle():
                    self._settings[k] = v.get()
                    self._save_settings()
                return toggle
            rf = tk.Frame(f, bg=C["panel"])
            rf.pack(fill="x", padx=16, pady=4)
            tk.Label(rf, text=label, font=("Arial", 10), bg=C["panel"],
                     fg=C["txt"], anchor="w", width=22).pack(side="left")
            self._toggle_switch(rf, var, mk_toggle())

        # ── Профиль ──
        section_label("👤  ПРОФИЛЬ")
        uname_var = tk.StringVar(value=self._settings.get("username", "Водитель"))
        def uname_row(rf):
            e = tk.Entry(rf, textvariable=uname_var, font=("Arial", 10),
                         bg=C["input_bg"], fg=C["accent"], insertbackground=C["accent"],
                         relief="flat", width=14)
            e.pack(side="left", padx=(0,4))
            tk.Button(rf, text="💾", font=("Arial", 11),
                      bg=C["accent3"], fg=C["bg"], relief="flat", cursor="hand2",
                      command=lambda: [self._settings.update({"username": uname_var.get()}),
                                       self._save_settings()]).pack(side="left")
        row("Имя:", uname_row)

        # ── Данные ──
        section_label("🗑  УПРАВЛЕНИЕ ДАННЫМИ")
        tk.Button(f, text="  Очистить историю расчётов",
                  font=("Arial", 10), bg=C["accent2"], fg=C["bg"],
                  relief="flat", padx=12, pady=6, cursor="hand2",
                  command=self._clear_history).pack(padx=16, pady=4, fill="x")
        tk.Button(f, text="  Сбросить настройки",
                  font=("Arial", 10), bg=C["border"], fg=C["txt2"],
                  relief="flat", padx=12, pady=6, cursor="hand2",
                  command=self._reset_settings).pack(padx=16, pady=(0, 4), fill="x")

        # ── Инфо ──
        section_label("ℹ  ВЕРСИЯ")
        tk.Label(f, text="Fuel Calculator Pro v2.0\nPython + Tkinter\n© 2025",
                 font=("Courier New", 8), bg=C["panel"], fg=C["txt3"],
                 justify="left").pack(padx=16, pady=8, anchor="w")

    def _toggle_switch(self, parent, var, cmd):
        C = self.C
        cv = tk.Canvas(parent, width=42, height=22, highlightthickness=0,
                       bg=C["panel"], cursor="hand2")
        cv.pack(side="left")
        def draw():
            cv.delete("all")
            bg = C["accent"] if var.get() else C["border"]
            cv.create_rounded_rect = lambda *a, **kw: None
            cv.create_oval(0, 0, 42, 22, fill=bg, outline="")
            x = 26 if var.get() else 16
            cv.create_oval(x-8, 3, x+8, 19, fill="white", outline="")
        def toggle(e=None):
            var.set(not var.get())
            draw()
            cmd()
        cv.bind("<Button-1>", toggle)
        draw()

    def _toggle_settings(self):
        if self._settings_open:
            self.settings_panel.pack_forget()
            self._settings_open = False
        else:
            if self._sidebar_open:
                self._toggle_sidebar()
            self.settings_panel.pack(fill="y", side="right")
            self._settings_open = True
            self._apply_settings_theme()

    # ─────────────────────────────────────────────────────────────
    #  SECTIONS
    # ─────────────────────────────────────────────────────────────
    def _show_section(self, sec):
        self._section.set(sec)
        for w in self.content_frame.winfo_children():
            w.destroy()
        if sec == "calculator":
            self._build_calculator()
        elif sec == "profile":
            self._build_profile()
        elif sec == "history":
            self._build_history_section()
        elif sec == "about":
            self._build_about()

    # ─────────────────────────────────────────────────────────────
    #  CALCULATOR SECTION
    # ─────────────────────────────────────────────────────────────
    def _build_calculator(self):
        C = self.C
        root = self.content_frame
        root.configure(bg=C["bg"])

        main = tk.Frame(root, bg=C["bg"])
        main.pack(fill="both", expand=True, padx=16, pady=12)
        main.columnconfigure(0, weight=3)
        main.columnconfigure(1, weight=2)
        main.rowconfigure(0, weight=1)

        left = tk.Frame(main, bg=C["bg"])
        left.grid(row=0, column=0, sticky="nsew", padx=(0, 8))
        right = tk.Frame(main, bg=C["bg"])
        right.grid(row=0, column=1, sticky="nsew")

        self._build_vehicle_card(left)
        self._build_params_card(left)
        self._calc_btn = tk.Button(left, text="▶  РАССЧИТАТЬ МАРШРУТ",
                                    font=("Courier New", 13, "bold"),
                                    bg=C["accent"], fg=C["bg"],
                                    activebackground=C["accent2"],
                                    activeforeground=C["bg"],
                                    relief="flat", bd=0, cursor="hand2",
                                    height=2, command=self._calculate)
        self._calc_btn.pack(fill="x", pady=(8, 0))
        self._calc_btn.bind("<Enter>", lambda e: self._calc_btn.config(bg=C["accent2"]))
        self._calc_btn.bind("<Leave>", lambda e: self._calc_btn.config(bg=C["accent"]))

        self._build_result_card(right)

    def _build_vehicle_card(self, parent):
        C = self.C
        card = self._card(parent, "🚗  ТРАНСПОРТНОЕ СРЕДСТВО")
        card.pack(fill="x", pady=(0, 8))

        grid = tk.Frame(card, bg=C["card"])
        grid.pack(fill="x", padx=12, pady=(4, 12))
        self._veh_btns = {}
        vehicles = list(VEHICLES.items())
        for i, (emoji, (name, norm, tank)) in enumerate(vehicles):
            btn = tk.Button(grid, text=f"{emoji}\n{name}",
                            font=("Arial", 9),
                            bg=C["input_bg"], fg=C["txt2"],
                            relief="flat", bd=0, cursor="hand2", width=7,
                            command=lambda e=emoji, n=norm: self._pick_vehicle(e, n))
            btn.grid(row=i//4, column=i%4, padx=3, pady=3, sticky="ew")
            grid.columnconfigure(i%4, weight=1)
            self._veh_btns[emoji] = btn
        self._pick_vehicle(self.v_vehicle.get(), VEHICLES[self.v_vehicle.get()][1])

    def _pick_vehicle(self, emoji, norm):
        C = self.C
        self.v_vehicle.set(emoji)
        self.v_rate.set(str(norm))
        for e, btn in self._veh_btns.items():
            if e == emoji:
                btn.config(bg=C["accent"], fg=C["bg"], font=("Arial", 9, "bold"))
            else:
                btn.config(bg=C["input_bg"], fg=C["txt2"], font=("Arial", 9))

    def _build_params_card(self, parent):
        C = self.C
        card = self._card(parent, "⚙️  ПАРАМЕТРЫ")
        card.pack(fill="x", pady=(0, 8))

        du = self._settings.get("distance_unit", "км")
        for label, var, hint in [
            (f"📏 Расстояние ({du}):",    self.v_dist,  du),
            ("💧 Расход (л/100):",        self.v_rate,  "л/100"),
            (f"💰 Цена ({self._settings.get('currency','₽')}/л):", self.v_price, self._settings.get('currency','₽')+"/л"),
            ("🔄 Количество поездок:",    self.v_passes,"раз"),
        ]:
            self._input_row(card, label, var, hint)

        # Slider — load
        sf = tk.Frame(card, bg=C["card"])
        sf.pack(fill="x", padx=12, pady=(4, 8))
        tk.Label(sf, text="🏋️ Нагрузка / пробки:", font=("Arial", 10),
                 bg=C["card"], fg=C["txt2"]).pack(anchor="w")
        sr = tk.Frame(sf, bg=C["card"])
        sr.pack(fill="x", pady=4)
        sld = ttk.Scale(sr, from_=0, to=50, variable=self.v_load,
                        orient="horizontal", command=self._on_slider)
        sld.pack(side="left", fill="x", expand=True)
        self._lbl_load = tk.Label(sr, text="  0%", font=("Courier New", 10),
                                   bg=C["card"], fg=C["accent"], width=6)
        self._lbl_load.pack(side="left")

        # Fuel buttons
        ff = tk.Frame(card, bg=C["card"])
        ff.pack(fill="x", padx=12, pady=(0, 12))
        tk.Label(ff, text="⛽ Вид топлива:", font=("Arial", 10),
                 bg=C["card"], fg=C["txt2"]).pack(anchor="w")
        fbr = tk.Frame(ff, bg=C["card"])
        fbr.pack(fill="x", pady=4)
        self._fuel_btns = {}
        for name, (icon, price) in FUEL_DATA.items():
            btn = tk.Button(fbr, text=f"{icon} {name}", font=("Arial", 9),
                            bg=C["input_bg"], fg=C["txt2"],
                            relief="flat", bd=0, padx=6, pady=3, cursor="hand2",
                            command=lambda n=name, p=price: self._pick_fuel(n, p))
            btn.pack(side="left", padx=2, pady=1)
            self._fuel_btns[name] = btn
        self._pick_fuel(self.v_fuel.get(), FUEL_DATA[self.v_fuel.get()][1])

    def _pick_fuel(self, name, price):
        C = self.C
        self.v_fuel.set(name)
        if price > 0:
            self.v_price.set(str(price))
        for n, btn in self._fuel_btns.items():
            if n == name:
                btn.config(bg=C["accent2"], fg=C["bg"])
            else:
                btn.config(bg=C["input_bg"], fg=C["txt2"])

    def _on_slider(self, v):
        self._lbl_load.config(text=f" +{int(float(v))}%")

    def _build_result_card(self, parent):
        C = self.C
        card = self._card(parent, "📊  РЕЗУЛЬТАТЫ")
        card.pack(fill="x", pady=(0, 8))

        if self._settings.get("show_gauge", True):
            gf = tk.Frame(card, bg=C["card"])
            gf.pack(pady=8)
            tk.Label(gf, text="РАСХОД л/100км", font=("Courier New", 8),
                     bg=C["card"], fg=C["txt3"]).pack()
            self.gauge = Gauge(gf, size=150, C=C)
            self.gauge.pack()
            self._lbl_gauge = tk.Label(gf, textvariable=self.r_per100,
                                        font=("Courier New", 18, "bold"),
                                        bg=C["card"], fg=C["accent"])
            self._lbl_gauge.pack(pady=2)
        else:
            self.gauge = None

        nums = tk.Frame(card, bg=C["card"])
        nums.pack(fill="x", padx=16, pady=(4, 12))
        blocks = [
            ("💧 Топлива:", self.r_liters, "л", C["accent"]),
            ("💰 Стоимость:", self.r_cost, self._settings.get("currency","₽"), C["accent2"]),
            ("🛣️ Запас хода:", self.r_range, self._settings.get("distance_unit","км"), C["accent3"]),
            ("☁️ CO₂:", self.r_co2, "г/км", C["warn"]),
        ]
        for title, var, unit, color in blocks:
            bf = tk.Frame(nums, bg=C["card"])
            bf.pack(fill="x", pady=4)
            tk.Label(bf, text=title, font=("Arial", 9),
                     bg=C["card"], fg=C["txt2"]).pack(anchor="w")
            rr = tk.Frame(bf, bg=C["card"])
            rr.pack(anchor="w")
            tk.Label(rr, textvariable=var, font=("Courier New", 17, "bold"),
                     bg=C["card"], fg=color).pack(side="left")
            tk.Label(rr, text=f"  {unit}", font=("Arial", 10),
                     bg=C["card"], fg=C["txt2"]).pack(side="left")

    # ─────────────────────────────────────────────────────────────
    #  CALCULATE
    # ─────────────────────────────────────────────────────────────
    def _calculate(self):
        try:
            dist  = float(self.v_dist.get())
            rate  = float(self.v_rate.get())
            price = float(self.v_price.get())
            load  = self.v_load.get()
            passes = self.v_passes.get()

            if dist <= 0 or rate <= 0:
                messagebox.showerror("Ошибка", "Расстояние и расход должны быть > 0")
                return

            adj = rate * (1 + load / 100)
            liters = (dist / 100) * adj * passes
            cost = liters * price
            veh = self.v_vehicle.get()
            tank = VEHICLES[veh][2]
            full_range = (tank / adj * 100) if adj else 0
            co2 = adj * 2.31  # кг/100км → г/км approx

            self.r_liters.set(f"{liters:.1f}")
            self.r_cost.set(f"{cost:,.0f}")
            self.r_per100.set(f"{adj:.1f}")
            self.r_range.set(f"{full_range:.0f}")
            self.r_co2.set(f"{co2:.0f}")

            if self.gauge:
                self.gauge.set_value(adj, max_v=40)

            self._flash_btn()
            if self._settings.get("notifications", True):
                self._show_toast(f"✅ {liters:.1f} л · {cost:,.0f} {self._settings.get('currency','₽')}")

            entry = {
                "time":     datetime.now().strftime("%d.%m.%Y %H:%M"),
                "vehicle":  VEHICLES[veh][0],
                "dist":     dist,
                "rate":     adj,
                "liters":   liters,
                "cost":     cost,
                "fuel":     self.v_fuel.get(),
                "passes":   passes,
            }
            self.history.insert(0, entry)
            self.history = self.history[:100]
            save_json(HISTORY_FILE, self.history)

        except ValueError:
            messagebox.showerror("Ошибка ввода", "Проверьте числовые поля")

    def _flash_btn(self):
        if not hasattr(self, "_calc_btn"):
            return
        C = self.C
        seq = [C["accent3"], C["accent"], C["accent3"], C["accent"]]
        for i, col in enumerate(seq):
            self.after(i * 100, lambda c=col: self._calc_btn.config(bg=c) if self._calc_btn.winfo_exists() else None)

    def _show_toast(self, msg):
        C = self.C
        toast = tk.Toplevel(self)
        toast.overrideredirect(True)
        toast.attributes("-alpha", 0.92)
        toast.configure(bg=C["accent3"])
        lbl = tk.Label(toast, text=msg, font=("Courier New", 11, "bold"),
                       bg=C["accent3"], fg=C["bg"], padx=20, pady=10)
        lbl.pack()
        # Position bottom-right of main window
        x = self.winfo_x() + self.winfo_width() - 280
        y = self.winfo_y() + self.winfo_height() - 80
        toast.geometry(f"+{x}+{y}")
        toast.after(2200, toast.destroy)

    # ─────────────────────────────────────────────────────────────
    #  PROFILE SECTION
    # ─────────────────────────────────────────────────────────────
    def _build_profile(self):
        C = self.C
        root = self.content_frame
        root.configure(bg=C["bg"])

        canvas = tk.Frame(root, bg=C["bg"])
        canvas.pack(fill="both", expand=True, padx=32, pady=24)

        # Avatar area
        av = tk.Frame(canvas, bg=C["card"],
                      highlightbackground=C["accent"], highlightthickness=2)
        av.pack(fill="x", pady=(0, 16))

        inner = tk.Frame(av, bg=C["card"])
        inner.pack(pady=24)

        uname = self._settings.get("username", "Водитель")
        fav_veh = self._settings.get("vehicle_emoji", "🚗")
        fav_name = VEHICLES.get(fav_veh, ("?",))[0]

        tk.Label(inner, text=fav_veh, font=("Arial", 52),
                 bg=C["card"], fg=C["accent"]).pack()
        tk.Label(inner, text=uname.upper(), font=("Courier New", 20, "bold"),
                 bg=C["card"], fg=C["txt"]).pack(pady=4)
        tk.Label(inner, text=f"Любимый транспорт: {fav_veh} {fav_name}",
                 font=("Arial", 11), bg=C["card"], fg=C["txt2"]).pack()

        # Stats
        if self.history:
            total_km = sum(h.get("dist", 0) for h in self.history)
            total_l = sum(h.get("liters", 0) for h in self.history)
            total_cost = sum(h.get("cost", 0) for h in self.history)
            total_trips = len(self.history)
        else:
            total_km = total_l = total_cost = total_trips = 0

        stats_card = self._card(canvas, "📈  СТАТИСТИКА")
        stats_card.pack(fill="x", pady=(0, 12))
        sg = tk.Frame(stats_card, bg=C["card"])
        sg.pack(fill="x", padx=16, pady=12)
        sg.columnconfigure((0,1,2,3), weight=1)
        stats = [
            ("🚗", "Поездок", str(total_trips), C["accent"]),
            ("📏", "км пройдено", f"{total_km:,.0f}", C["accent2"]),
            ("💧", "литров сожжено", f"{total_l:,.1f}", C["accent3"]),
            ("💰", "потрачено", f"{total_cost:,.0f}", C["warn"]),
        ]
        for i, (ico, lbl, val, col) in enumerate(stats):
            sf = tk.Frame(sg, bg=C["input_bg"],
                          highlightbackground=C["border"], highlightthickness=1)
            sf.grid(row=0, column=i, padx=4, pady=4, sticky="nsew")
            tk.Label(sf, text=ico, font=("Arial", 20),
                     bg=C["input_bg"], fg=col).pack(pady=(12,2))
            tk.Label(sf, text=val, font=("Courier New", 15, "bold"),
                     bg=C["input_bg"], fg=col).pack()
            tk.Label(sf, text=lbl, font=("Arial", 9),
                     bg=C["input_bg"], fg=C["txt2"]).pack(pady=(0, 10))

        # Fav vehicle selector
        fav_card = self._card(canvas, "⭐  ЛЮБИМЫЙ ТРАНСПОРТ")
        fav_card.pack(fill="x")
        fg2 = tk.Frame(fav_card, bg=C["card"])
        fg2.pack(fill="x", padx=16, pady=12)
        self._fav_btns = {}
        for j, (emoji, (name, _, __)) in enumerate(VEHICLES.items()):
            btn = tk.Button(fg2, text=f"{emoji}\n{name}", font=("Arial", 9),
                            bg=C["accent"] if emoji==fav_veh else C["input_bg"],
                            fg=C["bg"] if emoji==fav_veh else C["txt2"],
                            relief="flat", width=7, cursor="hand2",
                            command=lambda e=emoji: self._set_fav_veh(e))
            btn.grid(row=j//4, column=j%4, padx=3, pady=3, sticky="ew")
            fg2.columnconfigure(j%4, weight=1)
            self._fav_btns[emoji] = btn

    def _set_fav_veh(self, emoji):
        C = self.C
        self._settings["vehicle_emoji"] = emoji
        self._save_settings()
        for e, btn in self._fav_btns.items():
            btn.config(bg=C["accent"] if e==emoji else C["input_bg"],
                       fg=C["bg"] if e==emoji else C["txt2"])

    # ─────────────────────────────────────────────────────────────
    #  HISTORY SECTION
    # ─────────────────────────────────────────────────────────────
    def _build_history_section(self):
        C = self.C
        root = self.content_frame
        root.configure(bg=C["bg"])

        card = self._card(root, "📋  ИСТОРИЯ РАСЧЁТОВ")
        card.pack(fill="both", expand=True, padx=16, pady=12)

        toolbar = tk.Frame(card, bg=C["card"])
        toolbar.pack(fill="x", padx=12, pady=6)
        tk.Label(toolbar, text=f"Всего записей: {len(self.history)}",
                 font=("Arial", 10), bg=C["card"], fg=C["txt2"]).pack(side="left")
        tk.Button(toolbar, text="🗑  Очистить",
                  font=("Arial", 10), bg=C["accent2"], fg=C["bg"],
                  relief="flat", cursor="hand2", padx=10,
                  command=lambda: [self._clear_history(), self._build_history_section()]).pack(side="right")

        # Headers
        hf = tk.Frame(card, bg=C["border"])
        hf.pack(fill="x", padx=12, pady=(0, 2))
        for text, w in [("ДАТА", 14), ("ТС", 10), ("КМ", 7), ("РАСХОД", 9),
                         ("ЛИТРЫ", 8), ("СТОИМОСТЬ", 12), ("ТОПЛИВО", 10)]:
            tk.Label(hf, text=text, font=("Courier New", 9, "bold"), width=w,
                     bg=C["border"], fg=C["txt2"], anchor="w").pack(side="left", padx=2)

        # Scrollable list
        lf = tk.Frame(card, bg=C["card"])
        lf.pack(fill="both", expand=True, padx=12, pady=(0, 12))
        sc = ttk.Scrollbar(lf)
        sc.pack(side="right", fill="y")
        lb = tk.Listbox(lf, font=("Courier New", 10),
                         bg=C["input_bg"], fg=C["txt2"],
                         selectbackground=C["accent"], selectforeground=C["bg"],
                         relief="flat", bd=0, yscrollcommand=sc.set,
                         activestyle="none")
        lb.pack(fill="both", expand=True)
        sc.config(command=lb.yview)

        cur = self._settings.get("currency", "₽")
        for h in self.history:
            line = (f"  {h.get('time','?'):<17}"
                    f"{h.get('vehicle','?'):<11}"
                    f"{h.get('dist',0):<8.0f}"
                    f"{h.get('rate',0):<10.1f}"
                    f"{h.get('liters',0):<9.1f}"
                    f"{h.get('cost',0):<13,.0f}"
                    f"{h.get('fuel','?')}")
            lb.insert("end", line)
            lb.itemconfig("end", fg=C["txt"])

    # ─────────────────────────────────────────────────────────────
    #  ABOUT SECTION
    # ─────────────────────────────────────────────────────────────
    def _build_about(self):
        C = self.C
        root = self.content_frame
        root.configure(bg=C["bg"])

        outer = tk.Frame(root, bg=C["bg"])
        outer.pack(fill="both", expand=True, padx=40, pady=30)

        # Big logo
        logo_f = tk.Frame(outer, bg=C["card"],
                           highlightbackground=C["accent"], highlightthickness=2)
        logo_f.pack(fill="x", pady=(0, 20))
        linner = tk.Frame(logo_f, bg=C["card"])
        linner.pack(pady=30)
        tk.Label(linner, text="⛽", font=("Arial", 60),
                 bg=C["card"], fg=C["accent"]).pack()
        tk.Label(linner, text="FUEL CALCULATOR PRO",
                 font=("Courier New", 22, "bold"),
                 bg=C["card"], fg=C["accent"]).pack(pady=4)
        tk.Label(linner, text="Профессиональный инструмент для расчёта расхода топлива",
                 font=("Arial", 12), bg=C["card"], fg=C["txt2"]).pack()
        tk.Label(linner, text="Версия 2.0.0  ·  Python 3  ·  Tkinter",
                 font=("Courier New", 9), bg=C["card"], fg=C["txt3"]).pack(pady=(8, 0))

        features_card = self._card(outer, "✨  ВОЗМОЖНОСТИ")
        features_card.pack(fill="x", pady=(0, 16))
        features = [
            ("⚡", "Мгновенный расчёт", "Расчёт расхода для 8 видов транспорта"),
            ("🎨", "Несколько тем", "Cyber Blue, Neon Pink, Matrix Green, Amber Retro"),
            ("📋", "История расчётов", "Сохранение до 100 последних расчётов"),
            ("👤", "Профиль водителя", "Статистика поездок и любимый транспорт"),
            ("🔧", "Гибкие настройки", "Валюта, единицы измерения, уведомления"),
            ("📊", "Визуальный датчик", "Аналоговый индикатор расхода"),
        ]
        fg = tk.Frame(features_card, bg=C["card"])
        fg.pack(fill="x", padx=16, pady=12)
        fg.columnconfigure((0,1), weight=1)
        for i, (ico, title, desc) in enumerate(features):
            ff = tk.Frame(fg, bg=C["input_bg"],
                          highlightbackground=C["border"], highlightthickness=1)
            ff.grid(row=i//2, column=i%2, padx=6, pady=6, sticky="nsew")
            tk.Label(ff, text=ico, font=("Arial", 18),
                     bg=C["input_bg"], fg=C["accent"]).pack(anchor="w", padx=14, pady=(12, 2))
            tk.Label(ff, text=title, font=("Arial", 11, "bold"),
                     bg=C["input_bg"], fg=C["txt"]).pack(anchor="w", padx=14)
            tk.Label(ff, text=desc, font=("Arial", 9),
                     bg=C["input_bg"], fg=C["txt2"], wraplength=200, justify="left").pack(
                anchor="w", padx=14, pady=(0, 12))

        credits = tk.Frame(outer, bg=C["bg"])
        credits.pack(fill="x")
        tk.Label(credits,
                 text="Разработано с ❤️  для тех, кто считает каждый литр\n"
                      "Исходный код: Python 3 + Tkinter\n"
                      "© 2025 FuelCalc Pro. Все права защищены.",
                 font=("Courier New", 9), bg=C["bg"], fg=C["txt3"],
                 justify="center").pack(pady=12)

    # ─────────────────────────────────────────────────────────────
    #  HELPERS
    # ─────────────────────────────────────────────────────────────
    def _card(self, parent, title):
        C = self.C
        frame = tk.Frame(parent, bg=C["card"],
                         highlightbackground=C["border"], highlightthickness=1)
        hdr = tk.Frame(frame, bg=C["panel"])
        hdr.pack(fill="x")
        tk.Label(hdr, text=title, font=("Courier New", 9, "bold"),
                 bg=C["panel"], fg=C["txt2"]).pack(anchor="w", padx=12, pady=(7, 7))
        tk.Frame(frame, bg=C["border"], height=1).pack(fill="x")
        return frame

    def _input_row(self, parent, label, var, hint):
        C = self.C
        row = tk.Frame(parent, bg=C["card"])
        row.pack(fill="x", padx=12, pady=3)
        tk.Label(row, text=label, font=("Arial", 10), width=22, anchor="w",
                 bg=C["card"], fg=C["txt2"]).pack(side="left")
        e = tk.Entry(row, textvariable=var, font=("Courier New", 11),
                     bg=C["input_bg"], fg=C["accent"],
                     insertbackground=C["accent"],
                     relief="flat", bd=4, width=10)
        e.pack(side="left", padx=(0, 6))
        tk.Label(row, text=hint, font=("Arial", 9),
                 bg=C["card"], fg=C["txt3"]).pack(side="left")

    # ─────────────────────────────────────────────────────────────
    #  THEME & SETTINGS
    # ─────────────────────────────────────────────────────────────
    def _apply_theme(self):
        C = self.C
        self.configure(bg=C["bg"])
        self.topbar.configure(bg=C["panel"])
        self.top_sep.configure(bg=C["border_hi"])
        self.body.configure(bg=C["bg"])
        self.content_frame.configure(bg=C["bg"])
        self.burger_btn.configure(bg=C["panel"], fg=C["accent"],
                                   activebackground=C["bg2"])
        self.lbl_title.configure(bg=C["panel"], fg=C["accent"])
        self.lbl_sub.configure(bg=C["panel"], fg=C["txt2"])
        self.lbl_clock.configure(bg=C["panel"], fg=C["txt2"])
        self.settings_btn.configure(bg=C["accent"], fg=C["bg"],
                                     activebackground=C["accent2"])
        self.sidebar.configure(bg=C["sidebar"])
        self.settings_panel.configure(bg=C["panel"])
        for widget in self.topbar.winfo_children():
            try:
                if isinstance(widget, tk.Frame):
                    widget.configure(bg=C["panel"])
            except Exception:
                pass
        self._apply_sidebar_theme()
        self._apply_settings_theme()
        self._show_section(self._section.get())

    def _apply_sidebar_theme(self):
        C = self.C
        self.sidebar.configure(bg=C["sidebar"])
        self._refresh_sidebar_highlight()

    def _apply_settings_theme(self):
        C = self.C
        self.settings_panel.configure(bg=C["panel"])
        self._build_settings_content()
        sp_children = self.settings_panel.winfo_children()
        for w in sp_children:
            try: w.configure(bg=C["panel"])
            except: pass

    def _change_theme(self, name):
        if name in THEMES:
            self._settings["theme"] = name
            self.C = THEMES[name]
            self._save_settings()
            self._apply_theme()

    def _save_settings(self):
        save_json(SETTINGS_FILE, self._settings)

    def _reset_settings(self):
        if messagebox.askyesno("Сброс", "Сбросить все настройки?"):
            self._settings = {"theme": "cyber", "currency": "₽",
                               "distance_unit": "км", "username": "Водитель"}
            self._save_settings()
            self.C = THEMES["cyber"]
            self._apply_theme()

    def _clear_history(self):
        self.history = []
        save_json(HISTORY_FILE, self.history)

    # ─────────────────────────────────────────────────────────────
    #  CLOCK
    # ─────────────────────────────────────────────────────────────
    def _clock_tick(self):
        self.lbl_clock.config(
            text=datetime.now().strftime("⏱  %H:%M:%S   %d.%m.%Y"))
        self.after(1000, self._clock_tick)


# ══════════════════════════════════════════════════════════════════
#  RUN
# ══════════════════════════════════════════════════════════════════

def main():
    app = FuelCalcPro()
    style = ttk.Style(app)
    style.theme_use("clam")
    C = app.C
    style.configure("TScale",
                     background=C["card"],
                     troughcolor=C["input_bg"],
                     sliderthickness=20, sliderlength=20)
    style.configure("TScrollbar",
                     background=C["input_bg"],
                     troughcolor=C["card"],
                     borderwidth=0, arrowsize=12)
    style.configure("TCombobox",
                     fieldbackground=C["input_bg"],
                     background=C["panel"],
                     foreground=C["txt"],
                     selectbackground=C["accent"],
                     selectforeground=C["bg"])
    app.mainloop()


if __name__ == "__main__":
    main()
