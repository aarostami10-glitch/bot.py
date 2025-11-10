import sqlite3
from telegram import Bot, Update, InlineKeyboardMarkup, InlineKeyboardButton
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes

# =======================
# تنظیمات ربات
# =======================
BOT_TOKEN = "8444443288:AAHGIDqXD0J_0LkPDWUQbnyGDtNK4sI4obs"
CHANNEL_ID = "@Game_ET30"
ADMINS = ["@AmirAliT313", "@Alireza_AriYo"]

MAX_POINTS = 500
POLL_POINT_VALUE = 10

# =======================
# دیتابیس
# =======================
conn = sqlite3.connect('bot_data.db', check_same_thread=False)
cur = conn.cursor()

cur.execute('''CREATE TABLE IF NOT EXISTS users (user_id INTEGER PRIMARY KEY, points INTEGER DEFAULT 0)''')
cur.execute('''CREATE TABLE IF NOT EXISTS poll_points (user_id INTEGER PRIMARY KEY, points_pending INTEGER DEFAULT 0)''')
cur.execute('''CREATE TABLE IF NOT EXISTS posted_files (message_id INTEGER PRIMARY KEY, file_id TEXT, title TEXT, cost INTEGER)''')
cur.execute('''CREATE TABLE IF NOT EXISTS feedback (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, text TEXT, date TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')
conn.commit()

# =======================
# توابع کمکی
# =======================
def get_points(user_id):
    cur.execute("SELECT points FROM users WHERE user_id=?", (user_id,))
    row = cur.fetchone()
    return row[0] if row else 0

def set_points(user_id, points):
    cur.execute("INSERT INTO users(user_id, points) VALUES(?,?) ON CONFLICT(user_id) DO UPDATE SET points=?", (user_id, points, points))
    conn.commit()

def get_pending_points(user_id):
    cur.execute("SELECT points_pending FROM poll_points WHERE user_id=?", (user_id,))
    row = cur.fetchone()
    return row[0] if row else 0

def add_pending_points(user_id, points=POLL_POINT_VALUE):
    current = get_points(user_id)
    cur.execute("SELECT points_pending FROM poll_points WHERE user_id=?", (user_id,))
    row = cur.fetchone()
    pending = row[0] if row else 0
    total = current + pending + points
    if total > MAX_POINTS:
        points = MAX_POINTS - current - pending
        if points <= 0:
            return 0
    if row:
        cur.execute("UPDATE poll_points SET points_pending = points_pending + ? WHERE user_id=?", (points, user_id))
    else:
        cur.execute("INSERT INTO poll_points(user_id, points_pending) VALUES(?,?)", (user_id, points))
    conn.commit()
    return points

def withdraw_pending_points(user_id):
    cur.execute("SELECT points_pending FROM poll_points WHERE user_id=?", (user_id,))
    row = cur.fetchone()
    if not row or row[0] <= 0:
        return 0
    points = row[0]
    current = get_points(user_id)
    new_points = min(current + points, MAX_POINTS)
    set_points(user_id, new_points)
    cur.execute("UPDATE poll_points SET points_pending=0 WHERE user_id=?", (user_id,))
    conn.commit()
    return points

def get_all_files():
    cur.execute("SELECT rowid, title, file_id, cost FROM posted_files ORDER BY rowid ASC")
    return cur.fetchall()

def save_file(message_id, file_id, title, cost):
    cur.execute("INSERT OR REPLACE INTO posted_files(message_id, file_id, title, cost) VALUES(?,?,?,?)",
                (message_id, file_id, title, cost))
    conn.commit()

def save_feedback(user_id, text):
    cur.execute("INSERT INTO feedback(user_id, text) VALUES(?,?)", (user_id, text))
    conn.commit()

# =======================
# توابع ربات
# =======================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    set_points(user_id, get_points(user_id))
    keyboard = [
        [InlineKeyboardButton("شارژ امتیاز ⛽", callback_data="menu_charge")],
        [InlineKeyboardButton("ارتباط با ما 📞", callback_data="support")],
        [InlineKeyboardButton("آیتم‌ها 🪙", callback_data="menu_buy")],
        [InlineKeyboardButton("نظرات و پیشنهادات 📝", callback_data="feedback")]
    ]
    await update.message.reply_text("به ربات خوش آمدید!", reply_markup=InlineKeyboardMarkup(keyboard))

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id
    data = query.data

    if data == "menu_charge":
        current = get_points(user_id)
        pending = get_pending_points(user_id)
        text = f"موجودی: {current}/{MAX_POINTS}\nقابل برداشت: {pending} امتیاز"
        keyboard = []
        if pending > 0:
            keyboard.append([InlineKeyboardButton(f"برداشت امتیاز ({pending})", callback_data="withdraw_points")])
        keyboard.append([InlineKeyboardButton("بازگشت", callback_data="menu")])
        await query.message.reply_text(text, reply_markup=InlineKeyboardMarkup(keyboard))

    elif data == "withdraw_points":
        points = withdraw_pending_points(user_id)
        current = get_points(user_id)
        await query.message.reply_text(f"امتیاز قابل برداشت شما اضافه شد. امتیاز فعلی: {current}/{MAX_POINTS}")

    elif data == "support":
        keyboard = [[InlineKeyboardButton(admin, url=f"https://t.me/{admin[1:]}")] for admin in ADMINS]
        keyboard.append([InlineKeyboardButton("بازگشت", callback_data="menu")])
        await query.message.reply_text("برای ارتباط با ادمین‌ها روی نام آن‌ها کلیک کنید:", reply_markup=InlineKeyboardMarkup(keyboard))

    elif data == "menu_buy":
        files = get_all_files()
        if not files:
            await query.message.reply_text("هیچ آیتمی برای نمایش وجود ندارد.")
            return
        text = "لیست آیتم‌ها:\n\n"
        keyboard = []
        for f in files:
            idx, title, file_id, cost = f
            text += f"{idx}. {title}\nامتیاز: {cost}\n\n"
            keyboard.append([InlineKeyboardButton(f"{title} - خرید ({cost} امتیاز)", callback_data=f"buy_{idx}")])
        keyboard.append([InlineKeyboardButton("بازگشت", callback_data="menu")])
        await query.message.reply_text(text, reply_markup=InlineKeyboardMarkup(keyboard))

    elif data.startswith("buy_"):
        idx = int(data.split("_")[1])
        files = get_all_files()
        file = next((f for f in files if f[0] == idx), None)
        if not file:
            await query.message.reply_text("آیتم پیدا نشد.")
            return
        title, file_id, cost = file[1], file[2], file[3]
        user_points = get_points(user_id)
        if user_points < cost:
            await query.message.reply_text(f"امتیاز کافی ندارید. امتیاز فعلی شما: {user_points}/{MAX_POINTS}")
            return
        set_points(user_id, user_points - cost)
        await context.bot.send_document(chat_id=user_id, document=file_id, caption=f"دانلود: {title}")
        await query.message.reply_text(f"خرید موفق! امتیاز باقی‌مانده: {get_points(user_id)}/{MAX_POINTS}")

    elif data == "feedback":
        await query.message.reply_text("لطفاً نظر یا پیشنهاد خود را ارسال کنید:")

    elif data == "menu":
        keyboard = [
            [InlineKeyboardButton("شارژ امتیاز ⛽", callback_data="menu_charge")],
            [InlineKeyboardButton("ارتباط با ما 📞", callback_data="support")],
            [InlineKeyboardButton("آیتم‌ها 🪙", callback_data="menu_buy")],
            [InlineKeyboardButton("نظرات و پیشنهادات 📝", callback_data="feedback")]
        ]
        await query.message.reply_text("منوی اصلی:", reply_markup=InlineKeyboardMarkup(keyboard))

async def message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    text = update.message.text
    if text and not text.startswith("/"):
        save_feedback(user_id, text)
        for admin in ADMINS:
            await context.bot.send_message(chat_id=admin, text=f"نظر جدید از @{update.effective_user.username or user_id}:\n{text}")
        await update.message.reply_text("نظر شما ثبت شد. ممنون!")

async def channel_post_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = update.channel_post
    caption = msg.caption or ""
    cost = 0
    if "امتیاز:" in caption:
        try:
            cost = int(caption.split("امتیاز:")[1].split()[0])
        except:
            cost = 0
    save_file(msg.message_id, msg.document.file_id, msg.caption or "بدون عنوان", cost)

# =======================
# راه‌اندازی ربات
# =======================
app = ApplicationBuilder().token(BOT_TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(CallbackQueryHandler(button_handler))
app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), message_handler))
app.add_handler(MessageHandler(filters.ALL & filters.CHANNELS, channel_post_handler))

print("Bot started. Press Ctrl+C to stop.")
app.run_polling()# bot.py
