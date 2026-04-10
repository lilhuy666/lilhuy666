import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

public class Main {

    static class Article {
        private String title;
        private String content;

        public Article(String title, String content) {
            this.title = title;
            this.content = content;
        }

        public String getTitle() {
            return title;
        }

        public String getContent() {
            return content;
        }
    }

    static class NewsAggregator {
        private List<Article> articles = new ArrayList<>();

        public void addArticle(Article article) {
            articles.add(article);
        }

        public List<Article> searchByKeyword(String keyword) {
            List<Article> result = new ArrayList<>();

            for (Article article : articles) {
                if (article.getTitle().toLowerCase().contains(keyword.toLowerCase()) ||
                    article.getContent().toLowerCase().contains(keyword.toLowerCase())) {
                    result.add(article);
                }
            }

            return result;
        }

        public void printArticles(List<Article> list) {
            if (list.isEmpty()) {
                System.out.println("Ничего не найдено.");
                return;
            }

            for (Article article : list) {
                System.out.println("Заголовок: " + article.getTitle());
                System.out.println("Текст: " + article.getContent());
                System.out.println("-------------------------");
            }
        }
    }

    public static void main(String[] args) {
        NewsAggregator aggregator = new NewsAggregator();

        aggregator.addArticle(new Article(
                "Технологии будущего",
                "Искусственный интеллект развивается очень быстро"
        ));

        aggregator.addArticle(new Article(
                "Спорт сегодня",
                "Футбольная команда выиграла матч"
        ));

        aggregator.addArticle(new Article(
                "Новости экономики",
                "Курс валют изменился"
        ));

        Scanner scanner = new Scanner(System.in);

        System.out.print("Введите ключевое слово: ");
        String keyword = scanner.nextLine();

        List<Article> result = aggregator.searchByKeyword(keyword);

        System.out.println("\nРезультаты поиска:");
        aggregator.printArticles(result);
    }
}
