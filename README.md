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
