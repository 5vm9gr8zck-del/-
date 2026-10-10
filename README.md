# -import asyncio
from datetime import datetime, timedelta

from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.utils.keyboard import InlineKeyboardBuilder

# Замените на свой токен, полученный у @BotFather
BOT_TOKEN = "YOUR_BOT_TOKEN"

# Хранилище данных (в памяти). Для продакшена лучше использовать БД.
user_data = {}          # chat_id -> {tables_count, landing, orders: []}
active_reminders = {}   # chat_id -> datetime последней отправки

# Состояния FSM
class Form(StatesGroup):
    waiting_for_tables_count = State()
    waiting_for_landing = State()
    waiting_for_order_table = State()
    waiting_for_order_composition = State()
    waiting_for_order_comment = State()

# Клавиатура главного меню
def get_main_keyboard(chat_id: int) -> InlineKeyboardMarkup:
    reminder_status = "ВКЛ" if chat_id in active_reminders else "ВЫКЛ"
    builder = InlineKeyboardBuilder()
    builder.button(text="🍽 Количество столов", callback_data="set_tables")
    builder.button(text="🪑 Номер посадки", callback_data="set_landing")
    builder.button(text="📝 Добавить заказ", callback_data="add_order")
    builder.button(text=f"🔔 Напоминания: {reminder_status}", callback_data="toggle_reminder")
    builder.adjust(1)
    return builder.as_markup()

# /start и /help
async def cmd_start(message: types.Message):
    await message.answer(
        "Помощь хостесс и официантам в работе.\n\nВыберите действие:",
        reply_markup=get_main_keyboard(message.chat.id)
    )

async def cmd_help(message: types.Message):
    await cmd_start(message)

# --- Команда 1: количество столов ---
async def set_tables(callback: types.CallbackQuery, state: FSMContext):
    await callback.message.answer("Введите количество столов в ресторане:")
    await state.set_state(Form.waiting_for_tables_count)
    await callback.answer()

async def process_tables_count(message: types.Message, state: FSMContext):
    try:
        count = int(message.text.strip())
        if count <= 0:
            raise ValueError
    except ValueError:
        await message.answer("Пожалуйста, введите положительное число.")
        return

    chat_id = message.chat.id
    user_data.setdefault(chat_id, {})['tables_count'] = count
    await message.answer(f"✅ Количество столов сохранено: {count}")
    await state.clear()
    await message.answer("Выберите действие:", reply_markup=get_main_keyboard(chat_id))

# --- Команда 2: номер посадки ---
async def set_landing(callback: types.CallbackQuery, state: FSMContext):
    await callback.message.answer("Введите номер посадки:")
    await state.set_state(Form.waiting_for_landing)
    await callback.answer()

async def process_landing(message: types.Message, state: FSMContext):
    landing = message.text.strip()
    chat_id = message.chat.id
    user_data.setdefault(chat_id, {})['landing'] = landing
    await message.answer(f"✅ Номер посадки сохранён: {landing}")
    await state.clear()
    await message.answer("Выберите действие:", reply_markup=get_main_keyboard(chat_id))

# --- Команда 3: добавление заказа ---
async def add_order(callback: types.CallbackQuery, state: FSMContext):
    await callback.message.answer("Введите номер стола:")
    await state.set_state(Form.waiting_for_order_table)
    await callback.answer()

async def process_order_table(message: types.Message, state: FSMContext):
    await state.update_data(table=message.text.strip())
    await message.answer("Введите состав заказа:")
    await state.set_state(Form.waiting_for_order_composition)

async def process_order_composition(message: types.Message, state: FSMContext):
    await state.update_data(composition=message.text.strip())
    await message.answer("Введите комментарий к заказу:")
    await state.set_state(Form.waiting_for_order_comment)

async def process_order_comment(message: types.Message, state: FSMContext):
    comment = message.text.strip()
    data = await state.get_data()
    chat_id = message.chat.id

    order = {
        'table': data.get('table'),
        'composition': data.get('composition'),
        'comment': comment
    }
    user_data.setdefault(chat_id, {}).setdefault('orders', []).append(order)

    await message.answer(
        f"✅ Заказ для стола {order['table']} добавлен.\n"
        f"Состав: {order['composition']}\n"
        f"Комментарий: {order['comment']}"
    )
    await state.clear()
    await message.answer("Выберите действие:", reply_markup=get_main_keyboard(chat_id))

# --- Переключатель напоминаний ---
async def toggle_reminder(callback: types.CallbackQuery):
    chat_id = callback.message.chat.id
    if chat_id in active_reminders:
        del active_reminders[chat_id]
        status = "ВЫКЛ"
    else:
        active_reminders[chat_id] = datetime.now()
        status = "ВКЛ"

    await callback.message.edit_reply_markup(reply_markup=get_main_keyboard(chat_id))
    await callback.answer(f"Напоминания {status}")
    await callback.message.answer(f"🔔 Напоминания переключены: {status}")

# --- Фоновая задача для напоминаний каждые 30 минут ---
async def reminder_task(bot: Bot):
    while True:
        await asyncio.sleep(60)  # проверяем раз в минуту
        now = datetime.now()
        for chat_id, last_sent in list(active_reminders.items()):
            if now - last_sent >= timedelta(minutes=30):
                try:
                    await bot.send_message(
                        chat_id,
                        "Проверь свободные столы и напиши, какие есть и готовы к посадке"
                    )
                    active_reminders[chat_id] = now
                except Exception as e:
                    print(f"Ошибка отправки в чат {chat_id}: {e}")
                    # Если бот заблокирован, убираем из активных
                    del active_reminders[chat_id]

# --- Запуск ---
async def main():
    bot = Bot(token=BOT_TOKEN)
    dp = Dispatcher(storage=MemoryStorage())

    # Регистрация хендлеров
    dp.message.register(cmd_start, Command("start"))
    dp.message.register(cmd_help, Command("help"))

    dp.callback_query.register(set_tables, F.data == "set_tables")
    dp.callback_query.register(set_landing, F.data == "set_landing")
    dp.callback_query.register(add_order, F.data == "add_order")
    dp.callback_query.register(toggle_reminder, F.data == "toggle_reminder")

    dp.message.register(process_tables_count, Form.waiting_for_tables_count)
    dp.message.register(process_landing, Form.waiting_for_landing)
    dp.message.register(process_order_table, Form.waiting_for_order_table)
    dp.message.register(process_order_composition, Form.waiting_for_order_composition)
    dp.message.register(process_order_comment, Form.waiting_for_order_comment)

    # Запуск фоновой задачи
    asyncio.create_task(reminder_task(bot))

    print("Бот запущен...")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
