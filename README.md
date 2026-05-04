#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import tkinter as tk
from tkinter import ttk, messagebox
import math, json, os
from datetime import datetime


# ─────────────────────────────────────────────────────────────
#  FIX: имитация прозрачности (Tkinter не поддерживает alpha)
# ─────────────────────────────────────────────────────────────
def fade_color(hex_color, factor=0.25):
    hex_color = hex_color.lstrip("#")
    r = int(hex_color[0:2], 16)
    g = int(hex_color[2:4], 16)
    b = int(hex_color[4:6], 16)

    r = int(r * factor)
    g = int(g * factor)
    b = int(b * factor)

    return f"#{r:02x}{g:02x}{b:02x}"


# ─────────────────────────────────────────────────────────────
#  ТЕМЫ (FIXED)
# ─────────────────────────────────────────────────────────────
THEMES = {
    "cyber": {
        "bg": "#08091A",
        "bg2": "#0D1128",
        "panel": "#111830",
        "card": "#141C35",
        "input_bg": "#0A0F22",
        "accent": "#00E5FF",
        "accent2": "#FF4D6D",
        "accent3": "#39FF14",
        "warn": "#FFD600",
        "txt": "#D8ECFF",
        "txt2": "#5A7A99",
        "txt3": "#2A4060",
        "border": "#1A2B50",
        "border_hi": "#00E5FF",
        "sidebar": "#090E1F",
        "sidebar_hi": "#0A3A4A",  # FIX вместо #00E5FF22
    },
}

VEHICLES = {
    "🚗": ("Легковой", 8.5, 55),
    "🚙": ("Внедорожник", 12.0, 75),
}

FUEL_DATA = {
    "АИ-95": ("⛽", 65),
}

HISTORY_FILE = os.path.join(os.path.expanduser("~"), ".fuelcalc_pro.json")


# ─────────────────────────────────────────────────────────────
#  GAUGE (FIXED)
# ─────────────────────────────────────────────────────────────
class Gauge(tk.Canvas):
    def __init__(self, parent, size=160, C=None):
        super().__init__(parent, width=size, height=size, highlightthickness=0)
        self.size = size
        self.C = C
        self.cx = self.cy = size // 2
        self.r = size // 2 - 14
        self._val = 0
        self.configure(bg=C["card"])
        self._draw_base()

    def _draw_base(self):
        cx, cy, r = self.cx, self.cy, self.r
        C = self.C
        self.delete("all")

        # FIX: заменили alpha на fade_color
        for i in range(3, 0, -1):
            self.create_oval(
                cx - r - i * 3,
                cy - r - i * 3,
                cx + r + i * 3,
                cy + r + i * 3,
                outline=fade_color(C["accent"], 0.2),
                width=1,
            )

        self.create_oval(cx - r, cy - r, cx + r, cy + r,
                         outline=C["border"], width=2, fill=C["bg"])

        self.arc_id = self.create_arc(
            cx - r + 10, cy - r + 10,
            cx + r - 10, cy + r - 10,
            start=210, extent=0,
            outline="", fill=C["accent"]
        )

        self.needle = self.create_line(cx, cy, cx, cy - r + 20,
                                       fill=C["accent2"], width=3)

    def set_value(self, val, max_v=40):
        f = max(0, min(1, val / max_v))
        angle = math.radians(-210 + f * 240)
        cx, cy, r = self.cx, self.cy, self.r

        nx = cx + (r - 20) * math.cos(angle)
        ny = cy + (r - 20) * math.sin(angle)

        self.coords(self.needle, cx, cy, nx, ny)


# ─────────────────────────────────────────────────────────────
#  APP (МИНИМАЛЬНАЯ РАБОЧАЯ ВЕРСИЯ)
# ─────────────────────────────────────────────────────────────
class FuelCalcPro(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Fuel Calculator FIXED")
        self.geometry("500x400")

        self.C = THEMES["cyber"]

        self.v_dist = tk.StringVar(value="100")
        self.v_rate = tk.StringVar(value="8.5")
        self.v_price = tk.StringVar(value="65")

        self.r_cost = tk.StringVar(value="—")

        self._build()

    def _build(self):
        C = self.C
        self.configure(bg=C["bg"])

        frame = tk.Frame(self, bg=C["bg"])
        frame.pack(pady=20)

        tk.Entry(frame, textvariable=self.v_dist).pack(pady=5)
        tk.Entry(frame, textvariable=self.v_rate).pack(pady=5)
        tk.Entry(frame, textvariable=self.v_price).pack(pady=5)

        tk.Button(frame, text="Рассчитать",
                  command=self._calculate).pack(pady=10)

        tk.Label(frame, textvariable=self.r_cost,
                 fg=C["accent"]).pack(pady=10)

        self.gauge = Gauge(self, size=200, C=C)
        self.gauge.pack(pady=10)

    def _calculate(self):
        try:
            dist = float(self.v_dist.get())
            rate = float(self.v_rate.get())
            price = float(self.v_price.get())

            liters = dist / 100 * rate
            cost = liters * price

            self.r_cost.set(f"{cost:.0f} ₽")
            self.gauge.set_value(rate)

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")


# ─────────────────────────────────────────────────────────────
#  RUN
# ─────────────────────────────────────────────────────────────
if __name__ == "__main__":
    app = FuelCalcPro()
    app.mainloop()
