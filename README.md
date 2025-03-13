Вот пример кода на Java, который выполняет указанные действия. Более подробные комментарии добавлены для понимания:

import java.io.*;
import java.util.*;

public class SchoolJournal {
    static class Student {
        String lastName;
        String firstName;
        int grade;
        String subject;
        int mark;

        public Student(String lastName, String firstName, int grade, String subject, int mark) {
            this.lastName = lastName;
            this.firstName = firstName;
            this.grade = grade;
            this.subject = subject;
            this.mark = mark;
        }
    }

    public static void main(String[] args) {
        List<Student> students = new ArrayList<>();
        readStudentsFromFile("data school.txt", students);
        
        Map<Integer, List<Student>> classes = new TreeMap<>();
        for (Student student : students) {
            classes.putIfAbsent(student.grade, new ArrayList<>());
            classes.get(student.grade).add(student);
        }

        // 1. Вывод списка учеников с заданной оценкой
        int targetMark = 5; // Например, заданная оценка
        for (Map.Entry<Integer, List<Student>> entry : classes.entrySet()) {
            System.out.println("Класс " + entry.getKey() + ":");
            for (Student student : entry.getValue()) {
                if (student.mark == targetMark) {
                    System.out.println(student.lastName + " " + student.firstName + " - " + student.mark);
                }
            }
        }

        // 2. Сортировка классов по средней успеваемости
        classes.entrySet()
               .stream()
               .sorted(Comparator.comparingDouble(entry -> {
                   return entry.getValue().stream().mapToInt(stud -> stud.mark).average().orElse(0);
               }))
               .forEach(entry -> {
                   System.out.println("Класс " + entry.getKey() + " - Средняя успеваемость: " +
                           entry.getValue().stream().mapToInt(stud -> stud.mark).average().orElse(0));
               });

        // 3. Вывод списка учеников всех классов, отсортированный по фамилии для заданного предмета
        String targetSubject = "Математика"; // Например, заданный предмет
        List<Student> sortedStudentsBySubject = new ArrayList<>();
        for (Student student : students) {
            if (student.subject.equals(targetSubject)) {
                sortedStudentsBySubject.add(student);
            }
        }
        sortedStudentsBySubject.sort(Comparator.comparing(stud -> stud.lastName));
        System.out.println("Список учеников по предмету " + targetSubject + ":");
        for (Student student : sortedStudentsBySubject) {
            System.out.println(student.lastName + " " + student.firstName);
        }

        // 4. Запись ведомостей заданного класса по каждому предмету в файл
        int targetGrade = 10; // Например, заданный класс
        Map<String, List<Student>> subjects = new HashMap<>();
        for (Student student : students) {
            if (student.grade == targetGrade) {
                subjects.putIfAbsent(student.subject, new ArrayList<>());
                subjects.get(student.subject).add(student);
            }
        }

        try (BufferedWriter writer = new BufferedWriter(new FileWriter("class_" + targetGrade + "_journal.txt"))) {
            for (Map.Entry<String, List<Student>> entry : subjects.entrySet()) {
                writer.write("Предмет: " + entry.getKey() + "\n");
                for (Student student : entry.getValue()) {
                    writer.write(student.lastName + " " + student.firstName + " - " + student.mark + "\n");
                }
                writer.write("\n");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

        // 5. По имени и фамилии определение класса
        String queryLastName = "Иванов"; // Замени на нужную фамилию
        String queryFirstName = "Иван"; // Замени на нужное имя
        for (Student student : students) {
        
            if (student.lastName.equals(queryLastName) && student.firstName.equals(queryFirstName)) {
                System.out.println("Ученик " + queryFirstName + " " + queryLastName + " учится в классе " + student.grade);
                break;
            }
        }

        // 6. Определение предмета с самой высокой средней успеваемостью
        Map<String, List<Integer>> subjectMarks = new HashMap<>();
        for (Student student : students) {
            subjectMarks.putIfAbsent(student.subject, new ArrayList<>());
            subjectMarks.get(student.subject).add(student.mark);
        }

        String topSubject = "";
        double highestAverage = 0;
        for (Map.Entry<String, List<Integer>> entry : subjectMarks.entrySet()) {
            double average = entry.getValue().stream().mapToInt(m -> m).average().orElse(0);
            if (average > highestAverage) {
                highestAverage = average;
                topSubject = entry.getKey();
            }
        }
        System.out.println("Предмет с самой высокой средней успеваемостью: " + topSubject + " - Средняя оценка: " + highestAverage);
    }

    private static void readStudentsFromFile(String fileName, List<Student> students) {
        try (BufferedReader br = new BufferedReader(new FileReader(fileName))) {
            String line;
            while ((line = br.readLine()) != null) {
                String[] parts = line.split(" ");
                String lastName = parts[0];
                String firstName = parts[1];
                int grade = Integer.parseInt(parts[2]);
                String subject = parts[3];
                int mark = Integer.parseInt(parts[4]);
                students.add(new Student(lastName, firstName, grade, subject, mark));
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}


### Пояснение кода:
1. Структура данных: Определяем класс Student, который хранит данные о школьнике.
2. Чтение данных: Метод readStudentsFromFile считывает данные из файла и создает объекты Student.
3. Группировка по классам: Создаем Map для распределения студентов по классам.
4. Фильтрация и сортировка: Реализованы операции по фильтрации студентов по оценке и по предметам.
5. Запись результатов: Ведомости записываются в отдельные текстовые файлы для каждого класса.
6. Поиск по имени: Можем находить класс по имени и фамилии.
7. Определение успешности предмета: Рассчитываем среднюю успеваемость по предмету.

Этот код можно запускать в IntelliJ IDEA, предварительно создав файл "data school.txt" с необходимыми данными.
