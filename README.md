import telebot
import json
import time

TOKEN = "8186957673:AAGj0SFhIkY8QV-nDWQu-802KkdtO6YX36I"
bot = telebot.TeleBot(TOKEN)

ADMIN_USERNAME = "bumkach_stalar"
DATA_FILE = "users.json"
AD_COOLDOWN = 60  # секунды между нажатиями на "Смотреть рекламу"

def load_users():
    try:
        with open(DATA_FILE, "r") as f:
            return json.load(f)
    except:
        return {}

def save_users(data):
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=4)

@bot.message_handler(commands=['start'])
def start(msg):
    user_id = str(msg.from_user.id)
    users = load_users()
    if user_id not in users:
        users[user_id] = {
            "username": msg.from_user.username,
            "balance": 0,
            "ref": None,
            "refs": [],
            "kaspi": "",
            "withdraws": [],
            "last_ad_time": 0
        }
        ref = msg.text.split(" ")[1] if len(msg.text.split()) > 1 else None
        if ref and ref != user_id:
            users[user_id]["ref"] = ref
            if ref in users:
                users[ref]["balance"] += 1
                users[ref]["refs"].append(user_id)
    save_users(users)
    markup = telebot.types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.row("🎥 Смотреть рекламу", "💰 Баланс")
    markup.row("👥 Мои рефералы", "📤 Заказать выплату")
    bot.send_message(msg.chat.id, "💸 Хочешь зарабатывать с телефона?\n🎥 Смотри рекламу, приглашай друзей и выводи деньги!", reply_markup=markup)

@bot.message_handler(func=lambda message: message.text == "🎥 Смотреть рекламу")
def watch_ad(msg):
    user_id = str(msg.from_user.id)
    users = load_users()
    now = time.time()
    last_time = users[user_id].get("last_ad_time", 0)
    if now - last_time < AD_COOLDOWN:
        remaining = int(AD_COOLDOWN - (now - last_time))
        bot.send_message(msg.chat.id, f"⏳ Подождите {remaining} секунд перед следующим просмотром.")
        return
    users[user_id]["balance"] += 1
    users[user_id]["last_ad_time"] = now
    save_users(users)
    bot.send_message(msg.chat.id, "✅ Спасибо за просмотр! Вам начислен 1₸.")

@bot.message_handler(func=lambda message: message.text == "💰 Баланс")
def balance(msg):
    user_id = str(msg.from_user.id)
    users = load_users()
    bal = users.get(user_id, {}).get("balance", 0)
    bot.send_message(msg.chat.id, f"💰 Ваш баланс: {bal}₸")

@bot.message_handler(func=lambda message: message.text == "👥 Мои рефералы")
def referrals(msg):
    user_id = str(msg.from_user.id)
    users = load_users()
    refs = users.get(user_id, {}).get("refs", [])
    bot.send_message(msg.chat.id, f"👥 У вас {len(refs)} рефералов.\n🧾 Список ID: {', '.join(refs) if refs else 'нет'}")

@bot.message_handler(func=lambda message: message.text == "📤 Заказать выплату")
def withdraw(msg):
    msg_ask = bot.send_message(msg.chat.id, "💳 Введите номер Kaspi для вывода:")
    bot.register_next_step_handler(msg_ask, process_kaspi_step)

def process_kaspi_step(msg):
    user_id = str(msg.from_user.id)
    users = load_users()
    users[user_id]["kaspi"] = msg.text.strip()
    save_users(users)
    msg_sum = bot.send_message(msg.chat.id, "💰 Введите сумму для вывода:")
    bot.register_next_step_handler(msg_sum, process_withdraw_amount)

def process_withdraw_amount(msg):
    user_id = str(msg.from_user.id)
    amount = int(msg.text.strip())
    users = load_users()
    if users[user_id]["balance"] >= amount:
        users[user_id]["balance"] -= amount
        users[user_id]["withdraws"].append({
            "amount": amount,
            "kaspi": users[user_id]["kaspi"]
        })
        save_users(users)
        bot.send_message(msg.chat.id, f"✅ Заявка на {amount}₸ успешно отправлена. Выплата на: {users[user_id]['kaspi']}")
    else:
        bot.send_message(msg.chat.id, "❌ Недостаточно средств на балансе.")

@bot.message_handler(commands=['admin'])
def admin(msg):
    if msg.from_user.username != ADMIN_USERNAME:
        return bot.send_message(msg.chat.id, "⛔ Доступ только для администратора.")
    users = load_users()
    total = len(users)
    total_balance = sum(u["balance"] for u in users.values())
    text = f"👑 Админ-панель:\n👥 Пользователей: {total}\n💰 Всего баллов: {total_balance}\n📬 Последние выплаты:"
    for u in users.values():
        for w in u.get("withdraws", [])[-3:]:
            text += f"\n - {w['kaspi']}: {w['amount']}₸"
    bot.send_message(msg.chat.id, text)

bot.polling()
