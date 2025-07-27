import logging
from telegram import Update, InlineKeyboardMarkup, InlineKeyboardButton, ChatAdministratorRights
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    ConversationHandler,
    ContextTypes,
    filters,
)

# Включаем логгирование (для отладки)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Список ID админов и ID группы
ADMINS = [123456789, 987654321]  # 🔁 Замени на свои Telegram ID
GROUP_ID = -1001234567890        # 🔁 Замени на свой ID группы

# Состояния анкеты
CHOOSING_ROLE, GET_NAME, GET_AGE, GET_HEIGHT, GET_LANGS, GET_EMAIL = range(6)
user_data = {}

roles = {
    "Модератор": "mod",
    "Администратор": "admin",
    "Хелпер": "helper"
}

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [
            InlineKeyboardButton("Модератор", callback_data='Модератор'),
            InlineKeyboardButton("Администратор", callback_data='Администратор'),
            InlineKeyboardButton("Хелпер", callback_data='Хелпер')
        ]
    ]
    await update.message.reply_text(
        "👋 Кем хотите быть?\nВыберите роль:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )
    return CHOOSING_ROLE

async def choose_role(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    role = query.data
    user_data[query.from_user.id] = {"role": role}
    await query.message.reply_text("❗ Укажите ваше имя и фамилию:")
    return GET_NAME

async def get_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data[update.message.from_user.id]["name"] = update.message.text
    await update.message.reply_text("📅 Укажите ваш возраст:")
    return GET_AGE

async def get_age(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data[update.message.from_user.id]["age"] = update.message.text
    await update.message.reply_text("📏 Укажите ваш рост:")
    return GET_HEIGHT

async def get_height(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data[update.message.from_user.id]["height"] = update.message.text
    await update.message.reply_text("🌐 Укажите, какими языками владеете:")
    return GET_LANGS

async def get_langs(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data[update.message.from_user.id]["langs"] = update.message.text
    await update.message.reply_text("📧 Введите вашу почту:")
    return GET_EMAIL

async def get_email(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.message.from_user.id
    data = user_data[uid]
    data["email"] = update.message.text

    text = (
        f"✅ Новая заявка на вступление:\n\n"
        f"👤 Имя: {data['name']}\n"
        f"🎯 Роль: {data['role']}\n"
        f"📅 Возраст: {data['age']} лет\n"
        f"📏 Рост: {data['height']} см\n"
        f"🌐 Языки: {data['langs']}\n"
        f"📧 Почта: {data['email']}\n\n"
        "🧵 Отправлено из бота."
    )

    buttons = [
        [
            InlineKeyboardButton("✅ Принять", callback_data=f"accept_{uid}"),
            InlineKeyboardButton("❌ Отклонить", callback_data=f"reject_{uid}")
        ]
    ]

    for admin_id in ADMINS:
        await context.bot.send_message(chat_id=admin_id, text=text, reply_markup=InlineKeyboardMarkup(buttons))

    await update.message.reply_text("📨 Ваша заявка отправлена администрации!")
    return ConversationHandler.END

async def handle_decision(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    action, uid_str = query.data.split('_')
    uid = int(uid_str)

    if action == "accept":
        await context.bot.send_message(chat_id=uid, text="🎉 Поздравляю, вас приняли!")
        if uid in user_data:
            role = roles[user_data[uid]["role"]]
            rights = ChatAdministratorRights(
                is_anonymous=False,
                can_change_info=False,
                can_delete_messages=True,
                can_restrict_members=True,
                can_invite_users=True,
                can_pin_messages=(role == "admin"),
                can_manage_topics=(role == "helper"),
                can_promote_members=(role == "admin")
            )
            try:
                await context.bot.promote_chat_member(
                    chat_id=GROUP_ID,
                    user_id=uid,
                    privileges=rights
                )
                await context.bot.set_chat_administrator_custom_title(GROUP_ID, uid, user_data[uid]["role"])
            except Exception as e:
                await query.message.reply_text(f"❗ Ошибка назначения: {e}")

    elif action == "reject":
        await context.bot.send_message(chat_id=uid, text="😔 Простите, вашу заявку не приняли.")

    await query.edit_message_text("✅ Решение обработано.")

def main():
    app = ApplicationBuilder().token("🔐_ТВОЙ_ТОКЕН_ТУТ_").build()

    conv_handler = ConversationHandler(
        entry_points=[CommandHandler("start", start)],
        states={
            CHOOSING_ROLE: [CallbackQueryHandler(choose_role)],
            GET_NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_name)],
            GET_AGE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_age)],
