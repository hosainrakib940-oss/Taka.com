# Taka.com
@Takadotcom_bot
# app.py
import os
import sqlite3
import random
import json
from flask import Flask, request, abort
import telebot
from telebot import types
from datetime import datetime

# ====== CONFIG ======
BOT_TOKEN = os.environ.get("BOT_TOKEN", "YOUR_BOT_TOKEN_HERE")
WEBHOOK_PATH = f"/webhook/{BOT_TOKEN}"
ADMIN_ID = int(os.environ.get("ADMIN_ID", "0"))  # set your telegram id as admin
DATABASE = os.environ.get("DATABASE_PATH", "bot_data.db")
PORT = int(os.environ.get("PORT", 5000))

bot = telebot.TeleBot(BOT_TOKEN, parse_mode="HTML")
app = Flask(__name__)

# ====== DB HELPERS ======
def get_conn():
    conn = sqlite3.connect(DATABASE, check_same_thread=False)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_conn()
    cur = conn.cursor()
    # users: id, username, points, created_at
    cur.execute("""
    CREATE TABLE IF NOT EXISTS users(
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        points INTEGER DEFAULT 0,
        created_at TEXT
    )
    """)
    # transactions log
    cur.execute("""
    CREATE TABLE IF NOT EXISTS transactions(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        amount INTEGER,
        reason TEXT,
        ts TEXT
    )
    """)
    # spin history
    cur.execute("""
    CREATE TABLE IF NOT EXISTS spin_history(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        result TEXT,
        points INTEGER,
        ts TEXT
    )
    """)
    # quiz questions
    cur.execute("""
    CREATE TABLE IF NOT EXISTS quiz_questions(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        question TEXT,
        options TEXT,
        answer_index INTEGER
    )
    """)
    # quiz attempts
    cur.execute("""
    CREATE TABLE IF NOT EXISTS quiz_attempts(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        question_id INTEGER,
        correct INTEGER,
        ts TEXT
    )
    """)
    # lucky draw events
    cur.execute("""
    CREATE TABLE IF NOT EXISTS lucky_events(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT,
        ticket_price INTEGER,
        created_at TEXT,
        drawn INTEGER DEFAULT 0,
        winner_id INTEGER
    )
    """)
    # tickets
    cur.execute("""
    CREATE TABLE IF NOT EXISTS tickets(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        event_id INTEGER,
        user_id INTEGER,
        ts TEXT
    )
    """)
    conn.commit()

    # seed sample quiz questions if none exist
    cur.execute("SELECT COUNT(*) as cnt FROM quiz_questions")
    if cur.fetchone()[0] == 0:
        sample = [
            {
                "question": "বাংলাদেশের স্বাধীনতার বছর কখন?",
                "options": ["1970", "1971", "1972", "1969"],
                "answer": 1
            },
            {
                "question": "পৃথিবীর সবচেয়ে বড় মহাসাগর কোনটি?",
                "options": ["আট্লান্টিক", "ইন্ডিয়ান", "প্রশান্ত", "আর্কটিক"],
                "answer": 2
            },
            {
                "question": "মানবদেহে সবচেয়ে বড় অঙ্গ কোনটি?",
                "options": ["লিভার", "হৃদয়", "চামড়া", "ফুসফুস"],
                "answer": 2
            },
            {
                "question": "HTML কী ধরনের ভাষা?",
                "options": ["Programming", "Markup", "Styling", "Database"],
                "answer": 1
            },
            {
                "question": "পাইথন কোন প্রতিষ্ঠাতা কোম্পানীর তৈরি?",
                "options": ["Google", "Microsoft", "Python Software Foundation", "Apple"],
                "answer": 2
            }
        ]
        for q in sample:
            cur.execute("INSERT INTO quiz_questions(question, options, answer_index) VALUES (?,?,?)",
                        (q['question'], json.dumps(q['options'], ensure_ascii=False), q['answer']))
        conn.commit()
    conn.close()


def ensure_user(user):
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT * FROM users WHERE user_id = ?", (user.id,))
    r = cur.fetchone()
    if not r:
        cur.execute("INSERT INTO users(user_id, username, points, created_at) VALUES (?, ?, ?, ?)",
                    (user.id, getattr(user, "username", ""), 50, datetime.utcnow().isoformat()))
        cur.execute("INSERT INTO transactions(user_id, amount, reason, ts) VALUES (?, ?, ?, ?)",
                    (user.id, 50, "signup_bonus", datetime.utcnow().isoformat()))
        conn.commit()
    conn.close()


def change_points(user_id, amount, reason=""):
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("UPDATE users SET points = points + ? WHERE user_id = ?", (amount, user_id))
    cur.execute("INSERT INTO transactions(user_id, amount, reason, ts) VALUES (?, ?, ?, ?)",
                (user_id, amount, reason, datetime.utcnow().isoformat()))
    conn.commit()
    conn.close()


def get_points(user_id):
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT points FROM users WHERE user_id = ?", (user_id,))
    r = cur.fetchone()
    conn.close()
    return r["points"] if r else 0

# ====== BOT HANDLERS ======
@bot.message_handler(commands=["start"])
def start_handler(msg):
    ensure_user(msg.from_user)
    txt = ("👋 হ্যালো! বট-এ স্বাগতম।\n\n"
           "💎 তোমার আজকের বোনাস: 50 পয়েন্ট (একবারই)\n"
           "🎮 কুইজ, স্পিন, লাকি ড্র - খেলে পয়েন্ট জিতে নাও।\n\n"
           "কমান্ডসমূহ:\n"
           "/balance - তোমার পয়েন্ট\n"
           "/spin - Spin (free/paid variants)\n"
           "/quiz - র্যান্ডম কুইজ\n"
           "/lucky - লাকি ড্র মেনু\n"
           "/help - সাহায্য")
    bot.send_message(msg.chat.id, txt)

@bot.message_handler(commands=["help"])
def help_handler(msg):
    txt = ("Available Commands:\n"
           "/balance - দেখো তোমার পয়েন্ট\n"
           "/spin - Spin গেম\n"
           "/quiz - Quiz খেলা\n"
           "/lucky - Lucky Draw (buy tickets / draw)\n"
           "/profile - তোমার উৎসব/ইতিহাস")
    bot.send_message(msg.chat.id, txt)

@bot.message_handler(commands=["balance"])
def balance_handler(msg):
    ensure_user(msg.from_user)
    pts = get_points(msg.from_user.id)
    bot.send_message(msg.chat.id, f"💰 তোমার ব্যালান্স: {pts} পয়েন্ট")

# ====== SPIN GAME ======
SPIN_SLOTS = [
    ("Small Win", 5),
    ("Jackpot", 200),
    ("Try Again", 0),
    ("Medium Win", 30),
    ("Lose", -5),
    ("Bonus", 50),
    ("Tiny", 2),
    ("Big Win", 100),
]

@bot.message_handler(commands=["spin"])
def spin_menu(msg):
    ensure_user(msg.from_user)
    kb = types.InlineKeyboardMarkup()
    kb.add(types.InlineKeyboardButton("Free Spin (cooldown)", callback_data="spin_free"))
    kb.add(types.InlineKeyboardButton("Paid Spin (20 pts)", callback_data="spin_paid"))
    bot.send_message(msg.chat.id, "🎡 Spin Menu: চয়ন করো", reply_markup=kb)

@bot.callback_query_handler(func=lambda c: c.data and c.data.startswith("spin_"))
def spin_callback(call):
    user = call.from_user
    ensure_user(user)
    action = call.data.split("_")[1]
    if action == "free":
        slot = random.choice(SPIN_SLOTS)
        label, pts = slot
        change_points(user.id, pts, reason=f"spin_free:{label}")
        conn = get_conn()
        conn.execute("INSERT INTO spin_history(user_id,result,points,ts) VALUES (?,?,?,?)",
                     (user.id, label, pts, datetime.utcnow().isoformat()))
        conn.commit(); conn.close()
        bot.answer_callback_query(call.id, f"{label}! {pts} পয়েন্ট", show_alert=True)
        bot.send_message(call.message.chat.id, f"🎉 Result: <b>{label}</b>\nPoints: {pts}\nNow Balance: {get_points(user.id)}")
    elif action == "paid":
        price = 20
        current = get_points(user.id)
        if current < price:
            bot.answer_callback_query(call.id, "পয়েন্ট কম আছে", show_alert=True)
            return
        change_points(user.id, -price, "spin_paid_entry")
        slot = random.choice(SPIN_SLOTS)
        label, pts = slot
        change_points(user.id, pts, reason=f"spin_paid:{label}")
        conn = get_conn()
        conn.execute("INSERT INTO spin_history(user_id,result,points,ts) VALUES (?,?,?,?)",
                     (user.id, label, pts, datetime.utcnow().isoformat()))
        conn.commit(); conn.close()
        bot.answer_callback_query(call.id, f"{label}! {pts} পয়েন্ট", show_alert=True)
        bot.send_message(call.message.chat.id, f"🎉 Paid Spin Result: <b>{label}</b>\nPoints: {pts}\nNow Balance: {get_points(user.id)}")

# ====== QUIZ GAME ======
@bot.message_handler(commands=["quiz"])
def quiz_start(msg):
    ensure_user(msg.from_user)
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT * FROM quiz_questions ORDER BY RANDOM() LIMIT 1")
    q = cur.fetchone()
    conn.close()
    if not q:
        bot.send_message(msg.chat.id, "কোনো কুইজ প্রশ্ন পাওয়া যায়নি। (Admin যোগ করতে হবে)")
        return
    options = json.loads(q["options"])
    kb = types.InlineKeyboardMarkup()
    for i, opt in enumerate(options):
        kb.add(types.InlineKeyboardButton(opt, callback_data=f"quiz_{q['id']}_{i}"))
    bot.send_message(msg.chat.id, f"❓ <b>{q['question']}</b>", reply_markup=kb)

@bot.callback_query_handler(func=lambda c: c.data and c.data.startswith("quiz_"))
def quiz_answer(call):
    user = call.from_user
    ensure_user(user)
    _, qid, idx = call.data.split("_")
    qid = int(qid); idx = int(idx)
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT * FROM quiz_questions WHERE id = ?", (qid,))
    q = cur.fetchone()
    if not q:
        bot.answer_callback_query(call.id, "Invalid question.", show_alert=True); conn.close(); return
    correct = 1 if idx == q["answer_index"] else 0
    cur.execute("INSERT INTO quiz_attempts(user_id,question_id,correct,ts) VALUES (?,?,?,?)",
                (user.id, qid, correct, datetime.utcnow().isoformat()))
    if correct:
        reward = 30
        change_points(user.id, reward, reason=f"quiz_correct_q{qid}")
        bot.answer_callback_query(call.id, "সঠিক উত্তর! পুরস্কার পেলেই — 30 পয়েন্ট", show_alert=True)
        bot.send_message(call.message.chat.id, f"✅ সঠিক! তুমি পেয়েছো {reward} পয়েন্ট। এখন ব্যালান্স: {get_points(user.id)}")
    else:
        bot.answer_callback_query(call.id, "ভুল উত্তর।", show_alert=True)
        bot.send_message(call.message.chat.id, f"❌ ভুল উত্তর। সঠিক উত্তর ছিল অপশন #{q['answer_index']+1}")
    conn.commit(); conn.close()

# ====== LUCKY DRAW ======
@bot.message_handler(commands=["lucky"])
def lucky_menu(msg):
    ensure_user(msg.from_user)
    kb = types.InlineKeyboardMarkup()
    kb.add(types.InlineKeyboardButton("List Events", callback_data="lucky_list"))
    kb.add(types.InlineKeyboardButton("My Tickets", callback_data="lucky_my"))
    if msg.from_user.id == ADMIN_ID:
        kb.add(types.InlineKeyboardButton("Create Event (admin)", callback_data="lucky_create"))
        kb.add(types.InlineKeyboardButton("Draw Winner (admin)", callback_data="lucky_draw"))
    bot.send_message(msg.chat.id, "🎟 Lucky Draw Menu", reply_markup=kb)

@bot.callback_query_handler(func=lambda c: c.data and c.data.startswith("lucky_"))
def lucky_cb(call):
    user = call.from_user
    ensure_user(user)
    action = call.data.split("_",1)[1]
    conn = get_conn(); cur = conn.cursor()
    if action == "list":
        cur.execute("SELECT * FROM lucky_events WHERE drawn = 0")
        rows = cur.fetchall()
        if not rows:
            bot.send_message(call.message.chat.id, "কোনো চলমান ইভেন্ট নেই।")
        else:
            text = "🎪 চলমান Lucky Events:\n\n"
            for r in rows:
                text += f"ID: {r['id']} | {r['name']} | Ticket: {r['ticket_price']} pts\n"
                kb = types.InlineKeyboardMarkup()
                kb.add(types.InlineKeyboardButton("Buy Ticket", callback_data=f"lucky_buy_{r['id']}"))
                bot.send_message(call.message.chat.id, text, reply_markup=kb)
                text = ""
    elif action.startswith("buy_"):
        pass
    elif action == "my":
        cur.execute("SELECT e.name, COUNT(t.id) as cnt FROM tickets t JOIN lucky_events e ON e.id=t.event_id WHERE t.user_id=? GROUP BY e.id", (user.id,))
        rows = cur.fetchall()
        if not rows:
            bot.send_message(call.message.chat.id, "তোমার কোনো টিকেট নেই।")
        else:
            text = "🎫 তোমার টিকেটসমূহ:\n"
            for r in rows:
                text += f"{r['name']}: {r['cnt']} টিকেট\n"
            bot.send_message(call.message.chat.id, text)
    elif action == "create":
        if user.id != ADMIN_ID:
            bot.answer_callback_query(call.id, "Not allowed", show_alert=True); conn.close(); return
        cur.execute("INSERT INTO lucky_events(name,ticket_price,created_at) VALUES (?,?,?)",
                    ("Default Event", 10, datetime.utcnow().isoformat()))
        conn.commit()
        bot.send_message(call.message.chat.id, "Event created: Default Event (10 pts ticket)")
    elif action == "draw":
        if user.id != ADMIN_ID:
            bot.answer_callback_query(call.id, "Not allowed", show_alert=True); conn.close(); return
        cur.execute("SELECT * FROM lucky_events WHERE drawn = 0")
        events = cur.fetchall()
        if not events:
            bot.send_message(call.message.chat.id, "কোনো চলমান event নেই।"); conn.close(); return
        for ev in events:
            cur.execute("SELECT * FROM tickets WHERE event_id = ?", (ev["id"],))
            tickets = cur.fetchall()
            if not tickets:
                bot.send_message(call.message.chat.id, f"No tickets sold for {ev['name']}. Skipping.")
                continue
            winner_ticket = random.choice(tickets)
            winner_id = winner_ticket["user_id"]
            cur.execute("UPDATE lucky_events SET drawn=1, winner_id=? WHERE id=?", (winner_id, ev["id"]))
            pot = len(tickets) * ev["ticket_price"]
            change_points(winner_id, pot, reason=f"lucky_win_event_{ev['id']}")
            bot.send_message(call.message.chat.id, f"🎉 Event <b>{ev['name']}</b> winner: <a href='tg://user?id={winner_id}'>User</a>\nPrize: {pot} pts")
        conn.commit(); conn.close()
    conn.close()

@bot.callback_query_handler(func=lambda c: c.data and c.data.startswith("lucky_buy_"))
def lucky_buy(call):
    user = call.from_user
    ensure_user(user)
    _, _, ev_id = call.data.split("_")
    ev_id = int(ev_id)
    conn = get_conn(); cur = conn.cursor()
    cur.execute("SELECT * FROM lucky_events WHERE id = ? AND drawn = 0", (ev_id,))
    ev = cur.fetchone()
    if not ev:
        bot.answer_callback_query(call.id, "Event not available", show_alert=True); conn.close(); return
    price = ev["ticket_price"]
    if get_points(user.id) < price:
        bot.answer_callback_query(call.id, "পয়েন্ট কম আছে", show_alert=True); conn.close(); return
    change_points(user.id, -price, reason=f"ticket_buy_event_{ev_id}")
    cur.execute("INSERT INTO tickets(event_id,user_id,ts) VALUES (?,?,?)", (ev_id, user.id, datetime.utcnow().isoformat()))
    conn.commit(); conn.close()
    bot.answer_callback_query(call.id, "Ticket bought!", show_alert=True)
    bot.send_message(call.message.chat.id, f"🎟 টিকেট ক্রয় সম্পন্ন! Event: {ev['name']} | Price: {price} pts\nNow Balance: {get_points(user.id)}")

# ====== ADMIN COMMANDS ======
@bot.message_handler(commands=["add_quiz"])
def add_quiz_handler(msg):
    if msg.from_user.id != ADMIN_ID:
        bot.reply_to(msg, "Not allowed")
        return
    try:
        payload = msg.text.partition(" ")[2]
        data = json.loads(payload)
        q = data["question"]
        options = data["options"]
        answer = int(data["answer"])
        conn = get_conn(); cur = conn.cursor()
        cur.execute("INSERT INTO quiz_questions(question,options,answer_index) VALUES (?,?,?)",
                    (q, json.dumps(options, ensure_ascii=False), answer))
        conn.commit(); conn.close()
        bot.reply_to(msg, "Quiz added.")
    except Exception as e:
        bot.reply_to(msg, f"Error: {e}\nUsage: /add_quiz {{\"question\":\"..\",\"options\":[\"a\",\"b\"],\"answer\":0}}")

@bot.message_handler(commands=["create_lucky"])
def create_lucky(msg):
    if msg.from_user.id != ADMIN_ID:
        bot.reply_to(msg, "Not allowed"); return
    try:
        payload = msg.text.partition(" ")[2]
        name, price = payload.split("|",1)
        price = int(price.strip())
        conn = get_conn(); cur = conn.cursor()
        cur.execute("INSERT INTO lucky_events(name,ticket_price,created_at) VALUES (?,?,?)",
                    (name.strip(), price, datetime.utcnow().isoformat()))
        conn.commit(); conn.close()
        bot.reply_to(msg, "Lucky event created.")
    except Exception as e:
        bot.reply_to(msg, "Usage: /create_lucky Event Name | 10")

# ====== WEBHOOK ROUTE ======
@app.route(WEBHOOK_PATH, methods=['POST'])
def webhook():
    if request.headers.get('content-type') == 'application/json':
        json_string = request.get_data().decode('utf-8')
        update = telebot.types.Update.de_json(json_string)
        bot.process_new_updates([update])
        return "", 200
    else:
        abort(403)

# ====== SETUP ======
if __name__ == "__main__":
    init_db()
    PUBLIC_URL = os.environ.get("PUBLIC_URL")
    if PUBLIC_URL:
        webhook_url = PUBLIC_URL + WEBHOOK_PATH
        bot.remove_webhook()
        bot.set_webhook(url=webhook_url)
        print(f"Webhook set to: {webhook_url}")
    else:
        print("PUBLIC_URL not set; remember to set webhook manually.")
    app.run(host="0.0.0.0", port=PORT)
