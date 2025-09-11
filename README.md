Конечно! Вот простой и понятный пример программы на C# в Visual Studio, которая:

- запрашивает у пользователя координаты трех вершин треугольника,
- вычисляет площадь и периметр,
- выводит результат на экран.

using System;

class Triangle
{
    static void Main()
    {
        // Ввод координат вершин
        Console.WriteLine("Введите координаты 3 вершин треугольника:");

        Console.Write("x1 = ");
        double x1 = Convert.ToDouble(Console.ReadLine());
        Console.Write("y1 = ");
        double y1 = Convert.ToDouble(Console.ReadLine());

        Console.Write("x2 = ");
        double x2 = Convert.ToDouble(Console.ReadLine());
        Console.Write("y2 = ");
        double y2 = Convert.ToDouble(Console.ReadLine());

        Console.Write("x3 = ");
        double x3 = Convert.ToDouble(Console.ReadLine());
        Console.Write("y3 = ");
        double y3 = Convert.ToDouble(Console.ReadLine());

        // Функция для вычисления длины стороны между двумя точками
        double Distance(double xa, double ya, double xb, double yb)
        {
            return Math.Sqrt(Math.Pow(xa - xb, 2) + Math.Pow(ya - yb, 2));
        }

        // Длины сторон
        double a = Distance(x1, y1, x2, y2);
        double b = Distance(x2, y2, x3, y3);
        double c = Distance(x3, y3, x1, y1);

        // Периметр
        double perimeter = a + b + c;

        // Полупериметр для формулы Герона
        double s = perimeter / 2;

        // Площадь по формуле Герона
        double area = Math.Sqrt(s * (s - a) * (s - b) * (s - c));

        // Вывод результата
        Console.WriteLine($"Периметр треугольника: {perimeter:F2}");
        Console.WriteLine($"Площадь треугольника: {area:F2}");
    }
}


Как использовать:

1. Создайте новый проект консольного приложения в Visual Studio.
2. Вставьте этот код в основной файл Program.cs.
3. Запустите программу, вводите координаты поочередно.
4. Получите результат на экране.

Если хотите, могу помочь оформить программу с валидацией ввода или интерфейсом Windows Forms. Нужно?
