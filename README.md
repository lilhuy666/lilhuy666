Вот код Java с использованием лямбда-функций для решения поставленных задач:

import java.util.*;
import java.util.stream.Collectors;

public class TemperatureAndWords {

    public static void main(String[] args) {
        // Задача 1: Дневник температуры за март
        int[] temperatures = {1, -2, -5, -2, -4, 3, -6, -2, 1, 1, 0, -1, 0, 3, -1, 2, 5, 2, 4, 4, 0, 6, 1, 4, 6, -1, 2, 4, 7, 11};

        // Количество дней с отрицательной температурой
        long negativeDays = Arrays.stream(temperatures).filter(temp -> temp < 0).count();
        System.out.println("Количество дней с отрицательной температурой: " + negativeDays);

        // Были ли дни, когда температура выше 10 градусов
        boolean aboveTenDegrees = Arrays.stream(temperatures).anyMatch(temp -> temp > 10);
        System.out.println("Были ли дни с температурой выше 10 градусов: " + (aboveTenDegrees ? "Да" : "Нет"));

        // Максимальная температура в первую неделю марта
        int maxFirstWeekTemp = Arrays.stream(Arrays.copyOfRange(temperatures, 0, 7)).max().orElse(Integer.MIN_VALUE);
        System.out.println("Максимальная температура в первую неделю марта: " + maxFirstWeekTemp);

        // Средняя температура за месяц
        double averageTemp = Arrays.stream(temperatures).average().orElse(Double.NaN);
        System.out.println("Средняя температура за месяц: " + averageTemp);

        // Задача 2: Подсчет слов и сумма целых чисел в предложении
        String sentence = "apple, oranges, bananas, grapes, 3, 10, cherries";
        String[] words = sentence.split("[, ]+");

        // Количество слов, заканчивающихся на "es"
        long wordsEndingWithEs = Arrays.stream(words).filter(word -> word.endsWith("es")).count();
        System.out.println("Количество слов, заканчивающихся на 'es': " + wordsEndingWithEs);

        // Сортировка слов по возрастанию длины
        List<String> sortedWords = Arrays.stream(words).sorted(Comparator.comparingInt(String::length)).collect(Collectors.toList());
        System.out.println("Слова, упорядоченные по возрастанию длины: " + sortedWords);

        // Сумма целых чисел
        int sumOfIntegers = Arrays.stream(words)
                .filter(word -> word.matches("-?\\d+"))  // проверяем, является ли слово целым числом
                .mapToInt(Integer::parseInt)
                .sum();
        System.out.println("Сумма целых чисел в предложении: " + sumOfIntegers);
    }
}


Этот код решает обе задачи с использованием Java Stream API и лямбда-выражений.
