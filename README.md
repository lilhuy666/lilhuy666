import requests
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# ==========================
# 🔑 НАСТРОЙКИ
# ==========================
TELEGRAM_TOKEN = "8659598412:AAGidVRLwWRllRe38IOMjbHpzi_Rnry0CM4"
NEWS_API_KEY = "85a73564e3764f2baedb52f1422a3603"

# ==========================
# 📡 МОДУЛЬ АГРЕГАЦИИ НОВОСТЕЙ
# ==========================
def get_news(keyword: str):
    url = "https://newsapi.org/v2/everything"

    params = {
        "q": keyword,
        "language": "ru",
        "sortBy": "publishedAt",
        "apiKey": NEWS_API_KEY,
        "pageSize": 5
    }

    response = requests.get(url, params=params)
    data = response.json()

    articles = []

    if data.get("status") == "ok":
        for article in data["articles"]:
            title = article["title"]
            url = article["url"]
            source = article["source"]["name"]

            articles.append(f"📰 {title}\nИсточник: {source}\n{url}\n")

    return articles


# ==========================
# 🤖 КОМАНДЫ БОТА
# ==========================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Привет! 👋\n\n"
        "Напиши команду:\n"
        "/news слово\n\n"
        "Например:\n"
        "/news технологии"
    )


async def news(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("❗ Укажи ключевое слово\nПример: /news AI")
        return

    keyword = " ".join(context.args)

    await update.message.reply_text(f"🔎 Ищу новости по запросу: {keyword}...")

    articles = get_news(keyword)

    if not articles:
        await update.message.reply_text("😔 Новости не найдены")
        return

    for article in articles:
        await update.message.reply_text(article)


# ==========================
# 🚀 ЗАПУСК БОТА
# ==========================
def main():
    app = ApplicationBuilder().token(TELEGRAM_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("news", news))

    print("Бот запущен...")
    app.run_polling()


if __name__ == "__main__":
    main()
