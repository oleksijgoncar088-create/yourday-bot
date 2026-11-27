[bot.py](https://github.com/user-attachments/files/35502286/bot.py)
import logging
import sqlite3
from datetime import datetime

from aiogram import Bot, Dispatcher
from aiogram.filters import Command
from aiogram.types import Message

BOT_TOKEN = "9067319770:AAFh-Ckw1rAfXAabeHLD5VYWzA2NDeMO3QU"

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s",
)
logger = logging.getLogger("habit_tracker")

DB_FILE = "habits.db"


def init_db() -> None:
    with sqlite3.connect(DB_FILE) as conn:
        conn.execute(
            """
            CREATE TABLE IF NOT EXISTS habits (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                habit_name TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
            """
        )


def add_habit(user_id: int, habit_name: str) -> None:
    with sqlite3.connect(DB_FILE) as conn:
        conn.execute(
            "INSERT INTO habits (user_id, habit_name) VALUES (?, ?)",
            (user_id, habit_name),
        )


def get_today_habits(user_id: int) -> list[tuple[str, str]]:
    with sqlite3.connect(DB_FILE) as conn:
        rows = conn.execute(
            """
            SELECT habit_name, created_at
            FROM habits
            WHERE user_id = ?
              AND DATE(created_at) = DATE('now', 'localtime')
            ORDER BY created_at
            """,
            (user_id,),
        ).fetchall()
    return [(name, created_at) for name, created_at in rows]


def get_all_habits(user_id: int) -> list[str]:
    with sqlite3.connect(DB_FILE) as conn:
        rows = conn.execute(
            """
            SELECT DISTINCT habit_name
            FROM habits
            WHERE user_id = ?
            ORDER BY habit_name
            """,
            (user_id,),
        ).fetchall()
    return [row[0] for row in rows]


HELP_TEXT = (
    "Привет! Я твой трекер привычек.\n"
    "Пиши /done <привычка>, чтобы отметить выполнение.\n"
    "Пример: /done вода\n"
    "/stats — посмотреть статистику за сегодня\n"
    "/list — список всех твоих привычек"
)

ERROR_TEXT = "Что-то сломалось, попробуй ещё раз"


async def cmd_start(message: Message) -> None:
    logger.info("Команда /start от user_id=%s", message.from_user.id)
    await message.answer(HELP_TEXT)


async def cmd_done(message: Message) -> None:
    user_id = message.from_user.id
    logger.info("Команда /done от user_id=%s", user_id)

    raw = (message.text or "").split(maxsplit=1)
    if len(raw) < 2 or not raw[1].strip():
        await message.answer("Укажи привычку: /done вода")
        return

    habit_name = raw[1].strip().lower()
    try:
        add_habit(user_id, habit_name)
    except Exception:
        logger.exception("Ошибка при записи привычки user_id=%s", user_id)
        await message.answer(ERROR_TEXT)
        return

    now = datetime.now().strftime("%H:%M")
    await message.answer(f"✅ Записал: {habit_name} в {now}")


async def cmd_stats(message: Message) -> None:
    user_id = message.from_user.id
    logger.info("Команда /stats от user_id=%s", user_id)

    try:
        today_habits = get_today_habits(user_id)
    except Exception:
        logger.exception("Ошибка при получении статистики user_id=%s", user_id)
        await message.answer(ERROR_TEXT)
        return

    if not today_habits:
        await message.answer("Сегодня пока пусто. Начни с /done <привычка>")
        return

    lines = [f"{name} — {created_at[11:16]}" for name, created_at in today_habits]
    lines.append(f"Всего за сегодня: {len(today_habits)} выполнений")
    await message.answer("\n".join(lines))


async def cmd_list(message: Message) -> None:
    user_id = message.from_user.id
    logger.info("Команда /list от user_id=%s", user_id)

    try:
        habits = get_all_habits(user_id)
    except Exception:
        logger.exception("Ошибка при получении списка привычек user_id=%s", user_id)
        await message.answer(ERROR_TEXT)
        return

    if not habits:
        await message.answer("У тебя пока нет привычек. Отметь первую: /done вода")
        return

    await message.answer("\n".join(habits))


async def main() -> None:
    init_db()
    bot = Bot(token=BOT_TOKEN)
    dp = Dispatcher()

    dp.message.register(cmd_start, Command("start"))
    dp.message.register(cmd_done, Command("done"))
    dp.message.register(cmd_stats, Command("stats"))
    dp.message.register(cmd_list, Command("list"))

    logger.info("Бот запущен")
    await dp.start_polling(bot)


if __name__ == "__main__":
    import asyncio

    asyncio.run(main())
