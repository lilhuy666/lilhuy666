import tkinter as tk
from tkinter import ttk, messagebox, filedialog, colorchooser
import json
import os
import datetime
from PIL import Image, ImageTk, ImageFilter, ImageDraw
import io
import base64

# ─────────────────────────────────────────────
#  Путь к файлу данных
# ─────────────────────────────────────────────
DATA_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)), "fuel_data.json")

# ─────────────────────────────────────────────
#  Палитра
# ─────────────────────────────────────────────
C = {
    "bg":          "#0F1923",
    "surface":     "#162030",
    "card":        "#1C2B3A",
    "card2":       "#1A2535",
    "accent":      "#00D4AA",
    "accent2":     "#FF6B35",
    "accent3":     "#4FC3F7",
    "text":        "#E8F4FD",
    "text_dim":    "#7A9BB5",
    "border":      "#2A3F55",
    "danger":      "#FF4757",
    "success":     "#2ED573",
    "warn":        "#FFA502",
    "overlay":     "#0F192380",
    "sidebar":     "#0C1520",
    "header":      "#111E2D",
}

FONT_H1   = ("Segoe UI", 22, "bold")
FONT_H2   = ("Segoe UI", 16, "bold")
FONT_H3   = ("Segoe UI", 13, "bold")
FONT_BODY = ("Segoe UI", 11)
FONT_SM   = ("Segoe UI", 9)
FONT_MONO = ("Consolas", 11)

# ─────────────────────────────────────────────
#  Утилиты данных
# ─────────────────────────────────────────────

def load_data():
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception:
            pass
    return {"vehicles": [], "history": [], "profile": {"name": "", "email": ""}, "settings": {"bg_image": "", "bg_color": C["bg"]}}

def save_data(data):
    try:
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception as e:
        print("Ошибка сохранения:", e)

# ─────────────────────────────────────────────
#  Кастомные виджеты
# ─────────────────────────────────────────────

class RoundedFrame(tk.Canvas):
    def __init__(self, parent, width, height, radius=16, bg=C["card"], border_color=C["border"], **kwargs):
        super().__init__(parent, width=width, height=height,
                         bg=parent.cget("bg") if hasattr(parent, "cget") else C["bg"],
                         highlightthickness=0, **kwargs)
        self._bg = bg
        self._radius = radius
        self._border = border_color
        self._w = width
        self._h = height
        self._draw()
        self.bind("<Configure>", self._on_resize)

    def _draw(self):
        self.delete("all")
        r = self._radius
        w, h = self._w, self._h
        self.create_rounded_rect(2, 2, w-2, h-2, r, fill=self._bg, outline=self._border, width=1)

    def create_rounded_rect(self, x1, y1, x2, y2, r, **kw):
        pts = [
            x1+r, y1,   x2-r, y1,
            x2, y1,     x2, y1+r,
            x2, y2-r,   x2, y2,
            x2-r, y2,   x1+r, y2,
            x1, y2,     x1, y2-r,
            x1, y1+r,   x1, y1,
        ]
        return self.create_polygon(pts, smooth=True, **kw)

    def _on_resize(self, e):
        self._w, self._h = e.width, e.height
        self._draw()


class GlowButton(tk.Frame):
    def __init__(self, parent, text, command=None, color=C["accent"], width=160, height=42, icon="", **kwargs):
        super().__init__(parent, bg=parent.cget("bg") if hasattr(parent, "cget") else C["bg"],
                         highlightthickness=0, **kwargs)
        self._color = color
        self._cmd = command
        self._hovered = False

        self.canvas = tk.Canvas(self, width=width, height=height, bg=self["bg"],
                                highlightthickness=0, cursor="hand2")
        self.canvas.pack()
        self._w, self._h = width, height
        self._text = (icon + "  " + text) if icon else text
        self._draw(False)

        self.canvas.bind("<Enter>", lambda e: self._draw(True))
        self.canvas.bind("<Leave>", lambda e: self._draw(False))
        self.canvas.bind("<Button-1>", lambda e: self._click())
        self.canvas.bind("<ButtonRelease-1>", lambda e: self._draw(True))

    def _draw(self, hover):
        self.canvas.delete("all")
        w, h = self._w, self._h
        r = h // 2
        color = self._color
        if hover:
            self.canvas.create_rounded_rect = lambda *a, **k: None
            # glow shadow
            for i in range(6, 0, -1):
                alpha_color = self._lighten(color, i * 8)
                self._rounded_rect(self.canvas, i, i, w-i, h-i, r-i+6,
                                   fill="", outline=alpha_color, width=1)
            self._rounded_rect(self.canvas, 0, 0, w, h, r, fill=color, outline="")
        else:
            dark = self._darken(color, 40)
            self._rounded_rect(self.canvas, 0, 0, w, h, r, fill=dark, outline=color, width=1.5)
        txt_color = C["bg"] if hover else color
        self.canvas.create_text(w//2, h//2, text=self._text, fill=txt_color,
                                font=("Segoe UI", 10, "bold"), anchor="center")

    def _rounded_rect(self, canvas, x1, y1, x2, y2, r, **kw):
        pts = [
            x1+r, y1,   x2-r, y1,
            x2, y1,     x2, y1+r,
            x2, y2-r,   x2, y2,
            x2-r, y2,   x1+r, y2,
            x1, y2,     x1, y2-r,
            x1, y1+r,   x1, y1,
        ]
        return canvas.create_polygon(pts, smooth=True, **kw)

    def _lighten(self, hex_color, amount):
        hex_color = hex_color.lstrip("#")
        r, g, b = int(hex_color[:2], 16), int(hex_color[2:4], 16), int(hex_color[4:], 16)
        return "#{:02x}{:02x}{:02x}".format(min(255,r+amount), min(255,g+amount), min(255,b+amount))

    def _darken(self, hex_color, amount):
        hex_color = hex_color.lstrip("#")
        r, g, b = int(hex_color[:2], 16), int(hex_color[2:4], 16), int(hex_color[4:], 16)
        return "#{:02x}{:02x}{:02x}".format(max(0,r-amount), max(0,g-amount), max(0,b-amount))

    def _click(self):
        self._draw(False)
        if self._cmd:
            self._cmd()


class StyledEntry(tk.Frame):
    def __init__(self, parent, placeholder="", width=220, **kwargs):
        bg = parent.cget("bg") if hasattr(parent, "cget") else C["bg"]
        super().__init__(parent, bg=bg, highlightthickness=0)
        self._placeholder = placeholder
        self._ph_active = True

        self.entry = tk.Entry(self, bg=C["card2"], fg=C["text_dim"],
                              insertbackground=C["accent"], relief="flat",
                              font=FONT_BODY, width=int(width/8),
                              highlightthickness=1, highlightbackground=C["border"],
                              highlightcolor=C["accent"])
        self.entry.pack(ipady=8, ipadx=10, fill="x")
        self.entry.insert(0, placeholder)
        self.entry.bind("<FocusIn>", self._on_focus_in)
        self.entry.bind("<FocusOut>", self._on_focus_out)

    def _on_focus_in(self, e):
        if self._ph_active:
            self.entry.delete(0, "end")
            self.entry.config(fg=C["text"])
            self._ph_active = False

    def _on_focus_out(self, e):
        if not self.entry.get():
            self.entry.insert(0, self._placeholder)
            self.entry.config(fg=C["text_dim"])
            self._ph_active = True

    def get(self):
        if self._ph_active:
            return ""
        return self.entry.get()

    def set(self, val):
        self.entry.delete(0, "end")
        if val:
            self.entry.insert(0, val)
            self.entry.config(fg=C["text"])
            self._ph_active = False
        else:
            self.entry.insert(0, self._placeholder)
            self.entry.config(fg=C["text_dim"])
            self._ph_active = True


class Label(tk.Label):
    def __init__(self, parent, text="", style="body", color=None, **kwargs):
        fonts = {"h1": FONT_H1, "h2": FONT_H2, "h3": FONT_H3, "body": FONT_BODY, "sm": FONT_SM, "mono": FONT_MONO}
        colors = {"h1": C["text"], "h2": C["text"], "h3": C["accent"], "body": C["text"], "sm": C["text_dim"], "mono": C["accent"]}
        bg = parent.cget("bg") if hasattr(parent, "cget") else C["bg"]
        super().__init__(parent, text=text, font=fonts.get(style, FONT_BODY),
                         fg=color or colors.get(style, C["text"]),
                         bg=bg, **kwargs)


# ─────────────────────────────────────────────
#  Главное приложение
# ─────────────────────────────────────────────

class FuelApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("⛽ FuelTrack Pro")
        self.geometry("1100x720")
        self.minsize(900, 600)
        self.configure(bg=C["bg"])

        self.data = load_data()
        self._bg_image = None
        self._bg_photo = None
        self._sidebar_open = False
        self._current_page = "calculator"
        self._anim_offset = tk.IntVar(value=-260)

        self._build_ui()
        self._load_bg()
        self._show_page("calculator")

        self.bind("<Configure>", self._on_resize)

    # ── Построение интерфейса ──────────────────

    def _build_ui(self):
        # Фоновый canvas
        self.bg_canvas = tk.Canvas(self, highlightthickness=0, bg=C["bg"])
        self.bg_canvas.place(x=0, y=0, relwidth=1, relheight=1)

        # Оверлей (затемнение при открытом сайдбаре)
        self.overlay = tk.Frame(self, bg="#0F1923", cursor="arrow")
        self.overlay.bind("<Button-1>", lambda e: self._close_sidebar())

        # ── Header ──
        self.header = tk.Frame(self, bg=C["header"], height=58)
        self.header.pack(fill="x", side="top")
        self.header.pack_propagate(False)

        # Бургер
        self.burger_btn = tk.Button(self.header, text="☰", font=("Segoe UI", 18),
                                    bg=C["header"], fg=C["accent"], relief="flat",
                                    activebackground=C["surface"], activeforeground=C["accent"],
                                    cursor="hand2", padx=14, command=self._toggle_sidebar)
        self.burger_btn.pack(side="left", pady=4)

        # Заголовок
        self.title_label = tk.Label(self.header, text="⛽  FuelTrack Pro",
                                    font=("Segoe UI", 15, "bold"), bg=C["header"], fg=C["accent"])
        self.title_label.pack(side="left", padx=8)

        # Кнопка настроек
        self.settings_btn = tk.Button(self.header, text="⚙", font=("Segoe UI", 17),
                                      bg=C["header"], fg=C["text_dim"], relief="flat",
                                      activebackground=C["surface"], activeforeground=C["accent"],
                                      cursor="hand2", padx=14, command=self._open_settings)
        self.settings_btn.pack(side="right", pady=4)

        # Индикатор страницы
        self.page_label = tk.Label(self.header, text="Калькулятор",
                                   font=("Segoe UI", 10), bg=C["header"], fg=C["text_dim"])
        self.page_label.pack(side="right", padx=4)

        # ── Sidebar ──
        self.sidebar = tk.Frame(self, bg=C["sidebar"], width=260)
        self.sidebar.place(x=-260, y=58, relheight=1, height=-58, width=260)
        self._build_sidebar()

        # ── Контент ──
        self.content = tk.Frame(self, bg=C["bg"])
        self.content.pack(fill="both", expand=True)

        # Страницы
        self.pages = {}
        for name, cls in [("calculator", CalcPage), ("profile", ProfilePage),
                          ("history", HistoryPage), ("about", AboutPage)]:
            frame = cls(self.content, self)
            frame.place(relx=0, rely=0, relwidth=1, relheight=1)
            self.pages[name] = frame

    def _build_sidebar(self):
        # Шапка sidebar
        header = tk.Frame(self.sidebar, bg=C["sidebar"], height=70)
        header.pack(fill="x")
        header.pack_propagate(False)
        tk.Label(header, text="⛽ FuelTrack", font=("Segoe UI", 14, "bold"),
                 bg=C["sidebar"], fg=C["accent"]).pack(side="left", padx=20, pady=20)

        sep = tk.Frame(self.sidebar, bg=C["border"], height=1)
        sep.pack(fill="x", padx=16)

        # Пункты меню
        menu_items = [
            ("calculator", "🧮", "Калькулятор"),
            ("profile",    "🚗", "Профиль & Авто"),
            ("history",    "📋", "История"),
            ("about",      "ℹ️",  "О приложении"),
        ]
        self._menu_buttons = {}
        for page, icon, label in menu_items:
            btn = self._sidebar_item(self.sidebar, icon, label, page)
            self._menu_buttons[page] = btn

        # Нижняя часть
        bottom = tk.Frame(self.sidebar, bg=C["sidebar"])
        bottom.pack(side="bottom", fill="x", pady=20)
        tk.Frame(bottom, bg=C["border"], height=1).pack(fill="x", padx=16, pady=(0,12))
        tk.Label(bottom, text="v1.0  •  by Тарасов К.А.", font=FONT_SM,
                 bg=C["sidebar"], fg=C["text_dim"]).pack(padx=20)

    def _sidebar_item(self, parent, icon, label, page):
        frame = tk.Frame(parent, bg=C["sidebar"], cursor="hand2")
        frame.pack(fill="x", pady=2, padx=10)

        inner = tk.Frame(frame, bg=C["sidebar"])
        inner.pack(fill="x")

        icon_l = tk.Label(inner, text=icon, font=("Segoe UI", 14), bg=C["sidebar"], fg=C["text_dim"], width=3)
        icon_l.pack(side="left", padx=(10,6), pady=10)
        text_l = tk.Label(inner, text=label, font=("Segoe UI", 11), bg=C["sidebar"], fg=C["text_dim"], anchor="w")
        text_l.pack(side="left", fill="x", expand=True)

        accent_bar = tk.Frame(inner, bg=C["sidebar"], width=3)
        accent_bar.pack(side="right", fill="y")

        def on_enter(e):
            if self._current_page != page:
                for w in [inner, icon_l, text_l]: w.config(bg=C["surface"])
        def on_leave(e):
            if self._current_page != page:
                for w in [inner, icon_l, text_l]: w.config(bg=C["sidebar"])
        def on_click(e):
            self._show_page(page)
            self._close_sidebar()

        for w in [frame, inner, icon_l, text_l]:
            w.bind("<Enter>", on_enter)
            w.bind("<Leave>", on_leave)
            w.bind("<Button-1>", on_click)

        return (inner, icon_l, text_l, accent_bar)

    # ── Навигация ──────────────────────────────

    def _show_page(self, name):
        # Сброс стиля предыдущего
        if self._current_page in self._menu_buttons:
            inner, icon_l, text_l, bar = self._menu_buttons[self._current_page]
            for w in [inner, icon_l, text_l]: w.config(bg=C["sidebar"])
            bar.config(bg=C["sidebar"])

        self._current_page = name
        # Активный стиль
        if name in self._menu_buttons:
            inner, icon_l, text_l, bar = self._menu_buttons[name]
            for w in [inner, icon_l, text_l]: w.config(bg=C["card"])
            icon_l.config(fg=C["accent"])
            text_l.config(fg=C["text"])
            bar.config(bg=C["accent"])

        labels = {"calculator": "Калькулятор", "profile": "Профиль & Авто",
                  "history": "История", "about": "О приложении"}
        self.page_label.config(text=labels.get(name, ""))

        for n, frame in self.pages.items():
            if n == name:
                frame.lift()
                if hasattr(frame, "refresh"):
                    frame.refresh()
            else:
                frame.lower()

    def _toggle_sidebar(self):
        if self._sidebar_open:
            self._close_sidebar()
        else:
            self._open_sidebar()

    def _open_sidebar(self):
        self._sidebar_open = True
        self.overlay.place(x=0, y=58, relwidth=1, relheight=1)
        self.overlay.lift()
        self.sidebar.lift()
        self._animate_sidebar(target=0)

    def _close_sidebar(self):
        self._sidebar_open = False
        self._animate_sidebar(target=-260, on_done=lambda: self.overlay.place_forget())

    def _animate_sidebar(self, target, step=20, on_done=None):
        cur = int(self.sidebar.place_info().get("x", -260))
        if cur == target:
            if on_done: on_done()
            return
        new = cur + step if cur < target else cur - step
        if (step > 0 and new >= target) or (step < 0 and new <= target):
            new = target
        self.sidebar.place_configure(x=new)
        self.after(12, lambda: self._animate_sidebar(target, step, on_done))

    # ── Фон ───────────────────────────────────

    def _load_bg(self):
        path = self.data["settings"].get("bg_image", "")
        if path and os.path.exists(path):
            self._apply_bg_image(path)

    def _apply_bg_image(self, path):
        try:
            img = Image.open(path)
            w, h = self.winfo_width() or 1100, self.winfo_height() or 720
            img = img.resize((w, h), Image.LANCZOS)
            # Затемнение
            overlay = Image.new("RGBA", img.size, (15, 25, 35, 180))
            if img.mode != "RGBA":
                img = img.convert("RGBA")
            img = Image.alpha_composite(img, overlay)
            self._bg_photo = ImageTk.PhotoImage(img)
            self.bg_canvas.delete("all")
            self.bg_canvas.create_image(0, 0, anchor="nw", image=self._bg_photo)
            self._bg_image = path
        except Exception as e:
            print("Ошибка загрузки фона:", e)

    def _on_resize(self, e):
        if self._bg_image:
            self._apply_bg_image(self._bg_image)

    # ── Настройки ─────────────────────────────

    def _open_settings(self):
        win = tk.Toplevel(self)
        win.title("Настройки")
        win.geometry("420x340")
        win.configure(bg=C["bg"])
        win.resizable(False, False)
        win.grab_set()

        tk.Label(win, text="⚙  Настройки", font=FONT_H2, bg=C["bg"], fg=C["accent"]).pack(pady=(20,4))
        tk.Frame(win, bg=C["border"], height=1).pack(fill="x", padx=24, pady=8)

        # Фон — изображение
        frm = tk.Frame(win, bg=C["surface"], padx=20, pady=16)
        frm.pack(fill="x", padx=24, pady=6)
        tk.Label(frm, text="🖼  Фоновое изображение", font=FONT_H3, bg=C["surface"], fg=C["text"]).pack(anchor="w")
        tk.Label(frm, text="Выберите изображение для фона приложения", font=FONT_SM, bg=C["surface"], fg=C["text_dim"]).pack(anchor="w", pady=(2,10))

        path_var = tk.StringVar(value=self.data["settings"].get("bg_image", "") or "Не выбрано")
        path_lbl = tk.Label(frm, textvariable=path_var, font=("Consolas",9), bg=C["card2"],
                            fg=C["text_dim"], padx=8, pady=4, wraplength=300, anchor="w")
        path_lbl.pack(fill="x", pady=(0,8))

        def pick_image():
            fp = filedialog.askopenfilename(
                title="Выберите изображение",
                filetypes=[("Изображения", "*.png *.jpg *.jpeg *.bmp *.gif *.webp"), ("Все файлы", "*.*")])
            if fp:
                path_var.set(fp)
                self.data["settings"]["bg_image"] = fp
                save_data(self.data)
                self._apply_bg_image(fp)

        def clear_image():
            path_var.set("Не выбрано")
            self.data["settings"]["bg_image"] = ""
            save_data(self.data)
            self._bg_image = None
            self.bg_canvas.delete("all")

        btn_row = tk.Frame(frm, bg=C["surface"])
        btn_row.pack(fill="x")
        GlowButton(btn_row, "Выбрать файл", pick_image, C["accent"], 170, 36, "📂").pack(side="left", padx=(0,8))
        GlowButton(btn_row, "Сбросить", clear_image, C["danger"], 120, 36, "✕").pack(side="left")

        tk.Frame(win, bg=C["border"], height=1).pack(fill="x", padx=24, pady=10)
        GlowButton(win, "Закрыть", win.destroy, C["accent"], 140, 38).pack(pady=4)

    def refresh_pages(self):
        for frame in self.pages.values():
            if hasattr(frame, "refresh"):
                frame.refresh()


# ─────────────────────────────────────────────
#  Страница: КАЛЬКУЛЯТОР
# ─────────────────────────────────────────────

class CalcPage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=C["bg"])
        self.app = app
        self._build()

    def _build(self):
        # Скролл
        canvas = tk.Canvas(self, bg=C["bg"], highlightthickness=0)
        scrollbar = ttk.Scrollbar(self, orient="vertical", command=canvas.yview)
        self.scroll_frame = tk.Frame(canvas, bg=C["bg"])
        self.scroll_frame.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.create_window((0, 0), window=self.scroll_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)
        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        canvas.bind_all("<MouseWheel>", lambda e: canvas.yview_scroll(int(-1*(e.delta/120)), "units"))

        wrap = self.scroll_frame
        pad = tk.Frame(wrap, bg=C["bg"])
        pad.pack(fill="both", expand=True, padx=36, pady=24)

        # Заголовок
        hdr = tk.Frame(pad, bg=C["bg"])
        hdr.pack(fill="x", pady=(0,20))
        tk.Label(hdr, text="🧮", font=("Segoe UI", 32), bg=C["bg"], fg=C["accent"]).pack(side="left", padx=(0,14))
        col = tk.Frame(hdr, bg=C["bg"])
        col.pack(side="left")
        tk.Label(col, text="Калькулятор расхода топлива", font=FONT_H1, bg=C["bg"], fg=C["text"]).pack(anchor="w")
        tk.Label(col, text="Точный расчёт затрат и расхода для любого маршрута", font=FONT_BODY, bg=C["bg"], fg=C["text_dim"]).pack(anchor="w")

        # Выбор авто
        car_card = self._card(pad, "🚗  Транспортное средство")
        self._car_var = tk.StringVar(value="— Без привязки к авто —")
        self._car_combo = ttk.Combobox(car_card, textvariable=self._car_var,
                                       font=FONT_BODY, state="readonly", width=40)
        self._style_combo()
        self._car_combo.pack(anchor="w", pady=(0,4))
        tk.Label(car_card, text="Выберите авто из профиля или введите данные вручную",
                 font=FONT_SM, bg=C["card"], fg=C["text_dim"]).pack(anchor="w")
        self._car_combo.bind("<<ComboboxSelected>>", self._on_car_select)

        # Две колонки ввода
        row = tk.Frame(pad, bg=C["bg"])
        row.pack(fill="x", pady=(0,10))

        # Левая колонка
        left = tk.Frame(row, bg=C["bg"])
        left.pack(side="left", fill="both", expand=True, padx=(0,8))

        lcard = self._card(left, "📏  Расстояние и топливо")
        self._dist_entry = self._field(lcard, "Расстояние (км)", "Например: 350")
        self._consumption_entry = self._field(lcard, "Расход топлива (л/100 км)", "Например: 8.5")
        self._price_entry = self._field(lcard, "Цена топлива (руб/л)", "Например: 58.50")

        # Правая колонка
        right = tk.Frame(row, bg=C["bg"])
        right.pack(side="left", fill="both", expand=True, padx=(8,0))

        rcard = self._card(right, "⚙️  Параметры расчёта")
        self._fuel_type_var = tk.StringVar(value="АИ-95")
        tk.Label(rcard, text="Тип топлива", font=FONT_SM, bg=C["card"], fg=C["text_dim"]).pack(anchor="w", pady=(0,4))
        fuel_combo = ttk.Combobox(rcard, textvariable=self._fuel_type_var,
                                  values=["АИ-92", "АИ-95", "АИ-98", "Дизель", "Газ (LPG)", "Электро"],
                                  font=FONT_BODY, state="readonly", width=22)
        fuel_combo.pack(anchor="w", pady=(0,10))

        self._passengers_entry = self._field(rcard, "Пассажиры (чел.)", "1")
        self._route_entry = self._field(rcard, "Название маршрута", "Например: Москва → Тула")
        self._note_entry = self._field(rcard, "Заметка", "Необязательно")

        # Кнопки
        btn_row = tk.Frame(pad, bg=C["bg"])
        btn_row.pack(fill="x", pady=(6,16))
        GlowButton(btn_row, "Рассчитать", self._calculate, C["accent"], 180, 44, "⚡").pack(side="left", padx=(0,12))
        GlowButton(btn_row, "Очистить", self._clear, C["text_dim"], 140, 44, "🗑").pack(side="left")

        # Результат
        self.result_frame = tk.Frame(pad, bg=C["bg"])
        self.result_frame.pack(fill="x")

    def _card(self, parent, title):
        outer = tk.Frame(parent, bg=C["card"], highlightthickness=1,
                         highlightbackground=C["border"])
        outer.pack(fill="x", pady=8)
        tk.Label(outer, text=title, font=FONT_H3, bg=C["card"], fg=C["accent"]).pack(anchor="w", padx=16, pady=(12,6))
        tk.Frame(outer, bg=C["border"], height=1).pack(fill="x", padx=16, pady=(0,10))
        inner = tk.Frame(outer, bg=C["card"])
        inner.pack(fill="x", padx=16, pady=(0,14))
        return inner

    def _field(self, parent, label, placeholder):
        tk.Label(parent, text=label, font=FONT_SM, bg=C["card"], fg=C["text_dim"]).pack(anchor="w", pady=(6,2))
        e = tk.Entry(parent, bg=C["card2"], fg=C["text"], insertbackground=C["accent"],
                     relief="flat", font=FONT_BODY, width=28,
                     highlightthickness=1, highlightbackground=C["border"], highlightcolor=C["accent"])
        e.pack(anchor="w", ipady=7, ipadx=8)
        e.insert(0, placeholder)
        e.bind("<FocusIn>", lambda ev, en=e, ph=placeholder: self._ph_in(ev, en, ph))
        e.bind("<FocusOut>", lambda ev, en=e, ph=placeholder: self._ph_out(ev, en, ph))
        e._placeholder = placeholder
        return e

    def _ph_in(self, e, entry, ph):
        if entry.get() == ph:
            entry.delete(0, "end")
            entry.config(fg=C["text"])

    def _ph_out(self, e, entry, ph):
        if not entry.get():
            entry.insert(0, ph)
            entry.config(fg=C["text_dim"])

    def _get_val(self, entry):
        v = entry.get()
        if v == entry._placeholder:
            return ""
        return v

    def _style_combo(self):
        style = ttk.Style()
        style.configure("TCombobox", fieldbackground=C["card2"], background=C["card2"],
                        foreground=C["text"], selectbackground=C["accent"],
                        arrowcolor=C["accent"])

    def refresh(self):
        vehicles = self.app.data.get("vehicles", [])
        names = ["— Без привязки к авто —"] + [f"{v['brand']} {v['model']} ({v['year']})" for v in vehicles]
        self._car_combo["values"] = names
        if self._car_var.get() not in names:
            self._car_var.set(names[0])

    def _on_car_select(self, e):
        sel = self._car_var.get()
        if sel == "— Без привязки к авто —":
            return
        idx = list(self._car_combo["values"]).index(sel) - 1
        if 0 <= idx < len(self.app.data["vehicles"]):
            v = self.app.data["vehicles"][idx]
            if v.get("consumption"):
                self._consumption_entry.delete(0, "end")
                self._consumption_entry.insert(0, str(v["consumption"]))
                self._consumption_entry.config(fg=C["text"])
            if v.get("fuel_type"):
                self._fuel_type_var.set(v["fuel_type"])

    def _calculate(self):
        try:
            dist_raw = self._get_val(self._dist_entry)
            cons_raw = self._get_val(self._consumption_entry)
            price_raw = self._get_val(self._price_entry)

            if not dist_raw or not cons_raw or not price_raw:
                messagebox.showwarning("Недостаточно данных", "Заполните расстояние, расход и цену топлива!")
                return

            dist = float(dist_raw.replace(",", "."))
            cons = float(cons_raw.replace(",", "."))
            price = float(price_raw.replace(",", "."))

            if dist <= 0 or cons <= 0 or price <= 0:
                messagebox.showwarning("Ошибка", "Значения должны быть больше нуля!")
                return

            total_fuel = dist * cons / 100
            total_cost = total_fuel * price
            cost_per_km = total_cost / dist

            pax_raw = self._get_val(self._passengers_entry)
            pax = int(pax_raw) if pax_raw and pax_raw.isdigit() and int(pax_raw) > 0 else 1
            cost_per_person = total_cost / pax

            self._show_result(dist, cons, price, total_fuel, total_cost, cost_per_km, pax, cost_per_person)

            # Сохранение в историю
            car_name = self._car_var.get()
            if car_name == "— Без привязки к авто —":
                car_name = "Без авто"

            record = {
                "id": datetime.datetime.now().strftime("%Y%m%d%H%M%S"),
                "date": datetime.datetime.now().strftime("%d.%m.%Y %H:%M"),
                "route": self._get_val(self._route_entry) or "Маршрут не указан",
                "note": self._get_val(self._note_entry) or "",
                "vehicle": car_name,
                "fuel_type": self._fuel_type_var.get(),
                "distance": dist,
                "consumption": cons,
                "price": price,
                "total_fuel": round(total_fuel, 2),
                "total_cost": round(total_cost, 2),
                "cost_per_km": round(cost_per_km, 2),
                "passengers": pax,
                "cost_per_person": round(cost_per_person, 2),
            }
            self.app.data["history"].insert(0, record)
            if len(self.app.data["history"]) > 200:
                self.app.data["history"] = self.app.data["history"][:200]
            save_data(self.app.data)

        except ValueError:
            messagebox.showerror("Ошибка ввода", "Проверьте правильность введённых числовых значений.")

    def _show_result(self, dist, cons, price, total_fuel, total_cost, cost_per_km, pax, cpp):
        for w in self.result_frame.winfo_children():
            w.destroy()

        outer = tk.Frame(self.result_frame, bg=C["card"], highlightthickness=2,
                         highlightbackground=C["accent"])
        outer.pack(fill="x", pady=8)

        tk.Label(outer, text="✅  Результаты расчёта", font=FONT_H3, bg=C["card"], fg=C["accent"]).pack(anchor="w", padx=16, pady=(14,4))
        tk.Frame(outer, bg=C["accent"], height=1).pack(fill="x", padx=16, pady=(0,12))

        grid = tk.Frame(outer, bg=C["card"])
        grid.pack(fill="x", padx=16, pady=(0,16))

        items = [
            ("⛽", "Расход топлива", f"{total_fuel:.2f} л",      C["accent"]),
            ("💰", "Общая стоимость", f"{total_cost:,.0f} ₽",    C["accent2"]),
            ("📍", "Стоимость / км", f"{cost_per_km:.2f} ₽/км",  C["accent3"]),
            ("👥", f"На {pax} чел.", f"{cpp:,.0f} ₽/чел.",        C["warn"]),
        ]
        for col_i, (icon, label, value, color) in enumerate(items):
            cell = tk.Frame(grid, bg=C["card2"], padx=14, pady=12)
            cell.grid(row=0, column=col_i, padx=6, pady=4, sticky="nsew")
            grid.columnconfigure(col_i, weight=1)
            tk.Label(cell, text=icon, font=("Segoe UI", 20), bg=C["card2"], fg=color).pack()
            tk.Label(cell, text=value, font=("Segoe UI", 15, "bold"), bg=C["card2"], fg=color).pack()
            tk.Label(cell, text=label, font=FONT_SM, bg=C["card2"], fg=C["text_dim"]).pack()

        tk.Label(outer, text=f"📊  Исходные данные: {dist} км  •  {cons} л/100км  •  {price} ₽/л",
                 font=FONT_SM, bg=C["card"], fg=C["text_dim"]).pack(pady=(0,10))

    def _clear(self):
        for entry, ph in [(self._dist_entry, "Например: 350"),
                          (self._consumption_entry, "Например: 8.5"),
                          (self._price_entry, "Например: 58.50"),
                          (self._passengers_entry, "1"),
                          (self._route_entry, "Например: Москва → Тула"),
                          (self._note_entry, "Необязательно")]:
            entry.delete(0, "end")
            entry.insert(0, ph)
            entry.config(fg=C["text_dim"])
        for w in self.result_frame.winfo_children():
            w.destroy()


# ─────────────────────────────────────────────
#  Страница: ПРОФИЛЬ
# ─────────────────────────────────────────────

class ProfilePage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=C["bg"])
        self.app = app
        self._build()

    def _build(self):
        canvas = tk.Canvas(self, bg=C["bg"], highlightthickness=0)
        sb = ttk.Scrollbar(self, orient="vertical", command=canvas.yview)
        self.sf = tk.Frame(canvas, bg=C["bg"])
        self.sf.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.create_window((0,0), window=self.sf, anchor="nw")
        canvas.configure(yscrollcommand=sb.set)
        canvas.pack(side="left", fill="both", expand=True)
        sb.pack(side="right", fill="y")
        canvas.bind_all("<MouseWheel>", lambda e: canvas.yview_scroll(int(-1*(e.delta/120)), "units"))

        self.pad = tk.Frame(self.sf, bg=C["bg"])
        self.pad.pack(fill="both", expand=True, padx=36, pady=24)
        self._draw()

    def _draw(self):
        for w in self.pad.winfo_children():
            w.destroy()

        hdr = tk.Frame(self.pad, bg=C["bg"])
        hdr.pack(fill="x", pady=(0,20))
        tk.Label(hdr, text="🚗", font=("Segoe UI",32), bg=C["bg"], fg=C["accent"]).pack(side="left", padx=(0,14))
        col = tk.Frame(hdr, bg=C["bg"]); col.pack(side="left")
        tk.Label(col, text="Профиль и транспортные средства", font=FONT_H1, bg=C["bg"], fg=C["text"]).pack(anchor="w")
        tk.Label(col, text="Управляйте своими автомобилями и личными данными", font=FONT_BODY, bg=C["bg"], fg=C["text_dim"]).pack(anchor="w")

        # Профиль пользователя
        pcard = tk.Frame(self.pad, bg=C["card"], highlightthickness=1, highlightbackground=C["border"])
        pcard.pack(fill="x", pady=(0,16))
        tk.Label(pcard, text="👤  Личные данные", font=FONT_H3, bg=C["card"], fg=C["accent"]).pack(anchor="w", padx=16, pady=(12,6))
        tk.Frame(pcard, bg=C["border"], height=1).pack(fill="x", padx=16)

        pinner = tk.Frame(pcard, bg=C["card"]); pinner.pack(fill="x", padx=16, pady=14)
        left = tk.Frame(pinner, bg=C["card"]); left.pack(side="left", fill="x", expand=True)
        right = tk.Frame(pinner, bg=C["card"]); right.pack(side="left")

        profile = self.app.data.get("profile", {})
        tk.Label(left, text="Имя", font=FONT_SM, bg=C["card"], fg=C["text_dim"]).pack(anchor="w")
        self._name_e = tk.Entry(left, bg=C["card2"], fg=C["text"], insertbackground=C["accent"],
                                relief="flat", font=FONT_BODY, width=30,
                                highlightthickness=1, highlightbackground=C["border"], highlightcolor=C["accent"])
        self._name_e.insert(0, profile.get("name",""))
        self._name_e.pack(anchor="w", ipady=7, ipadx=8, pady=(2,10))

        tk.Label(left, text="Email", font=FONT_SM, bg=C["card"], fg=C["text_dim"]).pack(anchor="w")
        self._email_e = tk.Entry(left, bg=C["card2"], fg=C["text"], insertbackground=C["accent"],
                                 relief="flat", font=FONT_BODY, width=30,
                                 highlightthickness=1, highlightbackground=C["border"], highlightcolor=C["accent"])
        self._email_e.insert(0, profile.get("email",""))
        self._email_e.pack(anchor="w", ipady=7, ipadx=8)

        GlowButton(right, "Сохранить", self._save_profile, C["accent"], 130, 38, "💾").pack(padx=(20,0))

        # Мои авто
        vcard = tk.Frame(self.pad, bg=C["card"], highlightthickness=1, highlightbackground=C["border"])
        vcard.pack(fill="x", pady=(0,16))
        vhdr = tk.Frame(vcard, bg=C["card"]); vhdr.pack(fill="x", padx=16, pady=(12,6))
        tk.Label(vhdr, text="🚘  Мои автомобили", font=FONT_H3, bg=C["card"], fg=C["accent"]).pack(side="left")
        GlowButton(vhdr, "Добавить авто", self._open_add_vehicle, C["success"], 170, 34, "➕").pack(side="right")
        tk.Frame(vcard, bg=C["border"], height=1).pack(fill="x", padx=16)

        self._vehicles_frame = tk.Frame(vcard, bg=C["card"])
        self._vehicles_frame.pack(fill="x", padx=16, pady=14)
        self._draw_vehicles()

    def _draw_vehicles(self):
        for w in self._vehicles_frame.winfo_children():
            w.destroy()
        vehicles = self.app.data.get("vehicles", [])
        if not vehicles:
            tk.Label(self._vehicles_frame, text="Автомобили не добавлены. Нажмите «Добавить авто».",
                     font=FONT_BODY, bg=C["card"], fg=C["text_dim"]).pack(pady=20)
            return

        fuel_icons = {"АИ-92":"⛽","АИ-95":"⛽","АИ-98":"⛽","Дизель":"🛢","Газ (LPG)":"🔵","Электро":"⚡"}
        for i, v in enumerate(vehicles):
            row = tk.Frame(self._vehicles_frame, bg=C["card2"],
                           highlightthickness=1, highlightbackground=C["border"])
            row.pack(fill="x", pady=4)

            icon = fuel_icons.get(v.get("fuel_type",""), "🚗")
            tk.Label(row, text="🚗", font=("Segoe UI",22), bg=C["card2"], fg=C["accent3"]).pack(side="left", padx=14, pady=10)

            info = tk.Frame(row, bg=C["card2"]); info.pack(side="left", fill="x", expand=True, pady=10)
            tk.Label(info, text=f"{v['brand']} {v['model']}  {v['year']}",
                     font=FONT_H3, bg=C["card2"], fg=C["text"]).pack(anchor="w")
            details = f"{icon} {v.get('fuel_type','—')}   •   {v.get('consumption','—')} л/100км"
            if v.get("plate"): details += f"   •   {v['plate']}"
            tk.Label(info, text=details, font=FONT_SM, bg=C["card2"], fg=C["text_dim"]).pack(anchor="w")

            btn_col = tk.Frame(row, bg=C["card2"]); btn_col.pack(side="right", padx=10)
            GlowButton(btn_col, "Изменить", lambda idx=i: self._open_edit_vehicle(idx), C["accent3"], 110, 30, "✏️").pack(pady=2)
            GlowButton(btn_col, "Удалить", lambda idx=i: self._delete_vehicle(idx), C["danger"], 110, 30, "🗑").pack(pady=2)

    def _open_add_vehicle(self):
        self._vehicle_dialog(None)

    def _open_edit_vehicle(self, idx):
        self._vehicle_dialog(idx)

    def _vehicle_dialog(self, idx):
        win = tk.Toplevel(self.app)
        win.title("Добавить авто" if idx is None else "Изменить авто")
        win.geometry("440x480")
        win.configure(bg=C["bg"])
        win.resizable(False, False)
        win.grab_set()

        existing = self.app.data["vehicles"][idx] if idx is not None else {}

        tk.Label(win, text="🚗  " + ("Добавить автомобиль" if idx is None else "Изменить автомобиль"),
                 font=FONT_H2, bg=C["bg"], fg=C["accent"]).pack(pady=(20,4))
        tk.Frame(win, bg=C["border"], height=1).pack(fill="x", padx=24, pady=8)

        frm = tk.Frame(win, bg=C["surface"], padx=24, pady=16)
        frm.pack(fill="x", padx=24)

        def field(label, key, placeholder=""):
            tk.Label(frm, text=label, font=FONT_SM, bg=C["surface"], fg=C["text_dim"]).pack(anchor="w", pady=(8,2))
            e = tk.Entry(frm, bg=C["card2"], fg=C["text"], insertbackground=C["accent"],
                         relief="flat", font=FONT_BODY, width=36,
                         highlightthickness=1, highlightbackground=C["border"], highlightcolor=C["accent"])
            e.pack(anchor="w", ipady=7, ipadx=8)
            val = existing.get(key, "")
            if val: e.insert(0, str(val))
            return e

        e_brand = field("Марка *", "brand", "Toyota")
        e_model = field("Модель *", "model", "Camry")
        e_year  = field("Год выпуска *", "year", "2020")
        e_cons  = field("Расход топлива (л/100 км)", "consumption", "8.5")
        e_plate = field("Гос. номер", "plate", "А123БВ77")

        tk.Label(frm, text="Тип топлива", font=FONT_SM, bg=C["surface"], fg=C["text_dim"]).pack(anchor="w", pady=(8,2))
        fuel_var = tk.StringVar(value=existing.get("fuel_type","АИ-95"))
        ttk.Combobox(frm, textvariable=fuel_var,
                     values=["АИ-92","АИ-95","АИ-98","Дизель","Газ (LPG)","Электро"],
                     font=FONT_BODY, state="readonly", width=24).pack(anchor="w")

        def save():
            brand = e_brand.get().strip()
            model = e_model.get().strip()
            year  = e_year.get().strip()
            if not brand or not model or not year:
                messagebox.showwarning("Ошибка", "Заполните марку, модель и год!", parent=win)
                return
            v = {
                "brand": brand, "model": model, "year": year,
                "consumption": e_cons.get().strip(),
                "plate": e_plate.get().strip(),
                "fuel_type": fuel_var.get(),
            }
            if idx is None:
                self.app.data["vehicles"].append(v)
            else:
                self.app.data["vehicles"][idx] = v
            save_data(self.app.data)
            self._draw_vehicles()
            self.app.pages["calculator"].refresh()
            win.destroy()

        tk.Frame(win, bg=C["bg"]).pack(expand=True)
        btn_row = tk.Frame(win, bg=C["bg"]); btn_row.pack(pady=16)
        GlowButton(btn_row, "Сохранить", save, C["accent"], 140, 40, "💾").pack(side="left", padx=8)
        GlowButton(btn_row, "Отмена", win.destroy, C["text_dim"], 120, 40).pack(side="left")

    def _delete_vehicle(self, idx):
        v = self.app.data["vehicles"][idx]
        if messagebox.askyesno("Удалить", f"Удалить {v['brand']} {v['model']}?"):
            del self.app.data["vehicles"][idx]
            save_data(self.app.data)
            self._draw_vehicles()
            self.app.pages["calculator"].refresh()

    def _save_profile(self):
        self.app.data["profile"]["name"] = self._name_e.get().strip()
        self.app.data["profile"]["email"] = self._email_e.get().strip()
        save_data(self.app.data)
        messagebox.showinfo("Сохранено", "Профиль успешно обновлён!")

    def refresh(self):
        self._draw()


# ─────────────────────────────────────────────
#  Страница: ИСТОРИЯ
# ─────────────────────────────────────────────

class HistoryPage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=C["bg"])
        self.app = app
        self._filter_var = tk.StringVar(value="Все")
        self._search_var = tk.StringVar()
        self._build()

    def _build(self):
        hdr_frame = tk.Frame(self, bg=C["bg"])
        hdr_frame.pack(fill="x", padx=36, pady=(24,0))

        tk.Label(hdr_frame, text="📋", font=("Segoe UI",32), bg=C["bg"], fg=C["accent"]).pack(side="left", padx=(0,14))
        col = tk.Frame(hdr_frame, bg=C["bg"]); col.pack(side="left")
        tk.Label(col, text="История расчётов", font=FONT_H1, bg=C["bg"], fg=C["text"]).pack(anchor="w")
        tk.Label(col, text="Все сохранённые расчёты расхода топлива", font=FONT_BODY, bg=C["bg"], fg=C["text_dim"]).pack(anchor="w")

        # Панель фильтров
        filter_frame = tk.Frame(self, bg=C["surface"])
        filter_frame.pack(fill="x", padx=36, pady=12)
        tk.Label(filter_frame, text="🔍 Поиск:", font=FONT_BODY, bg=C["surface"], fg=C["text_dim"]).pack(side="left", padx=(12,4), pady=10)
        search_e = tk.Entry(filter_frame, textvariable=self._search_var, bg=C["card2"], fg=C["text"],
                            insertbackground=C["accent"], relief="flat", font=FONT_BODY, width=22,
                            highlightthickness=1, highlightbackground=C["border"], highlightcolor=C["accent"])
        search_e.pack(side="left", ipady=6, ipadx=8, pady=10)
        search_e.bind("<KeyRelease>", lambda e: self._draw_records())

        GlowButton(filter_frame, "Очистить историю", self._clear_history, C["danger"], 170, 34, "🗑").pack(side="right", padx=12, pady=10)
        GlowButton(filter_frame, "Обновить", self.refresh, C["accent3"], 130, 34, "🔄").pack(side="right", padx=4, pady=10)

        # Список
        canvas = tk.Canvas(self, bg=C["bg"], highlightthickness=0)
        sb = ttk.Scrollbar(self, orient="vertical", command=canvas.yview)
        self.sf = tk.Frame(canvas, bg=C["bg"])
        self.sf.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.create_window((0,0), window=self.sf, anchor="nw")
        canvas.configure(yscrollcommand=sb.set)
        canvas.pack(side="left", fill="both", expand=True)
        sb.pack(side="right", fill="y")
        canvas.bind_all("<MouseWheel>", lambda e: canvas.yview_scroll(int(-1*(e.delta/120)), "units"))

        self.records_frame = tk.Frame(self.sf, bg=C["bg"])
        self.records_frame.pack(fill="both", expand=True, padx=36, pady=10)
        self._draw_records()

    def _draw_records(self):
        for w in self.records_frame.winfo_children():
            w.destroy()

        history = self.app.data.get("history", [])
        q = self._search_var.get().lower().strip()
        if q:
            history = [r for r in history if q in r.get("route","").lower() or
                       q in r.get("vehicle","").lower() or q in r.get("note","").lower()]

        if not history:
            tk.Label(self.records_frame, text="История пуста. Выполните расчёт в разделе «Калькулятор».",
                     font=FONT_BODY, bg=C["bg"], fg=C["text_dim"]).pack(pady=60)
            return

        # Статистика
        total_cost = sum(r.get("total_cost",0) for r in history)
        total_fuel = sum(r.get("total_fuel",0) for r in history)
        total_km   = sum(r.get("distance",0) for r in history)
        stat = tk.Frame(self.records_frame, bg=C["card2"],
                        highlightthickness=1, highlightbackground=C["border"])
        stat.pack(fill="x", pady=(0,16))
        stats_data = [
            ("📊", f"{len(history)}", "расчётов"),
            ("⛽", f"{total_fuel:.1f} л", "топлива"),
            ("📍", f"{total_km:,.0f} км", "пробег"),
            ("💰", f"{total_cost:,.0f} ₽", "затрат"),
        ]
        for icon, val, lbl in stats_data:
            cell = tk.Frame(stat, bg=C["card2"]); cell.pack(side="left", expand=True, pady=12, padx=10)
            tk.Label(cell, text=icon, font=("Segoe UI",16), bg=C["card2"], fg=C["accent"]).pack()
            tk.Label(cell, text=val, font=("Segoe UI",13,"bold"), bg=C["card2"], fg=C["accent"]).pack()
            tk.Label(cell, text=lbl, font=FONT_SM, bg=C["card2"], fg=C["text_dim"]).pack()

        for r in history:
            self._record_card(r)

    def _record_card(self, r):
        card = tk.Frame(self.records_frame, bg=C["card"],
                        highlightthickness=1, highlightbackground=C["border"])
        card.pack(fill="x", pady=5)

        top = tk.Frame(card, bg=C["card"]); top.pack(fill="x", padx=16, pady=(12,4))
        tk.Label(top, text=r.get("route","Маршрут"), font=FONT_H3, bg=C["card"], fg=C["text"]).pack(side="left")
        tk.Label(top, text=r.get("date",""), font=FONT_SM, bg=C["card"], fg=C["text_dim"]).pack(side="right")

        mid = tk.Frame(card, bg=C["card"]); mid.pack(fill="x", padx=16, pady=(0,4))
        parts = [
            f"🚗 {r.get('vehicle','')}",
            f"⛽ {r.get('fuel_type','')}",
            f"📍 {r.get('distance','')} км",
            f"💧 {r.get('total_fuel','')} л",
            f"💰 {r.get('total_cost','')} ₽",
        ]
        tk.Label(mid, text="   •   ".join(parts), font=FONT_SM, bg=C["card"], fg=C["text_dim"]).pack(side="left")

        if r.get("note"):
            tk.Label(card, text=f"📝 {r['note']}", font=FONT_SM, bg=C["card"], fg=C["text_dim"]).pack(anchor="w", padx=16, pady=(0,4))

        bot = tk.Frame(card, bg=C["card2"]); bot.pack(fill="x")
        metrics = [
            ("💰  Стоимость", f"{r.get('total_cost',0):,.0f} ₽",    C["accent2"]),
            ("⛽  Топливо",   f"{r.get('total_fuel',0):.2f} л",      C["accent"]),
            ("📍  ₽/км",     f"{r.get('cost_per_km',0):.2f} ₽",     C["accent3"]),
            ("👥  На чел.",  f"{r.get('cost_per_person',0):,.0f} ₽", C["warn"]),
        ]
        for lbl, val, clr in metrics:
            cell = tk.Frame(bot, bg=C["card2"]); cell.pack(side="left", expand=True, pady=8)
            tk.Label(cell, text=lbl, font=FONT_SM, bg=C["card2"], fg=C["text_dim"]).pack()
            tk.Label(cell, text=val, font=("Segoe UI",12,"bold"), bg=C["card2"], fg=clr).pack()

        def del_record(rid=r["id"]):
            self.app.data["history"] = [x for x in self.app.data["history"] if x["id"] != rid]
            save_data(self.app.data)
            self._draw_records()

        GlowButton(card, "Удалить", del_record, C["danger"], 100, 28).pack(anchor="e", padx=16, pady=8)

    def _clear_history(self):
        if messagebox.askyesno("Очистить историю", "Удалить всю историю расчётов?"):
            self.app.data["history"] = []
            save_data(self.app.data)
            self._draw_records()

    def refresh(self):
        self._draw_records()


# ─────────────────────────────────────────────
#  Страница: О ПРИЛОЖЕНИИ
# ─────────────────────────────────────────────

class AboutPage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=C["bg"])
        self.app = app
        self._build()

    def _build(self):
        canvas = tk.Canvas(self, bg=C["bg"], highlightthickness=0)
        sb = ttk.Scrollbar(self, orient="vertical", command=canvas.yview)
        sf = tk.Frame(canvas, bg=C["bg"])
        sf.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.create_window((0,0), window=sf, anchor="nw")
        canvas.configure(yscrollcommand=sb.set)
        canvas.pack(side="left", fill="both", expand=True)
        sb.pack(side="right", fill="y")

        pad = tk.Frame(sf, bg=C["bg"])
        pad.pack(fill="both", expand=True, padx=60, pady=36)

        # Hero
        hero = tk.Frame(pad, bg=C["surface"], highlightthickness=2, highlightbackground=C["accent"])
        hero.pack(fill="x", pady=(0,24))
        tk.Label(hero, text="⛽", font=("Segoe UI",56), bg=C["surface"], fg=C["accent"]).pack(pady=(30,0))
        tk.Label(hero, text="FuelTrack Pro", font=("Segoe UI",28,"bold"), bg=C["surface"], fg=C["text"]).pack()
        tk.Label(hero, text="Профессиональный калькулятор расхода топлива", font=FONT_BODY,
                 bg=C["surface"], fg=C["text_dim"]).pack(pady=(4,4))
        tk.Label(hero, text="Версия 1.0.0", font=FONT_SM, bg=C["surface"], fg=C["accent"]).pack(pady=(0,24))

        def section(icon, title, text):
            frm = tk.Frame(pad, bg=C["card"], highlightthickness=1, highlightbackground=C["border"])
            frm.pack(fill="x", pady=8)
            hdr = tk.Frame(frm, bg=C["card"]); hdr.pack(fill="x", padx=20, pady=(16,8))
            tk.Label(hdr, text=icon, font=("Segoe UI",20), bg=C["card"], fg=C["accent"]).pack(side="left", padx=(0,10))
            tk.Label(hdr, text=title, font=FONT_H3, bg=C["card"], fg=C["text"]).pack(side="left")
            tk.Frame(frm, bg=C["border"], height=1).pack(fill="x", padx=20)
            tk.Label(frm, text=text, font=FONT_BODY, bg=C["card"], fg=C["text_dim"],
                     wraplength=700, justify="left").pack(anchor="w", padx=20, pady=(10,20))

        section("📋", "Описание приложения",
            "FuelTrack Pro — это мощный и удобный инструмент для расчёта расхода и стоимости "
            "топлива для любых транспортных средств. Приложение позволяет точно планировать "
            "бюджет поездок, вести учёт всех маршрутов и анализировать затраты на топливо.")

        section("✨", "Возможности",
            "🧮  Точный расчёт расхода топлива и стоимости поездки\n"
            "🚗  Управление парком транспортных средств в профиле\n"
            "📋  Полная история расчётов с фильтрацией и поиском\n"
            "👥  Расчёт стоимости на каждого пассажира\n"
            "🖼  Настраиваемый фон приложения\n"
            "⛽  Поддержка всех видов топлива: бензин, дизель, газ, электро\n"
            "💾  Автоматическое сохранение данных")

        section("👨‍💻", "Разработчик",
            "Приложение разработано Тарасовым Кириллом Анатольевичем.\n\n"
            "Создано с любовью к автомобилям и стремлением упростить ежедневные "
            "расчёты каждого автовладельца. Приложение написано на языке Python "
            "с использованием библиотек tkinter и Pillow.\n\n"
            "«Каждая поездка — это история. FuelTrack Pro помогает её записать.»")

        section("🛠", "Технологии",
            "Python 3.12  •  tkinter (GUI)  •  Pillow (работа с изображениями)  •  JSON (хранение данных)\n\n"
            "Приложение работает полностью офлайн. Все данные хранятся локально на вашем компьютере "
            "в файле fuel_data.json рядом с программой.")

        # Footer
        footer = tk.Frame(pad, bg=C["surface"])
        footer.pack(fill="x", pady=(16,0))
        tk.Label(footer, text="© 2024  Тарасов Кирилл Анатольевич  •  FuelTrack Pro v1.0.0",
                 font=FONT_SM, bg=C["surface"], fg=C["text_dim"]).pack(pady=16)


# ─────────────────────────────────────────────
#  Точка входа
# ─────────────────────────────────────────────

if __name__ == "__main__":
    try:
        from PIL import Image, ImageTk
    except ImportError:
        import subprocess, sys
        print("Устанавливаю Pillow...")
        subprocess.check_call([sys.executable, "-m", "pip", "install", "Pillow"])
        from PIL import Image, ImageTk

    app = FuelApp()
    app.mainloop()
