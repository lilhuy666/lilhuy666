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
            String lowerKeyword = keyword.toLowerCase();

            for (Article article : articles) {
                if (article.getTitle().toLowerCase().contains(lowerKeyword) ||
                    article.getContent().toLowerCase().contains(lowerKeyword)) {
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

            int index = 1;
            for (Article article : list) {
                System.out.println(index++ + ". " + article.getTitle());
                System.out.println(article.getContent());
                System.out.println("-------------------------");
            }
        }
    }

    public static void main(String[] args) {
        NewsAggregator aggregator = new NewsAggregator();

        String[] keywords = {
            "технологии","искусственный интеллект","роботы","интернет","программирование",
            "java","python","разработка","стартап","инновации",
            "спорт","футбол","баскетбол","теннис","олимпиада",
            "здоровье","медицина","вакцина","питание","фитнес",
            "экономика","деньги","инвестиции","бизнес","рынок",
            "политика","выборы","правительство","закон","реформа",
            "образование","школа","университет","курсы","обучение",
            "наука","космос","физика","химия","биология",
            "природа","экология","климат","животные","лес",
            "путешествия","туризм","отпуск","страны","город",
            "еда","ресторан","кухня","рецепт","напитки",
            "музыка","фильмы","сериалы","игры","развлечения",
            "культура","искусство","театр","музей","выставка",
            "авто","машины","транспорт","дороги","электромобили",
            "безопасность","кибербезопасность","данные","хакеры","пароли",
            "погода","зима","лето","осень","весна",
            "работа","карьера","резюме","собеседование","офис",
            "дом","ремонт","дизайн","интерьер","мебель",
            "финансы","кредит","банк","налоги","страхование"
        };

        // создаём 100 статей
        for (int i = 0; i < keywords.length; i++) {
            String keyword = keywords[i];

            String title = "Новость про " + keyword;
            String content = "Это статья о теме: " + keyword +
                    ". Здесь обсуждаются " + keyword +
                    " и связанные вопросы. Также слово " + keyword +
                    " используется для поиска.";

            aggregator.addArticle(new Article(title, content));
        }

        Scanner scanner = new Scanner(System.in);

        System.out.print("Введите ключевое слово: ");
        String keyword = scanner.nextLine();

        if (keyword.trim().isEmpty()) {
            System.out.println("Введите непустое слово!");
            return;
        }

        List<Article> result = aggregator.searchByKeyword(keyword);

        System.out.println("\nРезультаты поиска:");
        aggregator.printArticles(result);

        scanner.close();
    }
}
