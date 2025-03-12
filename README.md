
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
