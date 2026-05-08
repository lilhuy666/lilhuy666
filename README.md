import json
import os
import hashlib
import datetime
from tkinter import *
from tkinter import ttk, messagebox, simpledialog
from tkcalendar import DateEntry
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

# ======================== СИСТЕМА ПОЛЬЗОВАТЕЛЕЙ ========================
DATA_FILE = "fuelmaster_data.json"

class UserManager:
    def __init__(self):
        self.users = {}
        self.current_user = None
        self.load_data()
    
    def load_data(self):
        if os.path.exists(DATA_FILE):
            try:
                with open(DATA_FILE, "r", encoding="utf-8") as f:
                    data = json.load(f)
                    self.users = data.get("users", {})
            except:
                self.users = {}
    
    def save_data(self):
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump({"users": self.users}, f, indent=4, ensure_ascii=False)
    
    def hash_password(self, password):
        return hashlib.sha256(password.encode()).hexdigest()
    
    def register(self, username, password, vehicle_model="Не указано"):
        if username in self.users:
            return False, "Пользователь уже существует!"
        if len(password) < 4:
            return False, "Пароль должен быть минимум 4 символа!"
        self.users[username] = {
            "password": self.hash_password(password),
            "vehicle_model": vehicle_model,
            "history": [],
            "created_at": str(datetime.datetime.now())
        }
        self.save_data()
        return True, "Регистрация успешна!"
    
    def login(self, username, password):
        if username not in self.users:
            return False, "Пользователь не найден!"
        if self.users[username]["password"] != self.hash_password(password):
            return False, "Неверный пароль!"
        self.current_user = username
        return True, f"Добро пожаловать, {username}!"
    
    def logout(self):
        self.current_user = None
    
    def add_calculation(self, username, data):
        if username in self.users:
            self.users[username]["history"].append(data)
            self.save_data()
    
    def get_user_history(self, username):
        return self.users.get(username, {}).get("history", [])
    
    def update_vehicle(self, username, model):
        if username in self.users:
            self.users[username]["vehicle_model"] = model
            self.save_data()

# ======================== ГЛАВНОЕ ПРИЛОЖЕНИЕ ========================
class FuelMasterApp:
    def __init__(self, root):
        self.root = root
        self.root.title("FuelMaster Pro - Супер Калькулятор Расхода Топлива")
        self.root.geometry("1200x700")
        self.root.configure(bg="#1e1e2f")
        self.user_manager = UserManager()
        
        # Стили
        self.style = ttk.Style()
        self.style.theme_use("clam")
        self.style.configure("TLabel", background="#1e1e2f", foreground="#ffffff", font=("Segoe UI", 11))
        self.style.configure("TButton", font=("Segoe UI", 11, "bold"), padding=8)
        self.style.configure("TEntry", font=("Segoe UI", 11), padding=5)
        self.style.configure("TFrame", background="#1e1e2f")
        
        self.show_login_screen()
    
    def clear_window(self):
        for widget in self.root.winfo_children():
            widget.destroy()
    
    def show_login_screen(self):
        self.clear_window()
        
        main_frame = Frame(self.root, bg="#1e1e2f")
        main_frame.pack(expand=True)
        
        Label(main_frame, text="🚗 FuelMaster Pro", font=("Segoe UI", 32, "bold"), bg="#1e1e2f", fg="#ffaa44").pack(pady=30)
        Label(main_frame, text="Ваш персональный помощник по топливу", font=("Segoe UI", 14), bg="#1e1e2f", fg="#cccccc").pack(pady=5)
        
        frame = Frame(main_frame, bg="#2a2a3b", padx=30, pady=30, relief=GROOVE, bd=2)
        frame.pack(pady=30)
        
        Label(frame, text="Логин", bg="#2a2a3b", fg="white", font=("Segoe UI", 12)).grid(row=0, column=0, pady=10, padx=10, sticky="e")
        self.login_entry = Entry(frame, font=("Segoe UI", 12), width=20, bg="#3a3a4f", fg="white", insertbackground="white")
        self.login_entry.grid(row=0, column=1, pady=10, padx=10)
        
        Label(frame, text="Пароль", bg="#2a2a3b", fg="white", font=("Segoe UI", 12)).grid(row=1, column=0, pady=10, padx=10, sticky="e")
        self.pass_entry = Entry(frame, font=("Segoe UI", 12), width=20, show="*", bg="#3a3a4f", fg="white", insertbackground="white")
        self.pass_entry.grid(row=1, column=1, pady=10, padx=10)
        
        btn_frame = Frame(frame, bg="#2a2a3b")
        btn_frame.grid(row=2, column=0, columnspan=2, pady=20)
        
        Button(btn_frame, text="Вход", command=self.login, bg="#2ecc71", fg="white", font=("Segoe UI", 11, "bold"), padx=20, pady=5).pack(side=LEFT, padx=10)
        Button(btn_frame, text="Регистрация", command=self.show_register, bg="#3498db", fg="white", font=("Segoe UI", 11, "bold"), padx=20, pady=5).pack(side=LEFT, padx=10)
    
    def show_register(self):
        reg_win = Toplevel(self.root)
        reg_win.title("Регистрация")
        reg_win.geometry("400x350")
        reg_win.configure(bg="#2a2a3b")
        reg_win.grab_set()
        
        Label(reg_win, text="Регистрация нового аккаунта", font=("Segoe UI", 16, "bold"), bg="#2a2a3b", fg="#ffaa44").pack(pady=20)
        
        frame = Frame(reg_win, bg="#2a2a3b")
        frame.pack(pady=10)
        
        Label(frame, text="Логин:", bg="#2a2a3b", fg="white", font=("Segoe UI", 11)).grid(row=0, column=0, pady=10, padx=10, sticky="e")
        reg_login = Entry(frame, font=("Segoe UI", 11), width=20, bg="#3a3a4f", fg="white")
        reg_login.grid(row=0, column=1, pady=10, padx=10)
        
        Label(frame, text="Пароль:", bg="#2a2a3b", fg="white", font=("Segoe UI", 11)).grid(row=1, column=0, pady=10, padx=10, sticky="e")
        reg_pass = Entry(frame, font=("Segoe UI", 11), width=20, show="*", bg="#3a3a4f", fg="white")
        reg_pass.grid(row=1, column=1, pady=10, padx=10)
        
        Label(frame, text="Модель авто:", bg="#2a2a3b", fg="white", font=("Segoe UI", 11)).grid(row=2, column=0, pady=10, padx=10, sticky="e")
        reg_vehicle = Entry(frame, font=("Segoe UI", 11), width=20, bg="#3a3a4f", fg="white")
        reg_vehicle.grid(row=2, column=1, pady=10, padx=10)
        
        def do_register():
            res, msg = self.user_manager.register(reg_login.get(), reg_pass.get(), reg_vehicle.get())
            messagebox.showinfo("Результат", msg)
            if res:
                reg_win.destroy()
        
        Button(reg_win, text="Зарегистрироваться", command=do_register, bg="#2ecc71", fg="white", font=("Segoe UI", 11, "bold"), padx=20).pack(pady=20)
    
    def login(self):
        res, msg = self.user_manager.login(self.login_entry.get(), self.pass_entry.get())
        if res:
            self.show_main_menu()
        else:
            messagebox.showerror("Ошибка", msg)
    
    def show_main_menu(self):
        self.clear_window()
        
        # Верхняя панель
        top_bar = Frame(self.root, bg="#2a2a3b", height=60)
        top_bar.pack(fill=X)
        Label(top_bar, text=f"👤 {self.user_manager.current_user}", font=("Segoe UI", 12), bg="#2a2a3b", fg="#ffaa44").pack(side=LEFT, padx=20, pady=15)
        Button(top_bar, text="Выйти", command=self.logout, bg="#e74c3c", fg="white", font=("Segoe UI", 10), padx=15).pack(side=RIGHT, padx=20, pady=10)
        Button(top_bar, text="Профиль", command=self.show_profile, bg="#9b59b6", fg="white", font=("Segoe UI", 10), padx=15).pack(side=RIGHT, padx=10, pady=10)
        
        # Основной контент
        notebook = ttk.Notebook(self.root)
        notebook.pack(fill=BOTH, expand=True, padx=10, pady=10)
        
        # Вкладка калькулятора
        self.calc_tab = Frame(notebook, bg="#1e1e2f")
        notebook.add(self.calc_tab, text="📊 Калькулятор")
        self.setup_calculator()
        
        # Вкладка истории
        self.history_tab = Frame(notebook, bg="#1e1e2f")
        notebook.add(self.history_tab, text="📜 История")
        self.setup_history()
        
        # Вкладка аналитики
        self.analytics_tab = Frame(notebook, bg="#1e1e2f")
        notebook.add(self.analytics_tab, text="📈 Аналитика")
        self.setup_analytics()
        
        # Вкладка советов
        self.tips_tab = Frame(notebook, bg="#1e1e2f")
        notebook.add(self.tips_tab, text="💡 Эко-советы")
        self.setup_tips()
    
    def setup_calculator(self):
        calc_frame = Frame(self.calc_tab, bg="#1e1e2f", padx=30, pady=30)
        calc_frame.pack(expand=True)
        
        Label(calc_frame, text="✨ Калькулятор расхода топлива ✨", font=("Segoe UI", 24, "bold"), bg="#1e1e2f", fg="#ffaa44").pack(pady=20)
        
        fields = [
            ("📏 Пройденное расстояние (км):", "distance"),
            ("⛽ Израсходовано топлива (л):", "fuel"),
            ("💰 Цена за литр (руб):", "price")
        ]
        
        self.calc_entries = {}
        for text, key in fields:
            frame = Frame(calc_frame, bg="#1e1e2f")
            frame.pack(pady=10)
            Label(frame, text=text, font=("Segoe UI", 12), bg="#1e1e2f", fg="white", width=25, anchor="e").pack(side=LEFT, padx=10)
            entry = Entry(frame, font=("Segoe UI", 12), width=15, bg="#3a3a4f", fg="white", insertbackground="white")
            entry.pack(side=LEFT, padx=10)
            self.calc_entries[key] = entry
        
        self.result_label = Label(calc_frame, text="", font=("Segoe UI", 14), bg="#1e1e2f", fg="#2ecc71")
        self.result_label.pack(pady=20)
        
        Button(calc_frame, text="🚀 РАССЧИТАТЬ", command=self.calculate, bg="#f39c12", fg="white", font=("Segoe UI", 14, "bold"), padx=30, pady=10).pack(pady=20)
    
    def calculate(self):
        try:
            dist = float(self.calc_entries["distance"].get())
            fuel = float(self.calc_entries["fuel"].get())
            price = float(self.calc_entries["price"].get())
            
            if dist <= 0 or fuel <= 0 or price <= 0:
                raise ValueError
            
            consumption = (fuel / dist) * 100
            cost_per_km = (fuel * price) / dist
            total_cost = fuel * price
            
            result_text = f"📊 Расход: {consumption:.2f} л/100км\n💰 Стоимость км: {cost_per_km:.2f} руб\n💸 Общая стоимость: {total_cost:.2f} руб"
            self.result_label.config(text=result_text)
            
            # Сохраняем в историю
            history_entry = {
                "date": str(datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")),
                "distance": dist,
                "fuel": fuel,
                "price": price,
                "consumption": round(consumption, 2),
                "cost_per_km": round(cost_per_km, 2),
                "total_cost": round(total_cost, 2)
            }
            self.user_manager.add_calculation(self.user_manager.current_user, history_entry)
            self.refresh_history()
            self.refresh_analytics()
            messagebox.showinfo("Успех", "Расчёт сохранён в истории!")
        except:
            messagebox.showerror("Ошибка", "Введите корректные положительные числа!")
    
    def setup_history(self):
        self.history_tree = ttk.Treeview(self.history_tab, columns=("date", "dist", "fuel", "consumption", "total"), show="headings", height=20)
        self.history_tree.heading("date", text="Дата")
        self.history_tree.heading("dist", text="Км")
        self.history_tree.heading("fuel", text="Литры")
        self.history_tree.heading("consumption", text="л/100км")
        self.history_tree.heading("total", text="Стоимость (руб)")
        
        self.history_tree.column("date", width=180)
        self.history_tree.column("dist", width=100)
        self.history_tree.column("fuel", width=100)
        self.history_tree.column("consumption", width=120)
        self.history_tree.column("total", width=150)
        
        scrollbar = Scrollbar(self.history_tab, orient=VERTICAL, command=self.history_tree.yview)
        self.history_tree.configure(yscrollcommand=scrollbar.set)
        
        self.history_tree.pack(side=LEFT, fill=BOTH, expand=True, padx=10, pady=10)
        scrollbar.pack(side=RIGHT, fill=Y)
        
        Button(self.history_tab, text="🗑 Очистить историю", command=self.clear_history, bg="#e74c3c", fg="white", font=("Segoe UI", 10)).pack(pady=10)
        
        self.refresh_history()
    
    def refresh_history(self):
        for item in self.history_tree.get_children():
            self.history_tree.delete(item)
        history = self.user_manager.get_user_history(self.user_manager.current_user)
        for entry in reversed(history):
            self.history_tree.insert("", 0, values=(
                entry["date"],
                entry["distance"],
                entry["fuel"],
                entry["consumption"],
                entry["total_cost"]
            ))
    
    def clear_history(self):
        if messagebox.askyesno("Подтверждение", "Удалить всю историю расчётов?"):
            self.user_manager.users[self.user_manager.current_user]["history"] = []
            self.user_manager.save_data()
            self.refresh_history()
            self.refresh_analytics()
            messagebox.showinfo("Успех", "История очищена!")
    
    def setup_analytics(self):
        self.analytics_frame = Frame(self.analytics_tab, bg="#1e1e2f")
        self.analytics_frame.pack(fill=BOTH, expand=True)
        self.refresh_analytics()
    
    def refresh_analytics(self):
        for widget in self.analytics_frame.winfo_children():
            widget.destroy()
        
        history = self.user_manager.get_user_history(self.user_manager.current_user)
        if len(history) == 0:
            Label(self.analytics_frame, text="Нет данных для аналитики. Выполните расчёты!", font=("Segoe UI", 16), bg="#1e1e2f", fg="white").pack(expand=True)
            return
        
        fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(10, 4))
        fig.patch.set_facecolor("#1e1e2f")
        
        dates = [h["date"][:10] for h in history[-10:]]
        consumptions = [h["consumption"] for h in history[-10:]]
        
        ax1.plot(dates, consumptions, marker='o', color='#ffaa44', linewidth=2)
        ax1.set_title("Динамика расхода топлива", color="white")
        ax1.set_ylabel("л/100км", color="white")
        ax1.tick_params(colors="white")
        ax1.set_facecolor("#2a2a3b")
        for spine in ax1.spines.values():
            spine.set_color("white")
        
        costs = [h["total_cost"] for h in history[-10:]]
        ax2.bar(dates, costs, color='#2ecc71')
        ax2.set_title("Стоимость поездок", color="white")
        ax2.set_ylabel("Руб.", color="white")
        ax2.tick_params(colors="white")
        ax2.set_facecolor("#2a2a3b")
        for spine in ax2.spines.values():
            spine.set_color("white")
        
        canvas = FigureCanvasTkAgg(fig, master=self.analytics_frame)
        canvas.draw()
        canvas.get_tk_widget().pack(expand=True)
        
        avg_cons = sum(h["consumption"] for h in history) / len(history)
        total_spent = sum(h["total_cost"] for h in history)
        Label(self.analytics_frame, text=f"📊 Средний расход: {avg_cons:.2f} л/100км | 💰 Всего потрачено: {total_spent:.2f} руб", 
              font=("Segoe UI", 14), bg="#1e1e2f", fg="#ffaa44").pack(pady=10)
    
    def setup_tips(self):
        tips = [
            "🔧 Поддерживайте оптимальное давление в шинах - экономия до 5% топлива.",
            "🚗 Избегайте резких ускорений и торможений.",
            "📦 Уберите лишний вес из багажника - каждые 50 кг увеличивают расход на 2%.",
            "🌡 Используйте кондиционер умеренно - на скорости выше 80 км/ч лучше открыть окна.",
            "⛽ Заправляйтесь на проверенных АЗС с качественным топливом.",
            "🛠 Регулярно меняйте масло и воздушный фильтр.",
            "📈 Планируйте маршрут заранее, избегая пробок.",
            "⚡ На коротких поездках двигатель не успевает прогреться - расход выше на 20%."
        ]
        
        frame = Frame(self.tips_tab, bg="#1e1e2f")
        frame.pack(expand=True, fill=BOTH, padx=50, pady=50)
        
        Label(frame, text="💚 Эко-советы для экономии топлива 💚", font=("Segoe UI", 22, "bold"), bg="#1e1e2f", fg="#2ecc71").pack(pady=30)
        
        for tip in tips:
            Label(frame, text=tip, font=("Segoe UI", 12), bg="#1e1e2f", fg="#cccccc", anchor="w", justify=LEFT).pack(anchor="w", pady=8)
    
    def show_profile(self):
        user_data = self.user_manager.users[self.user_manager.current_user]
        profile_win = Toplevel(self.root)
        profile_win.title("Профиль пользователя")
        profile_win.geometry("450x350")
        profile_win.configure(bg="#2a2a3b")
        
        Label(profile_win, text="👤 Мой профиль", font=("Segoe UI", 20, "bold"), bg="#2a2a3b", fg="#ffaa44").pack(pady=20)
        
        frame = Frame(profile_win, bg="#2a2a3b")
        frame.pack(pady=20)
        
        Label(frame, text=f"Логин: {self.user_manager.current_user}", font=("Segoe UI", 12), bg="#2a2a3b", fg="white").pack(anchor="w", pady=5)
        Label(frame, text=f"Модель авто: {user_data['vehicle_model']}", font=("Segoe UI", 12), bg="#2a2a3b", fg="white").pack(anchor="w", pady=5)
        Label(frame, text=f"Дата регистрации: {user_data['created_at']}", font=("Segoe UI", 10), bg="#2a2a3b", fg="#aaaaaa").pack(anchor="w", pady=5)
        Label(frame, text=f"Всего расчётов: {len(user_data['history'])}", font=("Segoe UI", 12), bg="#2a2a3b", fg="#2ecc71").pack(anchor="w", pady=10)
        
        def change_vehicle():
            new_model = simpledialog.askstring("Смена авто", "Введите новую модель автомобиля:", parent=profile_win)
            if new_model:
                self.user_manager.update_vehicle(self.user_manager.current_user, new_model)
                messagebox.showinfo("Успех", "Модель авто обновлена!")
                profile_win.destroy()
                self.show_profile()
        
        Button(profile_win, text="✏️ Изменить модель авто", command=change_vehicle, bg="#3498db", fg="white", font=("Segoe UI", 10), padx=20).pack(pady=10)
        Button(profile_win, text="Закрыть", command=profile_win.destroy, bg="#95a5a6", fg="white", padx=20).pack(pady=10)
    
    def logout(self):
        self.user_manager.logout()
        self.show_login_screen()

# ======================== ЗАПУСК ========================
if __name__ == "__main__":
    root = Tk()
    app = FuelMasterApp(root)
    root.mainloop()
