import os
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes
import gspread
from oauth2client.service_account import ServiceAccountCredentials
import math

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
PILOTS_PER_PAGE = 10

# --- Store user selections ---
user_selection = {}

def get_next_seven_days():
    today = datetime.today()
    return [(today + timedelta(days=i)).strftime("%Y-%m-%d") for i in range(7)]

# --- Helper: build summary text ---
def build_summary(user_id, editing_wave=None):
    selection = user_selection[user_id]
    text = f"🛩 Squadron: {selection['squadron']}\n📅 Date: {selection['date']}\n\n"

    for wave in ["Wave 1", "Wave 2", "Wave 3", "Wave 4"]:
        names = selection["pilots"].get(wave, [])
        if names:
            text += f"{wave}:\n"
            for name in names:
                callsign = next((p["callsign"] for p in PILOT_LIST if p["name"] == name), "")
                text += f"- {name} ({callsign})\n"
        else:
            text += f"{wave}: —\n"
        text += "\n"

    if editing_wave:
        text += f"Now editing: {editing_wave}"
    return text

# --- Start ---
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("140 Squadron", callback_data="squadron|140")],
        [InlineKeyboardButton("143 Squadron", callback_data="squadron|143")],
        [InlineKeyboardButton("145 Squadron", callback_data="squadron|145")]
    ]
    await update.message.reply_text("Select a squadron:",
                                    reply_markup=InlineKeyboardMarkup(keyboard))

# --- Build paginated pilot keyboard ---
def build_pilot_keyboard(user_id, wave, page=1):
    selection = user_selection[user_id]["pilots"][wave]
    start_idx = (page - 1) * PILOTS_PER_PAGE
    end_idx = start_idx + PILOTS_PER_PAGE
    pilots_page = PILOT_LIST[start_idx:end_idx]

    keyboard = []
    for p in pilots_page:
        if p["name"] in selection:
            keyboard.append([InlineKeyboardButton(f"❌ {p['name']} ({p['callsign']})",
                                                  callback_data=f"remove|{wave}|{p['name']}|{page}")])
        else:
            keyboard.append([InlineKeyboardButton(f"➕ {p['name']} ({p['callsign']})",
                                                  callback_data=f"add|{wave}|{p['name']}|{page}")])

    # Pagination buttons
    total_pages = math.ceil(len(PILOT_LIST)/PILOTS_PER_PAGE)
    nav_buttons = []
    if page > 1:
        nav_buttons.append(InlineKeyboardButton("⬅ Previous", callback_data=f"page|{wave}|{page-1}"))
    if page < total_pages:
        nav_buttons.append(InlineKeyboardButton("Next ➡", callback_data=f"page|{wave}|{page+1}"))
    if nav_buttons:
        keyboard.append(nav_buttons)

    # Back & Finish
    keyboard.append([InlineKeyboardButton("⬅ Back to Waves", callback_data=f"back|{user_selection[user_id]['date']}")])
    keyboard.append([InlineKeyboardButton("✔ Finish", callback_data="done")])

    return InlineKeyboardMarkup(keyboard)

# --- Handle button clicks ---
async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    data = query.data.split("|")
    user_id = query.from_user.id

    # Squadron selected
    if data[0] == "squadron":
        squadron = data[1]
        user_selection[user_id] = {"squadron": squadron, "date": None,
                                   "pilots": {"Wave 1": [], "Wave 2": [], "Wave 3": [], "Wave 4": []}}

        dates = get_next_seven_days()
        keyboard = [[InlineKeyboardButton(d, callback_data=f"date|{d}")] for d in dates]
        await query.edit_message_text(f"🛩 Squadron: {squadron}\nSelect flying date:",
                                      reply_markup=InlineKeyboardMarkup(keyboard))

    # Date selected
    elif data[0] == "date":
        flying_date = data[1]
        user_selection[user_id]["date"] = flying_date

        keyboard = [
            [InlineKeyboardButton("Wave 1", callback_data=f"wave|{flying_date}|Wave 1")],
            [InlineKeyboardButton("Wave 2", callback_data=f"wave|{flying_date}|Wave 2")],
            [InlineKeyboardButton("Wave 3", callback_data=f"wave|{flying_date}|Wave 3")],
            [InlineKeyboardButton("Wave 4", callback_data=f"wave|{flying_date}|Wave 4")],
            [InlineKeyboardButton("✔ Finish", callback_data="done")]
        ]
        await query.edit_message_text(f"🛩 Squadron: {user_selection[user_id]['squadron']}\n📅 Date: {flying_date}\nSelect a wave:",
                                      reply_markup=InlineKeyboardMarkup(keyboard))

    # Wave selected
    elif data[0] == "wave":
        flying_date, wave = data[1], data[2]
        markup = build_pilot_keyboard(user_id, wave)
        await query.edit_message_text(build_summary(user_id, editing_wave=wave),
                                      reply_markup=markup)

    # Add pilot
    elif data[0] == "add":
        wave, pilot_name, page = data[1], data[2], int(data[3])
        if pilot_name not in user_selection[user_id]["pilots"][wave]:
            user_selection[user_id]["pilots"][wave].append(pilot_name)
        markup = build_pilot_keyboard(user_id, wave, page)
        await query.edit_message_text(build_summary(user_id, editing_wave=wave),
                                      reply_markup=markup)

    # Remove pilot
    elif data[0] == "remove":
        wave, pilot_name, page = data[1], data[2], int(data[3])
        if pilot_name in user_selection[user_id]["pilots"][wave]:
            user_selection[user_id]["pilots"][wave].remove(pilot_name)
        markup = build_pilot_keyboard(user_id, wave, page)
        await query.edit_message_text(build_summary(user_id, editing_wave=wave),
                                      reply_markup=markup)

    # Pagination
    elif data[0] == "page":
        wave, page = data[1], int(data[2])
        markup = build_pilot_keyboard(user_id, wave, page)
        await query.edit_message_text(build_summary(user_id, editing_wave=wave),
                                      reply_markup=markup)

    # Back to wave menu
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

    # Final summary
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

worker: python main.py
