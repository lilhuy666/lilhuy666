Traceback (most recent call last):
  File "C:\Users\Кирилл\PycharmProjects\PythonProject1\main.py", line 9, in <module>
    conn = psycopg2.connect(
           ^^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject1\.venv\Lib\site-packages\psycopg2\__init__.py", line 135, in connect
    conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
psycopg2.OperationalError: connection to server at "localhost" (::1), port 5432 failed: Connection refused (0x0000274D/10061)
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused (0x0000274D/10061)
	Is the server running on that host and accepting TCP/IP connections?
