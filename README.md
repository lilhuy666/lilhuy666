import os
from collections import defaultdict

# Чтение данных из файла
def read_data(file_path):
    students = []
    with open(file_path, 'r', encoding='utf-8') as file:
        for line in file:
            surname, name, class_num, subject, grade = line.strip().split()
            students.append((surname, name, int(class_num), subject, int(grade)))
    return students

# Создание структуры данных
def create_class_journal(students):
    class_journal = defaultdict(list)
    for surname, name, class_num, subject, grade in students:
        class_journal[class_num].append((surname, name, subject, grade))
    return class_journal

# Фильтрация по оценкам
def filter_by_grade(class_journal, grade):
    for class_num, students in class_journal.items():
        filtered_students = [s for s in students if s[3] == grade]
        if filtered_students:
            print(f"Class {class_num}:")
            for surname, name, subject, _ in filtered_students:
                print(f"{surname} {name}")

# Расчет средней успеваемости
def average_grades(class_journal):
    averages = {}
    for class_num, students in class_journal.items():
        total_grade = sum(s[3] for s in students)
        average = total_grade / len(students) if students else 0
        averages[class_num] = average
    return averages

# Сортировка классов по средней успеваемости
def sort_classes_by_average(averages):
    return sorted(averages.items(), key=lambda x: x[1], reverse=True)

# Сортировка по фамилии для заданного предмета
def sort_by_surname(class_journal, subject):
    all_students = []
    for students in class_journal.values():
        for student in students:
            if student[2] == subject:
                all_students.append(student)
    all_students.sort(key=lambda x: x[0])  # Сортировка по фамилии
    return all_students

# Запись ведомостей в файлы
def write_class_reports(class_journal):
    for class_num, students in class_journal.items():
        with open(f'class_{class_num}.txt', 'w', encoding='utf-8') as file:
            for surname, name, subject, grade in students:
                file.write(f"{surname} {name} {subject} {grade}\n")

# Поиск класса по имени и фамилии
def find_class_by_name(class_journal, surname, name):
    for class_num, students in class_journal.items():
        for student in students:
            if student[0] == surname and student[1] == name:
                return class_num
    return None

# Определение предмета с самой высокой средней успеваемостью
def subject_with_highest_average(students):
    subject_grades = defaultdict(list)
    for surname, name, class_num, subject, grade in students:
        subject_grades[subject].append(grade)
    
    averages = {subject: sum(grades) / len(grades) for subject, grades in subject_grades.items()}
    return max(averages.items(), key=lambda x: x[1])

# Основная функция
def main():
    file_path = 'school.txt'
    students = read_data(file_path)
    class_journal = create
