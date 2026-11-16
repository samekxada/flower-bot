import asyncio
import logging
import os
import json

from dotenv import load_dotenv
from aiohttp import web
from aiogram import Bot, Dispatcher, types
from aiogram.filters import CommandStart
from aiogram.utils.keyboard import ReplyKeyboardBuilder, InlineKeyboardBuilder

# =========================
# НАСТРОЙКИ
# =========================

load_dotenv()
TOKEN = os.getenv("BOT_TOKEN")
if not TOKEN: raise RuntimeError("❌ BOT_TOKEN не найден!")

try: ADMIN_CHAT_ID = int(os.getenv("ADMIN_CHAT_ID", "0"))
except ValueError: ADMIN_CHAT_ID = 0

SUPPORT_PHONE = "+998 (90) 931-31-07"

# ⚠️ СЮДА ВСТАВЬ СВОЮ НОВУЮ ССЫЛКУ С GITHUB PAGES!
WEBAPP_URL = "https://samekxada.github.io/flower-bot/?v=1" 

logging.basicConfig(level=logging.INFO, format="%(asctime)s | %(levelname)s | %(message)s")
logger = logging.getLogger(__name__)
bot = Bot(token=TOKEN)
dp = Dispatcher()

# =========================
# ТЕКСТЫ
# =========================

TEXTS = {
    "ru": {
        "welcome": "🌸 <b>Добро пожаловать в Fleur!</b>\n\nЦветы, которые скажут больше слов. Нажмите на кнопку <b>«🛍 Сделать заказ»</b> ниже, чтобы открыть каталог.",
        "addr_btn": "📍 Узнать Адрес",
        "lang_btn": "🌐 Язык / Til",
        "address": "📍 Демо-адрес: Ташкент, Мирабадский район.",
        
        "ask_anonymous": "🕵️‍♂️ Это <b>анонимная доставка</b>? (Получатель не узнает, от кого цветы)",
        "anon_yes": "🤫 Да, анонимно",
        "anon_no": "🙋‍♂️ Нет, от меня",
        
        "phone_req": "📱 Отправьте ваш номер телефона для связи (курьер может уточнить детали):",
        "phone_btn": "📞 Отправить номер",
        "phone_wrong": "❌ Пожалуйста, отправьте СВОЙ номер.",
        
        "ask_address": "📍 Напишите адрес доставки и желаемое время (например: <i>'ул. Амира Темура 10, к 18:00'</i>):",
        
        "thanks": "✅ Спасибо! Ваш заказ принят.\n\n📞 Номер для связи: {}\n📞 Служба поддержки: {}",
        "thanks_warn": "✅ Заказ принят.\n\n📞 Номер: {}\n⚠️ Если не свяжутся за 10 минут — позвоните: {}"
    },
    "uz": {
        "welcome": "🌸 <b>Fleur do'koniga xush kelibsiz!</b>\n\nKatalog ochish uchun pastdagi <b>«🛍 Buyurtma berish»</b> tugmasini bosing.",
        "addr_btn": "📍 Manzilni bilish",
        "lang_btn": "🌐 Язык / Til",
        "address": "📍 Demo-manzil: Toshkent, Mirobod tumani.",
        
        "ask_anonymous": "🕵️‍♂️ Bu <b>anonim yetkazib berishmi</b>? (Qabul qiluvchi kimdan ekanligini bilmaydi)",
        "anon_yes": "🤫 Ha, anonim",
        "anon_no": "🙋‍♂️ Yo'q, mendan",
        
        "phone_req": "📱 Kuryer bog'lanishi uchun telefon raqamingizni yuboring:",
        "phone_btn": "📞 Raqamni yuborish",
        "phone_wrong": "❌ Iltimos, O'Z raqamingizni yuboring.",
        
        "ask_address": "📍 Yetkazib berish manzilini va vaqtini yozing (masalan: <i>'Amir Temur ko'chasi 10, 18:00 gacha'</i>):",
        
        "thanks": "✅ Rahmat! Buyurtmangiz qabul qilindi.\n\n📞 Aloqa: {}\n📞 Yordam: {}",
        "thanks_warn": "✅ Buyurtma qabul qilindi.\n\n📞 Aloqa: {}\n⚠️ 10 daqiqada bog'lanmasa — qo'ng'iroq qiling: {}"
    }
}

user_languages = {}
webapp_carts = {} 
pending_orders = {} 

# =========================
# ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ
# =========================

def get_lang(user_id): return user_languages.get(user_id, "ru")

def build_phone_keyboard(lang):
    builder = ReplyKeyboardBuilder()
    builder.add(types.KeyboardButton(text=TEXTS[lang]["phone_btn"], request_contact=True))
    return builder.as_markup(resize_keyboard=True)

def get_main_keyboard(lang):
    builder = ReplyKeyboardBuilder()
    builder.add(types.KeyboardButton(text="🛍 Сделать заказ", web_app=types.WebAppInfo(url=WEBAPP_URL)))
    builder.add(types.KeyboardButton(text=TEXTS[lang]["addr_btn"]), types.KeyboardButton(text=TEXTS[lang]["lang_btn"]))
    builder.adjust(1)
    return builder.as_markup(resize_keyboard=True)

# =========================
# ОТПРАВКА ЗАКАЗА АДМИНУ
# =========================

async def finalize_order(user_id, lang, address=None):
    data = pending_orders.pop(user_id, None)
    if not data: return

    phone = data["phone"]
    is_anonymous = data.get("anonymous", False)
    currency = "сум" if lang == "ru" else "so'm"
    
    order_text = ""
    total = 0
    for item in data["cart"]:
        subtotal = item['price'] * item['qty']
        total += subtotal
        order_text += f"• {item['name']} × {item['qty']} = {subtotal:,} {currency}\n"
    
    anon_text = "🤫 <b>АНОНИМНАЯ ДОСТАВКА</b>\n" if is_anonymous else ""
    address_text = f"\n📍 <b>Адрес:</b> {address}\n" if address else ""

    admin_message = (
        f"🔔 <b>НОВЫЙ ЗАКАЗ ЦВЕТОВ!</b>\n---------------------------\n"
        f"👤 Клиент: {data['first_name']}\n🔗 Telegram: {data['username']}\n📞 Телефон: {phone}\n"
        f"{anon_text}"
        f"---------------------------\n🌸 ЗАКАЗ:\n{order_text}\n💰 ИТОГО: {total:,} {currency}\n{address_text}"
        f"---------------------------\n"
    )

    notified = False
    if ADMIN_CHAT_ID:
        admin_kb = InlineKeyboardBuilder()
        admin_kb.button(text="✅ Завершить заказ", callback_data=f"complete_{user_id}")
        
        try:
            await bot.send_message(ADMIN_CHAT_ID, admin_message, reply_markup=admin_kb.as_markup(), parse_mode="HTML")
            notified = True
        except Exception as e: logger.error(f"Ошибка админу: {e}")
            
    thanks_key = "thanks" if notified else "thanks_warn"
    await bot.send_message(user_id, TEXTS[lang][thanks_key].format(phone, SUPPORT_PHONE), reply_markup=get_main_keyboard(lang))

# =========================
# ЗАВЕРШЕНИЕ ЗАКАЗА
# =========================

@dp.callback_query(lambda c: c.data.startswith("complete_"))
async def complete_order(callback: types.CallbackQuery):
    client_id = int(callback.data.split("_")[1])
    try:
        await bot.send_message(client_id, "🛵 <b>Отличные новости!</b>\n\nВаш заказ передан курьеру. Ожидайте доставку в ближайшее время.", parse_mode="HTML")
        await callback.message.edit_text(text=callback.message.text + "\n\n✅ <b>СТАТУС: ДОСТАВЛЯЕТСЯ</b>", parse_mode="HTML")
        await callback.answer("Клиент уведомлен! 🛵", show_alert=False)
    except Exception as e:
        await callback.answer(f"Ошибка: {e}", show_alert=True)

# =========================
# ХЕНДЛЕРЫ: СТАРТ И ЯЗЫК
# =========================

@dp.message(CommandStart())
async def cmd_start(message: types.Message):
    builder = ReplyKeyboardBuilder()
    builder.add(types.KeyboardButton(text="🇷🇺 Русский"), types.KeyboardButton(text="🇺🇿 O'zbekcha"))
    builder.adjust(2)
    await message.answer("🇷🇺 Выберите язык / 🇺🇿 Tilni tanlang:", reply_markup=builder.as_markup(resize_keyboard=True))

@dp.message(lambda msg: msg.text in ["🇷🇺 Русский", "🇺🇿 O'zbekcha"])
async def select_language(message: types.Message):
    lang = "ru" if message.text == "🇷🇺 Русский" else "uz"
    user_languages[message.from_user.id] = lang
    await message.answer(TEXTS[lang]["welcome"], reply_markup=get_main_keyboard(lang), parse_mode="HTML")

@dp.message(lambda msg: msg.text in ["🌐 Язык / Til", "🌐 Сменить язык / Tilni o'zgartirish"])
async def change_lang(message: types.Message):
    await cmd_start(message)

# =========================
# ПРИЕМ ДАННЫХ ИЗ MINI APP
# =========================

@dp.message(lambda msg: msg.web_app_data)
async def handle_web_app_data(message: types.Message):
    user_id = message.from_user.id
    lang = get_lang(user_id)
    logger.info(f"Получены данные из Web App: {message.web_app_data.data}")
    
    try:
        data = json.loads(message.web_app_data.data)
        items = data.get("items", [])
        
        if not items:
            await message.answer("Корзина пуста.")
            return
            
        webapp_carts[user_id] = items
        total = sum(item['price'] * item['qty'] for item in items)
        items_text = "\n".join([f"• {item['name']} × {item['qty']}" for item in items])
        
        builder = InlineKeyboardBuilder()
        builder.button(text=TEXTS[lang]["anon_yes"], callback_data="anon_yes")
        builder.button(text=TEXTS[lang]["anon_no"], callback_data="anon_no")
        builder.adjust(2)
        
        await message.answer(
            f"🌸 <b>Ваш заказ:</b>\n{items_text}\n\n💰 <b>Итого:</b> {total:,} сум\n\n"
            f"{TEXTS[lang]['ask_anonymous']}",
            reply_markup=builder.as_markup(),
            parse_mode="HTML"
        )
        
    except Exception as e:
        logger.error(f"Ошибка WebApp: {e}")

@dp.callback_query(lambda c: c.data in ["anon_yes", "anon_no"])
async def handle_anonymous_choice(callback: types.CallbackQuery):
    user_id = callback.from_user.id
    lang = get_lang(user_id)
    
    is_anon = callback.data == "anon_yes"
    
    if user_id in webapp_carts:
        pending_orders[user_id] = {
            "cart": webapp_carts.pop(user_id),
            "anonymous": is_anon,
            "username": f"@{callback.from_user.username}" if callback.from_user.username else "Нет",
            "first_name": callback.from_user.first_name
        }
    
    await callback.answer()
    await callback.message.delete()
    
    await callback.message.answer(TEXTS[lang]["phone_req"], reply_markup=build_phone_keyboard(lang))

# =========================
# ЗАПРОС КОНТАКТА И АДРЕСА
# =========================

@dp.message(lambda msg: msg.contact is not None and msg.from_user.id in pending_orders)
async def handle_contact(message: types.Message):
    user_id = message.from_user.id
    lang = get_lang(user_id)
    
    if message.contact.user_id != user_id:
        await message.answer(TEXTS[lang]["phone_wrong"], reply_markup=build_phone_keyboard(lang))
        return

    pending_orders[user_id]["phone"] = message.contact.phone_number
    
    await message.answer(TEXTS[lang]["ask_address"], reply_markup=types.ReplyKeyboardRemove(), parse_mode="HTML")

@dp.message(lambda msg: msg.from_user.id in pending_orders and "phone" in pending_orders[msg.from_user.id] and msg.text is not None)
async def handle_address(message: types.Message):
    user_id = message.from_user.id
    lang = get_lang(user_id)
    await finalize_order(user_id, lang, address=message.text.strip())

@dp.message(lambda msg: msg.text in ["📍 Узнать Адрес", "📍 Manzilni bilish"])
async def show_address(message: types.Message):
    await message.answer(TEXTS[get_lang(message.from_user.id)]["address"], reply_markup=get_main_keyboard(get_lang(message.from_user.id)))

@dp.message()
async def fallback(message: types.Message):
    await message.answer("Нажмите на кнопку «🛍 Сделать заказ» внизу, чтобы открыть каталог цветов.", reply_markup=get_main_keyboard(get_lang(message.from_user.id)))

@dp.error()
async def error_handler(event: types.ErrorEvent):
    logger.error(f"Ошибка: {event.exception}", exc_info=event.exception)

# =========================
# WEBHOOK
# =========================

async def handle_webhook(request: web.Request):
    try:
        update_data = await request.json()
        await dp.feed_update(bot, types.Update(**update_data))
    except Exception as e: logger.error(f"Ошибка вебхука: {e}")
    return web.Response(status=200)

async def healthcheck(request: web.Request): return web.Response(text="Bot is running!")

async def on_startup(app: web.Application):
    webhook_host = os.getenv("RENDER_EXTERNAL_URL") or os.getenv("WEBHOOK_HOST")
    if not webhook_host: return
    webhook_url = f"{webhook_host}/webhook/{TOKEN}"
    try:
        await bot.delete_webhook(drop_pending_updates=True)
        await bot.set_webhook(webhook_url)
    except Exception as e: logger.error(f"Ошибка старта: {e}")

async def on_shutdown(app: web.Application): await bot.session.close()

if __name__ == "__main__":
    port = int(os.getenv("PORT", 8080))
    app = web.Application()
    app.router.add_post(f"/webhook/{TOKEN}", handle_webhook)
    app.router.add_get("/healthz", healthcheck)
    app.router.add_get("/", healthcheck)
    app.on_startup.append(on_startup)
    app.on_shutdown.append(on_shutdown)
    web.run_app(app, host="0.0.0.0", port=port)
