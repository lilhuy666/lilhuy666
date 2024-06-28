Пример решения задачи на Java с использованием рекурсии для определения количества городов:

import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.print("Введите количество поселков и городов: ");
        int numPoints = scanner.nextInt();
        
        int[][] roads = new int[numPoints][numPoints];
        
        System.out.println("Введите данные о дорогах:");
        for (int i = 0; i < numPoints; i++) {
            for (int j = 0; j < numPoints; j++) {
                roads[i][j] = scanner.nextInt();
            }
        }
        
        int numCities = countCities(roads, numPoints);
        
        System.out.println("Количество городов: " + numCities);
    }
    
    public static int countCities(int[][] roads, int numPoints) {
        boolean[] visited = new boolean[numPoints];
        int numCities = 0;
        
        for (int i = 0; i < numPoints; i++) {
            if (!visited[i]) {
                numCities++;
                exploreCities(roads, visited, numPoints, i);
            }
        }
        
        if (numCities == 1) {
            return -1;
        } else {
            return numCities;
        }
    }
    
    public static void exploreCities(int[][] roads, boolean[] visited, int numPoints, int point) {
        visited[point] = true;
        
        for (int i = 0; i < numPoints; i++) {
            if (roads[point][i] == 1 && !visited[i]) {
                exploreCities(roads, visited, numPoints, i);
            }
        }
    }
}


Объяснение:
- В данном примере мы сначала вводим количество поселков и городов через консоль.
- Затем пользователь вводит информацию о дорогах между каждой парой поселков и городов.
- Функция countCities определяет количество городов, путем исследования соединений между точками с помощью рекурсивной функции exploreCities.
- Если количество городов равно 1, то программа выводит "Количество городов: -1", иначе выводится фактическое количество городов.
- После выполнения программы пользователь получает информацию о количестве городов в зависимости от соединений по данным дорогам.
