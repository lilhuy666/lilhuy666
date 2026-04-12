import feedparser
from telegram import Update
from telegram.ext import Updater, CommandHandler, CallbackContext

# 🔑 Вставь сюда свой токен
TOKEN = "YOUR_BOT_TOKEN_HERE"

# Ключевые слова
KEYWORDS = ["AI", "technology", "python", "robot"]

# RSS-ленты
RSS_FEEDS = [
    "https://news.google.com/rss",
    "https://feeds.bbci.co.uk/news/technology/rss.xml"
]


def fetch_news():
    articles = []

    for url in RSS_FEEDS:
        feed = feedparser.parse(url)

        for entry in feed.entries[:10]:  # ограничим количество
            title = entry.title
            link = entry.link

            if any(k.lower() in title.lower() for k in KEYWORDS):
                articles.append(f"📰 {title}\n🔗 {link}")

    return articles


# Команда /start
def start(update: Update, context: CallbackContext):
    update.message.reply_text(
        "Привет! 🤖\nНапиши /news чтобы получить новости по ключевым словам."
    )


# Команда /news
def news(update: Update, context: CallbackContext):
    articles = fetch_news()

    if not articles:
        update.message.reply_text("Ничего не найдено 😢")
        return

    for article in articles[:5]:  # отправим только 5
        update.message.reply_text(article)


def main():
    updater = Updater(TOKEN, use_context=True)
    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("news", news))

    print("Бот запущен...")
    updater.start_polling()
    updater.idle()


if __name__ == "__main__":
    main()
