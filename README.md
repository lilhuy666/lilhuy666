Traceback (most recent call last):
  File "C:\Users\Кирилл\PycharmProjects\PythonProject9\main.py", line 45, in <module>
    init()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject9\main.py", line 24, in init
    conn = get_db()
           ^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject9\main.py", line 19, in get_db
    return psycopg2.connect(**DB)
                              ^^
NameError: name 'DB' is not defined
