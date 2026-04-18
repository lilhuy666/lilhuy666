import sys
import json
from datetime import datetime
from PyQt6.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QHBoxLayout,
    QLabel, QLineEdit, QPushButton, QListWidget, QMessageBox, QStackedWidget
)
from PyQt6.QtCore import Qt

DATA_FILE = "users.json"


def load_data():
    try:
        with open(DATA_FILE, "r") as f:
            return json.load(f)
    except:
        return {}


def save_data(data):
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=4)


class AuthScreen(QWidget):
    def __init__(self, app):
        super().__init__()
        self.app = app

        layout = QVBoxLayout()

        self.login_input = QLineEdit()
        self.login_input.setPlaceholderText("Логин")

        self.password_input = QLineEdit()
        self.password_input.setPlaceholderText("Пароль")
        self.password_input.setEchoMode(QLineEdit.EchoMode.Password)

        self.login_btn = QPushButton("Войти")
        self.register_btn = QPushButton("Регистрация")

        self.login_btn.clicked.connect(self.login)
        self.register_btn.clicked.connect(self.register)

        layout.addWidget(QLabel("Fuel Calculator"))
        layout.addWidget(self.login_input)
        layout.addWidget(self.password_input)
        layout.addWidget(self.login_btn)
        layout.addWidget(self.register_btn)

        self.setLayout(layout)

    def login(self):
        data = load_data()
        user = self.login_input.text()
        pwd = self.password_input.text()

        if user in data and data[user]["password"] == pwd:
            self.app.current_user = user
            self.app.load_main()
        else:
            QMessageBox.warning(self, "Ошибка", "Неверные данные")

    def register(self):
        data = load_data()
        user = self.login_input.text()
        pwd = self.password_input.text()

        if user in data:
            QMessageBox.warning(self, "Ошибка", "Пользователь уже есть")
            return

        data[user] = {
            "password": pwd,
            "history": []
        }
        save_data(data)
        QMessageBox.information(self, "Успех", "Регистрация успешна")


class MainScreen(QWidget):
    def __init__(self, app):
        super().__init__()
        self.app = app

        layout = QVBoxLayout()

        self.distance = QLineEdit()
        self.distance.setPlaceholderText("Км")

        self.fuel = QLineEdit()
        self.fuel.setPlaceholderText("Литры")

        self.price = QLineEdit()
        self.price.setPlaceholderText("Цена за литр")

        self.calc_btn = QPushButton("Рассчитать")
        self.calc_btn.clicked.connect(self.calculate)

        self.result = QLabel("Результат появится здесь")

        self.history = QListWidget()

        layout.addWidget(QLabel(f"Пользователь: {self.app.current_user}"))
        layout.addWidget(self.distance)
        layout.addWidget(self.fuel)
        layout.addWidget(self.price)
        layout.addWidget(self.calc_btn)
        layout.addWidget(self.result)
        layout.addWidget(QLabel("История"))
        layout.addWidget(self.history)

        self.setLayout(layout)
        self.load_history()

    def load_history(self):
        self.history.clear()
        data = load_data()
        user_data = data[self.app.current_user]["history"]

        for item in user_data[::-1]:
            self.history.addItem(item)

    def calculate(self):
        try:
            dist = float(self.distance.text())
            fuel = float(self.fuel.text())
            price = float(self.price.text())

            consumption = (fuel / dist) * 100
            total_cost = fuel * price

            result_text = f"{consumption:.2f} л/100км | {total_cost:.2f} ₽"
            self.result.setText(result_text)

            # сохраняем историю
            data = load_data()
            entry = f"{datetime.now().strftime('%d.%m %H:%M')} - {result_text}"
            data[self.app.current_user]["history"].append(entry)
            save_data(data)

            self.load_history()

        except:
            QMessageBox.warning(self, "Ошибка", "Проверьте ввод")


class App(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Fuel Calculator")
        self.setFixedSize(400, 500)

        self.current_user = None

        self.stack = QStackedWidget()
        self.auth = AuthScreen(self)
        self.stack.addWidget(self.auth)

        layout = QVBoxLayout()
        layout.addWidget(self.stack)
        self.setLayout(layout)

        self.setStyleSheet("""
            QWidget {
                background-color: #121212;
                color: white;
                font-size: 14px;
            }
            QLineEdit {
                padding: 8px;
                border: 1px solid #333;
                border-radius: 6px;
                background: #1e1e1e;
            }
            QPushButton {
                padding: 10px;
                background-color: #4CAF50;
                border: none;
                border-radius: 6px;
            }
            QPushButton:hover {
                background-color: #45a049;
            }
        """)

    def load_main(self):
        self.main = MainScreen(self)
        self.stack.addWidget(self.main)
        self.stack.setCurrentWidget(self.main)


if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = App()
    window.show()
    sys.exit(app.exec())
