import sys
from PyQt5.QtWidgets import (QApplication, QMainWindow, QWidget, QVBoxLayout, 
                              QPushButton, QLabel, QLineEdit, QTableWidget, 
                              QMessageBox, QComboBox, QFormLayout)
import sqlite3 # Или ваш драйвер БД

# ----------------------------------------------------------
# 1. ГЛОБАЛЬНЫЕ ПЕРЕМЕННЫЕ
# ----------------------------------------------------------
current_user_role = None # 'guest', 'client', 'manager', 'admin'

# ----------------------------------------------------------
# 2. ОКНО АВТОРИЗАЦИИ (Модуль 2)
# ----------------------------------------------------------
class LoginWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle('Авторизация')
        layout = QVBoxLayout()

        self.login_input = QLineEdit(placeholderText='Логин')
        self.password_input = QLineEdit(placeholderText='Пароль')
        layout.addWidget(self.login_input)
        layout.addWidget(self.password_input)

        login_btn = QPushButton('Войти')
        login_btn.clicked.connect(self.try_login)
        layout.addWidget(login_btn)

        guest_btn = QPushButton('Войти как гость')
        guest_btn.clicked.connect(self.login_as_guest)
        layout.addWidget(guest_btn)

        self.setLayout(layout)

    def try_login(self):
        global current_user_role
        # ЗДЕСЬ БУДЕТ ВАШ КОД:
        # - Взять логин и пароль из self.login_input.text() и self.password_input.text()
        # - Сделать SQL-запрос к вашей таблице пользователей
        # - При совпадении определить роль и установить current_user_role
        # - Открыть окно списка товаров (MainWindow)
        QMessageBox.information(self, 'Заглушка', 'Здесь будет проверка логина и пароля')
        pass 

    def login_as_guest(self):
        global current_user_role
        current_user_role = 'guest'
        # ЗДЕСЬ БУДЕТ ВАШ КОД:
        # - Открыть окно списка товаров (MainWindow)
        QMessageBox.information(self, 'Заглушка', 'Вход как гость')
        pass

# ----------------------------------------------------------
# 3. ГЛАВНОЕ ОКНО СО СПИСКОМ ТОВАРОВ (Модуль 2 и 3)
# ----------------------------------------------------------
class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle('Список товаров')

        # Виджет для таблицы
        self.table = QTableWidget()
        
        # Кнопки, которые видны только Админу/Менеджеру
        self.search_input = QLineEdit(placeholderText='Поиск...')
        self.filter_combo = QComboBox()
        self.sort_btn = QPushButton('Сорт. по кол-ву')
        self.add_btn = QPushButton('Добавить товар')
        self.delete_btn = QPushButton('Удалить товар')

        # ЗДЕСЬ БУДЕТ ВАШ КОД:
        # - Настроить layout
        # - Подключить кнопки к функциям (connect)
        # - Вызвать функцию refresh_table() для загрузки данных

    def refresh_table(self, search_text='', filter_value=None):
        global current_user_role
        # ЗДЕСЬ БУДЕТ ВАШ КОД:
        # - Сделать SELECT-запрос к таблице товаров с учетом search_text и filter_value
        # - В запросе уже использовать JOIN для категорий, производителей, скидок...
        # - Заполнить self.table данными из запроса
        # - ПРИМЕНИТЬ ЦВЕТА СТРОК (Модуль 2):
        #   - Если скидка > 15%, цвет фона #2E8B57
        #   - Если количество = 0, цвет фона голубой
        #   - Цена: старая перечеркнута красным, новая черным (использовать QTableWidgetItem)
        pass

    def delete_item(self):
        # ЗДЕСЬ БУДЕТ ВАШ КОД:
        # - Взять ID выбранного товара
        # - Проверить, есть ли этот ID в таблице заказов (Модуль 3)
        # - Если есть — показать QMessageBox с ошибкой
        # - Если нет — удалить и вызвать refresh_table()
        pass

# ----------------------------------------------------------
# 4. ЗАПУСК ПРИЛОЖЕНИЯ
# ----------------------------------------------------------
if __name__ == '__main__':
    app = QApplication(sys.argv)
    login = LoginWindow()
    login.show()
    sys.exit(app.exec_())
