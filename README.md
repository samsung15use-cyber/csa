import telebot
import sqlite3
import time
import os
from telebot import types
from datetime import datetime
import random
import string
from dotenv import load_dotenv

# ==================== ЗАГРУЗКА КОНФИГУРАЦИИ ИЗ ПЕРЕМЕННЫХ ОКРУЖЕНИЯ ====================
load_dotenv()

# ==================== КОНФИГУРАЦИЯ ====================
BOT_TOKEN = os.getenv("8495865415:AAHlUTXRynIkx9dHOzclHKc0G_sF4qZgffg")
# Получаем ID администраторов из переменной окружения (можно передать строку с несколькими ID, разделенными запятыми)
ADMIN_IDS_STR = os.getenv("ADMIN_IDS",1417003901 "")
ADMIN_IDS = [int(id.strip()) for id in ADMIN_IDS_STR.split(",") if id.strip()]

DB_NAME = os.getenv("DB_NAME", "gifts_bot.db")

# ==================== НАСТРОЙКИ РЕФЕРАЛЬНОЙ СИСТЕМЫ ====================
REFERRAL_BONUS = int(os.getenv("REFERRAL_BONUS", 5))
REFERRAL_PERCENT = int(os.getenv("REFERRAL_PERCENT", 5))

# ==================== ТОВАРЫ (ПОДАРКИ) С ЭМОДЗИ ====================
# Товары можно также вынести в .env, но для простоты оставим здесь.
GIFTS = {
    "🎁 Подарок": 22,
    "🌹 Роза": 22,
    "🚀 Ракета": 47,
    "💐 Букет": 47,
    "🎂 Торт": 47,
    "🍾 Шампанское": 47,
    "💎 Алмаз": 97,
    "💍 Кольцо": 97,
    "🏆 Кубок": 97,
    "🎄 Ёлочка": 50,
    "🧸 Новогодний мишка": 50,
    "❤️ Сердце 14 февраля": 50,
    "🧸 Мишка 14 февраля": 50,
    "🐻 Мишка": 13,
    "💖 Сердце": 13
}

# Проверяем, что токен загружен
if not BOT_TOKEN:
    raise ValueError("Не задан BOT_TOKEN в файле .env")

bot = telebot.TeleBot(BOT_TOKEN)

# ==================== ИНИЦИАЛИЗАЦИЯ БАЗЫ ДАННЫХ ====================
def init_database():
    """Инициализирует базу данных, создавая все необходимые таблицы."""
    # Если нужно пересоздать БД при каждом запуске (для разработки), можно раскомментировать:
    # if os.path.exists(DB_NAME):
    #     os.remove(DB_NAME)
    #     print("🗑️ Старая база данных удалена")
    
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        first_name TEXT,
        balance REAL DEFAULT 0,
        total_earned REAL DEFAULT 0,
        gifts_received INTEGER DEFAULT 0,
        referrer_id INTEGER DEFAULT NULL,
        referral_count INTEGER DEFAULT 0,
        referral_earnings REAL DEFAULT 0,
        registration_date TEXT,
        notifications INTEGER DEFAULT 1,
        invite_link TEXT UNIQUE,
        referral_code TEXT UNIQUE
    )
    ''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS sponsors (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        link TEXT NOT NULL,
        chat_id TEXT NOT NULL UNIQUE,
        date_added TEXT
    )
    ''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS referrals (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        referral_id INTEGER,
        date TEXT,
        earnings REAL DEFAULT 0
    )
    ''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS transactions (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        amount REAL,
        gift_name TEXT,
        transaction_type TEXT,
        description TEXT,
        date TEXT
    )
    ''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS temp_mailing (
        admin_id INTEGER PRIMARY KEY,
        text TEXT
    )
    ''')
    
    conn.commit()
    conn.close()
    print("✅ База данных готова (таблицы созданы или уже существуют)")

# ==================== РАБОТА СО СПОНСОРАМИ ====================
def get_sponsors():
    """Возвращает список всех спонсоров."""
    try:
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT name, link, chat_id FROM sponsors")
        sponsors = cursor.fetchall()
        conn.close()
        return [{"name": s[0], "link": s[1], "chat_id": s[2]} for s in sponsors]
    except Exception as e:
        print(f"Ошибка получения спонсоров: {e}")
        return []

def add_sponsor(name, link, chat_id):
    """Добавляет нового спонсора."""
    try:
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO sponsors (name, link, chat_id, date_added) VALUES (?, ?, ?, ?)",
            (name, link, chat_id, datetime.now().strftime("%Y-%m-%d %H:%M:%S"))
        )
        conn.commit()
        conn.close()
        return True
    except Exception as e:
        print(f"Ошибка добавления спонсора: {e}")
        return False

def delete_sponsor(chat_id):
    """Удаляет спонсора по chat_id."""
    try:
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("DELETE FROM sponsors WHERE chat_id = ?", (chat_id,))
        conn.commit()
        conn.close()
        return True
    except Exception as e:
        print(f"Ошибка удаления спонсора: {e}")
        return False

# ==================== РАБОТА С ПОЛЬЗОВАТЕЛЯМИ ====================
def get_user(user_id):
    """Возвращает данные пользователя по его ID."""
    try:
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
        user = cursor.fetchone()
        conn.close()
        return user
    except Exception as e:
        print(f"Ошибка получения пользователя: {e}")
        return None

def generate_unique_code():
    """Генерирует уникальный код для реферальной ссылки."""
    while True:
        code = ''.join(random.choices(string.ascii_letters + string.digits, k=8))
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT referral_code FROM users WHERE referral_code = ?", (code,))
        exists = cursor.fetchone()
        conn.close()
        if not exists:
            return code

def register_user(message):
    """Регистрирует нового пользователя или обновляет данные существующего."""
    try:
        user_id = message.from_user.id
        username = message.from_user.username or ""
        first_name = message.from_user.first_name or ""
        
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        
        cursor.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
        user = cursor.fetchone()
        
        if not user:
            referral_code = generate_unique_code()
            bot_username = bot.get_me().username
            invite_link = f"https://t.me/{bot_username}?start={referral_code}"
            
            referrer_id = None
            if message.text and message.text.startswith('/start '):
                ref_code = message.text[7:]
                cursor.execute("SELECT user_id FROM users WHERE referral_code = ?", (ref_code,))
                result = cursor.fetchone()
                if result:
                    referrer_id = result[0]
                    print(f"👤 Новый пользователь пришел по реферальной ссылке от {referrer_id}")
            
            cursor.execute('''
            INSERT INTO users 
            (user_id, username, first_name, registration_date, invite_link, referral_code, referrer_id) 
            VALUES (?, ?, ?, ?, ?, ?, ?)
            ''', (
                user_id, username, first_name, 
                datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                invite_link, referral_code, referrer_id
            ))
            
            if referrer_id:
                cursor.execute('''
                UPDATE users 
                SET balance = balance + ?, 
                    total_earned = total_earned + ?,
                    referral_count = referral_count + 1 
                WHERE user_id = ?
                ''', (REFERRAL_BONUS, REFERRAL_BONUS, referrer_id))
                
                cursor.execute('''
                INSERT INTO referrals (user_id, referral_id, date, earnings)
                VALUES (?, ?, ?, ?)
                ''', (referrer_id, user_id, datetime.now().strftime("%Y-%m-%d %H:%M:%S"), REFERRAL_BONUS))
                
                try:
                    bot.send_message(
                        referrer_id,
                        f"🎉 У тебя новый реферал!\n\n"
                        f"👤 {first_name or username or 'Пользователь'}\n"
                        f"💰 Начислено: +{REFERRAL_BONUS} ⭐"
                    )
                except Exception as e:
                    print(f"Не удалось отправить уведомление рефереру: {e}")
                
                print(f"✅ Реферал зарегистрирован: {user_id} приглашен {referrer_id}")
            
            conn.commit()
            print(f"✅ Новый пользователь зарегистрирован: {user_id}")
            print(f"🔗 Реферальная ссылка: {invite_link}")
        
        conn.close()
    except Exception as e:
        print(f"❌ Ошибка регистрации: {e}")

# ==================== ПРОВЕРКА ПОДПИСКИ ====================
def check_subscription(user_id):
    """
    Проверяет, подписан ли пользователь на всех спонсоров.
    Возвращает (True, []) если подписан на всех,
    иначе (False, список неотмеченных спонсоров).
    """
    sponsors = get_sponsors()
    if not sponsors:
        return True, []
    
    not_subscribed = []
    try:
        for sponsor in sponsors:
            try:
                chat_id = sponsor['chat_id']
                if str(chat_id).startswith('@'):
                    chat = bot.get_chat(chat_id)
                    chat_id = chat.id
                
                member = bot.get_chat_member(chat_id, user_id)
                if member.status in ['left', 'kicked']:
                    not_subscribed.append(sponsor)
            except Exception as e:
                print(f"Ошибка проверки подписки на {sponsor['name']}: {e}")
                not_subscribed.append(sponsor)
        
        return len(not_subscribed) == 0, not_subscribed
    except Exception as e:
        print(f"Общая ошибка проверки подписки: {e}")
        return False, sponsors

# ==================== ПРОВЕРКА АДМИНА ====================
def check_admin_status(user_id):
    """Проверяет, является ли пользователь администратором."""
    return user_id in ADMIN_IDS

# ==================== ИНЛАЙН КЛАВИАТУРЫ ====================
def main_menu_keyboard(user_id):
    """Клавиатура главного меню."""
    keyboard = types.InlineKeyboardMarkup(row_width=2)
    buttons = [
        types.InlineKeyboardButton("🎁 Подарки", callback_data="menu_gifts"),
        types.InlineKeyboardButton("⭐️ Заработать", callback_data="menu_earn"),
        types.InlineKeyboardButton("👤 Профиль", callback_data="menu_profile"),
    ]
    
    if check_admin_status(user_id):
        buttons.append(types.InlineKeyboardButton("⚙️ Админ панель", callback_data="menu_admin"))
    
    keyboard.add(*buttons)
    return keyboard

def admin_menu_keyboard():
    """Клавиатура админ-панели."""
    keyboard = types.InlineKeyboardMarkup(row_width=2)
    buttons = [
        types.InlineKeyboardButton("📊 Статистика", callback_data="admin_stats"),
        types.InlineKeyboardButton("👥 Пользователи", callback_data="admin_users"),
        types.InlineKeyboardButton("📢 Спонсоры", callback_data="admin_sponsors"),
        types.InlineKeyboardButton("📨 Рассылка", callback_data="admin_mailing"),
        types.InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")
    ]
    keyboard.add(*buttons)
    return keyboard

def subscription_keyboard():
    """Клавиатура для проверки подписки."""
    keyboard = types.InlineKeyboardMarkup(row_width=1)
    sponsors = get_sponsors()
    for sponsor in sponsors:
        keyboard.add(types.InlineKeyboardButton(
            text=f"📢 {sponsor['name']}", 
            url=sponsor['link']
        ))
    keyboard.add(types.InlineKeyboardButton(
        text="✅ Я подписался", 
        callback_data="check_subscription"
    ))
    return keyboard

def gifts_keyboard(user_balance):
    """Клавиатура со списком подарков."""
    keyboard = types.InlineKeyboardMarkup(row_width=2)
    row = []
    for i, (gift, price) in enumerate(GIFTS.items(), 1):
        button = types.InlineKeyboardButton(
            text=f"{gift} - {price} ⭐", 
            callback_data=f"buy_{gift}"
        )
        row.append(button)
        if i % 2 == 0:
            keyboard.row(*row)
            row = []
    if row:
        keyboard.row(*row)
    keyboard.row(types.InlineKeyboardButton("🔙 Назад", callback_data="back_to_main"))
    return keyboard

def earn_keyboard():
    """Клавиатура раздела заработка."""
    keyboard = types.InlineKeyboardMarkup(row_width=2)
    buttons = [
        types.InlineKeyboardButton("📋 Скопировать ссылку", callback_data="copy_link"),
        types.InlineKeyboardButton("👥 Мои рефералы", callback_data="my_referrals"),
        types.InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")
    ]
    keyboard.add(*buttons)
    return keyboard

def back_keyboard(dest="main"):
    """Универсальная кнопка 'Назад'."""
    keyboard = types.InlineKeyboardMarkup()
    if dest == "admin":
        keyboard.add(types.InlineKeyboardButton("🔙 Назад", callback_data="back_to_admin"))
    else:
        keyboard.add(types.InlineKeyboardButton("🔙 Назад", callback_data="back_to_main"))
    return keyboard

def back_to_earn_keyboard():
    """Кнопка возврата к разделу заработка."""
    keyboard = types.InlineKeyboardMarkup()
    keyboard.add(types.InlineKeyboardButton("🔙 Назад к заработку", callback_data="menu_earn"))
    return keyboard

def sponsors_management_keyboard():
    """Клавиатура управления спонсорами."""
    keyboard = types.InlineKeyboardMarkup(row_width=2)
    buttons = [
        types.InlineKeyboardButton("➕ Добавить", callback_data="sponsor_add"),
        types.InlineKeyboardButton("❌ Удалить", callback_data="sponsor_del"),
        types.InlineKeyboardButton("🗑️ Очистить", callback_data="sponsor_clear"),
        types.InlineKeyboardButton("🔙 Назад", callback_data="back_to_admin")
    ]
    keyboard.add(*buttons)
    return keyboard

# ==================== ОБРАБОТЧИКИ КОМАНД ====================
@bot.message_handler(commands=['start'])
def start_command(message):
    try:
        user_id = message.from_user.id
        register_user(message)
        
        is_subscribed, not_subscribed = check_subscription(user_id)
        
        if not is_subscribed:
            keyboard = types.InlineKeyboardMarkup(row_width=1)
            for sponsor in not_subscribed:
                keyboard.add(types.InlineKeyboardButton(
                    text=f"📢 {sponsor['name']}", 
                    url=sponsor['link']
                ))
            keyboard.add(types.InlineKeyboardButton(
                text="✅ Я подписался", 
                callback_data="check_subscription"
            ))
            
            bot.send_message(
                user_id,
                "🚫 **Для использования бота необходимо подписаться на спонсоров:**\n\n"
                "Подпишись на каналы ниже и нажми кнопку 'Я подписался'",
                parse_mode="Markdown",
                reply_markup=keyboard
            )
            return
        
        welcome_text = (
            "🎁 Добро пожаловать в бот подарков!\n\n"
            "✨ Здесь ты можешь зарабатывать звезды\n"
            "👥 Приглашай друзей и получай бонусы\n"
            f"💫 Зарабатывай {REFERRAL_PERCENT}% от трат рефералов\n"
            f"🎁 Бонус за друга: {REFERRAL_BONUS} ⭐"
        )
        
        bot.send_message(user_id, welcome_text, reply_markup=main_menu_keyboard(user_id))
    except Exception as e:
        print(f"Ошибка в start: {e}")

@bot.message_handler(commands=['ref'])
def ref_command(message):
    """Показать реферальную ссылку"""
    try:
        user_id = message.from_user.id
        is_subscribed, not_subscribed = check_subscription(user_id)
        
        if not is_subscribed:
            keyboard = types.InlineKeyboardMarkup(row_width=1)
            for sponsor in not_subscribed:
                keyboard.add(types.InlineKeyboardButton(
                    text=f"📢 {sponsor['name']}", 
                    url=sponsor['link']
                ))
            keyboard.add(types.InlineKeyboardButton(
                text="✅ Я подписался", 
                callback_data="check_subscription"
            ))
            
            bot.send_message(
                user_id,
                "🚫 **Для использования бота необходимо подписаться на спонсоров:**\n\n"
                "Подпишись на каналы ниже и нажми кнопку 'Я подписался'",
                parse_mode="Markdown",
                reply_markup=keyboard
            )
            return
            
        user = get_user(message.from_user.id)
        if user:
            bot.send_message(
                message.chat.id,
                f"🔗 **Твоя реферальная ссылка:**\n`{user[11]}`\n\n"
                f"📊 **Статистика:**\n"
                f"• Приглашено: {user[7]} чел.\n"
                f"• Заработано: {user[8]} ⭐",
                parse_mode="Markdown"
            )
    except Exception as e:
        print(f"Ошибка в ref: {e}")

@bot.callback_query_handler(func=lambda call: True)
def handle_callbacks(call):
    try:
        user_id = call.from_user.id
        user = get_user(user_id)
        
        # Проверка подписки для всех действий кроме проверки подписки
        if call.data != "check_subscription" and not call.data.startswith("admin_"):
            is_subscribed, not_subscribed = check_subscription(user_id)
            if not is_subscribed:
                bot.answer_callback_query(call.id, "❌ Сначала подпишись на спонсоров!", show_alert=True)
                
                keyboard = types.InlineKeyboardMarkup(row_width=1)
                for sponsor in not_subscribed:
                    keyboard.add(types.InlineKeyboardButton(
                        text=f"📢 {sponsor['name']}", 
                        url=sponsor['link']
                    ))
                keyboard.add(types.InlineKeyboardButton(
                    text="✅ Я подписался", 
                    callback_data="check_subscription"
                ))
                
                try:
                    bot.edit_message_text(
                        "🚫 **Для использования бота необходимо подписаться на спонсоров:**\n\n"
                        "Подпишись на каналы ниже и нажми кнопку 'Я подписался'",
                        user_id,
                        call.message.message_id,
                        parse_mode="Markdown",
                        reply_markup=keyboard
                    )
                except Exception as e:
                    print(f"Ошибка редактирования сообщения: {e}")
                return
        
        # ===== ГЛАВНОЕ МЕНЮ =====
        if call.data == "back_to_main":
            bot.edit_message_text(
                "🎁 Cassetov stars:",
                user_id,
                call.message.message_id,
                reply_markup=main_menu_keyboard(user_id)
            )
        
        elif call.data == "back_to_admin":
            if check_admin_status(user_id):
                bot.edit_message_text(
                    "⚙️ Админ панель:",
                    user_id,
                    call.message.message_id,
                    reply_markup=admin_menu_keyboard()
                )
        
        # ===== МЕНЮ ПОЛЬЗОВАТЕЛЯ =====
        elif call.data == "menu_gifts":
            if user:
                text = f"🎁 **Доступные подарки**\n💰 Твой баланс: {user[3]} ⭐\n\nВыбери подарок для покупки:"
                bot.edit_message_text(
                    text,
                    user_id,
                    call.message.message_id,
                    parse_mode="Markdown",
                    reply_markup=gifts_keyboard(user[3])
                )
        
        elif call.data == "menu_earn":
            if user:
                ref_link = user[11]  # invite_link
                text = (
                    "💰 **Как заработать звезды?**\n\n"
                    "👥 **Реферальная система**\n"
                    f"• За каждого друга: +{REFERRAL_BONUS} ⭐\n"
                    f"• {REFERRAL_PERCENT}% от всех трат друзей\n\n"
                    "🔗 **Твоя реферальная ссылка:**\n"
                    f"`{ref_link}`\n\n"
                    "📊 **Твоя статистика:**\n"
                    f"• Приглашено друзей: {user[7]} чел.\n"
                    f"• Заработано с рефералов: {user[8]} ⭐\n"
                    f"• Всего заработано: {user[4]} ⭐"
                )
                bot.edit_message_text(
                    text,
                    user_id,
                    call.message.message_id,
                    parse_mode="Markdown",
                    reply_markup=earn_keyboard()
                )
        
        elif call.data == "menu_profile":
            if user:
                conn = sqlite3.connect(DB_NAME)
                cursor = conn.cursor()
                cursor.execute("SELECT COUNT(*) FROM transactions WHERE user_id = ? AND transaction_type = 'purchase'", (user_id,))
                gifts_bought = cursor.fetchone()[0]
                conn.close()
                
                text = (
                    f"👤 **Профиль пользователя**\n\n"
                    f"🆔 ID: `{user_id}`\n"
                    f"📅 Регистрация: {user[9]}\n\n"
                    f"💰 **Баланс:** {user[3]} ⭐\n"
                    f"🎁 Куплено подарков: {gifts_bought}\n"
                    f"👥 Приглашено друзей: {user[7]}\n"
                    f"💫 Заработано всего: {user[4]} ⭐"
                )
                bot.edit_message_text(
                    text,
                    user_id,
                    call.message.message_id,
                    parse_mode="Markdown",
                    reply_markup=back_keyboard()
                )
        
        # ===== РЕФЕРАЛЬНЫЕ ФУНКЦИИ =====
        elif call.data == "copy_link":
            if user:
                ref_link = user[11]
                
                bot.send_message(
                    user_id,
                    f"🔗 **Твоя реферальная ссылка:**\n\n"
                    f"`{ref_link}`\n\n"
                    f"📋 **Как скопировать:**\n"
                    f"1. Нажми на текст ссылки выше\n"
                    f"2. Удерживай палец на ссылке (на телефоне) или выдели её мышкой (на компьютере)\n"
                    f"3. Выбери \"Копировать\" в появившемся меню\n\n"
                    f"📤 **Или отправь эту ссылку друзьям!**",
                    parse_mode="Markdown"
                )
                
                share_keyboard = types.InlineKeyboardMarkup()
                share_keyboard.add(types.InlineKeyboardButton(
                    text="📤 Поделиться ссылкой",
                    url=f"https://t.me/share/url?url={ref_link}&text=%F0%9F%8E%81%20%D0%9F%D0%BE%D0%BB%D1%83%D1%87%D0%B0%D0%B9%20%D0%BF%D0%BE%D0%B4%D0%B0%D1%80%D0%BA%D0%B8%20%D0%B7%D0%B0%20%D0%B7%D0%B2%D0%B5%D0%B7%D0%B4%D1%8B%21%20%D0%9F%D1%80%D0%B8%D1%81%D0%BE%D0%B5%D0%B4%D0%B8%D0%BD%D1%8F%D0%B9%D1%81%D1%8F%20%D0%BF%D0%BE%20%D0%BC%D0%BE%D0%B5%D0%B9%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B5%3A"
                ))
                share_keyboard.add(types.InlineKeyboardButton(
                    text="🔙 Назад к заработку",
                    callback_data="menu_earn"
                ))
                
                bot.send_message(
                    user_id,
                    "📤 Поделись ссылкой с друзьями:",
                    reply_markup=share_keyboard
                )
                
                bot.answer_callback_query(call.id, "✅ Ссылка отправлена в чат!", show_alert=False)
        
        elif call.data == "my_referrals":
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            cursor.execute('''
            SELECT u.username, u.first_name, r.date, r.earnings 
            FROM referrals r
            JOIN users u ON r.referral_id = u.user_id
            WHERE r.user_id = ?
            ORDER BY r.date DESC
            ''', (user_id,))
            referrals = cursor.fetchall()
            conn.close()
            
            if not referrals:
                text = "👥 У тебя пока нет рефералов.\nПриглашай друзей по своей ссылке и получай бонусы!"
            else:
                text = "👥 **Твои рефералы:**\n\n"
                for i, ref in enumerate(referrals, 1):
                    name = ref[1] or f"@{ref[0]}" if ref[0] else "Пользователь"
                    date = ref[2][:10] if ref[2] else "неизвестно"
                    earnings = ref[3]
                    text += f"{i}. {name}\n   📅 {date} | 💰 {earnings} ⭐\n\n"
            
            bot.edit_message_text(
                text,
                user_id,
                call.message.message_id,
                parse_mode="Markdown",
                reply_markup=back_keyboard()
            )
        
        # ===== ПОКУПКА ПОДАРКОВ =====
        elif call.data.startswith('buy_'):
            gift_name = call.data[4:]
            price = GIFTS.get(gift_name)
            
            if not price or not user:
                return
            
            if user[3] < price:
                bot.answer_callback_query(call.id, f"❌ Недостаточно звезд! Нужно: {price} ⭐", show_alert=True)
                return
            
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            
            cursor.execute("UPDATE users SET balance = balance - ?, gifts_received = gifts_received + 1 WHERE user_id = ?", 
                          (price, user_id))
            
            cursor.execute('''
            INSERT INTO transactions (user_id, amount, gift_name, transaction_type, description, date)
            VALUES (?, ?, ?, ?, ?, ?)
            ''', (user_id, -price, gift_name, "purchase", f"Покупка {gift_name}", 
                  datetime.now().strftime("%Y-%m-%d %H:%M:%S")))
            
            cursor.execute("SELECT referrer_id FROM users WHERE user_id = ?", (user_id,))
            referrer = cursor.fetchone()
            
            if referrer and referrer[0]:
                referrer_earnings = price * REFERRAL_PERCENT / 100
                cursor.execute("UPDATE users SET balance = balance + ?, total_earned = total_earned + ?, referral_earnings = referral_earnings + ? WHERE user_id = ?", 
                              (referrer_earnings, referrer_earnings, referrer_earnings, referrer[0]))
                
                cursor.execute('''
                INSERT INTO transactions (user_id, amount, transaction_type, description, date)
                VALUES (?, ?, ?, ?, ?)
                ''', (referrer[0], referrer_earnings, "referral_commission", 
                      f"{REFERRAL_PERCENT}% от покупки реферала", 
                      datetime.now().strftime("%Y-%m-%d %H:%M:%S")))
                
                cursor.execute("UPDATE referrals SET earnings = earnings + ? WHERE user_id = ? AND referral_id = ?", 
                              (referrer_earnings, referrer[0], user_id))
            
            conn.commit()
            conn.close()
            
            bot.answer_callback_query(call.id, f"✅ Покупка совершена!", show_alert=False)
            
            delivery_text = (
                f"🎁 **Покупка успешно оформлена!**\n\n"
                f"Ты купил: {gift_name}\n"
                f"Цена: {price} ⭐\n"
                f"Остаток на балансе: {user[3] - price} ⭐\n\n"
                f"⏱️ **Подарок будет доставлен в течение нескольких минут!**\n"
                f"Ожидай, скоро получишь свой подарок!"
            )
            
            bot.send_message(
                user_id,
                delivery_text,
                parse_mode="Markdown"
            )
            
            user = get_user(user_id)
            bot.edit_message_text(
                f"🎁 **Доступные подарки**\n💰 Твой баланс: {user[3]} ⭐\n\nВыбери подарок для покупки:",
                user_id,
                call.message.message_id,
                parse_mode="Markdown",
                reply_markup=gifts_keyboard(user[3])
            )
        
        # ===== ПРОВЕРКА ПОДПИСКИ =====
        elif call.data == "check_subscription":
            is_subscribed, not_subscribed = check_subscription(user_id)
            
            if is_subscribed:
                bot.edit_message_text(
                    "✅ Спасибо за подписку! Добро пожаловать!",
                    user_id,
                    call.message.message_id
                )
                bot.send_message(user_id, "🎁 Cassetov Stars:", reply_markup=main_menu_keyboard(user_id))
            else:
                keyboard = types.InlineKeyboardMarkup(row_width=1)
                for sponsor in not_subscribed:
                    keyboard.add(types.InlineKeyboardButton(
                        text=f"📢 {sponsor['name']}", 
                        url=sponsor['link']
                    ))
                keyboard.add(types.InlineKeyboardButton(
                    text="✅ Я подписался", 
                    callback_data="check_subscription"
                ))
                
                bot.answer_callback_query(call.id, "❌ Вы не подписались на всех спонсоров!", show_alert=True)
                bot.edit_message_text(
                    "🚫 **Для использования бота необходимо подписаться на спонсоров:**\n\n"
                    "Подпишись на каналы ниже и нажми кнопку 'Я подписался'",
                    user_id,
                    call.message.message_id,
                    parse_mode="Markdown",
                    reply_markup=keyboard
                )
        
        # ===== АДМИН МЕНЮ =====
        elif call.data == "menu_admin":
            if check_admin_status(user_id):
                bot.edit_message_text(
                    "⚙️ Админ панель:",
                    user_id,
                    call.message.message_id,
                    reply_markup=admin_menu_keyboard()
                )
            else:
                bot.answer_callback_query(call.id, "❌ У вас нет прав администратора!", show_alert=True)
        
        elif call.data == "admin_stats":
            if check_admin_status(user_id):
                conn = sqlite3.connect(DB_NAME)
                cursor = conn.cursor()
                
                cursor.execute("SELECT COUNT(*) FROM users")
                total_users = cursor.fetchone()[0]
                
                cursor.execute("SELECT COUNT(*) FROM users WHERE DATE(registration_date) = DATE('now')")
                new_today = cursor.fetchone()[0]
                
                cursor.execute("SELECT SUM(amount) FROM transactions WHERE transaction_type = 'purchase'")
                total_purchases = cursor.fetchone()[0] or 0
                
                cursor.execute("SELECT SUM(referral_earnings) FROM users")
                total_referral_paid = cursor.fetchone()[0] or 0
                
                cursor.execute("SELECT SUM(balance) FROM users")
                total_balance = cursor.fetchone()[0] or 0
                
                conn.close()
                
                text = (
                    "📊 **Статистика бота**\n\n"
                    f"👥 Всего пользователей: {total_users}\n"
                    f"📅 Новых сегодня: {new_today}\n"
                    f"💰 Всего покупок: {abs(total_purchases)} ⭐\n"
                    f"💫 Выплачено рефералам: {total_referral_paid} ⭐\n"
                    f"💎 Общий баланс: {total_balance} ⭐"
                )
                bot.edit_message_text(
                    text,
                    user_id,
                    call.message.message_id,
                    parse_mode="Markdown",
                    reply_markup=back_keyboard("admin")
                )
        
        elif call.data == "admin_users":
            if check_admin_status(user_id):
                conn = sqlite3.connect(DB_NAME)
                cursor = conn.cursor()
                cursor.execute("SELECT user_id, username, first_name, balance, referral_count, registration_date FROM users ORDER BY registration_date DESC LIMIT 10")
                users = cursor.fetchall()
                conn.close()
                
                text = "👥 **Последние 10 пользователей:**\n\n"
                for u in users:
                    name = u[2] or f"@{u[1]}" if u[1] else f"ID: {u[0]}"
                    text += f"• {name}\n"
                    text += f"  ID: `{u[0]}` | Баланс: {u[3]} ⭐ | Рефералов: {u[4]}\n"
                    text += f"  Дата: {u[5][:10]}\n\n"
                
                bot.edit_message_text(
                    text,
                    user_id,
                    call.message.message_id,
                    parse_mode="Markdown",
                    reply_markup=back_keyboard("admin")
                )
        
        elif call.data == "admin_sponsors":
            if check_admin_status(user_id):
                sponsors = get_sponsors()
                text = "📢 **Управление спонсорами**\n\n"
                
                if sponsors:
                    text += "**Текущие спонсоры:**\n"
                    for i, s in enumerate(sponsors, 1):
                        text += f"{i}. {s['name']} - {s['chat_id']}\n"
                else:
                    text += "Спонсоры отсутствуют\n"
                
                bot.edit_message_text(
                    text,
                    user_id,
                    call.message.message_id,
                    parse_mode="Markdown",
                    reply_markup=sponsors_management_keyboard()
                )
        
        elif call.data == "sponsor_add":
            if check_admin_status(user_id):
                bot.edit_message_text(
                    "📝 Отправьте мне данные спонсора в формате:\n`Название | ссылка | @канал`\n\nНапример:\n`Мой канал | https://t.me/mychannel | @mychannel`",
                    user_id,
                    call.message.message_id,
                    parse_mode="Markdown"
                )
                bot.register_next_step_handler_by_chat_id(user_id, process_add_sponsor)
        
        elif call.data == "sponsor_del":
            if check_admin_status(user_id):
                sponsors = get_sponsors()
                if not sponsors:
                    bot.answer_callback_query(call.id, "❌ Нет спонсоров для удаления", show_alert=True)
                    return
                
                text = "❌ Выберите спонсора для удаления:\n\n"
                keyboard = types.InlineKeyboardMarkup(row_width=1)
                for s in sponsors:
                    keyboard.add(types.InlineKeyboardButton(
                        text=f"{s['name']} - {s['chat_id']}",
                        callback_data=f"del_sponsor_{s['chat_id']}"
                    ))
                keyboard.add(types.InlineKeyboardButton("🔙 Назад", callback_data="admin_sponsors"))
                
                bot.edit_message_text(
                    text,
                    user_id,
                    call.message.message_id,
                    reply_markup=keyboard
                )
        
        elif call.data.startswith("del_sponsor_"):
            if check_admin_status(user_id):
                chat_id = call.data.replace("del_sponsor_", "")
                if delete_sponsor(chat_id):
                    bot.answer_callback_query(call.id, "✅ Спонсор удален!", show_alert=True)
                    sponsors = get_sponsors()
                    text = "📢 **Управление спонсорами**\n\n"
                    if sponsors:
                        text += "**Текущие спонсоры:**\n"
                        for i, s in enumerate(sponsors, 1):
                            text += f"{i}. {s['name']} - {s['chat_id']}\n"
                    else:
                        text += "Спонсоры отсутствуют\n"
                    
                    bot.edit_message_text(
                        text,
                        user_id,
                        call.message.message_id,
                        parse_mode="Markdown",
                        reply_markup=sponsors_management_keyboard()
                    )
        
        elif call.data == "sponsor_clear":
            if check_admin_status(user_id):
                conn = sqlite3.connect(DB_NAME)
                cursor = conn.cursor()
                cursor.execute("DELETE FROM sponsors")
                conn.commit()
                conn.close()
                bot.answer_callback_query(call.id, "✅ Все спонсоры удалены!", show_alert=True)
                
                bot.edit_message_text(
                    "📢 **Управление спонсорами**\n\nСпонсоры отсутствуют",
                    user_id,
                    call.message.message_id,
                    parse_mode="Markdown",
                    reply_markup=sponsors_management_keyboard()
                )
        
        elif call.data == "admin_mailing":
            if check_admin_status(user_id):
                bot.edit_message_text(
                    "📨 Введите текст для рассылки:",
                    user_id,
                    call.message.message_id
                )
                bot.register_next_step_handler_by_chat_id(user_id, process_mailing)
    
    except Exception as e:
        print(f"❌ Ошибка в callback: {e}")

def process_add_sponsor(message):
    try:
        user_id = message.from_user.id
        if not check_admin_status(user_id):
            return
        
        text = message.text
        parts = text.split('|')
        
        if len(parts) < 3:
            bot.send_message(
                user_id,
                "❌ Неверный формат. Используйте: Название | ссылка | @канал",
                reply_markup=admin_menu_keyboard()
            )
            return
        
        name = parts[0].strip()
        link = parts[1].strip()
        chat_id = parts[2].strip()
        
        if add_sponsor(name, link, chat_id):
            bot.send_message(user_id, f"✅ Спонсор {name} добавлен!", reply_markup=admin_menu_keyboard())
        else:
            bot.send_message(user_id, "❌ Ошибка: спонсор с таким @каналом уже существует", reply_markup=admin_menu_keyboard())
    except Exception as e:
        bot.send_message(user_id, f"❌ Ошибка: {e}", reply_markup=admin_menu_keyboard())

def process_mailing(message):
    try:
        admin_id = message.from_user.id
        text = message.text
        
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("INSERT OR REPLACE INTO temp_mailing (admin_id, text) VALUES (?, ?)", (admin_id, text))
        
        cursor.execute("SELECT user_id FROM users WHERE notifications = 1")
        users = cursor.fetchall()
        conn.close()
        
        bot.send_message(admin_id, f"📨 Рассылка началась... Всего пользователей: {len(users)}")
        
        success = 0
        failed = 0
        
        for user in users:
            try:
                bot.send_message(user[0], text)
                success += 1
                time.sleep(0.05)
            except Exception as e:
                print(f"Ошибка отправки пользователю {user[0]}: {e}")
                failed += 1
        
        bot.send_message(
            admin_id,
            f"✅ Рассылка завершена!\n\n📊 Успешно: {success}\n❌ Ошибок: {failed}",
            reply_markup=admin_menu_keyboard()
        )
    except Exception as e:
        bot.send_message(admin_id, f"❌ Ошибка: {e}", reply_markup=admin_menu_keyboard())

# ==================== ЗАПУСК БОТА ====================
if __name__ == "__main__":
    print("🚀 Бот запускается...")
    init_database()
    print(f"✅ Бот @{bot.get_me().username} готов к работе!")
    print(f"👑 Админы: {ADMIN_IDS}")
    print("🔄 Нажмите Ctrl+C для остановки")
    
    while True:
        try:
            bot.polling(none_stop=True)
        except Exception as e:
            print(f"❌ Ошибка: {e}")
            time.sleep(5)
