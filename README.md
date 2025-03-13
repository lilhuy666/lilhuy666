Вот пример реализации программы на Java, которая выполняет заданные действия с данными о школьниках. Перед началом убедитесь, что файл data_school.txt находится в рабочем каталоге.

import java.io.*;
import java.nio.file.*;
import java.util.*;

public class SchoolJournal {
    static class Student {
        String lastName;
        String firstName;
        int grade;
        String subject;
        int mark;

        Student(String lastName, String firstName, int grade, String subject, int mark) {
            this.lastName = lastName;
            this.firstName = firstName;
            this.grade = grade;
            this.subject = subject;
            this.mark = mark;
        }
    }

    public static void main(String[] args) {
        List<Student> students = readStudentsFromFile("data_school.txt");
        Scanner scanner = new Scanner(System.in);
        int choice;

        do {
            System.out.println("Меню:");
            System.out.println("1. Список учеников по заданной оценке");
            System.out.println("2. Сортировка классов по средней успеваемости");
            System.out.println("3. Список учеников по предмету, отсортированный по фамилии");
            System.out.println("4. Вывести ведомости заданного класса по каждому предмету");
            System.out.println("5. Определить класс ученика по имени и фамилии");
            System.out.println("6. Определить предмет с самой высокой средней успеваемостью");
            System.out.println("0. Выход");
            System.out.print("Выберите опцию: ");
            choice = scanner.nextInt();
            scanner.nextLine(); // Считываем новую строку

            switch (choice) {
                case 1:
                    listStudentsByGrade(scanner, students);
                    break;
                case 2:
                    sortClassesByAverageScore(students);
                    break;
                case 3:
                    listStudentsBySubject(scanner, students);
                    break;
                case 4:
                    createClassStatementsBySubject(scanner, students);
                    break;
                case 5:
                    findClassByStudentName(scanner, students);
                    break;
                case 6:
                    findSubjectWithHighestAverageMark(students);
                    break;
                case 0:
                    System.out.println("Выход из программы");
                    break;
                default:
                    System.out.println("Неверный выбор, попробуйте снова.");
            }
        } while (choice != 0);

        scanner.close();
    }

    static List<Student> readStudentsFromFile(String fileName) {
        List<Student> students = new ArrayList<>();
        try {
            List<String> lines = Files.readAllLines(Paths.get(fileName));
            for (String line : lines) {
                String[] parts = line.split(" ");
                String lastName = parts[0];
                String firstName = parts[1];
                int grade = Integer.parseInt(parts[2]);
                String subject = parts[3];
                int mark = Integer.parseInt(parts[4]);
                students.add(new Student(lastName, firstName, grade, subject, mark));
            }
        } catch (IOException e) {
            System.err.println("Ошибка при чтении файла: " + e.getMessage());
        }
        return students;
    }

    static void listStudentsByGrade(Scanner scanner, List<Student> students) {
        System.out.print("Введите оценку: ");
        int mark = scanner.nextInt();

        for (Student student : students) {
            if (student.mark == mark) {
                System.out.println(student.lastName + " " + student.firstName + " (Класс " + student.grade + ")");
            }
        }
    }

    static void sortClassesByAverageScore(List<Student> students) {
        Map<Integer, List<Integer>> grades = new HashMap<>();

        for (Student student : students) {
            grades.putIfAbsent(student.grade, new ArrayList<>());
            
            grades.get(student.grade).add(student.mark);
        }

        grades.entrySet().stream()
            .sorted((entry1, entry2) -> {
                double avg1 = entry1.getValue().stream().mapToInt(Integer::intValue).average().orElse(0);
                double avg2 = entry2.getValue().stream().mapToInt(Integer::intValue).average().orElse(0);
                return Double.compare(avg1, avg2);
            })
            .forEach(entry -> {
                int classNumber = entry.getKey();
                double average = entry.getValue().stream().mapToInt(Integer::intValue).average().orElse(0);
                System.out.printf("Класс %d: Средняя успеваемость = %.2f%n", classNumber, average);
            });
    }

    static void listStudentsBySubject(Scanner scanner, List<Student> students) {
        System.out.print("Введите предмет: ");
        String subject = scanner.nextLine();

        students.stream()
            .filter(student -> student.subject.equalsIgnoreCase(subject))
            .sorted(Comparator.comparing(student -> student.lastName))
            .forEach(student -> System.out.println(student.lastName + " " + student.firstName + " (Оценка: " + student.mark + ")"));
    }

    static void createClassStatementsBySubject(Scanner scanner, List<Student> students) {
        System.out.print("Введите номер класса: ");
        int classNumber = scanner.nextInt();
        Map<String, List<Student>> subjectMap = new HashMap<>();

        for (Student student : students) {
            if (student.grade == classNumber) {
                subjectMap.putIfAbsent(student.subject, new ArrayList<>());
                subjectMap.get(student.subject).add(student);
            }
        }

        try {
            for (Map.Entry<String, List<Student>> entry : subjectMap.entrySet()) {
                String subject = entry.getKey();
                List<Student> classStudents = entry.getValue();
                String fileName = "class_" + classNumber + "_" + subject + ".txt";
                try (BufferedWriter writer = new BufferedWriter(new FileWriter(fileName))) {
                    for (Student student : classStudents) {
                        writer.write(student.lastName + " " + student.firstName + " " + student.mark);
                        writer.newLine();
                    }
                }
                System.out.println("Сохранено в " + fileName);
            }
        } catch (IOException e) {
            System.err.println("Ошибка при записи в файл: " + e.getMessage());
        }
    }

    static void findClassByStudentName(Scanner scanner, List<Student> students) {
        System.out.print("Введите фамилию: ");
        String lastName = scanner.nextLine();
        System.out.print("Введите имя: ");
        String firstName = scanner.nextLine();

        for (Student student : students) {
            if (student.lastName.equalsIgnoreCase(lastName) && student.firstName.equalsIgnoreCase(firstName)) {
                System.out.println(firstName + " " + lastName + " учится в классе " + student.grade);
                return;
            }
        }
        System.out.println("Ученик не найден.");
    }

    static void findSubjectWithHighestAverageMark(List<Student> students) {
        Map<String, List<Integer>> subjectMarks = new HashMap<>();

        for (Student student : students) {
            subjectMarks.putIfAbsent(student.subject, new ArrayList<>());
            subjectMarks.get(student.subject).add(student.mark);
        }

        String highestSubject = null;
        double highestAverage = 0;

        for (Map.Entry<String, List<Integer>> entry : subjectMarks.entrySet()) {
            double avg = entry.getValue().stream().mapToInt(Integer::intValue).average().orElse(0);
            if (avg > highestAverage) {
                highestAverage = avg;
                highestSubject = entry.getKey();
            }
        }
        if (highestSubject != null) {
        
            System.out.printf("Предмет с самой высокой средней успеваемостью: %s (Средняя: %.2f)%n", highestSubject, highestAverage);
        }
    }
}


### Объяснение кода
1. Структура данных: Создан класс Student, представляющий информацию о школьниках.
2. Чтение из файла: Функция readStudentsFromFile загружает данные о школьниках из файла.
3. Меню: В основном методе реализовано меню для выбора различных функций.
4. Функции: Каждая задача реализована в своем методе. 
5. Сохранение в файл: Ведомости по каждому предмету сохраняются в отдельные файлы.

### Запуск программы
1. Откройте вашу среду разработки (например, IntelliJ IDEA).
2. Создайте новый проект и добавьте файл data_school.txt с данными учеников в корень проекта.
3. Скомпилируйте и запустите программу.

Программа будет выдавать результаты на консоль и сохранять данные в файлы по мере ваших запросов.
