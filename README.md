import telebot 
from telebot.types import InlineKeyboardButton, InlineKeyboardMarkup
import sqlite3
from datetime import datetime, timedelta
import time
import traceback
import pytz
import random
import string

# إعدادات البوت
TOKEN = "7786577745:AAHY8rO2lfUpZ3kWLYuUx833H0juKm4y6Go"
ADMIN_ID = 6937469902
bot = telebot.TeleBot(TOKEN)
TIMEZONE = pytz.timezone('Africa/Algiers')

# إعدادات نظام النقاط
POINTS_RATE = 250
POINTS_TO_DIAMONDS = {'points': 100, 'diamonds': 110}

# قواعد البيانات
def init_db():
    conn = sqlite3.connect('bot_data.db', check_same_thread=False)
    cursor = conn.cursor()
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS orders (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        username TEXT,
        first_name TEXT,
        last_name TEXT,
        player_id TEXT,
        option TEXT,
        original_price INTEGER,
        final_price INTEGER,
        payment_method TEXT,
        transaction_id TEXT,
        payment_proof TEXT,
        status TEXT DEFAULT '🔄 قيد المعالجة',
        timestamp TEXT,
        formatted_time TEXT
    )''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS discounts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        discount_amount INTEGER,
        target_item TEXT DEFAULT 'all',
        start_time TEXT,
        end_time TEXT,
        is_active INTEGER DEFAULT 0
    )''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS referrals (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        referrer_id INTEGER,
        referred_id INTEGER UNIQUE,
        timestamp TEXT
    )''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS user_points (
        user_id INTEGER PRIMARY KEY,
        points REAL DEFAULT 0,
        referral_count INTEGER DEFAULT 0,
        referral_code TEXT UNIQUE,
        total_earned REAL DEFAULT 0
    )''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS points_redemption (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        player_id TEXT,
        points_used REAL,
        status TEXT DEFAULT 'pending',
        timestamp TEXT
    )''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS referral_purchases (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        referrer_id INTEGER,
        referred_id INTEGER,
        amount INTEGER,
        points_given REAL DEFAULT 0,
        timestamp TEXT
    )''')
    
    conn.commit()
    conn.close()

init_db()

# معلومات الدفع
PAYMENT_INFO = {
    "USDT_TON": "UQDwhzPQ-gVQPqurWzMpP2AezP0uFRhPbUkoqb1frF2cOj7o",
    "PHONE_NUMBER": "+213XXXXXXXXX",
    "USDT_RATE": 250
}

BASE_PRICES = {
    "110 💎": 250,
    "341 💎": 750,
    "572 💎": 1250,
    "2398 💎": 5000,
    "📅 عضوية أسبوعية": 500,
    "📅 عضوية شهرية": 2200,
    "🎫 تصريح بويا": 700
}

user_data = {}

# ============= دوال مساعدة =============
def generate_referral_code(user_id):
    return f"REF{str(user_id)[-4:]}{''.join(random.choices(string.ascii_uppercase + string.digits, k=4))}"

def get_user_referral_code(user_id):
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()
    cursor.execute("SELECT referral_code FROM user_points WHERE user_id=?", (user_id,))
    result = cursor.fetchone()
    if result and result[0]:
        conn.close()
        return result[0]
    else:
        referral_code = generate_referral_code(user_id)
        cursor.execute("INSERT OR REPLACE INTO user_points (user_id, referral_code) VALUES (?, ?)", (user_id, referral_code))
        conn.commit()
        conn.close()
        return referral_code

def get_user_points(user_id):
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()
    cursor.execute("SELECT points, referral_count, total_earned FROM user_points WHERE user_id=?", (user_id,))
    result = cursor.fetchone()
    conn.close()
    if result:
        return result[0] or 0, result[1] or 0, result[2] or 0
    return 0, 0, 0

def add_user_points(user_id, points_to_add):
    current_points, ref_count, total_earned = get_user_points(user_id)
    new_points = current_points + points_to_add
    new_total = total_earned + points_to_add
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()
    cursor.execute("INSERT OR REPLACE INTO user_points (user_id, points, total_earned) VALUES (?, ?, ?)", (user_id, new_points, new_total))
    conn.commit()
    conn.close()
    return new_points

def calculate_points(amount):
    multiplier = amount / POINTS_RATE
    referred_points = round(5 * multiplier, 2)
    referrer_points = round(2.5 * multiplier, 2)
    return referred_points, referrer_points

def add_referral_points(referrer_id, referred_id, amount):
    referred_points, referrer_points = calculate_points(amount)
    
    if referred_points > 0 or referrer_points > 0:
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        
        cursor.execute('''
        INSERT INTO referral_purchases 
        (referrer_id, referred_id, amount, points_given, timestamp)
        VALUES (?, ?, ?, ?, datetime('now'))
        ''', (referrer_id, referred_id, amount, (referred_points + referrer_points)))
        
        if referred_points > 0:
            add_user_points(referred_id, referred_points)
            try:
                bot.send_message(
                    referred_id,
                    f"🎉 لقد حصلت على {referred_points} نقطة من شرائك بقيمة {amount}DZ!\n"
                    f"⭐ إجمالي نقاطك الآن: {get_user_points(referred_id)[0]}"
                )
            except:
                pass
        
        if referrer_points > 0:
            add_user_points(referrer_id, referrer_points)
            try:
                bot.send_message(
                    referrer_id,
                    f"🎉 لقد حصلت على {referrer_points} نقطة من شراء محولك بقيمة {amount}DZ!\n"
                    f"⭐ إجمالي نقاطك الآن: {get_user_points(referrer_id)[0]}"
                )
            except:
                pass
        
        conn.commit()
        conn.close()

def is_admin(user_id):
    return user_id == ADMIN_ID

def format_time(dt):
    return dt.strftime('%Y-%m-%d %H:%M:%S')

def get_current_prices():
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()
    cursor.execute("SELECT discount_amount, target_item FROM discounts WHERE is_active=1 AND datetime(end_time)>=datetime('now')")
    discounts = cursor.fetchall()
    conn.close()
    
    prices = BASE_PRICES.copy()
    discount_info = {}
    
    for amount, target in discounts:
        if target == 'all':
            for item in prices:
                prices[item] -= amount
                discount_info['all'] = amount
        else:
            prices[target] -= amount
            discount_info[target] = amount
    
    return prices, discount_info

# ============= نظام الإحالات =============
def handle_referral(user_id, referrer_id):
    try:
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        
        cursor.execute("SELECT * FROM referrals WHERE referred_id=?", (user_id,))
        if cursor.fetchone():
            conn.close()
            return False
            
        cursor.execute("INSERT INTO referrals (referrer_id, referred_id, timestamp) VALUES (?, ?, datetime('now'))", (referrer_id, user_id))
        
        cursor.execute('''
        INSERT OR IGNORE INTO user_points (user_id, referral_count) VALUES (?, 0)
        ''', (referrer_id,))
        
        cursor.execute('''
        UPDATE user_points SET referral_count = referral_count + 1 WHERE user_id=?
        ''', (referrer_id,))
        
        conn.commit()
        conn.close()
        
        try:
            points, ref_count, _ = get_user_points(referrer_id)
            bot.send_message(
                referrer_id,
                f"🎉 تم تسجيل إحالة جديدة باستخدام رابطك!\n"
                f"👤 عدد الإحالات الآن: {ref_count}\n\n"
                f"ستحصل على نقاط عند كل شراء يقوم به هذا المستخدم"
            )
        except:
            pass
        
        return True
    except Exception as e:
        print(f"Error in handle_referral: {e}")
        return False

# ============= واجهة المستخدم =============
@bot.message_handler(commands=['start', 'admin'])
def handle_commands(message):
    if is_admin(message.from_user.id) and message.text == '/admin':
        admin_panel(message)
    else:
        if len(message.text.split()) > 1:
            try:
                ref_code = message.text.split()[1]
                conn = sqlite3.connect('bot_data.db')
                cursor = conn.cursor()
                cursor.execute("SELECT user_id FROM user_points WHERE referral_code=?", (ref_code,))
                result = cursor.fetchone()
                if result:
                    handle_referral(message.from_user.id, result[0])
                conn.close()
            except:
                pass
        send_welcome(message)

def send_welcome(message):
    welcome_text = "👋 أهلاً بك في متجر FreeFire DZ!\n\n🔹 الرجاء إدخال **ID اللاعب** (9-10 أرقام):"
    msg = bot.send_message(message.chat.id, welcome_text)
    user_data[message.chat.id] = {
        "state": "waiting_for_id",
        "welcome_msg_id": msg.message_id,
        "username": message.from_user.username,
        "first_name": message.from_user.first_name,
        "last_name": message.from_user.last_name
    }

@bot.message_handler(func=lambda m: user_data.get(m.chat.id, {}).get("state") == "waiting_for_id")
def get_player_id(message):
    player_id = message.text.strip()
    
    if not player_id.isdigit() or len(player_id) not in [9, 10]:
        bot.reply_to(message, "❌ يجب أن يكون ID اللاعب 9-10 أرقام!\n🔁 حاول مرة أخرى:")
        return
    
    user_data[message.chat.id].update({
        "player_id": player_id,
        "state": "waiting_for_selection"
    })
    
    show_services_menu(message.chat.id)
    show_referral_button(message.chat.id)

def show_referral_button(chat_id):
    points, ref_count, _ = get_user_points(chat_id)
    referral_code = get_user_referral_code(chat_id)
    share_link = f"https://t.me/{bot.get_me().username}?start={referral_code}"
    
    markup = InlineKeyboardMarkup()
    markup.add(InlineKeyboardButton("🎁 نظام الإحالات", callback_data="referral_system"))
    
    bot.send_message(
        chat_id,
        f"🔗 <b>كود الإحالة الخاص بك:</b> <code>{referral_code}</code>\n"
        f"⭐ <b>نقاطك:</b> {points}\n"
        f"👥 <b>عدد الإحالات:</b> {ref_count}\n\n"
        f"<b>رابط الإحالة:</b>\n<code>{share_link}</code>",
        reply_markup=markup,
        parse_mode="HTML"
    )

def show_services_menu(chat_id):
    try:
        prices, discount_info = get_current_prices()
        markup = InlineKeyboardMarkup(row_width=2)
        
        if 'welcome_msg_id' in user_data.get(chat_id, {}):
            try:
                bot.delete_message(chat_id, user_data[chat_id]['welcome_msg_id'])
            except:
                pass
        
        message_text = "✨ <b>اختر الخدمة:</b>\n\n💎 <b>عروض الجواهر:</b>\n"
        
        if discount_info:
            message_text += "\n🎁 <b>خصومات:</b>\n"
            if 'all' in discount_info:
                message_text += f"• خصم عام: {discount_info['all']} دينار\n"
            for item, amount in discount_info.items():
                if item != 'all':
                    message_text += f"• خصم {amount} دينار على {item}\n"
        
        buttons = []
        for option, price in prices.items():
            original_price = BASE_PRICES[option]
            btn_text = f"{option} - {price}DZD"
            if price != original_price:
                btn_text += f" (وفر {original_price-price}DZD)"
            buttons.append(InlineKeyboardButton(btn_text, callback_data=f"service_{option}"))
        
        for i in range(0, len(buttons), 2):
            markup.add(*buttons[i:i+2])
        
        bot.send_message(chat_id, message_text, reply_markup=markup, parse_mode='HTML')
    except Exception as e:
        print(f"Error in show_services_menu: {e}")
        bot.send_message(chat_id, "⚠️ حدث خطأ، يرجى المحاولة لاحقاً")

@bot.callback_query_handler(func=lambda call: call.data.startswith("service_"))
def handle_service(call):
    try:
        option = call.data.replace("service_", "")
        prices, _ = get_current_prices()
        
        user_data[call.message.chat.id].update({
            "selected_option": option,
            "final_price": prices[option],
            "state": "waiting_for_payment_method"
        })

        markup = InlineKeyboardMarkup()
        markup.add(
            InlineKeyboardButton("💰 USDT (TON)", callback_data="pay_usdt_ton"),
            InlineKeyboardButton("📱 رصيد هاتفي", callback_data="pay_phone")
        )
        markup.add(InlineKeyboardButton("↩ رجوع", callback_data="back_to_services"))

        original_price = BASE_PRICES[option]
        discount_text = f" (وفر {original_price-prices[option]}DZD)" if prices[option] != original_price else ""
        
        bot.edit_message_text(
            chat_id=call.message.chat.id,
            message_id=call.message.message_id,
            text=f"💎 <b>الخدمة المختارة:</b> {option}\n💵 <b>السعر:</b> {prices[option]}DZD{discount_text}\n\n🔽 اختر طريقة الدفع:",
            reply_markup=markup,
            parse_mode='HTML'
        )
    except Exception as e:
        print(f"Error in handle_service: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ!")

@bot.callback_query_handler(func=lambda call: call.data in ["pay_usdt_ton", "pay_phone", "back_to_services"])
def handle_payment(call):
    chat_id = call.message.chat.id
    
    if call.data == "back_to_services":
        show_services_menu(chat_id)
        return
    
    user_data[chat_id]["payment_method"] = call.data
    user_data[chat_id]["state"] = "waiting_for_payment_proof"

    if call.data == "pay_usdt_ton":
        usdt_amount = user_data[chat_id]["final_price"] / PAYMENT_INFO["USDT_RATE"]
        text = f"""
💎 <b>الدفع عبر USDT (TON)</b>

🔹 <b>المبلغ:</b> {usdt_amount:.2f} USDT
🔹 <b>العنوان:</b> <code>{PAYMENT_INFO["USDT_TON"]}</code>

📸 أرسل لقطة إثبات الدفع"""
    else:
        text = f"""
📱 <b>الدفع عبر الرصيد</b>

🔹 <b>المبلغ:</b> {user_data[chat_id]["final_price"]} DZD
🔹 <b>الرقم:</b> {PAYMENT_INFO["PHONE_NUMBER"]}

🔢 أرسل رقم التحويل"""
    
    markup = InlineKeyboardMarkup()
    markup.add(InlineKeyboardButton("↩ رجوع", callback_data="back_to_payment_method"))
    
    bot.edit_message_text(
        chat_id=chat_id,
        message_id=call.message.message_id,
        text=text,
        reply_markup=markup,
        parse_mode='HTML'
    )

@bot.callback_query_handler(func=lambda call: call.data == "back_to_payment_method")
def back_to_payment(call):
    chat_id = call.message.chat.id
    option = user_data[chat_id]["selected_option"]
    price = user_data[chat_id]["final_price"]
    
    markup = InlineKeyboardMarkup()
    markup.add(
        InlineKeyboardButton("💰 USDT (TON)", callback_data="pay_usdt_ton"),
        InlineKeyboardButton("📱 رصيد هاتفي", callback_data="pay_phone")
    )
    markup.add(InlineKeyboardButton("↩ رجوع", callback_data="back_to_services"))

    original_price = BASE_PRICES[option]
    discount_text = f" (وفر {original_price-price}DZD)" if price != original_price else ""
    
    bot.edit_message_text(
        chat_id=chat_id,
        message_id=call.message.message_id,
        text=f"💎 <b>الخدمة المختارة:</b> {option}\n💵 <b>السعر:</b> {price}DZD{discount_text}\n\n🔽 اختر طريقة الدفع:",
        reply_markup=markup,
        parse_mode='HTML'
    )

@bot.message_handler(content_types=['photo', 'text'], func=lambda m: user_data.get(m.chat.id, {}).get("state") == "waiting_for_payment_proof")
def handle_payment_proof(message):
    chat_id = message.chat.id
    user_info = user_data[chat_id]
    
    if user_info["payment_method"] == "pay_phone":
        if not message.text.strip().isdigit():
            bot.reply_to(message, "❌ رقم التحويل غير صحيح! يرجى إرسال أرقام فقط.")
            return
        
        user_info["transaction_id"] = message.text.strip()
    elif message.content_type == 'photo':
        user_info["payment_proof"] = message.photo[-1].file_id
    else:
        bot.reply_to(message, "❌ يرجى إرسال لقطة شاشة للإثبات")
        return
    
    save_order(chat_id)

def save_order(chat_id):
    try:
        now = datetime.now(TIMEZONE)
        formatted_time = format_time(now)
        
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        
        user_info = user_data[chat_id]
        original_price = BASE_PRICES[user_info['selected_option']]
        
        cursor.execute('''
        INSERT INTO orders 
        (user_id, username, first_name, last_name, player_id, option, 
         original_price, final_price, payment_method, transaction_id, 
         payment_proof, status, timestamp, formatted_time)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        ''', (
            chat_id,
            user_info.get('username'),
            user_info.get('first_name'),
            user_info.get('last_name'),
            user_info['player_id'],
            user_info['selected_option'],
            original_price,
            user_info['final_price'],
            user_info['payment_method'],
            user_info.get('transaction_id'),
            user_info.get('payment_proof'),
            '🔄 قيد المعالجة',
            now.strftime('%Y-%m-%d %H:%M:%S'),
            formatted_time
        ))
        
        # إضافة نقاط الإحالة إذا كان المستخدم محالاً
        cursor.execute('''
        SELECT referrer_id FROM referrals WHERE referred_id=?
        ''', (chat_id,))
        referral = cursor.fetchone()
        
        if referral:
            referrer_id = referral[0]
            add_referral_points(referrer_id, chat_id, user_info['final_price'])
        
        conn.commit()
        conn.close()
        
        bot.send_message(chat_id, "✅ تم استلام طلبك بنجاح!\n⏳ سيتم معالجته قريباً")
        notify_admin(chat_id)
        user_data.pop(chat_id, None)
    except Exception as e:
        print(f"Error in save_order: {e}")
        traceback.print_exc()
        bot.send_message(chat_id, "⚠️ حدث خطأ في حفظ الطلب، يرجى المحاولة لاحقاً")

def notify_admin(chat_id):
    try:
        user_info = user_data.get(chat_id, {})
        if not user_info:
            return
        
        username = f"@{user_info.get('username')}" if user_info.get('username') else "بدون اسم مستخدم"
        full_name = f"{user_info.get('first_name', '')} {user_info.get('last_name', '')}".strip()
        
        admin_text = f"""
📦 <b>طلب جديد!</b>
━━━━━━━━━━━━━━
⏰ <b>الوقت:</b> {format_time(datetime.now(TIMEZONE))}
👤 <b>المستخدم:</b> {username}
👤 <b>الاسم:</b> {full_name if full_name else 'غير معروف'}
🎮 <b>ID اللاعب:</b> {user_info['player_id']}
🛒 <b>الخدمة:</b> {user_info['selected_option']}
💰 <b>السعر الأصلي:</b> {BASE_PRICES[user_info['selected_option']]} DZD
💵 <b>السعر النهائي:</b> {user_info['final_price']} DZD
💳 <b>طريقة الدفع:</b> {'USDT (TON)' if user_info['payment_method'] == 'pay_usdt_ton' else 'رصيد هاتفي'}
"""
        
        if user_info.get('transaction_id'):
            admin_text += f"\n🔢 <b>رقم التحويل:</b> {user_info['transaction_id']}"
        
        markup = InlineKeyboardMarkup()
        markup.add(InlineKeyboardButton("📋 عرض الطلبات", callback_data="view_orders"))
        
        if user_info.get('payment_proof') and user_info['payment_method'] == 'pay_usdt_ton':
            try:
                bot.send_photo(ADMIN_ID, user_info['payment_proof'], caption=admin_text, reply_markup=markup, parse_mode='HTML')
                return
            except:
                pass
        
        bot.send_message(ADMIN_ID, admin_text, reply_markup=markup, parse_mode='HTML')
    except Exception as e:
        print(f"Error in notify_admin: {e}")

# ============= لوحة المشرف =============
def admin_panel(message):
    try:
        markup = InlineKeyboardMarkup()
        markup.add(InlineKeyboardButton("💰 إدارة الخصومات", callback_data="manage_discounts"))
        markup.add(InlineKeyboardButton("📋 عرض الطلبات", callback_data="view_orders"))
        markup.add(InlineKeyboardButton("✉️ إرسال إشعار", callback_data="start_broadcast"))
        markup.add(InlineKeyboardButton("📊 إحصائيات الإحالات", callback_data="referral_stats"))
        
        bot.send_message(message.chat.id, "⚙️ <b>لوحة التحكم:</b>", reply_markup=markup, parse_mode='HTML')
    except Exception as e:
        print(f"Error in admin_panel: {e}")
        bot.send_message(message.chat.id, "⚠️ حدث خطأ في تحميل لوحة التحكم")

@bot.callback_query_handler(func=lambda call: call.data == "referral_stats")
def show_referral_stats(call):
    try:
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        
        # إحصائيات عامة
        cursor.execute("SELECT COUNT(*) FROM referrals")
        total_referrals = cursor.fetchone()[0]
        
        cursor.execute("SELECT COUNT(DISTINCT referrer_id) FROM referrals")
        active_referrers = cursor.fetchone()[0]
        
        cursor.execute("SELECT SUM(points_given) FROM referral_purchases")
        total_points = cursor.fetchone()[0] or 0
        
        cursor.execute("SELECT SUM(amount) FROM referral_purchases")
        total_sales = cursor.fetchone()[0] or 0
        
        # أفضل 10 محيلين
        cursor.execute('''
        SELECT u.user_id, u.username, COUNT(r.id) as ref_count, 
               SUM(rp.points_given) as total_points, SUM(rp.amount) as total_sales
        FROM user_points u
        LEFT JOIN referrals r ON u.user_id = r.referrer_id
        LEFT JOIN referral_purchases rp ON u.user_id = rp.referrer_id
        GROUP BY u.user_id
        ORDER BY ref_count DESC
        LIMIT 10
        ''')
        top_referrers = cursor.fetchall()
        
        conn.close()
        
        stats_text = f"""
📊 <b>إحصائيات الإحالات الشاملة</b>

• إجمالي الإحالات: {total_referrals}
• عدد المحيلين النشطين: {active_referrers}
• إجمالي النقاط الموزعة: {total_points}
• إجمالي المبيعات من الإحالات: {total_sales} DZD

🏆 <b>أفضل 10 محيلين:</b>
"""
        for i, (user_id, username, ref_count, points, sales) in enumerate(top_referrers, 1):
            stats_text += f"\n{i}. @{username if username else 'غير معروف'} - {ref_count} إحالة\n   ⭐ {points or 0} نقطة | 💰 {sales or 0} DZD"
        
        markup = InlineKeyboardMarkup()
        markup.add(InlineKeyboardButton("🔙 رجوع", callback_data="back_to_admin"))
        
        bot.edit_message_text(
            chat_id=call.message.chat.id,
            message_id=call.message.message_id,
            text=stats_text,
            reply_markup=markup,
            parse_mode='HTML'
        )
    except Exception as e:
        print(f"Error in referral stats: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ في جلب الإحصائيات")

@bot.callback_query_handler(func=lambda call: call.data == "manage_discounts")
def manage_discounts(call):
    try:
        markup = InlineKeyboardMarkup()
        markup.add(InlineKeyboardButton("➕ خصم على جميع المنتجات", callback_data="add_discount_all"))
        markup.add(InlineKeyboardButton("➕ خصم على منتج محدد", callback_data="add_discount_item"))
        markup.add(InlineKeyboardButton("❌ إلغاء الخصم الحالي", callback_data="cancel_discount"))
        markup.add(InlineKeyboardButton("↩️ رجوع", callback_data="back_to_admin"))
        
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        cursor.execute("SELECT discount_amount, target_item, end_time FROM discounts WHERE is_active=1 AND datetime(end_time)>=datetime('now')")
        active_discounts = cursor.fetchall()
        conn.close()
        
        if active_discounts:
            discount_text = "<b>الخصومات النشطة:</b>\n"
            for amount, target, end_time in active_discounts:
                end_dt = datetime.strptime(end_time, '%Y-%m-%d %H:%M:%S')
                time_left = end_dt - datetime.now()
                hours = time_left.seconds // 3600
                minutes = (time_left.seconds % 3600) // 60
                discount_text += f"\n- خصم {amount}DZ على {target if target!='all' else 'الكل'} (متبقي: {hours}س {minutes}د)"
        else:
            discount_text = "لا يوجد خصومات نشطة حالياً"
        
        bot.edit_message_text(
            chat_id=call.message.chat.id,
            message_id=call.message.message_id,
            text=discount_text,
            reply_markup=markup,
            parse_mode='HTML'
        )
    except Exception as e:
        print(f"Error in manage_discounts: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ في تحميل الخصومات")

@bot.callback_query_handler(func=lambda call: call.data == "add_discount_all")
def add_discount_all(call):
    try:
        msg = bot.send_message(call.message.chat.id, "أدخل مبلغ الخصم لجميع المنتجات (بالدينار):")
        bot.register_next_step_handler(msg, process_discount_amount, 'all')
    except Exception as e:
        print(f"Error in add_discount_all: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ، يرجى المحاولة مرة أخرى")

@bot.callback_query_handler(func=lambda call: call.data == "add_discount_item")
def add_discount_item(call):
    try:
        markup = InlineKeyboardMarkup(row_width=2)
        buttons = [InlineKeyboardButton(item, callback_data=f"discount_item_{item}") for item in BASE_PRICES.keys()]
        markup.add(*buttons)
        markup.add(InlineKeyboardButton("↩️ رجوع", callback_data="manage_discounts"))
        
        bot.edit_message_text(
            chat_id=call.message.chat.id,
            message_id=call.message.message_id,
            text="اختر المنتج لتطبيق الخصم عليه:",
            reply_markup=markup
        )
    except Exception as e:
        print(f"Error in add_discount_item: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ، يرجى المحاولة مرة أخرى")

@bot.callback_query_handler(func=lambda call: call.data.startswith("discount_item_"))
def select_item_for_discount(call):
    try:
        item = call.data.replace("discount_item_", "")
        user_data[call.message.chat.id] = {"discount_item": item}
        msg = bot.send_message(call.message.chat.id, f"أدخل مبلغ الخصم للمنتج {item} (بالدينار):")
        bot.register_next_step_handler(msg, process_discount_amount, item)
    except Exception as e:
        print(f"Error in select_item_for_discount: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ، يرجى المحاولة مرة أخرى")

def process_discount_amount(message, target):
    try:
        amount = int(message.text)
        if amount <= 0:
            raise ValueError
        
        msg = bot.send_message(message.chat.id, "أدخل مدة الخصم بالساعات:")
        user_data[message.chat.id].update({
            "discount_amount": amount,
            "discount_target": target
        })
        bot.register_next_step_handler(msg, process_discount_duration)
    except:
        bot.send_message(message.chat.id, "❌ المبلغ غير صالح! يرجى إدخال رقم صحيح موجب")
        manage_discounts(message)

def process_discount_duration(message):
    try:
        hours = int(message.text)
        if hours <= 0:
            raise ValueError
        
        amount = user_data[message.chat.id]["discount_amount"]
        target = user_data[message.chat.id]["discount_target"]
        
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        
        cursor.execute('UPDATE discounts SET is_active=0')
        
        end_time = datetime.now() + timedelta(hours=hours)
        cursor.execute('''
        INSERT INTO discounts 
        (discount_amount, target_item, start_time, end_time, is_active)
        VALUES (?, ?, datetime('now'), ?, 1)
        ''', (amount, target, end_time.strftime('%Y-%m-%d %H:%M:%S')))
        
        conn.commit()
        conn.close()
        
        bot.send_message(
            message.chat.id,
            f"✅ تم تفعيل خصم {amount}DZ على {target if target!='all' else 'الكل'} لمدة {hours} ساعة",
            reply_markup=InlineKeyboardMarkup().add(
                InlineKeyboardButton("↩️ رجوع", callback_data="manage_discounts")
            ),
            parse_mode='HTML'
        )
    except:
        bot.send_message(message.chat.id, "❌ المدة غير صالحة! يرجى إدخال عدد ساعات صحيح")
        manage_discounts(message)

@bot.callback_query_handler(func=lambda call: call.data == "cancel_discount")
def cancel_discount(call):
    try:
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        cursor.execute('UPDATE discounts SET is_active=0')
        conn.commit()
        conn.close()
        
        bot.answer_callback_query(call.id, "تم إلغاء جميع الخصومات النشطة")
        manage_discounts(call)
    except Exception as e:
        print(f"Error in cancel_discount: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ في إلغاء الخصم")

@bot.callback_query_handler(func=lambda call: call.data == "view_orders")
def view_orders(call):
    try:
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM orders ORDER BY timestamp DESC LIMIT 10')
        orders = cursor.fetchall()
        conn.close()
        
        if not orders:
            bot.answer_callback_query(call.id, "لا توجد طلبات حالياً")
            return
        
        for order in orders:
            username = f"@{order[2]}" if order[2] else "بدون اسم مستخدم"
            full_name = f"{order[3] or ''} {order[4] or ''}".strip()
            
            order_text = f"""
📝 <b>تفاصيل الطلب #{order[0]}</b>
━━━━━━━━━━━━━━
⏰ <b>الوقت:</b> {order[14]}
👤 <b>المستخدم:</b> {username}
👤 <b>الاسم:</b> {full_name if full_name else 'غير معروف'}
🎮 <b>ID اللاعب:</b> {order[5]}
🛒 <b>الخدمة:</b> {order[6]}
💰 <b>السعر الأصلي:</b> {order[7]} DZD
💵 <b>السعر النهائي:</b> {order[8]} DZD
💳 <b>طريقة الدفع:</b> {order[9]}
📌 <b>الحالة:</b> {order[12]}
"""
            
            if order[10]:
                order_text += f"\n🔢 <b>رقم التحويل:</b> {order[10]}"
            
            markup = InlineKeyboardMarkup()
            if order[12] == '🔄 قيد المعالجة':
                markup.row(
                    InlineKeyboardButton("✅ قبول", callback_data=f"approve_{order[0]}"),
                    InlineKeyboardButton("❌ رفض", callback_data=f"reject_{order[0]}")
                )
                markup.add(InlineKeyboardButton("🗑️ حذف", callback_data=f"delete_{order[0]}"))
            
            if order[11] and order[9] == 'pay_usdt_ton':
                try:
                    bot.send_photo(call.message.chat.id, order[11], caption=order_text, reply_markup=markup, parse_mode='HTML')
                    continue
                except:
                    pass
            
            bot.send_message(call.message.chat.id, order_text, reply_markup=markup, parse_mode='HTML')
    except Exception as e:
        print(f"Error in view_orders: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ في عرض الطلبات")

@bot.callback_query_handler(func=lambda call: call.data.startswith(("approve_", "reject_", "delete_")))
def handle_order_action(call):
    try:
        action, order_id = call.data.split("_")
        
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        
        if action == "delete":
            cursor.execute('DELETE FROM orders WHERE id=?', (order_id,))
            conn.commit()
            conn.close()
            bot.answer_callback_query(call.id, "تم حذف الطلب بنجاح")
            view_orders(call)
            return
            
        cursor.execute('SELECT user_id, option, status FROM orders WHERE id=?', (order_id,))
        order_info = cursor.fetchone()
        
        if not order_info:
            bot.answer_callback_query(call.id, "الطلب غير موجود")
            return
            
        user_id, option, current_status = order_info
        
        if current_status != '🔄 قيد المعالجة':
            bot.answer_callback_query(call.id, "لا يمكن تعديل حالة هذا الطلب")
            return
            
        if action == "approve":
            new_status = "✅ تم التنفيذ"
            user_message = f"""
🎉 <b>تم تنفيذ طلبك بنجاح!</b>
━━━━━━━━━━━━━━
🛒 <b>الخدمة:</b> {option}
📌 <b>الحالة:</b> {new_status}
شكراً لثقتك بنا! ❤️
"""
        else:
            new_status = "❌ مرفوض"
            user_message = f"""
⚠️ <b>تم رفض طلبك</b>
━━━━━━━━━━━━━━
🛒 <b>الخدمة:</b> {option}
📌 <b>الحالة:</b> {new_status}
للمساعدة، يرجى التواصل مع الدعم.
"""
        
        cursor.execute('UPDATE orders SET status=? WHERE id=?', (new_status, order_id))
        conn.commit()
        conn.close()
        
        try:
            bot.send_message(user_id, user_message, parse_mode='HTML')
        except Exception as e:
            print(f"Failed to notify user: {e}")
        
        bot.answer_callback_query(call.id, f"تم {new_status.split()[1]} الطلب")
        view_orders(call)
        
    except Exception as e:
        print(f"Error in handle_order_action: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ في معالجة الطلب")

@bot.callback_query_handler(func=lambda call: call.data == "start_broadcast")
def start_broadcast(call):
    try:
        msg = bot.send_message(call.message.chat.id, "📝 أدخل الرسالة التي تريد إرسالها للجميع:")
        bot.register_next_step_handler(msg, process_broadcast)
    except Exception as e:
        print(f"Error in start_broadcast: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ في بدء البث")

def process_broadcast(message):
    try:
        send_broadcast(message.chat.id, message.text)
    except Exception as e:
        print(f"Error in process_broadcast: {e}")
        bot.send_message(message.chat.id, "⚠️ حدث خطأ في إرسال البث")

def send_broadcast(admin_id, message_text):
    try:
        conn = sqlite3.connect('bot_data.db')
        cursor = conn.cursor()
        cursor.execute('SELECT DISTINCT user_id FROM orders')
        users = [user[0] for user in cursor.fetchall()]
        conn.close()
        
        success = 0
        failed = 0
        
        for user_id in users:
            try:
                bot.send_message(user_id, message_text)
                success += 1
                time.sleep(0.5)
            except:
                failed += 1
        
        bot.send_message(
            admin_id,
            f"📊 <b>نتائج الإرسال:</b>\n\n✅ نجح: {success}\n❌ فشل: {failed}",
            parse_mode='HTML'
        )
    except Exception as e:
        print(f"Error in send_broadcast: {e}")
        bot.send_message(admin_id, "⚠️ حدث خطأ في إرسال الإشعارات")

@bot.callback_query_handler(func=lambda call: call.data == "back_to_admin")
def back_to_admin(call):
    try:
        admin_panel(call.message)
    except Exception as e:
        print(f"Error in back_to_admin: {e}")
        bot.answer_callback_query(call.id, "⚠️ حدث خطأ في العودة للوحة التحكم")

@bot.callback_query_handler(func=lambda call: call.data == "referral_system")
def show_referral_system(call):
    points, ref_count, total_earned = get_user_points(call.from_user.id)
    referral_code = get_user_referral_code(call.from_user.id)
    share_link = f"https://t.me/{bot.get_me().username}?start={referral_code}"
    
    markup = InlineKeyboardMarkup()
    markup.add(InlineKeyboardButton("📜 شرح نظام الإحالات", callback_data="referral_info"))
    markup.add(InlineKeyboardButton("🛒 متجر النقاط", callback_data="points_shop"))
    markup.add(InlineKeyboardButton("📤 مشاركة الرابط", url=f"https://t.me/share/url?url={share_link}&text=انضم%20إلى%20متجر%20FreeFire%20DZ%20وحصل%20على%20نقاط%20مجانية!"))
    
    bot.edit_message_text(
        chat_id=call.message.chat.id,
        message_id=call.message.message_id,
        text=f"""🎁 <b>نظام الإحالات</b>

🔗 <b>كود الإحالة:</b> <code>{referral_code}</code>
⭐ <b>نقاطك الحالية:</b> {points}
💰 <b>إجمالي نقاطك:</b> {total_earned}
👥 <b>عدد الإحالات:</b> {ref_count}

<b>رابط الإحالة:</b>\n<code>{share_link}</code>

اختر أحد الخيارات:""",
        reply_markup=markup,
        parse_mode='HTML'
    )

@bot.callback_query_handler(func=lambda call: call.data == "referral_info")
def show_referral_info(call):
    referral_code = get_user_referral_code(call.from_user.id)
    share_link = f"https://t.me/{bot.get_me().username}?start={referral_code}"
    
    explanation = f"""🎁 <b>شرح نظام الإحالات المتطور:</b>

💰 <b>كيفية الحصول على النقاط:</b>
- كل <b>250 دج</b> يدفعها المحال = <b>5 نقاط له</b> + <b>2.5 نقطة للمحيل</b>
- مثال: إذا اشترى المحال بـ 750 دج:
  - ستحصل أنت على 7.5 نقاط (2.5 × 3)
  - سيحصل هو على 15 نقطة (5 × 3)

🔗 <b>رابط الإحالة الخاص بك:</b>
<code>{share_link}</code>"""
    
    markup = InlineKeyboardMarkup()
    markup.add(InlineKeyboardButton("🔙 رجوع", callback_data="back_to_referral"))
    markup.add(InlineKeyboardButton("📤 مشاركة الرابط", url=f"https://t.me/share/url?url={share_link}&text=انضم%20إلى%20متجرنا%20واحصل%20على%20نقاط%20مجانية%20لك%20وليا!"))
    
    bot.edit_message_text(
        chat_id=call.message.chat.id,
        message_id=call.message.message_id,
        text=explanation,
        reply_markup=markup,
        parse_mode='HTML'
    )

@bot.callback_query_handler(func=lambda call: call.data == "points_shop")
def show_points_shop(call):
    points, _, _ = get_user_points(call.from_user.id)
    
    if points >= POINTS_TO_DIAMONDS['points']:
        markup = InlineKeyboardMarkup()
        markup.add(InlineKeyboardButton("💎 استبدال النقاط", callback_data="redeem_points"))
        markup.add(InlineKeyboardButton("🔙 رجوع", callback_data="back_to_referral"))
        
        bot.edit_message_text(
            chat_id=call.message.chat.id,
            message_id=call.message.message_id,
            text=f"""🛒 <b>متجر النقاط</b>

⭐ <b>نقاطك الحالية:</b> {points}

💎 يمكنك استبدال <b>{POINTS_TO_DIAMONDS['points']} نقطة</b> بـ <b>{POINTS_TO_DIAMONDS['diamonds']} جوهرة</b>""",
            reply_markup=markup,
            parse_mode='HTML'
        )
    else:
        markup = InlineKeyboardMarkup()
        markup.add(InlineKeyboardButton("🔙 رجوع", callback_data="back_to_referral"))
        
        needed = POINTS_TO_DIAMONDS['points'] - points
        bot.edit_message_text(
            chat_id=call.message.chat.id,
            message_id=call.message.message_id,
            text=f"""🛒 <b>متجر النقاط</b>

⚠️ <b>نقاطك غير كافية!</b>

⭐ لديك: {points} نقطة
⭐ تحتاج: {needed} نقطة إضافية

يمكنك الحصول على المزيد من النقاط عن طريق إحالة الأصدقاء""",
            reply_markup=markup,
            parse_mode='HTML'
        )

@bot.callback_query_handler(func=lambda call: call.data == "redeem_points")
def redeem_points(call):
    points, _, _ = get_user_points(call.from_user.id)
    
    if points < POINTS_TO_DIAMONDS['points']:
        bot.answer_callback_query(call.id, "نقاطك غير كافية لاستبدالها")
        return
    
    msg = bot.send_message(
        call.message.chat.id,
        "🔢 الرجاء إدخال <b>ID اللاعب</b> (9-10 أرقام) الذي تريد إرسال الجواهر إليه:",
        parse_mode='HTML'
    )
    
    user_data[call.from_user.id] = {
        "state": "waiting_for_player_id_for_redeem",
        "points_to_redeem": POINTS_TO_DIAMONDS['points'],
        "diamonds_to_get": POINTS_TO_DIAMONDS['diamonds'],
        "points_msg_id": msg.message_id
    }

@bot.message_handler(func=lambda m: user_data.get(m.chat.id, {}).get("state") == "waiting_for_player_id_for_redeem")
def process_redeem_player_id(message):
    player_id = message.text.strip()
    
    if not player_id.isdigit() or len(player_id) not in [9, 10]:
        bot.reply_to(message, "❌ يجب أن يكون ID اللاعب 9-10 أرقام!\n🔁 حاول مرة أخرى:")
        return
    
    user_id = message.from_user.id
    user_data[user_id]['player_id'] = player_id
    
    try:
        bot.delete_message(message.chat.id, user_data[user_id]['points_msg_id'])
    except:
        pass
    
    markup = InlineKeyboardMarkup()
    markup.add(
        InlineKeyboardButton("✅ تأكيد الاستبدال", callback_data="confirm_redeem"),
        InlineKeyboardButton("❌ إلغاء", callback_data="cancel_redeem")
    )
    
    msg = bot.send_message(
        message.chat.id,
        f"""⚠️ <b>تأكيد عملية الاستبدال</b>

🔢 ID اللاعب: <code>{player_id}</code>
⭐ النقاط المستهلكة: {user_data[user_id]['points_to_redeem']}
💎 الجواهر المستلمة: {user_data[user_id]['diamonds_to_get']}

هل أنت متأكد من عملية الاستبدال؟""",
        reply_markup=markup,
        parse_mode='HTML'
    )
    
    user_data[user_id]['confirm_msg_id'] = msg.message_id

@bot.callback_query_handler(func=lambda call: call.data == "confirm_redeem")
def confirm_redeem(call):
    user_id = call.from_user.id
    if user_id not in user_data:
        bot.answer_callback_query(call.id, "انتهت الجلسة، يرجى المحاولة مرة أخرى")
        return
    
    player_id = user_data[user_id].get('player_id')
    points_to_redeem = user_data[user_id].get('points_to_redeem')
    diamonds_to_get = user_data[user_id].get('diamonds_to_get')
    
    if not player_id or not points_to_redeem or not diamonds_to_get:
        bot.answer_callback_query(call.id, "حدث خطأ، يرجى المحاولة مرة أخرى")
        return
    
    current_points, _, _ = get_user_points(user_id)
    if current_points < points_to_redeem:
        bot.answer_callback_query(call.id, "نقاطك غير كافية")
        return
    
    new_points = add_user_points(user_id, -points_to_redeem)
    
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()
    cursor.execute('''
    INSERT INTO points_redemption 
    (user_id, player_id, points_used, status, timestamp)
    VALUES (?, ?, ?, 'pending', datetime('now'))
    ''', (user_id, player_id, points_to_redeem))
    
    conn.commit()
    conn.close()
    
    try:
        username = f"@{call.from_user.username}" if call.from_user.username else "بدون اسم مستخدم"
        admin_text = f"""
⚠️ <b>طلب استبدال نقاط جديد!</b>

👤 المستخدم: {username} ({user_id})
🔢 ID اللاعب: {player_id}
⭐ النقاط المستهلكة: {points_to_redeem}
💎 الجواهر المطلوبة: {diamonds_to_get}

⏰ الوقت: {format_time(datetime.now(TIMEZONE))}"""
        markup = InlineKeyboardMarkup()
        markup.add(
            InlineKeyboardButton("✅ قبول", callback_data=f"approve_redemption_{cursor.lastrowid}"),
            InlineKeyboardButton("❌ رفض", callback_data=f"reject_redemption_{cursor.lastrowid}")
        )
        
        bot.send_message(ADMIN_ID, admin_text, reply_markup=markup, parse_mode='HTML')
    except Exception as e:
        print(f"Error sending admin notification: {e}")
    
    bot.edit_message_text(
        chat_id=call.message.chat.id,
        message_id=call.message.message_id,
        text=f"""✅ <b>تم تقديم طلب استبدال النقاط بنجاح!</b>

🔢 ID اللاعب: <code>{player_id}</code>
⭐ النقاط المستهلكة: {points_to_redeem}
💎 الجواهر المطلوبة: {diamonds_to_get}

سيتم مراجعة طلبك من قبل الإدارة وإعلامك بالقرار.""",
        parse_mode='HTML'
    )
    
    user_data.pop(user_id, None)

@bot.callback_query_handler(func=lambda call: call.data.startswith(("approve_redemption_", "reject_redemption_")))
def handle_redemption_decision(call):
    action, redemption_id = call.data.split("_")[2:]
    redemption_id = int(redemption_id)
    
    conn = sqlite3.connect('bot_data.db')
    cursor = conn.cursor()
    cursor.execute('''
    SELECT user_id, player_id, points_used FROM points_redemption 
    WHERE id=? AND status='pending'
    ''', (redemption_id,))
    redemption = cursor.fetchone()
    
    if not redemption:
        bot.answer_callback_query(call.id, "الطلب غير موجود أو تم معالجته مسبقاً")
        return
    
    user_id, player_id, points_used = redemption
    
    if action == "approve":
        cursor.execute('''
        UPDATE points_redemption SET status='approved' 
        WHERE id=?
        ''', (redemption_id,))
        
        try:
            bot.send_message(
                user_id,
                f"""🎉 <b>تمت الموافقة على طلب استبدال النقاط!</b>

🔢 ID اللاعب: <code>{player_id}</code>
💎 الجواهر المضافة: {POINTS_TO_DIAMONDS['diamonds']}

شكراً لاستخدامك نظام الإحالات!""",
                parse_mode='HTML'
            )
        except:
            pass
        
        bot.answer_callback_query(call.id, "تمت الموافقة على الطلب")
        bot.edit_message_text(
            chat_id=call.message.chat.id,
            message_id=call.message.message_id,
            text=f"""✅ <b>تمت الموافقة على الطلب #{redemption_id}</b>

👤 المستخدم: {user_id}
🔢 ID اللاعب: {player_id}
⭐ النقاط: {points_used}
💎 الجواهر: {POINTS_TO_DIAMONDS['diamonds']}""",
            parse_mode='HTML'
        )
    else:
        add_user_points(user_id, points_used)
        
        cursor.execute('''
        UPDATE points_redemption SET status='rejected' 
        WHERE id=?
        ''', (redemption_id,))
        
        try:
            bot.send_message(
                user_id,
                f"""⚠️ <b>تم رفض طلب استبدال النقاط</b>

🔢 ID اللاعب: <code>{player_id}</code>
⭐ النقاط المسترجعة: {points_used}

للمساعدة، يرجى التواصل مع الدعم.""",
                parse_mode='HTML'
            )
        except:
            pass
        
        bot.answer_callback_query(call.id, "تم رفض الطلب واستعادة النقاط")
        bot.edit_message_text(
            chat_id=call.message.chat.id,
            message_id=call.message.message_id,
            text=f"""❌ <b>تم رفض الطلب #{redemption_id}</b>

👤 المستخدم: {user_id}
🔢 ID اللاعب: {player_id}
⭐ النقاط المسترجعة: {points_used}""",
            parse_mode='HTML'
        )
    
    conn.commit()
    conn.close()

@bot.callback_query_handler(func=lambda call: call.data in ["back_to_referral", "cancel_redeem"])
def back_to_referral_menu(call):
    if call.data == "cancel_redeem":
        user_id = call.from_user.id
        if user_id in user_data:
            user_data.pop(user_id)
    
    show_referral_system(call)

if __name__ == "__main__":
    print("✅ البوت يعمل بنجاح!")
    bot.infinity_polling()