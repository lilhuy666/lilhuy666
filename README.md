import sys
from PyQt5.QtWidgets import *
from PyQt5.QtCore import Qt
from PyQt5.QtGui import QFont

# ===============================
# Данные
# ===============================
history = []

# ===============================
# Главное окно
# ===============================
class FuelApp(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("CalculatCar")
        self.setGeometry(100, 100, 1200, 750)

        self.setStyleSheet("""
            QWidget {
                background: #eef2f7;
                font-family: Arial;
            }
            QFrame#card {
                background: white;
                border-radius: 12px;
                padding: 15px;
            }
            QPushButton {
                background: #3b82f6;
                color: white;
                border-radius: 8px;
                padding: 10px;
            }
            QPushButton:hover {
                background: #2563eb;
            }
        """)

        self.initUI()

    # ===============================
    # UI
    # ===============================
    def initUI(self):
        main = QWidget()
        self.setCentralWidget(main)

        layout = QHBoxLayout(main)

        # ===============================
        # SIDEBAR
        # ===============================
        sidebar = QFrame()
        sidebar.setFixedWidth(220)
        sidebar.setStyleSheet("background:white;")
        side_layout = QVBoxLayout(sidebar)

        side_layout.addWidget(QLabel("👤 Профиль"))
        side_layout.addSpacing(20)

        btn1 = QPushButton("🧮 Калькулятор")
        btn2 = QPushButton("🕒 История")
        btn3 = QPushButton("ℹ️ О нас")

        side_layout.addWidget(btn1)
        side_layout.addWidget(btn2)
        side_layout.addWidget(btn3)
        side_layout.addStretch()

        layout.addWidget(sidebar)

        # ===============================
        # MAIN
        # ===============================
        content = QVBoxLayout()

        title = QLabel("Калькулятор расхода топлива")
        title.setFont(QFont("Arial", 18, QFont.Bold))
        content.addWidget(title)

        grid = QHBoxLayout()

        # ===============================
        # Левая карточка
        # ===============================
        left = QFrame(objectName="card")
        left_layout = QVBoxLayout(left)

        self.distance = QLineEdit("150")
        self.fuel = QLineEdit("12")
        self.price = QLineEdit("48.5")

        left_layout.addWidget(QLabel("Дистанция (км):"))
        left_layout.addWidget(self.distance)

        left_layout.addWidget(QLabel("Кол-во топлива (л):"))
        left_layout.addWidget(self.fuel)

        left_layout.addWidget(QLabel("Цена топлива (₽/л):"))
        left_layout.addWidget(self.price)

        calc_btn = QPushButton("Рассчитать")
        calc_btn.clicked.connect(self.calculate)
        left_layout.addWidget(calc_btn)

        grid.addWidget(left)

        # ===============================
        # Результаты
        # ===============================
        right = QFrame(objectName="card")
        right_layout = QVBoxLayout(right)

        self.result = QLabel("0.0 л/100 км")
        self.result.setFont(QFont("Arial", 16, QFont.Bold))

        self.cost = QLabel("0 ₽")
        self.cost.setFont(QFont("Arial", 18, QFont.Bold))

        right_layout.addWidget(QLabel("Расход топлива"))
        right_layout.addWidget(self.result)

        right_layout.addSpacing(20)

        right_layout.addWidget(QLabel("Стоимость поездки"))
        right_layout.addWidget(self.cost)

        grid.addWidget(right)

        # ===============================
        # История
        # ===============================
        history_card = QFrame(objectName="card")
        hist_layout = QVBoxLayout(history_card)

        self.history_list = QListWidget()

        hist_layout.addWidget(QLabel("Последние расчёты"))
        hist_layout.addWidget(self.history_list)

        grid.addWidget(history_card)

        content.addLayout(grid)

        layout.addLayout(content)

    # ===============================
    # Расчет
    # ===============================
    def calculate(self):
        try:
            d = float(self.distance.text())
            f = float(self.fuel.text())
            p = float(self.price.text())

            cons = f / d * 100
            cost = f * p

            self.result.setText(f"{cons:.1f} л/100 км")
            self.cost.setText(f"{cost:.0f} ₽")

            record = f"{cons:.1f} л/100км — {cost:.0f} ₽"
            history.append(record)

            self.history_list.clear()
            for item in history[::-1][:5]:
                self.history_list.addItem(item)

        except:
            QMessageBox.warning(self, "Ошибка", "Проверь данные!")

# ===============================
# Запуск
# ===============================
app = QApplication(sys.argv)
window = FuelApp()
window.show()
sys.exit(app.exec_())
