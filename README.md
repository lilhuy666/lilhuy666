Вот простой и понятный пример программы на C# в Visual Studio, которая вычисляет площадь и периметр треугольника по координатам его вершин.

using System;

class Triangle
{
    static void Main()
    {
        // Ввод координат вершин
        Console.WriteLine("Введите координаты треугольника (x1, y1, x2, y2, x3, y3):");
        double x1 = double.Parse(Console.ReadLine());
        double y1 = double.Parse(Console.ReadLine());
        double x2 = double.Parse(Console.ReadLine());
        double y2 = double.Parse(Console.ReadLine());
        double x3 = double.Parse(Console.ReadLine());
        double y3 = double.Parse(Console.ReadLine());

        // Функция для вычисления длины стороны
        double Distance(double xA, double yA, double xB, double yB)
        {
            return Math.Sqrt((xB - xA) * (xB - xA) + (yB - yA) * (yB - yA));
        }

        double a = Distance(x1, y1, x2, y2);
        double b = Distance(x2, y2, x3, y3);
        double c = Distance(x3, y3, x1, y1);

        double perimeter = a + b + c;

        // Полупериметр (для формулы Герона)
        double s = perimeter / 2;

        // Площадь по формуле Герона
        double area = Math.Sqrt(s * (s - a) * (s - b) * (s - c));

        Console.WriteLine($"Периметр треугольника: {perimeter:F2}");
        Console.WriteLine($"Площадь треугольника: {area:F2}");
    }
}


Как пользоваться:

1. Создайте новый консольный проект в Visual Studio.
2. Вставьте этот код в Program.cs.
3. Запустите программу, введите поочерёдно координаты трёх вершин — каждый параметр с новой строки.
4. Программа выведет площадь и периметр треугольника.

Если надо, могу помочь добавить проверку на правильность введённых данных или оформить программу с использованием методов и классов.

Хотите, сделаем более структурированный вариант с методами?
