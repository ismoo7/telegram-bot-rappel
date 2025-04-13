from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    ConversationHandler,
    ContextTypes,
    filters
)
from datetime import datetime, timedelta
import asyncio

# États de la conversation
CHOISIR_HEURE, CHOISIR_MINUTE, SAISIR_NOTE = range(3)

# Base de données en mémoire
rappels_utilisateur = {}

# Limite pour version gratuite
LIMITE_FREE = 5

# Commande /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    utilisateur_id = update.effective_user.id
    aujourd_hui = datetime.now().date()
    rappels_du_jour = [
        r for r in rappels_utilisateur.get(utilisateur_id, [])
        if r["date"].date() == aujourd_hui
    ]

    if len(rappels_du_jour) >= LIMITE_FREE:
        await update.message.reply_text("🚫 Tu as atteint la limite de 5 rappels aujourd’hui (version gratuite).")
        return ConversationHandler.END

    clavier = []
    for i in range(0, 24, 4):
        ligne = [InlineKeyboardButton(f"{h:02d}h", callback_data=f"{h:02d}") for h in range(i, i + 4)]
        clavier.append(ligne)

    await update.message.reply_text("🕐 Choisis une *heure* :", reply_markup=InlineKeyboardMarkup(clavier), parse_mode="Markdown")
    return CHOISIR_HEURE

# Choix heure
async def choisir_heure(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    context.user_data['heure'] = query.data

    clavier = []
    for i in range(0, 60, 15):
        ligne = [InlineKeyboardButton(f"{i+offset:02d} min", callback_data=f"{query.data}:{i+offset:02d}") for offset in range(0, 15, 5)]
        clavier.append(ligne)

    await query.edit_message_text("🕓 Choisis les *minutes* :", reply_markup=InlineKeyboardMarkup(clavier), parse_mode="Markdown")
    return CHOISIR_MINUTE

# Choix minutes
async def choisir_minute(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    context.user_data['heure_complete'] = query.data
    await query.edit_message_text("📝 Envoie le *texte du rappel* :", parse_mode="Markdown")
    return SAISIR_NOTE

# Réception de la note
async def recevoir_note(update: Update, context: ContextTypes.DEFAULT_TYPE):
    utilisateur_id = update.effective_user.id
    note = update.message.text
    heure = context.user_data['heure_complete']
    date_aujourdhui = datetime.now().strftime("%d-%m-%Y")
    date_str = f"{date_aujourdhui} {heure}"
    dt_rappel = datetime.strptime(date_str, "%d-%m-%Y %H:%M")
    dt_alerte = dt_rappel - timedelta(minutes=10)

    if utilisateur_id not in rappels_utilisateur:
        rappels_utilisateur[utilisateur_id] = []

    rappels_utilisateur[utilisateur_id].append({
        "note": note,
        "date": dt_rappel
    })

    await update.message.reply_text(
        f"✅ Rappel enregistré pour *{dt_rappel.strftime('%d/%m/%Y à %H:%M')}*
"
        f"📝 Note : _{note}_",
        parse_mode="Markdown"
    )

    await programmer_rappel(context, update.effective_chat.id, note, dt_alerte)
    return ConversationHandler.END

# Programmer rappel
async def programmer_rappel(context, chat_id, note, moment_alerte):
    maintenant = datetime.now()
    delai = (moment_alerte - maintenant).total_seconds()
    if delai > 0:
        await asyncio.sleep(delai)
    await context.bot.send_message(
        chat_id=chat_id,
        text=f"🔔 *Rappel dans 10 minutes !*
📝 {note}

🛒 Pour acheter ta licence @ismoo7",
        parse_mode="Markdown"
    )

# Annuler
async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    utilisateur_id = update.effective_user.id
    if utilisateur_id in rappels_utilisateur and rappels_utilisateur[utilisateur_id]:
        supprimé = rappels_utilisateur[utilisateur_id].pop()
        await update.message.reply_text(f"❌ Rappel supprimé : {supprimé['note']}")
    else:
        await update.message.reply_text("❌ Aucun rappel à annuler.")

# Lancement bot
app = ApplicationBuilder().token("8141393264:AAGyfDp6rWfiBtWSY5A1suh3PmjOLZko5mo").build()

conv_handler = ConversationHandler(
    entry_points=[CommandHandler("start", start)],
    states={
        CHOISIR_HEURE: [CallbackQueryHandler(choisir_heure)],
        CHOISIR_MINUTE: [CallbackQueryHandler(choisir_minute)],
        SAISIR_NOTE: [MessageHandler(filters.TEXT & ~filters.COMMAND, recevoir_note)],
    },
    fallbacks=[CommandHandler("cancel", cancel)],
)

app.add_handler(conv_handler)
app.add_handler(CommandHandler("cancel", cancel))
app.run_polling()
