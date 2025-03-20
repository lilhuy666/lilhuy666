import java.util.*;
import java.util.stream.Collectors;

public class Main {
    public static void main(String[] args) {
        // Дневник температуры
        int[] temperatures = {1, -2, -5, -2, -4, 3, -6, -2, 1, 1, 0, -1, 0, 3, -1, 2, 5, 2, 4, 4, 0, 6, 1, 4, 6, -1, 2, 4, 7, 11};

        long negativeDays = Arrays.stream(temperatures)
                .filter(t -> t < 0)
                .count();

        boolean above10 = Arrays.stream(temperatures)
                .anyMatch(t -> t > 10);

        int maxFirstWeek = Arrays.stream(Arrays.copyOfRange(temperatures, 0, 7))
                .max()
                .orElse(Integer.MIN_VALUE);

        double averageTemperature = Arrays.stream(temperatures)
                .average()
                .orElse(0);
        
        System.out.println("Количество дней с отрицательной температурой: " + negativeDays);
        System.out.println("Температура выше 10 градусов: " + above10);
        System.out.println("Максимальная температура в первую неделю: " + maxFirstWeek);
        System.out.println("Средняя температура за месяц: " + averageTemperature);

        // Предложение
        String sentence = "They used 233 features including 227 stylometric features and six novel social network-specific features like character-based ones numbers of alphabets, uppercase characters, special characters, word-based ones the total number of words, average word length, the number of words with 1 char, syntactic ones numbers of punctuation marks and functional words, the total number of sentences and many others";
        List<String> words = Arrays.asList(sentence.split("[ ,]+"));
        
        long countEndsWithEs = words.stream()
                .filter(word -> word.endsWith("es"))
                .count();

        List<String> sortedWords = words.stream()
                .sorted(Comparator.comparingInt(String::length))
                .collect(Collectors.toList());

        int sumOfIntegers = words.stream()
                .filter(word -> word.matches("-?\\d+"))
                .mapToInt(Integer::parseInt)
                .sum();

        System.out.println("Количество слов, заканчивающихся на 'es': " + countEndsWithEs);
        System.out.println("Слова упорядоченные по длине: " + sortedWords);
        System.out.println("Сумма целых чисел в предложении: " + sumOfIntegers);
    }
}
