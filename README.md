#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Приложение: Автомобильные перевозки — тарификация и маршруты
Файл: cargo_app.py
Запуск: python cargo_app.py
Без внешних библиотек (использует sqlite3, csv, heapq)
"""

import sqlite3
import os
import csv
import heapq
from typing import Dict, Tuple, List, Optional

DB_FILE = "cargo_app.db"

# ---------------------------
# Утилиты для БД
# ---------------------------
def get_conn():
    conn = sqlite3.connect(DB_FILE)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    if os.path.exists(DB_FILE):
        return
    conn = get_conn()
    cur = conn.cursor()
    # Города
    cur.execute("""
    CREATE TABLE cities (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT UNIQUE NOT NULL
    )""")
    # Дороги (ориентированные: from_city -> to_city, distance в км)
    cur.execute("""
    CREATE TABLE roads (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        from_city INTEGER NOT NULL,
        to_city INTEGER NOT NULL,
        distance_km REAL NOT NULL,
        FOREIGN KEY(from_city) REFERENCES cities(id),
        FOREIGN KEY(to_city) REFERENCES cities(id),
        UNIQUE(from_city, to_city)
    )""")
    # Транспортные средства (для проверки вместимости)
    cur.execute("""
    CREATE TABLE vehicles (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        max_weight_kg REAL NOT NULL,
        max_volume_m3 REAL NOT NULL,
        base_cost_multiplier REAL DEFAULT 1.0
    )""")
    # Примеры тарифов (можно расширить)
    cur.execute("""
    CREATE TABLE tariffs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        base_eur REAL DEFAULT 50.0,
        per_km_eur REAL DEFAULT 0.5,
        per_kg_eur REAL DEFAULT 0.02,
        per_m3_eur REAL DEFAULT 2.0,
        dangerous_multiplier REAL DEFAULT 1.5,
        urgent_multiplier REAL DEFAULT 1.25,
        fuel_surcharge_percent REAL DEFAULT 0.0
    )""")
    conn.commit()

    # Добавим тестовые данные
    cur.execute("INSERT INTO cities (name) VALUES (?)", ("Amsterdam",))
    cur.execute("INSERT INTO cities (name) VALUES (?)", ("Rotterdam",))
    cur.execute("INSERT INTO cities (name) VALUES (?)", ("Utrecht",))
    cur.execute("INSERT INTO cities (name) VALUES (?)", ("The Hague",))
    cur.execute("INSERT INTO cities (name) VALUES (?)", ("Eindhoven",))

    # дорожные связи (пример — несимметрично, но мы добавим оба направления)
    def add_road_by_names(a, b, d):
        cur.execute("SELECT id FROM cities WHERE name=?", (a,))
        ai = cur.fetchone()["id"]
        cur.execute("SELECT id FROM cities WHERE name=?", (b,))
        bi = cur.fetchone()["id"]
        cur.execute("INSERT INTO roads (from_city,to_city,distance_km) VALUES (?,?,?)", (ai, bi, d))

    add_road_by_names("Amsterdam", "Rotterdam", 75)
    add_road_by_names("Rotterdam", "Amsterdam", 75)
    add_road_by_names("Amsterdam", "Utrecht", 45)
    add_road_by_names("Utrecht", "Amsterdam", 45)
    add_road_by_names("Utrecht", "The Hague", 60)
    add_road_by_names("The Hague", "Rotterdam", 25)
    add_road_by_names("Rotterdam", "Eindhoven", 110)
    add_road_by_names("Eindhoven", "Utrecht", 80)

    # vehicles
    cur.execute("INSERT INTO vehicles (name, max_weight_kg, max_volume_m3, base_cost_multiplier) VALUES (?,?,?,?)",
               ("Sprinter", 1500, 12.0, 1.0))
    cur.execute("INSERT INTO vehicles (name, max_weight_kg, max_volume_m3, base_cost_multiplier) VALUES (?,?,?,?)",
               ("7.5t Truck", 7500, 40.0, 1.2))
    cur.execute("INSERT INTO vehicles (name, max_weight_kg, max_volume_m3, base_cost_multiplier) VALUES (?,?,?,?)",
               ("18t Truck", 18000, 90.0, 1.5))

    # tariffs
    cur.execute("""INSERT INTO tariffs (name, base_eur, per_km_eur, per_kg_eur, per_m3_eur, dangerous_multiplier, urgent_multiplier, fuel_surcharge_percent)
                   VALUES (?,?,?,?,?,?,?,?)""",
                ("Standard", 60.0, 0.6, 0.015, 1.8, 1.5, 1.25, 5.0))
    cur.execute("""INSERT INTO tariffs (name, base_eur, per_km_eur, per_kg_eur, per_m3_eur, dangerous_multiplier, urgent_multiplier, fuel_surcharge_percent)
                   VALUES (?,?,?,?,?,?,?,?)""",
                ("Economy", 40.0, 0.45, 0.01, 1.2, 1.5, 1.1, 3.0))
    conn.commit()
    conn.close()

# ---------------------------
# Граф и поиск маршрута
# ---------------------------
class CityGraph:
    def __init__(self):
        self.adj: Dict[int, List[Tuple[int, float]]] = {}  # node -> list of (neighbor, distance)

    @staticmethod
    def from_db() -> 'CityGraph':
        g = CityGraph()
        conn = get_conn()
        cur = conn.cursor()
        cur.execute("SELECT id FROM cities")
        for row in cur.fetchall():
            nid = row["id"]
            g.adj[nid] = []
        cur.execute("SELECT from_city,to_city,distance_km FROM roads")
        for r in cur.fetchall():
            g.adj.setdefault(r["from_city"], []).append((r["to_city"], float(r["distance_km"])))
        conn.close()
        return g

    def dijkstra(self, start: int, goal: int) -> Tuple[Optional[float], List[int]]:
        # Возвращает (distance, path_ids)
        dist = {node: float("inf") for node in self.adj}
        prev: Dict[int, Optional[int]] = {node: None for node in self.adj}
        dist[start] = 0.0
        heap = [(0.0, start)]
        while heap:
            d, u = heapq.heappop(heap)
            if d > dist[u]:
                continue
            if u == goal:
                break
            for v, w in self.adj.get(u, []):
                nd = d + w
                # аккуратный пошаговый расчёт: используем float суммирование, это нормально для km
                if nd < dist[v]:
                    dist[v] = nd
                    prev[v] = u
                    heapq.heappush(heap, (nd, v))
        if dist[goal] == float("inf"):
            return None, []
        # восстановление пути
        path = []
        cur = goal
        while cur is not None:
            path.append(cur)
            cur = prev[cur]
        path.reverse()
        return dist[goal], path

# ---------------------------
# Тарифный движок
# ---------------------------
class TariffEngine:
    def __init__(self, tariff_row):
        # tariff_row — sqlite Row с полями из tariffs
        self.name = tariff_row["name"]
        self.base = float(tariff_row["base_eur"])
        self.per_km = float(tariff_row["per_km_eur"])
        self.per_kg = float(tariff_row["per_kg_eur"])
        self.per_m3 = float(tariff_row["per_m3_eur"])
        self.dangerous_multiplier = float(tariff_row["dangerous_multiplier"])
        self.urgent_multiplier = float(tariff_row["urgent_multiplier"])
        self.fuel_surcharge_percent = float(tariff_row["fuel_surcharge_percent"])

    def calc(self, distance_km: float, weight_kg: float, volume_m3: float,
             is_dangerous: bool=False, is_urgent: bool=False, vehicle_multiplier: float=1.0,
             discount_percent: float=0.0) -> Dict[str, float]:
        """
        Расчёт стоимости с подробной разбивкой.
        Формула (пошагово):
          base_part = base
          distance_part = per_km * distance
          weight_part = per_kg * weight
          volume_part = per_m3 * volume
          subtotal = base_part + distance_part + weight_part + volume_part
          apply dangerous multiplier and urgent multiplier and vehicle multiplier
          fuel_surcharge = subtotal * fuel_surcharge_percent / 100
          apply discount_percent
        Возвращает словарь с деталями.
        """
        # пошаговые вычисления (внимание к арифметике)
        base_part = round(self.base, 2)
        distance_part = round(self.per_km * float(distance_km), 2)
        weight_part = round(self.per_kg * float(weight_kg), 2)
        volume_part = round(self.per_m3 * float(volume_m3), 2)

        subtotal = round(base_part + distance_part + weight_part + volume_part, 2)

        multiplier = 1.0
        if is_dangerous:
            multiplier *= self.dangerous_multiplier
        if is_urgent:
            multiplier *= self.urgent_multiplier
        multiplier *= vehicle_multiplier

        after_multiplier = round(subtotal * multiplier, 2)

        fuel_surcharge = round(after_multiplier * (self.fuel_surcharge_percent / 100.0), 2)

        total_before_discount = round(after_multiplier + fuel_surcharge, 2)

        discount_amount = round(total_before_discount * (discount_percent / 100.0), 2)

        total = round(total_before_discount - discount_amount, 2)

        return {
            "tariff_name": self.name,
            "base_part": base_part,
            "distance_part": distance_part,
            "weight_part": weight_part,
            "volume_part": volume_part,
            "subtotal": subtotal,
            "multiplier": multiplier,
            "after_multiplier": after_multiplier,
            "fuel_surcharge": fuel_surcharge,
            "total_before_discount": total_before_discount,
            "discount_percent": discount_percent,
            "discount_amount": discount_amount,
            "total": total
        }

# ---------------------------
# Примитивный CLI
# ---------------------------
def list_cities():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id,name FROM cities ORDER BY name")
    rows = cur.fetchall()
    conn.close()
    print("Города:")
    for r in rows:
        print(f"  {r['id']}: {r['name']}")

def list_roads():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        SELECT r.id, c1.name AS from_name, c2.name AS to_name, r.distance_km
        FROM roads r
        JOIN cities c1 ON r.from_city=c1.id
        JOIN cities c2 ON r.to_city=c2.id
        ORDER BY c1.name, c2.name
    """)
    rows = cur.fetchall()
    conn.close()
    print("Дороги:")
    for r in rows:
        print(f"  {r['id']}: {r['from_name']} -> {r['to_name']} = {r['distance_km']} km")

def list_vehicles():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id,name,max_weight_kg,max_volume_m3,base_cost_multiplier FROM vehicles")
    rows = cur.fetchall()
    conn.close()
    print("Транспортные средства:")
    for r in rows:
        print(f"  {r['id']}: {r['name']} (max {r['max_weight_kg']} kg, {r['max_volume_m3']} m3) multiplier={r['base_cost_multiplier']}")

def list_tariffs():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id,name,base_eur,per_km_eur FROM tariffs")
    rows = cur.fetchall()
    conn.close()
    print("Тарифы:")
    for r in rows:
        print(f"  {r['id']}: {r['name']} base={r['base_eur']}€/{r['per_km_eur']}€/km")

def get_city_id_by_name(name: str) -> Optional[int]:
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id FROM cities WHERE name=?", (name,))
    r = cur.fetchone()
    conn.close()
    return r["id"] if r else None

def get_city_name_by_id(cid: int) -> Optional[str]:
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT name FROM cities WHERE id=?", (cid,))
    r = cur.fetchone()
    conn.close()
    return r["name"] if r else None

def quote_route(from_city: str, to_city: str, weight_kg: float, volume_m3: float,
                vehicle_id: int, tariff_id: int, is_dangerous: bool=False, is_urgent: bool=False,
                discount_percent: float=0.0):
    # Проверки
    conn = get_conn()
    cur = conn.cursor()

    cur.execute("SELECT id FROM cities WHERE name=?", (from_city,))
    a = cur.fetchone()
    cur.execute("SELECT id FROM cities WHERE name=?", (to_city,))
    b = cur.fetchone()
    if not a or not b:
        print("Один из городов не найден — проверьте имена.")
        return
    start = a["id"]; goal = b["id"]

    # Граф и поиск пути
    g = CityGraph.from_db()
    dist_path = g.dijkstra(start, goal)
    if dist_path[0] is None:
        print(f"Маршрут из {from_city} в {to_city} не найден.")
        return
    distance_km, path_ids = dist_path
    # vehicle
    cur.execute("SELECT * FROM vehicles WHERE id=?", (vehicle_id,))
    veh = cur.fetchone()
    if not veh:
        print("Транспортное средство не найдено.")
        return
    if weight_kg > veh["max_weight_kg"]:
        print(f"Внимание: вес {weight_kg} кг превышает грузоподъемность {veh['max_weight_kg']} кг для выбранного ТС.")
    if volume_m3 > veh["max_volume_m3"]:
        print(f"Внимание: объём {volume_m3} м3 превышает вместимость {veh['max_volume_m3']} m3 для выбранного ТС.")

    cur.execute("SELECT * FROM tariffs WHERE id=?", (tariff_id,))
    tariff_row = cur.fetchone()
    if not tariff_row:
        print("Тариф не найден.")
        return
    te = TariffEngine(tariff_row)

    vehicle_multiplier = float(veh["base_cost_multiplier"])

    details = te.calc(distance_km=distance_km, weight_kg=weight_kg, volume_m3=volume_m3,
                      is_dangerous=is_dangerous, is_urgent=is_urgent,
                      vehicle_multiplier=vehicle_multiplier, discount_percent=discount_percent)

    # Вывод
    print("="*60)
    print(f"Котировка: {from_city} -> {to_city}")
    print("Маршрут (по ID городов):", " -> ".join(str(x) for x in path_ids))
    print("Маршрут (по именам):", " -> ".join(get_city_name_by_id(x) for x in path_ids))
    print(f"Общая дистанция: {distance_km:.2f} km")
    print("Транспорт:", veh["name"], f"(mult={vehicle_multiplier})")
    print("Тариф:", te.name)
    print("- Детали расчёта -")
    for k in ["base_part","distance_part","weight_part","volume_part","subtotal",
              "multiplier","after_multiplier","fuel_surcharge","total_before_discount",
              "discount_percent","discount_amount","total"]:
        v = details[k]
        if isinstance(v, float):
            print(f"{k:22s}: {v:8.2f} €" if "percent" not in k else f"{k:22s}: {v:8.2f}")
        else:
            print(f"{k:22s}: {v}")
    print("="*60)
    conn.close()
    return details

# ---------------------------
# CSV import/export
# ---------------------------
def export_routes_csv(path="routes_export.csv"):
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        SELECT c1.name as from_city, c2.name as to_city, r.distance_km
        FROM roads r JOIN cities c1 ON r.from_city=c1.id JOIN cities c2 ON r.to_city=c2.id
    """)
    rows = cur.fetchall()
    conn.close()
    with open(path, "w", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        writer.writerow(["from_city","to_city","distance_km"])
        for r in rows:
            writer.writerow([r["from_city"], r["to_city"], r["distance_km"]])
    print("Экспорт выполнен в", path)

def import_routes_csv(path="routes_import.csv"):
    if not os.path.exists(path):
        print("Файл не найден:", path)
        return
    conn = get_conn()
    cur = conn.cursor()
    with open(path, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            a = row["from_city"].strip(); b = row["to_city"].strip(); d = float(row["distance_km"])
            # ensure cities exist
            cur.execute("INSERT OR IGNORE INTO cities (name) VALUES (?)", (a,))
            cur.execute("INSERT OR IGNORE INTO cities (name) VALUES (?)", (b,))
            conn.commit()
            cur.execute("SELECT id FROM cities WHERE name=?", (a,))
            ai = cur.fetchone()["id"]
            cur.execute("SELECT id FROM cities WHERE name=?", (b,))
            bi = cur.fetchone()["id"]
            cur.execute("INSERT OR IGNORE INTO roads (from_city,to_city,distance_km) VALUES (?,?,?)", (ai,bi,d))
    conn.commit()
    conn.close()
    print("Импорт завершен.")

# ---------------------------
# Меню
# ---------------------------
def menu():
    init_db()  # создаёт БД если не существует
    while True:
        print("\n=== Автомобильные перевозки — меню ===")
        print("1. Список городов")
        print("2. Список дорог")
        print("3. Список транспортных средств")
        print("4. Список тарифов")
        print("5. Котировка маршрута")
        print("6. Экспорт дорог в CSV")
        print("7. Импорт дорог из CSV")
        print("8. Добавить город")
        print("9. Добавить дорогу")
        print("0. Выход")
        choice = input("Выбери опцию: ").strip()
        if choice == "1":
            list_cities()
        elif choice == "2":
            list_roads()
        elif choice == "3":
            list_vehicles()
        elif choice == "4":
            list_tariffs()
        elif choice == "5":
            from_city = input("Откуда (имя города): ").strip()
            to_city = input("Куда (имя города): ").strip()
            weight = float(input("Вес грузa (кг): ").strip())
            volume = float(input("Объём (м3): ").strip())
            print("Выбери транспорт по id:")
            list_vehicles()
            vehicle_id = int(input("id транспортного средства: ").strip())
            print("Выбери тариф по id:")
            list_tariffs()
            tariff_id = int(input("id тарифа: ").strip())
            is_dangerous = input("Опасный груз? (y/N): ").strip().lower() == "y"
            is_urgent = input("Срочно? (y/N): ").strip().lower() == "y"
            discount = float(input("Скидка % (0 если нет): ").strip() or 0.0)
            quote_route(from_city, to_city, weight, volume, vehicle_id, tariff_id, is_dangerous, is_urgent, discount)
        elif choice == "6":
            path = input("Путь для CSV (routes_export.csv): ").strip() or "routes_export.csv"
            export_routes_csv(path)
        elif choice == "7":
            path = input("CSV файл для импорта (routes_import.csv): ").strip() or "routes_import.csv"
            import_routes_csv(path)
        elif choice == "8":
            name = input("Имя города: ").strip()
            conn = get_conn()
            cur = conn.cursor()
            cur.execute("INSERT OR IGNORE INTO cities (name) VALUES (?)", (name,))
            conn.commit(); conn.close()
            print("Добавлено (или уже существует).")
        elif choice == "9":
            from_name = input("Откуда (имя города): ").strip()
            to_name = input("Куда (имя города): ").strip()
            dist = float(input("Дистанция (км): ").strip())
            conn = get_conn(); cur = conn.cursor()
            # ensure cities exist
            cur.execute("INSERT OR IGNORE INTO cities (name) VALUES (?)", (from_name,))
            cur.execute("INSERT OR IGNORE INTO cities (name) VALUES (?)", (to_name,))
            conn.commit()
            cur.execute("SELECT id FROM cities WHERE name=?", (from_name,)); ai = cur.fetchone()["id"]
            cur.execute("SELECT id FROM cities WHERE name=?", (to_name,)); bi = cur.fetchone()["id"]
            cur.execute("INSERT OR REPLACE INTO roads (from_city,to_city,distance_km) VALUES (?,?,?)", (ai,bi,dist))
            conn.commit(); conn.close()
            print("Дорога добавлена/обновлена.")
        elif choice == "0":
            print("Пока!")
            break
        else:
            print("Неправильный выбор, попробуй ещё.")

if __name__ == "__main__":
    menu()
