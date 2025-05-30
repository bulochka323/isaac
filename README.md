import asyncio
import datetime
import logging
from aiogram import Bot, Dispatcher, Router, types
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton
from aiogram.filters import Command

API_TOKEN = '7988509866:AAGF5vKCTorl56eC_Z8vD-LkLdCVN2CfSjg'

bot = Bot(token=API_TOKEN)
dp = Dispatcher()
router = Router()
dp.include_router(router)

menu = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(text="👶 Почати вирощувати Айзека")],
        [KeyboardButton(text="🌱 Стан Айзека"), KeyboardButton(text="🔄 Перезапустити Айзека")],
        [KeyboardButton(text="🍕 Піца"), KeyboardButton(text="🍏 Яблуко"), KeyboardButton(text="🍭 Цукерка")],
        [KeyboardButton(text="🛌 Відпочити"), KeyboardButton(text="⚽ Пограти"), KeyboardButton(text="📚 Почитати")],
        [KeyboardButton(text="⚔️ Викликати на дуель")]
    ],
    resize_keyboard=True
)

user_states = {}
duel_requests = {}  # user_id: chat_id

def get_user_state(user_id):
    if user_id not in user_states:
        user_states[user_id] = {
            "birth_time": datetime.datetime.now(),
            "age": 0,
            "hunger": 50,
            "energy": 50,
            "mood": 50,
            "alive": True
        }
    return user_states[user_id]

def update_age_and_life(state):
    now = datetime.datetime.now()
    state["age"] = (now - state["birth_time"]).days
    if state["hunger"] <= 0 or state["energy"] <= 0 or state["mood"] <= 0:
        state["alive"] = False

def get_stage(age):
    if age <= 1:
        return "👶 Немовля"
    elif age <= 3:
        return "🧒 Дитина"
    elif age <= 6:
        return "🧑 Підліток"
    else:
        return "👨 Дорослий"

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    await message.answer("Привіт! Це бот 'Вирости Айзека' 🌱", reply_markup=menu)

@dp.message(lambda msg: msg.text == "👶 Почати вирощувати Айзека")
async def start_isaac(message: types.Message):
    user_id = message.from_user.id
    user_states[user_id] = {
        "birth_time": datetime.datetime.now(),
        "age": 0,
        "hunger": 50,
        "energy": 50,
        "mood": 50,
        "alive": True
    }
    await message.answer("Айзек народився! 👶\nСлідкуй за його станом, щоб він не помер!")

@dp.message(lambda msg: msg.text == "🔄 Перезапустити Айзека")
async def restart_isaac(message: types.Message):
    user_id = message.from_user.id
    user_states[user_id] = {
        "birth_time": datetime.datetime.now(),
        "age": 0,
        "hunger": 50,
        "energy": 50,
        "mood": 50,
        "alive": True
    }
    await message.answer("🔄 Айзек був перезапущений!\nВін знову немовля 👶. Починай піклуватися про нього заново.")

@dp.message(lambda msg: msg.text == "🌱 Стан Айзека")
async def isaac_status(message: types.Message):
    state = get_user_state(message.from_user.id)
    update_age_and_life(state)

    if not state["alive"]:
        await message.answer("☠️ На жаль, Айзек помер.\nПочни знову натиснувши 👶.")
        return

    stage = get_stage(state["age"])
    await message.answer(
        f"{stage} Айзек:\n"
        f"📅 Вік: {state['age']} днів\n"
        f"🍗 Голод: {state['hunger']}/100\n"
        f"💤 Енергія: {state['energy']}/100\n"
        f"😊 Настрій: {state['mood']}/100"
    )

@dp.message(lambda msg: msg.text in ["🍕 Піца", "🍏 Яблуко", "🍭 Цукерка"])
async def feed_isaac(message: types.Message):
    state = get_user_state(message.from_user.id)
    if not state["alive"]:
        await message.answer("☠️ Айзек мертвий. Натисни 👶, щоб почати знову.")
        return

    food = message.text
    if food == "🍕 Піца":
        state["hunger"] = min(100, state["hunger"] + 25)
        response = "Айзек з'їв піцу! 🍕"
    elif food == "🍏 Яблуко":
        state["hunger"] = min(100, state["hunger"] + 15)
        response = "Айзек з'їв яблуко! 🍏"
    elif food == "🍭 Цукерка":
        state["hunger"] = min(100, state["hunger"] + 5)
        state["energy"] = min(100, state["energy"] + 5)
        state["mood"] = min(100, state["mood"] + 10)
        response = "Айзек поласував цукеркою! 🍭 Він трохи підбадьорився!"

    await message.answer(response)

@dp.message(lambda msg: msg.text in ["🛌 Відпочити", "⚽ Пограти", "📚 Почитати"])
async def activity_isaac(message: types.Message):
    state = get_user_state(message.from_user.id)
    if not state["alive"]:
        await message.answer("☠️ Айзек мертвий. Натисни 👶, щоб почати знову.")
        return

    activity = message.text
    if activity == "🛌 Відпочити":
        state["energy"] = min(100, state["energy"] + 30)
        state["mood"] = min(100, state["mood"] + 10)
        response = "Айзек добре відпочив. 🛌"
    elif activity == "⚽ Пограти":
        state["energy"] = max(0, state["energy"] - 10)
        state["hunger"] = max(0, state["hunger"] - 10)
        state["mood"] = min(100, state["mood"] + 25)
        response = "Айзек пограв і щасливий! ⚽"
    elif activity == "📚 Почитати":
        state["energy"] = max(0, state["energy"] - 5)
        state["mood"] = min(100, state["mood"] + 15)
        response = "Айзек прочитав книжку і в захваті! 📚"

    await message.answer(response)

@dp.message(lambda msg: msg.text == "⚔️ Викликати на дуель")
async def duel_request(message: types.Message):
    chat_id = message.chat.id
    user_id = message.from_user.id
    username = message.from_user.full_name

    if message.chat.type == "private":
        await message.answer("Дуелі можливі лише в групах.")
        return

    duel_requests[user_id] = chat_id
    await message.answer(f"{username} хоче дуель! Відповідь командою /duel_accept")

@dp.message(Command("duel_accept"))
async def duel_accept(message: types.Message):
    if message.chat.type == "private":
        await message.answer("Дуелі можливі лише в групах.")
        return

    responder_id = message.from_user.id
    challenger_id = next((uid for uid, cid in duel_requests.items() if cid == message.chat.id and uid != responder_id), None)

    if not challenger_id:
        await message.answer("Немає активного виклику на дуель.")
        return

    user1 = get_user_state(challenger_id)
    user2 = get_user_state(responder_id)

    if not user1["alive"] or not user2["alive"]:
        await message.answer("Обидва учасники повинні бути живими.")
        return

    score1 = user1["mood"] + user1["energy"] + user1["hunger"]
    score2 = user2["mood"] + user2["energy"] + user2["hunger"]

    winner_id, loser_id = (challenger_id, responder_id) if score1 >= score2 else (responder_id, challenger_id)

    user_states[winner_id]["mood"] = min(100, user_states[winner_id]["mood"] + 10)
    user_states[loser_id]["mood"] = max(0, user_states[loser_id]["mood"] - 10)

    winner_name = message.from_user.full_name if winner_id == responder_id else "Суперник"
    loser_name = message.from_user.full_name if loser_id == responder_id else "Суперник"

    await message.answer(
        f"⚔️ Дуель завершена!\n"
        f"🏆 Переможець: {winner_name} (+10 до настрою)\n"
        f"😵 Програвший: {loser_name} (-10 до настрою)"
    )

    del duel_requests[challenger_id]

# Запуск бота
async def main():
    logging.basicConfig(level=logging.INFO)
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
