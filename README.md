import os
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes
import gspread
from oauth2client.service_account import ServiceAccountCredentials

TOKEN = os.getenv("BOT_TOKEN")

# --- Load Google Sheet ---
def load_pilot_data():
    scope = ["https://spreadsheets.google.com/feeds",
             "https://www.googleapis.com/auth/drive"]
    creds = ServiceAccountCredentials.from_json_keyfile_name("service_account.json", scope)
    client = gspread.authorize(creds)

    sheet = client.open("PilotLibrary").sheet1
    records = sheet.get_all_records()

    data = []
    for row in records:
        name = row['Pilot Name']
        callsign = row['Callsign']
        data.append({'name': name, 'callsign': callsign})
    return data

PILOT_LIST = load_pilot_data()

# --- Store user selections ---
user_selection = {}

def get_next_seven_days():
    today = datetime.today()
    return [(today + timedelta(days=i)).strftime("%Y-%m-%d") for i in range(7)]

# --- Helper: build summary text ---
def build_summary(user_id, editing_wave=None):
    selection = user_selection[user_id]
    text = f"📅 Date: {selection['date']}\n\n"
    for wave in ["Wave 1", "Wave 2", "Wave 3", "Wave 4"]:
        names = selection["pilots"].get(wave, [])
        if names:
            text += f"{wave}: {', '.join(names)}\n"
        else:
            text += f"{wave}: —\n"
    if editing_wave:
        text += f"\nNow editing: {editing_wave}"
    return text

# --- Start ---
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    dates = get_next_seven_days()
    keyboard = [[InlineKeyboardButton(d, callback_data=f"date|{d}")] for d in dates]
    await update.message.reply_text("Select flying date:", reply_markup=InlineKeyboardMarkup(keyboard))

# --- Handle button clicks ---
async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    data = query.data.split("|")
    user_id = query.from_user.id

    # 1. Date selected
    if data[0] == "date":
        flying_date = data[1]
        user_selection[user_id] = {"date": flying_date, "pilots": {"Wave 1": [], "Wave 2": [], "Wave 3": [], "Wave 4": []}}

        keyboard = [
            [InlineKeyboardButton("Wave 1", callback_data=f"wave|{flying_date}|Wave 1")],
            [InlineKeyboardButton("Wave 2", callback_data=f"wave|{flying_date}|Wave 2")],
            [InlineKeyboardButton("Wave 3", callback_data=f"wave|{flying_date}|Wave 3")],
            [InlineKeyboardButton("Wave 4", callback_data=f"wave|{flying_date}|Wave 4")],
            [InlineKeyboardButton("✔ Finish", callback_data="done")]
        ]
        await query.edit_message_text(f"📅 Date: {flying_date}\nSelect a wave:",
                                      reply_markup=InlineKeyboardMarkup(keyboard))

    # 2. Wave selected → show pilot list
    elif data[0] == "wave":
        flying_date, wave = data[1], data[2]

        selection = user_selection[user_id]["pilots"][wave]

        keyboard = []
        for p in PILOT_LIST:
            if p["name"] in selection:
                keyboard.append([InlineKeyboardButton(f"❌ {p['name']} ({p['callsign']})",
                                                      callback_data=f"remove|{wave}|{p['name']}")])
            else:
                keyboard.append([InlineKeyboardButton(f"➕ {p['name']} ({p['callsign']})",
                                                      callback_data=f"add|{wave}|{p['name']}")])

        keyboard.append([InlineKeyboardButton("⬅ Back to Waves", callback_data=f"back|{flying_date}")])
        keyboard.append([InlineKeyboardButton("✔ Finish", callback_data="done")])

        await query.edit_message_text(build_summary(user_id, editing_wave=wave),
                                      reply_markup=InlineKeyboardMarkup(keyboard))

    # 3. Add pilot
    elif data[0] == "add":
        wave, pilot_name = data[1], data[2]
        if pilot_name not in user_selection[user_id]["pilots"][wave]:
            user_selection[user_id]["pilots"][wave].append(pilot_name)

        keyboard = []
        for p in PILOT_LIST:
            if p["name"] in user_selection[user_id]["pilots"][wave]:
                keyboard.append([InlineKeyboardButton(f"❌ {p['name']} ({p['callsign']})",
                                                      callback_data=f"remove|{wave}|{p['name']}")])
            else:
                keyboard.append([InlineKeyboardButton(f"➕ {p['name']} ({p['callsign']})",
                                                      callback_data=f"add|{wave}|{p['name']}")])

        keyboard.append([InlineKeyboardButton("⬅ Back to Waves", callback_data=f"back|{user_selection[user_id]['date']}")])
        keyboard.append([InlineKeyboardButton("✔ Finish", callback_data="done")])

        await query.edit_message_text(build_summary(user_id, editing_wave=wave),
                                      reply_markup=InlineKeyboardMarkup(keyboard))

    # 4. Remove pilot
    elif data[0] == "remove":
        wave, pilot_name = data[1], data[2]
        if pilot_name in user_selection[user_id]["pilots"][wave]:
            user_selection[user_id]["pilots"][wave].remove(pilot_name)

        keyboard = []
        for p in PILOT_LIST:
            if p["name"] in user_selection[user_id]["pilots"][wave]:
                keyboard.append([InlineKeyboardButton(f"❌ {p['name']} ({p['callsign']})",
                                                      callback_data=f"remove|{wave}|{p['name']}")])
            else:
                keyboard.append([InlineKeyboardButton(f"➕ {p['name']} ({p['callsign']})",
                                                      callback_data=f"add|{wave}|{p['name']}")])

        keyboard.append([InlineKeyboardButton("⬅ Back to Waves", callback_data=f"back|{user_selection[user_id]['date']}")])
        keyboard.append([InlineKeyboardButton("✔ Finish", callback_data="done")])

        await query.edit_message_text(build_summary(user_id, editing_wave=wave),
                                      reply_markup=InlineKeyboardMarkup(keyboard))

    # 5. Back to wave menu
    elif data[0] == "back":
        flying_date = data[1]
        keyboard = [
            [InlineKeyboardButton("Wave 1", callback_data=f"wave|{flying_date}|Wave 1")],
            [InlineKeyboardButton("Wave 2", callback_data=f"wave|{flying_date}|Wave 2")],
            [InlineKeyboardButton("Wave 3", callback_data=f"wave|{flying_date}|Wave 3")],
            [InlineKeyboardButton("Wave 4", callback_data=f"wave|{flying_date}|Wave 4")],
            [InlineKeyboardButton("✔ Finish", callback_data="done")]
        ]
        await query.edit_message_text(build_summary(user_id),
                                      reply_markup=InlineKeyboardMarkup(keyboard))

    # 6. Final summary
    elif data[0] == "done":
        summary = build_summary(user_id)
        await query.edit_message_text(f"✅ Final Selection\n\n{summary}")

# --- Run bot ---
def main():
    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(button))
    print("Bot is running...")
    app.run_polling()

if __name__ == "__main__":
    main()
python-telegram-bot==20.6
gspread==5.7.0
oauth2client==4.1.3

# Telegram Pilot Selection Bot ✈️

A Telegram bot to select pilots from squadrons, waves, and dates.

## Setup

1. Clone this repo or import it into Replit.
2. In Replit, go to **Secrets** (🔑) and add:
   - Key: `BOT_TOKEN`
   - Value: (your Telegram BotFather token)
3. Run the bot.

## Usage

- `/start` → Select Squadron → Date → Wave → Pilots
- Multiple pilots can be selected.
- Final summary shows Squadron, Date, Wave, Pilots.
- Edit `data.json` to update squadrons, waves, and pilot callsigns.
