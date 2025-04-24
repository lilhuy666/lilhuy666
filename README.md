import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

class Application {
    private int number;
    private int score;

    public Application(int number, int score) {
        this.number = number;
        this.score = score;
    }

    public int getNumber() {
        return number;
    }

    public int getScore() {
        return score;
    }
}

class Contest {
    private double prizeFund;
    private List<Application> applications;

    public Contest(double prizeFund) {
        this.prizeFund = prizeFund;
        this.applications = new ArrayList<>();
    }

    public void addApplication(Application application) {
        applications.add(application);
    }

    public List<Application> getWinners() {
        double averageScore = calculateAverageScore();
        List<Application> winners = new ArrayList<>();
        for (Application app : applications) {
            if (app.getScore() > averageScore) {
                winners.add(app);
            }
        }
        return winners;
    }

    private double calculateAverageScore() {
        double totalScore = 0;
        for (Application app : applications) {
            totalScore += app.getScore();
        }
        return applications.size() > 0 ? totalScore / applications.size() : 0;
    }

    public void distributePrizes() {
        List<Application> winners = getWinners();
        double totalScoreOfWinners = 0;
        for (Application winner : winners) {
            totalScoreOfWinners += winner.getScore();
        }

        for (Application winner : winners) {
            double prize = (winner.getScore() / totalScoreOfWinners) * prizeFund;
            System.out.printf("Заявка #%d: Оценка = %d, Премия = %.2f%n", winner.getNumber(), winner.getScore(), prize);
        }
    }
}

public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        try {
            System.out.print("Введите размер премиального фонда первого конкурса: ");
            double prizeFund1 = scanner.nextDouble();
            Contest contest1 = new Contest(prizeFund1);

            System.out.print("Введите количество заявок первого конкурса: ");
            int numberOfApplications1 = scanner.nextInt();
            for (int i = 1; i <= numberOfApplications1; i++) {
                System.out.printf("Введите оценку для заявки #%d: ", i);
                int score = scanner.nextInt();
                contest1.addApplication(new Application(i, score));
            }

            System.out.print("Введите размер премиального фонда второго конкурса: ");
            double prizeFund2 = scanner.nextDouble();
            Contest contest2 = new Contest(prizeFund2);

            System.out.print("Введите количество заявок второго конкурса: ");
            int numberOfApplications2 = scanner.nextInt();
            for (int i = 1; i <= numberOfApplications2; i++) {
                System.out.printf("Введите оценку для заявки #%d: ", i);
                int score = scanner.nextInt();
                contest2.addApplication(new Application(i, score));
            }

            System.out.println("\nПобедители первого конкурса:");
            contest1.distributePrizes();

            System.out.println("\nПобедители второго конкурса:");
            contest2.distributePrizes();
        } catch (Exception e) {
            System.out.println("Произошла ошибка ввода данных. Пожалуйста, убедитесь, что вы вводите правильные значения.");
        } finally {
            scanner.close();
        }
    }
}
