Для выполнения данной задачи вам потребуется написать программу, которая будет обрабатывать файл data school.txt, анализировать данные и выводить результаты согласно указанным требованиям. Обратите внимание на последовательность действий и логику, которая будет использована для создания программы.

Вот примерный план решения:

1. Чтение данных из файла
   - Откройте файл data school.txt и считайте данные построчно.
   - Разделите каждую строку на фамилию, имя, класс, предмет и оценку.

2. Структурирование данных
   - Создайте словари или списки для хранения информации о школьниках, классах и предметах:
     - Словарь для хранения классов и оценок.
     - Словарь для хранения оценок по каждому предмету.

3. Фильтрация и сортировка
   - Для каждого класса выведите список учеников с заданной оценкой.
   - Рассчитайте среднюю успеваемость в каждом классе и отсортируйте классы по этой средней успеваемости.
   - Для заданного предмета создайте список учеников всех классов, отсортированный по фамилии.

4. Вывод результатов
   - Выведите данные на консоль и сохраните их в отдельные файлы:
     - Ведомости класса по каждому предмету.
   - Создайте функцию для поиска класса по имени и фамилии ученика.
   - Определите предмет с самой высокой средней успеваемостью.

Пример кода на Python:

import csv
from collections import defaultdict

# Чтение данных из файла
students = []
with open('data school.txt', 'r') as file:
    reader = csv.reader(file, delimiter=' ')
    for row in reader:
        surname, name, class_num, subject, grade = row
        students.append((surname, name, int(class_num), subject, int(grade)))

# Структурирование данных
classes = defaultdict(list)
subjects = defaultdict(lambda: defaultdict(list))

for surname, name, class_num, subject, grade in students:
    classes[class_num].append((surname, name, grade))
    subjects[subject][class_num].append(grade)

# Функция для вывода учеников с заданной оценкой
def print_students_by_grade(class_num, target_grade):
    print(f"Class {class_num} with grade {target_grade}:")
    for surname, name, grade in classes[class_num]:
        if grade == target_grade:
            print(f"{surname} {name} - {grade}")

# Сортировка классов по средней успеваемости
average_scores = {}
for class_num, class_students in classes.items():
    if class_students:  # Проверяем, что есть ученики в классе
        avg = sum(grade for _, _, grade in class_students) / len(class_students)
        average_scores[class_num] = avg

sorted_classes = dict(sorted(average_scores.items(), key=lambda item: item[1]))

# Вывод учеников по предмету, отсортированный по фамилии
def print_students_by_subject(subject):
    all_students = [(surname, name, grade) for class_num in classes for surname, name, grade in classes[class_num] if subject in subjects and class_num in subjects[subject]]
    all_students.sort(key=lambda x: x[0])  # Сортировка по фамилии
    for surname, name, grade in all_students:
        print(f"{surname} {name} - {grade}")

# Поиск класса по имени и фамилии
def find_class_by_name(surname, name):
    for class_num, class_students in classes.items():
        for s, n, _ in class_students:
            if s == surname and n == name:
                return class_num
    return None

# Определение предмета с самой высокой средней успеваемостью
def get_best_subject():
    subject_averages = {}
    for subject, class_grades in subjects.items():
        total = 0
        count = 0
        for class_num, grades in class_grades.items():
            total += sum(grades)
            count += len(grades)
        subject_averages[subject] = total / count if count > 0 else 0
    
    best_subject = max(subject_averages, key=subject_averages.get)
    return best_subject

# Пример использования функций
print_students_by_grade(5, 4)  # Укажите класс и оценку
print_students_by_subject("Math")  # Укажите предмет
student_class = find_class_by_name("Иванов", "Иван")  # Укажите ФИО
best_subject = get_best_subject()
print(f"Best subject: {best_subject}")
