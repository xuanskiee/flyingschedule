import os
import json
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes

# Load pilot library from data.json
with open("data.json", "r") as f:
    DATA = json.load(f)

# Bot token from Replit secret
TOKEN = os.getenv("BOT_TOKEN")

# Store temporary user choices
user_selection = {}

# Generate next 7 days
def get_next_seven_days():
    today = datetime.today()
    return [(today + timedelta(days=i)).strftime("%Y-%m-%d") for i in range(7)]

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [[InlineKeyboardButton(sq, callback_data=f"squadron|{sq}")]
                for sq in DATA.keys()]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text("Select a squadron:", reply_markup=reply_markup)

async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    data = query.data.split("|")
    user_id = query.from_user.id

    # 1. Squadron selection
    if data[0] == "squadron":
        squadron = data[1]
        user_selection[user_id] = {"squadron": squadron, "date": None, "pilots": {}}

        dates = get_next_seven_days()
        keyboard = [[InlineKeyboardButton(d, callback_data=f"date|{squadron}|{d}")] for d in dates]
        await query.edit_message_text(f"Squadron: {squadron}\nSelect flying date:",
                                      reply_markup=InlineKeyboardMarkup(keyboard))

    # 2. Date selection
    elif data[0] == "date":
        squadron, flying_date = data[1], data[2]
        user_selection[user_id]["date"] = flying_date
        user_selection[user_id]["pilots"] = {wave: [] for wave in DATA[squadron].keys()}

        # Show all pilots from all waves in one menu
        keyboard = []
        for wave, pilots in DATA[squadron].items():
            keyboard.append([InlineKeyboardButton(f"--- {wave} ---", callback_data="ignore")])
            for p in pilots:
                keyboard.append([InlineKeyboardButton(f"{p['name']} ({p['callsign']})",
                                                      callback_data=f"pilot|{squadron}|{flying_date}|{wave}|{p['name']}")])
        keyboard.append([InlineKeyboardButton("✔️ Done", callback_data="done")])

        await query.edit_message_text(
            f"Squadron: {squadron}\nDate: {flying_date}\nSelect pilots from any wave:",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

    # 3. Pilot selection
    elif data[0] == "pilot":
        squadron, flying_date, wave, pilot_name = data[1], data[2], data[3], data[4]

        if pilot_name not in user_selection[user_id]["pilots"][wave]:
            user_selection[user_id]["pilots"][wave].append(pilot_name)

        # Refresh pilot list
        keyboard = []
        for w, pilots in DATA[squadron].items():
            keyboard.append([InlineKeyboardButton(f"--- {w} ---", callback_data="ignore")])
            for p in pilots:
                keyboard.append([InlineKeyboardButton(f"{p['name']} ({p['callsign']})",
                                                      callback_data=f"pilot|{squadron}|{flying_date}|{w}|{p['name']}")])
        keyboard.append([InlineKeyboardButton("✔️ Done", callback_data="done")])

        await query.edit_message_text(
            f"✅ Added {pilot_name}\n\nSelect more pilots or press Done:",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

    # 4. Final summary
    elif data[0] == "done":
        squadron = user_selection[user_id]["squadron"]
        flying_date = user_selection[user_id]["date"]
        selected_pilots = user_selection[user_id]["pilots"]

        summary_lines = []
        for wave, names in selected_pilots.items():
            if names:
                pilots_text = []
                for p in DATA[squadron][wave]:
                    if p["name"] in names:
                        pilots_text.append(f"{p['name']} ({p['callsign']})")
                if pilots_text:
                    summary_lines.append(f"- {wave}: " + ", ".join(pilots_text))

        if not summary_lines:
            await query.edit_message_text("⚠️ No pilots selected. Please choose at least one.")
            return

        await query.edit_message_text(
            f"✅ Final Selection\n\n"
            f"**Squadron:** {squadron}\n"
            f"**Date:** {flying_date}\n"
            f"**Pilots:**\n" + "\n".join(summary_lines)
        )

def main():
    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(button))
    print("Bot is running...")
    app.run_polling()

if __name__ == "__main__":
    main()

{
  "140 Squadron": {
    "Wave 1": [
      {"name": "Pilot 140-1", "callsign": "Viper 1"},
      {"name": "Pilot 140-2", "callsign": "Viper 2"}
    ],
    "Wave 2": [
      {"name": "Pilot 140-3", "callsign": "Viper 3"},
      {"name": "Pilot 140-4", "callsign": "Viper 4"}
    ]
  },
  "143 Squadron": {
    "Wave 1": [
      {"name": "Pilot 143-1", "callsign": "Eagle 1"},
      {"name": "Pilot 143-2", "callsign": "Eagle 2"}
    ],
    "Wave 2": [
      {"name": "Pilot 143-3", "callsign": "Eagle 3"},
      {"name": "Pilot 143-4", "callsign": "Eagle 4"}
    ]
  },
  "145 Squadron": {
    "Wave 1": [
      {"name": "Pilot 145-1", "callsign": "Falcon 1"},
      {"name": "Pilot 145-2", "callsign": "Falcon 2"}
    ],
    "Wave 2": [
      {"name": "Pilot 145-3", "callsign": "Falcon 3"},
      {"name": "Pilot 145-4", "callsign": "Falcon 4"}
    ]
  }
}

python-telegram-bot==20.6

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
