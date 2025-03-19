Вот пример Java кода, который выполняет указанные задачи с использованием лямбда-выражений:

import java.util.*;
import java.util.regex.*;
import java.util.stream.*;

public class Main {
    public static void main(String[] args) {
        String input = "Привет, это тестовое предложение с числами 5 и 10 и словами, которые заканчиваются на es, такие как roses, buses.";
        
        // Подсчет слов, заканчивающихся на "es"
        long countEndsWithEs = Arrays.stream(input.split("[ ,]+"))
                .filter(word -> word.endsWith("es"))
                .count();

        // Упорядочивание слов по длине
        List<String> sortedWords = Arrays.stream(input.split("[ ,]+"))
                .distinct()
                .sorted(Comparator.comparingInt(String::length))
                .collect(Collectors.toList());

        // Подсчет суммы целых чисел
        int sumOfIntegers = Arrays.stream(Pattern.compile("\\d+").matcher(input).results()
                .map(MatchResult::group)
                .mapToInt(Integer::parseInt)
                .toArray())
                .sum();

        // Вывод результатов
        System.out.println("Количество слов, заканчивающихся на 'es': " + countEndsWithEs);
        System.out.println("Слова, упорядоченные по возрастанию длины: " + sortedWords);
        System.out.println("Сумма целых чисел: " + sumOfIntegers);
    }
}


Этот код выполняет все указанные задачи: подсчитывает слова с окончанием "es", упорядочивает слова по длине и суммирует целые числа.
