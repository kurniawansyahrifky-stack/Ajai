import asyncio
import json
import time
import os
import zipfile
import random
from telethon import TelegramClient, events, Button
from telethon.errors import FloodWaitError
from telethon.tl.functions.channels import GetParticipantsRequest
from telethon.tl.types import ChannelParticipantsRecent

# ===== CONFIG =====
API_ID = 33370509
API_HASH = "669af6caebf2aca264b16cf8b40d37b2"
BOT_TOKEN = "8307438695:AAEU8MPX2BSj7EH5VOjHVhtfqloS7olHObg"
OWNER_ID = 8209644174

LOG_GROUP = -1003812690414

# SPEED CONFIG
BATCH_SIZE = 3
WORKERS = 3
DELAY_MIN = 0.7
DELAY_MAX = 1.2
MAX_MSG_LEN = 3500

# ⚠️ FIX: JANGAN .start DI SINI
client = TelegramClient("bot", API_ID, API_HASH)

# ===== DATABASE =====
DB_FILE = "data.json"

if not os.path.exists(DB_FILE):
    with open(DB_FILE, "w") as f:
        json.dump({"partners": {}, "groups": [], "status": True}, f)

def load_db():
    with open(DB_FILE) as f:
        return json.load(f)

def save_db(data):
    with open(DB_FILE, "w") as f:
        json.dump(data, f, indent=2)

# ===== SAFE SEND =====
async def safe_send(chat_id, text, buttons=None):
    while True:
        try:
            return await client.send_message(chat_id, text, buttons=buttons, parse_mode="md")
        except FloodWaitError as e:
            await asyncio.sleep(min(e.seconds, 10))
        except:
            await asyncio.sleep(1)

# ===== START =====
@client.on(events.NewMessage(pattern="/start"))
async def start(event):
    await event.reply(
        "🚀 BOT TAGALL AKTIF\n\n"
        "Kirim link partner untuk mulai tagall",
        buttons=[[Button.inline("📊 STATUS", b"status")]]
    )

# ===== HELP OWNER =====
@client.on(events.NewMessage(pattern="/help"))
async def help_cmd(event):
    if event.sender_id != OWNER_ID:
        return

    await event.reply(
        "⚙️ OWNER MENU\n\n"
        "/addpartner link\n"
        "/addbot idgroup\n"
        "/on /off\n"
        "/backup\n"
        "/restore\n"
        "/ping"
    )

# ===== PING =====
@client.on(events.NewMessage(pattern="/ping"))
async def ping(event):
    if event.sender_id == OWNER_ID:
        await event.reply("🏓 BOT HIDUP")

# ===== ON OFF =====
@client.on(events.NewMessage(pattern="/on"))
async def on_bot(event):
    if event.sender_id != OWNER_ID:
        return
    db = load_db()
    db["status"] = True
    save_db(db)
    await event.reply("✅ BOT ON")

@client.on(events.NewMessage(pattern="/off"))
async def off_bot(event):
    if event.sender_id != OWNER_ID:
        return
    db = load_db()
    db["status"] = False
    save_db(db)
    await event.reply("⛔ BOT OFF")

# ===== GET USERS =====
async def get_users(chat_id):
    users = []
    offset = 0

    while True:
        participants = await client(GetParticipantsRequest(
            chat_id,
            ChannelParticipantsRecent(),
            offset,
            100,
            hash=0
        ))

        if not participants.users:
            break

        for u in participants.users:
            if not u.bot and u.first_name:
                users.append(f"[{u.first_name}](tg://user?id={u.id})")

        offset += len(participants.users)

    return users

# ===== SPLIT =====
def split_message(base, users):
    msgs = []
    current = base

    for u in users:
        if len(current) + len(u) + 1 > MAX_MSG_LEN:
            msgs.append(current)
            current = base + "\n\n" + u
        else:
            current += " " + u

    msgs.append(current)
    return msgs

# ===== ZIP =====
def create_zip(users, partner):
    name = f"tagall_{int(time.time())}.zip"
    with open("result.txt", "w", encoding="utf-8") as f:
        f.write(f"PARTNER: {partner}\nTOTAL: {len(users)}\n\n")
        for u in users:
            f.write(u + "\n")

    with zipfile.ZipFile(name, "w") as z:
        z.write("result.txt")

    os.remove("result.txt")
    return name

# ===== GLOBAL =====
running = {}

# ===== WORKER =====
async def worker(chat_id, queue, base, stop_flag, progress):
    while not queue.empty() and not stop_flag["stop"]:
        batch = []

        for _ in range(BATCH_SIZE):
            if queue.empty():
                break
            batch.append(queue.get_nowait())

        msgs = split_message(base, batch)

        for text in msgs:
            if stop_flag["stop"]:
                break

            await safe_send(
                chat_id,
                text,
                buttons=[[Button.inline("⛔ STOP", b"stop")]]
            )

        progress["count"] += len(batch)
        await asyncio.sleep(random.uniform(DELAY_MIN, DELAY_MAX))

# ===== TAGALL =====
async def start_tagall(event, partner, text, duration):
    db = load_db()

    msg = await event.reply("🚀 TAGALL DIMULAI", buttons=[[Button.inline("⛔ STOP", b"stop")]])

    stop_flag = {"stop": False}
    running[event.chat_id] = stop_flag

    all_users = []

    await client.send_message(LOG_GROUP, f"🚀 START\n{partner}")

    for group in db["groups"]:
        users = await get_users(group)
        all_users.extend(users)

        queue = asyncio.Queue()
        for u in users:
            queue.put_nowait(u)

        base = f"""✅ MENTION DIMULAI

🔗 {partner}
⏱ {duration//60} menit
🎯 {group}

{text}
"""

        progress = {"count": 0, "total": len(users)}

        async def progress_loop():
            while not stop_flag["stop"]:
                try:
                    await msg.edit(
                        f"⚡ PROGRESS\n{progress['count']}/{progress['total']}",
                        buttons=[[Button.inline("⛔ STOP", b"stop")]]
                    )
                except:
                    pass
                await asyncio.sleep(5)

        prog_task = asyncio.create_task(progress_loop())

        tasks = [
            worker(group, queue, base, stop_flag, progress)
            for _ in range(WORKERS)
        ]

        await asyncio.gather(*tasks)

        stop_flag["stop"] = True
        prog_task.cancel()

        await safe_send(group, "✅ MENTION SELESAI")

    zip_file = create_zip(all_users, partner)

    await client.send_message(LOG_GROUP, f"✅ DONE\n{len(all_users)} user", file=zip_file)
    os.remove(zip_file)

    await msg.edit("✅ TAGALL SELESAI")

# ===== STOP BUTTON =====
@client.on(events.CallbackQuery(data=b"stop"))
async def stop_handler(event):
    if event.chat_id in running:
        running[event.chat_id]["stop"] = True
        await event.edit("⛔ DIHENTIKAN")

# ===== ADD PARTNER =====
@client.on(events.NewMessage(pattern="/addpartner (.*)"))
async def add_partner(event):
    if event.sender_id != OWNER_ID:
        return

    db = load_db()
    link = event.pattern_match.group(1)

    db["partners"][link] = {"last": 0, "durasi": 300}
    save_db(db)

    await event.reply("✅ Partner ditambah")

# ===== ADD GROUP =====
@client.on(events.NewMessage(pattern="/addbot (.*)"))
async def add_group(event):
    if event.sender_id != OWNER_ID:
        return

    db = load_db()
    group = event.pattern_match.group(1)

    if group not in db["groups"]:
        db["groups"].append(group)
        save_db(db)

    await event.reply("✅ Group ditambah")

# ===== REQUEST =====
@client.on(events.NewMessage(pattern="https://t.me/.*"))
async def request(event):
    db = load_db()

    if not db["status"]:
        return await event.reply("❌ BOT OFF")

    link = event.raw_text.strip()

    if link not in db["partners"]:
        return await event.reply("❌ BUKAN PARTNER")

    now = time.time()
    last = db["partners"][link]["last"]

    if now - last < 86400:
        return await event.reply("❌ 1x sehari")

    db["partners"][link]["last"] = now
    save_db(db)

    durasi = db["partners"][link]["durasi"]

    await start_tagall(event, link, "PROMO PARTNER", durasi)

# ===== BACKUP =====
@client.on(events.NewMessage(pattern="/backup"))
async def backup(event):
    if event.sender_id == OWNER_ID:
        await event.reply(file=DB_FILE)

# ===== RESTORE =====
@client.on(events.NewMessage(pattern="/restore"))
async def restore(event):
    if event.sender_id != OWNER_ID:
        return

    file = await event.download_media()
    os.replace(file, DB_FILE)

    await event.reply("✅ RESTORE")

# ===== CLEAN CACHE =====
def clean():
    for f in os.listdir():
        if f.endswith(".zip") or f.endswith(".session-journal"):
            try:
                os.remove(f)
            except:
                pass

# ===== AUTO RESTART =====
async def auto_restart():
    while True:
        await asyncio.sleep(86400)
        clean()
        await client.send_message(LOG_GROUP, "♻️ RESTART")
        os.execl("/usr/bin/python3", "python3", __file__)

# ===== MAIN =====
async def main():
    await client.start(bot_token=BOT_TOKEN)

    print("BOT RUNNING...")

    # ✅ LOG KE GROUP PAS ONLINE
    await client.send_message(LOG_GROUP, "✅ BOT ONLINE (SYSTEMD AKTIF)")

    asyncio.create_task(auto_restart())

    await client.run_until_disconnected()

if __name__ == "__main__":
    asyncio.run(main())
