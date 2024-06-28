import java.util.Scanner;

public class SettlementsGraph {

    private static int[][] graph;
    private static boolean[] visited;

    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // Вводим количество поселков и дорог
        System.out.print("Введите количество поселков: ");
        int settlements = scanner.nextInt();
        System.out.print("Введите количество дорог: ");
        int roads = scanner.nextInt();

        graph = new int[settlements][settlements];
        visited = new boolean[settlements];

        // Вводим информацию о дорогах
        System.out.println("Введите информацию о дорогах:");
        for (int i = 0; i < roads; i++) {
            int from = scanner.nextInt();
            int to = scanner.nextInt();
            graph[from][to] = 1;
        }

        // Выполняем поиск для определения одиночного города
        for (int i = 0; i < settlements; i++) {
            if (!visited[i]) {
                if (!hasAdjacent(i)) {
                    System.out.println("Ответ: " + i);
                    return;
                }
            }
        }
        System.out.println("Ответ: -1");
    }

    private static boolean hasAdjacent(int settlement) {
        visited[settlement] = true;
        for (int i = 0; i < graph[settlement].length; i++) {
            if (graph[settlement][i] == 1 && !visited[i]) {
                return true;
            }
        }
        return false;
    }
}
