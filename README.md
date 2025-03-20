```java
import java.util.Arrays;
import java.util.Comparator;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
import java.util.stream.Stream;

public class StringProcessing {
    
    public static void main(String[] args) {
        String sentence = "This is a test, with some words ending in 'es', like cases, boxes.";
        
        // Подсчет количества слов, оканчивающихся на 'es'
        long esWordsCount = Arrays.stream(sentence.split("[\\s,\\.,!?]+"))
                .filter(word -> word.endsWith("es"))
                .count();
        System.out.println("Количество слов, оканчивающихся на 'es': " + esWordsCount);
        
        // Упорядочить слова по длине
        Stream<String> sortedByLength = Arrays.stream(sentence.split("[\\s,\\.,!?]+"))
                .sorted(Comparator.comparingInt(String::length));
        System.out.println("Слова, отсортированные по длине:");
        sortedByLength.forEach(System.out::println);
        
        // Сумма всех целых чисел в строке
        Pattern pattern = Pattern.compile("\\d+");
        Matcher matcher = pattern.matcher(sentence);
        int sumOfNumbers = 0;
        while (matcher.find()) {
            sumOfNumbers += Integer.parseInt(match...
