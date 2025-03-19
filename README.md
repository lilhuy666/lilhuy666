Вот пример Java кода с использованием лямбда-функций для анализа дневника температуры:

import java.util.Arrays;

public class TemperatureAnalysis {
    public static void main(String[] args) {
        int[] temperatures = {2, -5, -2, -4, 3, -6, -2, -1, 5, 1, 1, 0, -1, 0, 3, -1, 2, 5, 2, 4, 4, 0, 6, 1, 4, 6, -1, 2, 4, 7, 11};

        // Количество дней с отрицательной температурой
        long countNegativeDays = Arrays.stream(temperatures).filter(temp -> temp < 0).count();
        System.out.println("Количество дней с отрицательной температурой: " + countNegativeDays);

        // Были ли дни, когда температура оказалась выше 10 градусов
        boolean hasAbove10 = Arrays.stream(temperatures).anyMatch(temp -> temp > 10);
        System.out.println("Были ли дни выше 10 градусов: " + hasAbove10);

        // Максимальная температура в первую неделю марта
        int maxFirstWeek = Arrays.stream(Arrays.copyOfRange(temperatures, 0, 7)).max().orElse(Integer.MIN_VALUE);
        System.out.println("Максимальная температура в первую неделю марта: " + maxFirstWeek);

        // Средняя температура за месяц
        double averageTemperature = Arrays.stream(temperatures).average().orElse(Double.NaN);
        System.out.println("Средняя температура за месяц: " + averageTemperature);
    }
}


Этот код выполняет следующие действия:
1. Считывает массив температур.
2. С помощью лямбда-выражений подсчитывает количество дней с отрицательной температурой.
3. Проверяет, были ли дни, когда температура превышала 10 градусов.
4. Находит максимальную температуру в первую неделю марта.
5. Рассчитывает среднюю температуру за месяц.
