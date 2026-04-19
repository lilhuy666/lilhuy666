Exception in Tkinter callback
Traceback (most recent call last):
  File "C:\Users\Кирилл\AppData\Local\Programs\Python\Python312\Lib\tkinter\__init__.py", line 1968, in __call__
    return self.func(*args)
           ^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject5\main.py", line 130, in register
    if e in data["users"]:
            ~~~~^^^^^^^^^
KeyError: 'users'
Exception in Tkinter callback
Traceback (most recent call last):
  File "C:\Users\Кирилл\AppData\Local\Programs\Python\Python312\Lib\tkinter\__init__.py", line 1968, in __call__
    return self.func(*args)
           ^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject5\main.py", line 116, in login
    if e in data["users"] and data["users"][e]["password"] == p:
            ~~~~^^^^^^^^^
KeyError: 'users'
