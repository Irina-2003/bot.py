# bot.py
import os
import csv
import asyncio
from datetime import date
from aiogram import Bot, Dispatcher, types
from aiogram.filters import Command

BOT_TOKEN = os.getenv("BOT_TOKEN")
CHAT_ID = os.getenv("CHANNEL_USERNAME")
DB_FILE = "wishes.csv"

if not BOT_TOKEN or not CHAT_ID:
    raise ValueError("Не заданы BOT_TOKEN или CHAT_ID в переменных окружения!")

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

def init_db():
    if not os.path.exists(DB_FILE):
        with open(DB_FILE, "w", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            writer.writerow(["id", "text", "name", "send_date", "status"])

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    await message.answer(
        "Привет! Напиши пожелание в формате:\n"
        "Текст пожелания | ГГГГ-ММ-ДД\n\n"
        "Пример: Счастья и любви! | 2026-12-31"
    )

@dp.message()
async def handle_wish(message: types.Message):
    text = message.text
    parts = text.split("|")

    if len(parts) != 2:
        await message.answer(
            "Пожалуйста, отправь в формате: Текст пожелания | ГГГГ-ММ-ДД\n"
            "Пример: Вы лучшая пара! | 2027-01-01"
        )
        return

    wish_text, send_date = parts[0].strip(), parts[1].strip()

    try:
        from datetime import datetime
        parsed = datetime.strptime(send_date, "%Y-%m-%d")
        if parsed.date() < date.today():
            await message.answer("Дата должна быть сегодняшней или будущей.")
            return
    except ValueError:
        await message.answer("Неверный формат даты. Используй ГГГГ-ММ-ДД, например 2026-09-30")
        return

    with open(DB_FILE, "r", encoding="utf-8") as f:
        reader = csv.reader(f)
        rows = list(reader)
        new_id = len(rows)

    with open(DB_FILE, "a", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        writer.writerow([new_id, wish_text, message.from_user.full_name, send_date, "pending"])

    await message.answer(f"Спасибо! Твое пожелание будет опубликовано {send_date}.")

async def daily_poster():
    while True:
        today = date.today().isoformat()
        to_send = []

        with open(DB_FILE, "r", newline="", encoding="utf-8") as f:
            reader = csv.DictReader(f)
            rows = list(reader)

        for row in rows:
            if row["status"] == "pending" and row["send_date"] == today:
                to_send.append(row)

        for wish in to_send:
            text_to_post = (
                f"💌 Пожелание от {wish['name']}\n\n"
                f"{wish['text']}"
            )
            try:
                await bot.send_message(CHAT_ID, text_to_post)
                wish["status"] = "sent"
            except Exception as e:
                print(f"Ошибка отправки: {e}")

        fieldnames = ["id", "text", "name", "send_date", "status"]
        with open(DB_FILE, "w", newline="", encoding="utf-8") as f:
            import csv as csv_mod
            writer = csv_mod.DictWriter(f, fieldnames=fieldnames)
            writer.writeheader()
            writer.writerows(rows)

        await asyncio.sleep(86400)

async def main():
    init_db()
    asyncio.create_task(daily_poster())
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
