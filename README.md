import java.io.*;
import java.util.*;

class Student {
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

public class SchoolJournal {
    private static Map<Integer, List<Student>> classMap = new HashMap<>();
    private static Map<String, Map<Integer, List<Student>>> subjectMap = new HashMap<>();

    public static void main(String[] args) {
        readDataFromFile("school.txt");
        printStudentsByGrade(5, 4); // Пример: выводим учеников 5 класса с оценкой 4
        sortClassesByAveragePerformance();
        printStudentsBySubject("Math");
        writeGradeReports();
        findStudentClass("Иванов", "Иван");
        findSubjectWithHighestAverage();
    }

    private static void readDataFromFile(String filename) {
        try (BufferedReader br = new BufferedReader(new FileReader(filename))) {
            String line;
            while ((line = br.readLine()) != null) {
                String[] parts = line.split(" ");
                String lastName = parts[0];
                String firstName = parts[1];
                int grade = Integer.parseInt(parts[2]);
                String subject = parts[3];
                int mark = Integer.parseInt(parts[4]);

                Student student = new Student(lastName, firstName, grade, subject, mark);
                classMap.computeIfAbsent(grade, k -> new ArrayList<>()).add(student);
                subjectMap.computeIfAbsent(subject, k -> new HashMap<>())
                        .computeIfAbsent(grade, k -> new ArrayList<>()).add(student);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void printStudentsByGrade(int grade, int mark) {
        List<Student> students = classMap.get(grade);
        if (students != null) {
            System.out.println("Учащиеся " + grade + " класса с оценкой " + mark + ":");
            for (Student student : students) {
                if (student.mark == mark) {
                    System.out.println(student.lastName + " " + student.firstName + " - " + student.mark);
                }
            }
        }
    }

    private static void sortClassesByAveragePerformance() {
        Map<Integer, Double> averageMap = new HashMap<>();
        for (Map.Entry<Integer, List<Student>> entry : classMap.entrySet()) {
            int grade = entry.getKey();
            List<Student> students = entry.getValue();
            double average = students.stream().mapToInt(s -> s.mark).average().orElse(0);
            averageMap.put(grade, average);
        }
        averageMap.entrySet().stream()
                .sorted(Map.Entry.comparingByValue())
                .forEach(entry -> System.out.println("Класс " + entry.getKey() + " - Средняя оценка: " + entry.getValue()));
    }

    private static void printStudentsBySubject(String subject) {
        List<Student> allStudents = new ArrayList<>();
        for (Map<Integer, List<Student>> gradeMap : subjectMap.values()) {
            List<Student> students = gradeMap.get(subject);
            if (students != null) {
                allStudents.addAll(students);
            }
        }
        allStudents.sort(Comparator.comparing(s -> s.lastName));
        System.out.println("Учащиеся по предмету " + subject + ":");
        for (Student student : allStudents) {
            System.out.println(student.lastName + " " + student.firstName + " - " + student.mark);
        }
    }

    private static void writeGradeReports() {
        for (Map.Entry<Integer, List<Student>> entry : classMap.entrySet()) {
            int grade = entry.getKey();
            List<Student> students = entry.getValue();
            try (BufferedWriter bw = new BufferedWriter(new FileWriter("grade_report_" + grade + ".txt"))) {
                for (Student student : students) {
                    bw.write(student.lastName + " " + student.firstName + " - " + student.subject + ": " + student.mark);
                    bw.newLine();
                }
            } catch (IOException
