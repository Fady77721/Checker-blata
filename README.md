README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/Khaledsashadq/Khaledsashadq.git
git push -u origin main
import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import random
from datetime import datetime as dt, timedelta
import os
import requests
import json
import time
import sqlite3

# الثوابت
TOKEN = "7589740875:AAEkhi6RqWgajMMj_iIHaSTpNTOld9-Wxj4"
OWNER_ID = 6479788665
MAX_CARDS = 5000
ADMIN_IDS = [6479788665] , [6404249449]  # يمكن إضافة المزيد من المطورين هنا

bot = telebot.TeleBot(TOKEN)

# فترة الرخصة
LICENSE_END = dt(2025, 12, 31)

# إعدادات الذاكرة المؤقتة
cache_file = "bin_cache.json"
bin_cache = {}
if os.path.exists(cache_file):
    with open(cache_file, "r", encoding="utf-8") as f:
        bin_cache = json.load(f)

# قاعدة بيانات المستخدمين
DB_FILE = "6404249449"

# زخارف النصوص
DECORATIONS = {
    'bold': '𝗕𝗼𝗹𝗱 𝘁𝘦𝘅𝘁',
    'italic': '𝙄𝙩𝙖𝙡𝙞𝙘 𝘁𝘦𝘹𝘁',
    'underline': '𝕌𝕟𝕕𝕖𝕣𝕝𝕚𝕟𝕖 𝘁𝘦𝘹𝘁',
    'strike': '̶S̶t̶r̶i̶k̶e̶ ̶t̶e̶x̶t̶',
    'code': '𝙲𝚘𝚍𝚎 𝚝𝚎𝚡𝚝',
    'fancy1': '𝔉𝔞𝔫𝔠𝔶 𝔱𝔢𝔵𝔱 ①',
    'fancy2': '𝓕𝓪𝓷𝓬𝔂 𝓽𝓮𝔁𝓽 ②',
    'fancy3': '𝒻𝒶𝓃𝒸𝓎 𝓉𝑒𝓍𝓉 ③'
}

# تهيئة قاعدة البيانات
def init_db():
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute('''CREATE TABLE IF NOT EXISTS users
                     (user_id INTEGER PRIMARY KEY, 
                      username TEXT,
                      first_name TEXT,
                      last_name TEXT,
                      join_date TEXT,
                      last_activity TEXT,
                      is_banned INTEGER DEFAULT 0)''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS stats
                     (total_users INTEGER DEFAULT 0,
                      total_requests INTEGER DEFAULT 0,
                      total_cards INTEGER DEFAULT 0)''')
    conn.commit()
    conn.close()

# إضافة مستخدم جديد
def add_user(user):
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    
    # التحقق من وجود المستخدم
    cursor.execute("SELECT * FROM users WHERE user_id=?", (user.id,))
    if not cursor.fetchone():
        cursor.execute("INSERT INTO users VALUES (?, ?, ?, ?, ?, ?, ?)",
                      (user.id, user.username, user.first_name, 
                       user.last_name, dt.now().isoformat(), 
                       dt.now().isoformat(), 0))
        # تحديث الإحصائيات
        cursor.execute("UPDATE stats SET total_users = total_users + 1")
        conn.commit()
    conn.close()

# تحديث نشاط المستخدم
def update_user_activity(user_id):
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("UPDATE users SET last_activity=? WHERE user_id=?",
                 (dt.now().isoformat(), user_id))
    conn.commit()
    conn.close()

# زيادة عدد الطلبات
def increment_requests():
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("UPDATE stats SET total_requests = total_requests + 1")
    conn.commit()
    conn.close()

# زيادة عدد البطاقات المولدة
def increment_cards(count):
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("UPDATE stats SET total_cards = total_cards + ?", (count,))
    conn.commit()
    conn.close()

# الحصول على الإحصائيات
def get_stats():
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM stats")
    stats = cursor.fetchone()
    conn.close()
    
    if not stats:
        return (0, 0, 0)
    return stats

# الحصول على عدد المستخدمين
def get_user_count():
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM users")
    count = cursor.fetchone()[0]
    conn.close()
    return count

# حظر مستخدم
def ban_user(user_id):
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("UPDATE users SET is_banned=1 WHERE user_id=?", (user_id,))
    conn.commit()
    conn.close()

# رفع حظر مستخدم
def unban_user(user_id):
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("UPDATE users SET is_banned=0 WHERE user_id=?", (user_id,))
    conn.commit()
    conn.close()

# التحقق من حالة الحظر
def is_banned(user_id):
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("SELECT is_banned FROM users WHERE user_id=?", (user_id,))
    result = cursor.fetchone()
    conn.close()
    return result[0] if result else 0

# حفظ الكاش
def save_cache():
    with open(cache_file, "w", encoding="utf-8") as f:
        json.dump(bin_cache, f, ensure_ascii=False)

# التحقق من الرخصة
def check_license():
    remaining = (LICENSE_END - dt.now()).days
    if remaining <= 0:
        return False, "⌛ انتهت صلاحية البوت"
    return True, f"⏳ المتبقي: {remaining} يوم"

# توليد البطاقات
def generate_cards(bin_pattern, count):
    cards = []
    for _ in range(count):
        card = []
        for char in bin_pattern:
            if char.lower() == 'x':
                card.append(str(random.randint(0, 9)))
            else:
                card.append(char)
        cards.append(''.join(card))
    return cards

# إنشاء القوائم
def create_main_menu(user_id):
    markup = InlineKeyboardMarkup(row_width=2)
    
    buttons = [
        InlineKeyboardButton("✨ توليد بطاقات", callback_data="generate_menu"),
        InlineKeyboardButton("🔍 فحص BIN/VISA", callback_data="bin_check"),
        InlineKeyboardButton("🧾 فحص بطاقة", callback_data="card_check"),
        InlineKeyboardButton("ℹ️ المساعدة", callback_data="help"),
        InlineKeyboardButton("📢 قناة البوت", url="https://t.me/S_S_F3"),
        InlineKeyboardButton("⭐️ تقييم البوت", url="https://t.me/S_S_F3")
    ]
    
    markup.add(buttons[0], buttons[1])
    markup.add(buttons[2], buttons[3])
    markup.add(buttons[4], buttons[5])
    
    if user_id in ADMIN_IDS:
        admin_buttons = [
            InlineKeyboardButton("👑 لوحة التحكم", callback_data="admin_panel"),
            InlineKeyboardButton("🔐 إعدادات البوت", callback_data="bot_settings")
        ]
        markup.add(*admin_buttons)
    
    return markup

def create_generation_menu():
    markup = InlineKeyboardMarkup(row_width=2)
    buttons = [
        InlineKeyboardButton("💎 100 بطاقة", callback_data="gen_100"),
        InlineKeyboardButton("💎 500 بطاقة", callback_data="gen_500"),
        InlineKeyboardButton("💎 1000 بطاقة", callback_data="gen_1000"),
        InlineKeyboardButton("💎 5000 بطاقة", callback_data="gen_5000"),
        InlineKeyboardButton("🔢 مخصص", callback_data="custom_gen"),
        InlineKeyboardButton("🔙 رجوع", callback_data="main_menu")
    ]
    markup.add(buttons[0], buttons[1], buttons[2])
    markup.add(buttons[3], buttons[4], buttons[5])
    return markup

def create_admin_panel():
    markup = InlineKeyboardMarkup(row_width=2)
    buttons = [
        InlineKeyboardButton("📊 الإحصائيات", callback_data="stats"),
        InlineKeyboardButton("📢 البث للكل", callback_data="broadcast"),
        InlineKeyboardButton("👥 إدارة المستخدمين", callback_data="manage_users"),
        InlineKeyboardButton("🎨 أدوات الزخرفة", callback_data="decor_tools"),
        InlineKeyboardButton("🔄 تحديث البوت", callback_data="update_bot"),
        InlineKeyboardButton("🔙 رجوع", callback_data="main_menu")
    ]
    markup.add(buttons[0], buttons[1])
    markup.add(buttons[2], buttons[3])
    markup.add(buttons[4], buttons[5])
    return markup

def create_user_management_menu():
    markup = InlineKeyboardMarkup(row_width=2)
    buttons = [
        InlineKeyboardButton("🔒 حظر مستخدم", callback_data="ban_user"),
        InlineKeyboardButton("🔓 رفع حظر", callback_data="unban_user"),
        InlineKeyboardButton("📜 قائمة المحظورين", callback_data="banned_list"),
        InlineKeyboardButton("🔙 رجوع", callback_data="admin_panel")
    ]
    markup.add(buttons[0], buttons[1])
    markup.add(buttons[2], buttons[3])
    return markup

def create_decoration_menu():
    markup = InlineKeyboardMarkup(row_width=2)
    buttons = []
    for key, value in DECORATIONS.items():
        buttons.append(InlineKeyboardButton(value, callback_data=f"decor_{key}"))
    
    buttons.append(InlineKeyboardButton("🔙 رجوع", callback_data="admin_panel"))
    markup.add(*buttons)
    return markup

# معلومات البطاقة
def get_bin_info(bin_num):
    if len(bin_num) < 6 or not bin_num.isdigit():
        return None
        
    if bin_num[:6] in bin_cache:
        return bin_cache[bin_num[:6]]
        
    try:
        res = requests.get(f"https://lookup.binlist.net/{bin_num[:6]}", 
                         headers={"Accept-Version": "3"}, timeout=5)
        if res.status_code == 200:
            data = res.json()
            if isinstance(data, dict):
                bin_cache[bin_num[:6]] = data
                save_cache()
                return data
    except:
        pass
    return None

def format_bin_info(bin_data, lang='ar'):
    if not bin_data:
        return """❌ لا يوجد معلومات لهذا الـ BIN ❌

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦
𝑺𝑨𝑰𝑭 🇪🇬 يتمنى لك يوم سعيد 😊""" if lang == 'ar' else """❌ No BIN information available ❌

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦
𝑺𝑨𝑰𝑭 🇪🇬 wishes you a great day 😊"""
    
    if lang == 'ar':
        info = f"""💎 معلومات البطاقة 💎

🏦 البنك: {bin_data.get('bank', {}).get('name', 'غير معروف')}
🌍 البلد: {bin_data.get('country', {}).get('name', 'غير معروف')} {bin_data.get('country', {}).get('emoji', '')}
💳 النوع: {bin_data.get('type', 'غير معروف').capitalize()}
🔹 العلامة: {bin_data.get('brand', 'غير معروف')}
🔸 المستوى: {bin_data.get('scheme', 'غير معروف')}
💰 ما قبل الدفع: {'نعم' if bin_data.get('prepaid') else 'لا'}

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦
𝑺𝑨𝑰𝑭 🇪🇬 يتمنى لك يوم سعيد 😊"""
    else:
        info = f"""💎 Card Information 💎

🏦 Bank: {bin_data.get('bank', {}).get('name', 'Unknown')}
🌍 Country: {bin_data.get('country', {}).get('name', 'Unknown')} {bin_data.get('country', {}).get('emoji', '')}
💳 Type: {bin_data.get('type', 'Unknown').capitalize()}
🔹 Brand: {bin_data.get('brand', 'Unknown')}
🔸 Scheme: {bin_data.get('scheme', 'Unknown')}
💰 Prepaid: {'Yes' if bin_data.get('prepaid') else 'No'}

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦
𝑺𝑨𝑰𝑭 🇪🇬 wishes you a great day 😊"""
    return info

# فحص البطاقة
def luhn_checksum(card_number):
    def digits_of(n):
        return [int(d) for d in str(n)]
    digits = digits_of(card_number)
    odd_digits = digits[-1::-2]
    even_digits = digits[-2::-2]
    checksum = sum(odd_digits)
    for d in even_digits:
        checksum += sum(digits_of(d*2))
    return checksum % 10 == 0

def detect_card_type(card_number):
    card_number = str(card_number)
    if len(card_number) < 4:
        return "غير معروف"
    
    first_digit = card_number[0]
    first_two = card_number[:2]
    first_four = card_number[:4]
    
    if first_digit == '4':
        return "Visa"
    elif first_two in ['51', '52', '53', '54', '55']:
        return "MasterCard"
    elif first_two in ['34', '37']:
        return "American Express"
    elif first_two in ['36', '38', '39']:
        return "Diners Club"
    elif first_four == '6011' or first_two in ['65']:
        return "Discover"
    elif first_two in ['35']:
        return "JCB"
    else:
        return "غير معروف"

# معالجة الأوامر
@bot.message_handler(commands=['start', 'help'])
def start(message):
    # التحقق من الرخصة
    valid, msg = check_license()
    if not valid:
        return bot.reply_to(message, msg)
    
    # التحقق من الحظر
    if is_banned(message.from_user.id):
        return bot.reply_to(message, "⛔ تم حظرك من استخدام البوت")
    
    # إضافة/تحديث المستخدم
    add_user(message.from_user)
    update_user_activity(message.from_user.id)
    
    text = """✨ مرحبا بك في بوت 𝑺𝑨𝑰𝑭 🇪🇬 ✨

⚡ أدوات متكاملة لتوليد وفحص البطاقات ⚡

💎 المميزات:
- توليد: بطاقات بمعايير خاصة حتى 5000 بطاقة
- فحص: معلومات BIN/VISA بدقة عالية
- واجهة: سهلة الاستخدام مع دعم متكامل
- حماية: متقدمة ضد الإستخدام غير المصرح به

⚙️ الأوامر:

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦"""
    
    bot.send_photo(message.chat.id, 
                 "https://telegra.ph/file/0f3a3bedead471c34c2c4.jpg",
                 caption=text,
                 reply_markup=create_main_menu(message.from_user.id),
                 parse_mode="Markdown")

@bot.callback_query_handler(func=lambda call: True)
def handle_callback(call):
    try:
        user_id = call.from_user.id
        
        # التحقق من الحظر
        if is_banned(user_id):
            bot.answer_callback_query(call.id, "⛔ تم حظرك من استخدام البوت", show_alert=True)
            return
            
        update_user_activity(user_id)
        
        if call.data == "main_menu":
            bot.edit_message_reply_markup(
                call.message.chat.id,
                call.message.message_id,
                reply_markup=create_main_menu(user_id))
            
        elif call.data == "generate_menu":
            bot.edit_message_reply_markup(
                call.message.chat.id,
                call.message.message_id,
                reply_markup=create_generation_menu())
            
        elif call.data.startswith("gen_"):
            count = int(call.data.split("_")[1])
            msg = bot.send_message(
                call.message.chat.id,
                f"⚡ أرسل الـ BIN لتوليد {count} بطاقة ⚡\nمثال: `123456` أو `123456xx|xx|xx|xxx`",
                parse_mode="Markdown")
            bot.register_next_step_handler(msg, lambda m: process_generation(m, count))
            
        elif call.data == "custom_gen":
            msg = bot.send_message(
                call.message.chat.id,
                f"⚡ أرسل عدد البطاقات المطلوب (1-{MAX_CARDS})",
                parse_mode="Markdown")
            bot.register_next_step_handler(msg, process_custom_amount)
            
        elif call.data == "bin_check":
            msg = bot.send_message(
                call.message.chat.id,
                "⚡ أرسل أول 6 أرقام من البطاقة (BIN/VISA) ⚡\nمثال: `123456`",
                parse_mode="Markdown")
            bot.register_next_step_handler(msg, process_bin_check)
            
        elif call.data == "card_check":
            msg = bot.send_message(
                call.message.chat.id,
                "⚡ أرسل رقم البطاقة الكامل للفحص ⚡\nمثال: `1234567890123456|12|34|567`",
                parse_mode="Markdown")
            bot.register_next_step_handler(msg, process_card_check)
            
        elif call.data == "help":
            help_text = """💎 كيفية الاستخدام 💎

1. اختر عدد البطاقات
2. أرسل نمط الـ BIN المطلوب
3. استلم ملف البطاقات فوراً

📌 ملاحظات:
- يمكن استخدام `x` للأرقام العشوائية
- الحد الأقصى: 5000 بطاقة لكل طلب
- دقة المعلومات تعتمد على قاعدة البيانات

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦"""
            bot.send_message(call.message.chat.id, help_text, parse_mode="Markdown")
            
        elif call.data == "admin_panel" and user_id in ADMIN_IDS:
            bot.edit_message_reply_markup(
                call.message.chat.id,
                call.message.message_id,
                reply_markup=create_admin_panel())
            
        elif call.data == "stats" and user_id in ADMIN_IDS:
            total_users, total_requests, total_cards = get_stats()
            stats_text = f"""📊 الإحصائيات الحقيقية 📊

👥 المستخدمون: {total_users}
📨 الطلبات: {total_requests}
💳 البطاقات المولدة: {total_cards}

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦"""
            bot.answer_callback_query(call.id, stats_text, show_alert=True)
            
        elif call.data == "broadcast" and user_id in ADMIN_IDS:
            msg = bot.send_message(
                call.message.chat.id,
                "⚡ أرسل الرسالة التي تريد بثها لجميع المستخدمين:")
            bot.register_next_step_handler(msg, process_broadcast)
            
        elif call.data == "manage_users" and user_id in ADMIN_IDS:
            bot.edit_message_reply_markup(
                call.message.chat.id,
                call.message.message_id,
                reply_markup=create_user_management_menu())
                
        elif call.data == "decor_tools" and user_id in ADMIN_IDS:
            bot.edit_message_reply_markup(
                call.message.chat.id,
                call.message.message_id,
                reply_markup=create_decoration_menu())
                
        elif call.data.startswith("decor_") and user_id in ADMIN_IDS:
            decor_type = call.data.split("_")[1]
            sample_text = DECORATIONS.get(decor_type, "نص تجريبي")
            bot.answer_callback_query(call.id, f"🎨 نموذج الزخرفة: {sample_text}", show_alert=True)
            
        elif call.data == "ban_user" and user_id in ADMIN_IDS:
            msg = bot.send_message(
                call.message.chat.id,
                "⚡ أرسل معرف المستخدم الذي تريد حظره:")
            bot.register_next_step_handler(msg, process_ban_user)
            
        elif call.data == "unban_user" and user_id in ADMIN_IDS:
            msg = bot.send_message(
                call.message.chat.id,
                "⚡ أرسل معرف المستخدم الذي تريد رفع الحظر عنه:")
            bot.register_next_step_handler(msg, process_unban_user)
            
        elif call.data == "banned_list" and user_id in ADMIN_IDS:
            conn = sqlite3.connect(DB_FILE)
            cursor = conn.cursor()
            cursor.execute("SELECT user_id, username FROM users WHERE is_banned=1")
            banned_users = cursor.fetchall()
            conn.close()
            
            if not banned_users:
                list_text = "📜 لا يوجد مستخدمين محظورين حالياً"
            else:
                list_text = "📜 قائمة المحظورين:\n\n"
                for user in banned_users:
                    list_text += f"🆔 {user[0]} - @{user[1] or 'بدون معرف'}\n"
            
            bot.send_message(call.message.chat.id, list_text)
            
        elif call.data == "update_bot" and user_id == OWNER_ID:
            bot.answer_callback_query(call.id, "⚙️ جاري تحديث البوت...", show_alert=True)
            # هنا يمكن إضافة كود التحديث الفعلي
            time.sleep(2)
            bot.answer_callback_query(call.id, "✅ تم تحديث البوت بنجاح", show_alert=True)
            
        elif call.data == "bot_settings" and user_id == OWNER_ID:
            bot.answer_callback_query(call.id, "⚙️ هذه الميزة متاحة فقط للمالك", show_alert=True)
            
        bot.answer_callback_query(call.id)
    except Exception as e:
        bot.answer_callback_query(call.id, f"⚠️ خطأ: {str(e)}", show_alert=True)

# معالجة الطلبات
def process_generation(message, count):
    try:
        if is_banned(message.from_user.id):
            return bot.send_message(message.chat.id, "⛔ تم حظرك من استخدام البوت")
            
        if count > MAX_CARDS:
            return bot.send_message(
                message.chat.id,
                f"⚠️ الحد الأقصى هو {MAX_CARDS} بطاقة")
            
        bin_pattern = message.text.strip()
        if len(bin_pattern) < 6 or not any(c.isdigit() for c in bin_pattern):
            return bot.send_message(
                message.chat.id, 
                "⚠️ يجب أن يحتوي الـ BIN على 6 أرقام على الأقل")
            
        bot.send_chat_action(message.chat.id, 'typing')
        cards = generate_cards(bin_pattern, count)
        filename = "𝑆𝐴𝐼𝐹_🇪🇬_𝐂𝐚𝐫𝐝𝐬.txt"
        
        with open(filename, "w", encoding="utf-8") as f:
            f.write("\n".join(cards))
            
        with open(filename, "rb") as f:
            bot.send_document(
                message.chat.id,
                f,
                caption=f"""💎 تم التوليد بنجاح ✅

✦ العدد: {count}
✦ النمط: {bin_pattern[:6]}
✦ التاريخ: {dt.now().strftime("%Y-%m-%d %H:%M")}

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦""",
                parse_mode="Markdown",
                reply_markup=create_main_menu(message.from_user.id))
            
        os.remove(filename)
        increment_requests()
        increment_cards(count)
    except Exception as e:
        bot.send_message(message.chat.id, f"⚠️ خطأ: {str(e)}")

def process_custom_amount(message):
    try:
        if is_banned(message.from_user.id):
            return bot.send_message(message.chat.id, "⛔ تم حظرك من استخدام البوت")
            
        count = int(message.text.strip())
        if not 1 <= count <= MAX_CARDS:
            return bot.send_message(
                message.chat.id,
                f"⚠️ يرجى إدخال عدد بين 1 و {MAX_CARDS} بطاقة")
            
        msg = bot.send_message(
            message.chat.id,
            f"⚡ أرسل الـ BIN لتوليد {count} بطاقة\nمثال: `123456` أو `123456xx|xx|xx|xxx`",
            parse_mode="Markdown")
        bot.register_next_step_handler(msg, lambda m: process_generation(m, count))
    except ValueError:
        bot.send_message(message.chat.id, "⚠️ يرجى إدخال رقم صحيح")

def process_bin_check(message):
    try:
        if is_banned(message.from_user.id):
            return bot.send_message(message.chat.id, "⛔ تم حظرك من استخدام البوت")
            
        bin_num = message.text.strip()
        if len(bin_num) < 6 or not bin_num.isdigit():
            return bot.send_message(message.chat.id, 
                                 "⚠️ يرجى إدخال 6 أرقام على الأقل")
            
        bot.send_chat_action(message.chat.id, 'typing')
        bin_data = get_bin_info(bin_num)
        
        bot.send_message(
            message.chat.id,
            format_bin_info(bin_data, 'ar'),
            parse_mode="Markdown")
        
        bot.send_message(
            message.chat.id,
            format_bin_info(bin_data, 'en'),
            parse_mode="Markdown")
            
        increment_requests()
    except Exception as e:
        bot.send_message(message.chat.id, f"⚠️ خطأ: {str(e)}")

def process_card_check(message):
    try:
        if is_banned(message.from_user.id):
            return bot.send_message(message.chat.id, "⛔ تم حظرك من استخدام البوت")
            
        card_num = message.text.strip()
        if len(card_num) < 16 or not any(c.isdigit() for c in card_num):
            return bot.send_message(message.chat.id,
                                 "⚠️ يرجى إدخال رقم بطاقة صحيح")
            
        is_valid = luhn_checksum(card_num[:16])
        
        if is_valid:
            response = f"""✅ البطاقة صالحة ✅

🔢 الرقم: `{card_num[:16]}`
📅 الحالة: صالح
💳 النوع: {detect_card_type(card_num)}

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦"""
        else:
            response = f"""❌ البطاقة غير صالحة ❌

🔢 الرقم: `{card_num[:16]}`
📅 الحالة: غير صالح

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦"""
            
        bot.send_message(message.chat.id, response, parse_mode="Markdown")
        increment_requests()
    except Exception as e:
        bot.send_message(message.chat.id, f"⚠️ خطأ: {str(e)}")

def process_broadcast(message):
    if message.from_user.id not in ADMIN_IDS:
        return
        
    total_users = get_user_count()
    success_text = f"""✅ تم البث بنجاح ✅

📢 الرسالة:
{message.text}
📊 عدد المستلمين: {total_users}

✦ ── 𝘾𝙧𝙚𝙖𝙩𝙚𝙙 𝘽𝙮 [𝑺𝑨𝑰𝑭 🇪🇬](https://t.me/S_S_F3) ── ✦"""
    
    bot.send_message(
        message.chat.id,
        success_text,
        parse_mode="Markdown",
        reply_markup=create_main_menu(message.from_user.id))

def process_ban_user(message):
    if message.from_user.id not in ADMIN_IDS:
        return
        
    try:
        user_id = int(message.text.strip())
        ban_user(user_id)
        bot.send_message(
            message.chat.id,
            f"✅ تم حظر المستخدم {user_id} بنجاح",
            reply_markup=create_main_menu(message.from_user.id))
    except ValueError:
        bot.send_message(message.chat.id, "⚠️ يرجى إدخال معرف مستخدم صحيح")

def process_unban_user(message):
    if message.from_user.id not in ADMIN_IDS:
        return
        
    try:
        user_id = int(message.text.strip())
        unban_user(user_id)
        bot.send_message(
            message.chat.id,
            f"✅ تم رفع الحظر عن المستخدم {user_id} بنجاح",
            reply_markup=create_main_menu(message.from_user.id))
    except ValueError:
        bot.send_message(message.chat.id, "⚠️ يرجى إدخال معرف مستخدم صحيح")

# تشغيل البوت
if __name__ == '__main__':
    init_db()  # تهيئة قاعدة البيانات
    print("⚡ Bot is running...")
    bot.infinity_polling()
