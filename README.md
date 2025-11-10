# file: telegram_crypto_shop.py
# Reikalavimai: python-telegram-bot==13.15, requests
import logging
import requests
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import Updater, CommandHandler, CallbackQueryHandler, CallbackContext

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# === CONFIG ===
TOKEN = "8451050098:AAH0P3E7MlsM4iCUnrvwTnCx_0oonpou5qA
"            # <-- įdėk savo Telegram bot token
NOW_API_KEY = "74f32715-08cd-4bb6-9e99-6fc18b4a4eab"  # <-- įdėk NOWPayments API key (ar kitos paslaugos)
NOW_API_BASE = "https://api.nowpayments.io/v1"

# Produktai (galima plėsti)
PRODUCTS = {
    "kedainiai": [
        {"id": "zaza_2g", "title": "zaza 2g", "price_eur": 35},
        {"id": "zaza_5g", "title": "zaza 5g", "price_eur": 65},
    ],
    "kaunas": [
        {"id": "zaza_2g", "title": "zaza 2g", "price_eur": 35},
        {"id": "zaza_5g", "title": "zaza 5g", "price_eur": 65},
    ]
}
# Test photo (pakeisk į savo nuotrauką arba URL)
PRODUCT_PHOTO_URL = "https://via.placeholder.com/800x600.png?text=Arbata+zaza"

# === HELPERS ===
def build_keyboard(button_rows):
    return InlineKeyboardMarkup([[InlineKeyboardButton(text=b[0], callback_data=b[1]) for b in row] for row in button_rows])

def create_now_invoice(amount_eur: float, pay_currency: str, order_id: str, description: str, callback_url: str=None):
    """
    Sukuria invoice per NOWPayments. Grąžina dict su 'invoice_url' ir 'id' arba raises.
    pay_currency: "SOL" or "LTC" etc.
    """
    url = f"{NOW_API_BASE}/invoice"
    headers = {"x-api-key": NOW_API_KEY, "Content-Type": "application/json"}
    data = {
        "price_amount": amount_eur,
        "price_currency": "eur",
        "pay_currency": pay_currency,
        "order_id": order_id,
        "order_description": description
    }
    if callback_url:
        data["ipn_callback_url"] = callback_url
    resp = requests.post(url, json=data, headers=headers, timeout=15)
    resp.raise_for_status()
    return resp.json()

def get_now_invoice_status(invoice_id: str):
    url = f"{NOW_API_BASE}/invoice/{invoice_id}"
    headers = {"x-api-key": NOW_API_KEY}
    resp = requests.get(url, headers=headers, timeout=15)
    resp.raise_for_status()
    return resp.json()

# === HANDLERS ===
def start(update: Update, context: CallbackContext):
    text = "Sveikas! Nori įsigyti arbatos? Paspausk „Pirkti“."
    keyboard = build_keyboard([[("Pirkti", "buy")]])
    update.message.reply_text(text, reply_markup=keyboard)

def buy_callback(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    # Paklausti miesto
    kb = [
        [("Kėdainiai", "city_kedainiai"), ("Kaunas", "city_kaunas")],
        [("Atšaukti", "cancel")]
    ]
    query.edit_message_text("Pasirink miestą:", reply_markup=build_keyboard(kb))

def city_selected(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    data = query.data
    if data == "city_kedainiai":
        city = "kedainiai"
    else:
        city = "kaunas"
    context.user_data['city'] = city

    # Parodyti produktus pagal miestą
    items = PRODUCTS.get(city, [])
    kb = []
    for p in items:
        kb.append([(f"{p['title']} — {p['price_eur']}€", f"product_{p['id']}")])
    kb.append([("Atgal", "buy"), ("Atšaukti", "cancel")])
    query.edit_message_text(f"Mieste: {city.capitalize()}. Pasirink produktą:", reply_markup=build_keyboard(kb))

def product_selected(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    data = query.data  # e.g. product_zaza_2g
    product_id = data.replace("product_", "")
    # surasti produktą
    city = context.user_data.get('city')
    product = None
    for p in PRODUCTS.get(city, []):
        if p['id'] == product_id:
            product = p
            break
    if not product:
        query.edit_message_text("Produktas nerastas. Bandyk dar kartą.")
        return
    context.user_data['product'] = product
    # Siųsti pasirinkimo su mygtukais: mokėti / info
    kb = [
        [("Mokėti / Piniginė", "wallet"), ("Atgal į miestą", "buy")],
        [("Atšaukti", "cancel")]
    ]
    query.edit_message_text(f"Pasirinkai: {product['title']} — {product['price_eur']}€\n\nAr nori tęsti?", reply_markup=build_keyboard(kb))

def wallet_menu(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    # Papildyti ar atgal
    kb = [
        [("Papildyti", "topup")],
        [("Atgal", "product_back"), ("Atšaukti", "cancel")]
    ]
    query.edit_message_text("Piniginė: pasirink veiksmą", reply_markup=build_keyboard(kb))

def topup_menu(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    # Galimybė pasirinkti sumą: pasiūlyti produktų kainas arba custom
    product = context.user_data.get('product')
    if not product:
        query.edit_message_text("Produktas nepasirinktas. Grįžk ir pasirink produktą.")
        return
    price = product['price_eur']
    kb = [
        [(f"Apmokėti {price}€ (rekomenduojama)", f"pay_amount_{price}")],
        [("Įvesti kitą sumą", "pay_custom")],
        [("Atgal", "wallet"), ("Atšaukti", "cancel")]
    ]
    query.edit_message_text("Pasirink sumą, kurią norėsi apmokėti:", reply_markup=build_keyboard(kb))

def pay_amount_selected(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    data = query.data  # pay_amount_35
    amount = float(data.replace("pay_amount_", ""))
    context.user_data['amount_eur'] = amount
    # Pasirink kriptovaliutą
    kb = [
        [("SOL", "paycoin_SOL"), ("LTC", "paycoin_LTC")],
        [("Atgal", "topup"), ("Atšaukti", "cancel")]
    ]
    query.edit_message_text(f"Sumą: {amount}€\nPasirink kriptovaliutą:", reply_markup=build_keyboard(kb))

def pay_custom(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    # Paprašyti įvesti sumą: dėl paprastumo naudojam dialogą su tekstu
    query.edit_message_text("Įvesk sumą eurais (pvz. 20).")
    context.user_data['expecting_custom_amount'] = True

def text_message(update: Update, context: CallbackContext):
    # Čia gaunamas vartotojo įvestas custom amount
    if context.user_data.get('expecting_custom_amount'):
        txt = update.message.text.replace(",", ".").strip()
        try:
            amount = float(txt)
        except:
            update.message.reply_text("Neteisingas formatas. Parašyk skaičių, pvz. 20")
            return
        context.user_data['expecting_custom_amount'] = False
        context.user_data['amount_eur'] = amount
        kb = [
            [("SOL", "paycoin_SOL"), ("LTC", "paycoin_LTC")],
            [("Atšaukti", "cancel")]
        ]
        update.message.reply_text(f"Sumą nustatėme: {amount}€. Pasirink mokėjimo valiutą:", reply_markup=build_keyboard(kb))
        return
    update.message.reply_text("Nenustatytas veiksmas. Naudok mygtukus.")

def choose_coin(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    coin = query.data.replace("paycoin_", "")
    amount = context.user_data.get('amount_eur')
    product = context.user_data.get('product')
    if not amount or not product:
        query.edit_message_text("Trūksta informacijos. Grįžk ir pasirink produktą bei sumą.")
        return

    # Sukurti mokėjimo invoice (NOWPayments)
    order_id = f"{update.effective_user.id}_{product['id']}_{int(amount*100)}"
    description = f"Pirkinys: {product['title']} ({product['id']})"

    try:
        inv = create_now_invoice(amount_eur=amount, pay_currency=coin, order_id=order_id, description=description)
        invoice_url = inv.get('invoice_url')
        invoice_id = inv.get('id')
    except Exception as e:
        logger.exception("Invoice creation failed")
        query.edit_message_text(f"Klaida sukuriant mokėjimą: {e}")
        return

    # Išsaugome invoice id, kad vėliau galėtume patikrinti
    context.user_data['invoice_id'] = invoice_id
    context.user_data['invoice_url'] = invoice_url

    kb = [
        [("Atidaryti mokėjimo nuorodą", "open_invoice")],
        [("Patikrinti apmokėjimą", "check_payment")],
        [("Atšaukti", "cancel")]
    ]
    # Siųsti nuorodą vartotojui
    query.edit_message_text(f"Generated invoice for {amount}€ in {coin}.\n\nNuoroda: {invoice_url}\n\nKai apmokėsi spausk „Patikrinti apmokėjimą“.", reply_markup=build_keyboard(kb))

def open_invoice(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    url = context.user_data.get('invoice_url')
    if not url:
        query.edit_message_text("Nuoroda nerasta.")
        return
    # edit_message_text doesn't open link; send as message
    query.message.reply_text(f"Atsidaryk nuorodą mokėjimui: {url}")

def check_payment(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    inv_id = context.user_data.get('invoice_id')
    if not inv_id:
        query.edit_message_text("Invoice ID nerastas. Pabandyk dar kartą sukurti mokėjimą.")
        return
    try:
        status = get_now_invoice_status(inv_id)
    except Exception as e:
        logger.exception("Failed to check invoice")
        query.edit_message_text(f"Klaida tikrinant mokėjimą: {e}")
        return
    current_status = status.get('status')
    # NOWPayments statuses: waiting, confirmed, partially_paid, finished, expired, cancelled
    if current_status in ("finished", "confirmed"):
        # Siunčiame nuotrauką / patvirtinimą
        chat_id = update.effective_user.id
        product = context.user_data.get('product')
        caption = f"Apmokėta! Siunčiu tavo prekę: {product['title']}"
        context.bot.send_photo(chat_id=chat_id, photo=PRODUCT_PHOTO_URL, caption=caption)
        query.edit_message_text("Mokėjimas patvirtintas. Nuotrauka išsiųsta.")
    else:
        query.edit_message_text(f"Mokėjimo būsena: {current_status}. Jei jau apmokėjai, palauk arba pabandyk vėliau.")

def cancel(update: Update, context: CallbackContext):
    # bendras atšaukimo handleris
    if update.callback_query:
        update.callback_query.answer()
        update.callback_query.edit_message_text("Operacija atšaukta.")
    else:
        update.message.reply_text("Operacija atšaukta.")

def product_back(update: Update, context: CallbackContext):
    # grąžinti į produktų listą
    city = context.user_data.get('city')
    if not city:
        update.callback_query.edit_message_text("Miesto nėra. Grįžk prie pradžios.")
        return
    items = PRODUCTS.get(city, [])
    kb = []
    for p in items:
        kb.append([(f"{p['title']} — {p['price_eur']}€", f"product_{p['id']}")])
    kb.append([("Atgal", "buy"), ("Atšaukti", "cancel")])
    update.callback_query.edit_message_text(f"Mieste: {city.capitalize()}. Pasirink produktą:", reply_markup=build_keyboard(kb))

def unknown(update: Update, context: CallbackContext):
    update.message.reply_text("Naudok mygtukus, prašau.")

# === Main ===
def main():
    updater = Updater(TOKEN, use_context=True)
    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CallbackQueryHandler(buy_callback, pattern="^buy$"))
    dp.add_handler(CallbackQueryHandler(city_selected, pattern="^city_"))
    dp.add_handler(CallbackQueryHandler(product_selected, pattern="^product_"))
    dp.add_handler(CallbackQueryHandler(wallet_menu, pattern="^wallet$"))
    dp.add_handler(CallbackQueryHandler(topup_menu, pattern="^topup$"))
    dp.add_handler(CallbackQueryHandler(pay_amount_selected, pattern="^pay_amount_"))
    dp.add_handler(CallbackQueryHandler(pay_custom, pattern="^pay_custom$"))
    dp.add_handler(CallbackQueryHandler(choose_coin, pattern="^paycoin_"))
    dp.add_handler(CallbackQueryHandler(open_invoice, pattern="^open_invoice$"))
    dp.add_handler(CallbackQueryHandler(check_payment, pattern="^check_payment$"))
    dp.add_handler(CallbackQueryHandler(cancel, pattern="^cancel$"))
    dp.add_handler(CallbackQueryHandler(product_back, pattern="^product_back$"))

    dp.add_handler(MessageHandler((~Filters.command) & Filters.text, text_message))
    dp.add_handler(MessageHandler(Filters.command, unknown))

    updater.start_polling()
    logger.info("Botas paleistas. Ctrl-C kad sustabdyti.")
    updater.idle()

if __name__ == "__main__":
    main()
