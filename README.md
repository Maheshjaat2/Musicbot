import asyncio
import os
import re
import time
from typing import Dict, List, Optional, Tuple

from pyrogram import Client, filters, types
from pyrogram.enums import ChatMemberStatus, ChatType
from pyrogram.errors import RPCError
from pyrogram.types import Message

from pytgcalls import PyTgCalls, idle
from pytgcalls.types import Update
from pytgcalls.types.input_stream import AudioPiped
from pytgcalls.types.stream import StreamAudioEnded

import yt_dlp
from humanfriendly import format_timespan

try:
    import config  # user-provided
except Exception:
    raise SystemExit("Please create config.py (copy from config.py.example) and fill values.")

# ---- Globals ----
app = Client(
    name=getattr(__import__("config"), "config").SESSION_NAME if "config" in globals() else "musicbot",
    api_id=config.API_ID,
    api_hash=config.API_HASH,
    bot_token=config.BOT_TOKEN,
    in_memory=True,
)

pytg = PyTgCalls(app)

# per-chat music queues
queues: Dict[int, List[dict]] = {}
current: Dict[int, Optional[dict]] = {}

# AFK store: user_id -> {"since": ts, "reason": str}
afk: Dict[int, dict] = {}

# Warnings: chat_id -> user_id -> int
warnings: Dict[int, Dict[int, int]] = {}

DOWNLOAD_DIR = getattr(config, "DOWNLOAD_DIR", "downloads")
MAX_QUEUE_LENGTH = getattr(config, "MAX_QUEUE_LENGTH", 25)

os.makedirs(DOWNLOAD_DIR, exist_ok=True)

YDL_OPTS = {
    "format": "bestaudio/best",
    "outtmpl": os.path.join(DOWNLOAD_DIR, "%(id)s.%(ext)s"),
    "noplaylist": True,
    "quiet": True,
    "no_warnings": True,
    "geo_bypass": True,
    "prefer_ffmpeg": True,
}

YDL_SEARCH = {
    **YDL_OPTS,
    "default_search": "ytsearch",
    "max_downloads": 1,
}

# ---- Utils ----

def is_sudo(user_id: int) -> bool:
    return user_id in getattr(config, "SUDO_USER_IDS", [])

async def is_admin(chat_id: int, user_id: int) -> bool:
    try:
        m = await app.get_chat_member(chat_id, user_id)
        return m.status in (ChatMemberStatus.OWNER, ChatMemberStatus.ADMINISTRATOR) or is_sudo(user_id)
    except RPCError:
        return False

def human(obj: dict) -> str:
    title = obj.get("title") or "Unknown"
    duration = obj.get("duration")
    if duration:
        return f"{title} [{format_timespan(duration)}]"
    return title

def parse_target_user(msg: Message) -> Optional[int]:
    if msg.reply_to_message:
        return msg.reply_to_message.from_user.id if msg.reply_to_message.from_user else None
    if msg.entities:
        for ent in msg.entities:
            if ent.type == "mention" and msg.text:
                uname = msg.text[ent.offset+1:ent.offset+ent.length]
                try:
                    u = asyncio.get_event_loop().run_until_complete(app.get_users(uname))
                    return u.id
                except Exception:
                    pass
            if ent.type == "text_mention" and ent.user:
                return ent.user.id
    m = re.findall(r"\d{7,}", msg.text or "")
    if m:
        return int(m[0])
    return None

# ---- Music: download & queue ----

def ytdl_extract(query: str) -> Tuple[str, str, int]:
    """
    Returns (filepath, title, duration_seconds)
    """
    opts = YDL_OPTS if re.match(r"^https?://", query) else YDL_SEARCH
    with yt_dlp.YoutubeDL(opts) as ydl:
        info = ydl.extract_info(query, download=True)
        if "entries" in info:
            info = info["entries"][0]
        path = ydl.prepare_filename(info)
        base, _ = os.path.splitext(path)
        for ext in (".m4a", ".webm", ".mp3", ".opus"):
            if os.path.exists(base + ext):
                path = base + ext
                break
        return path, info.get("title", "Unknown"), int(info.get("duration") or 0)

async def start_stream(chat_id: int, path: str):
    await pytg.join_group_call(chat_id, AudioPiped(path))

async def change_stream(chat_id: int, path: str):
    await pytg.change_stream(chat_id, AudioPiped(path))

async def stop_stream(chat_id: int):
    try:
        await pytg.leave_group_call(chat_id)
    except Exception:
        pass

async def play_next(chat_id: int):
    q = queues.get(chat_id, [])
    if not q:
        current[chat_id] = None
        await stop_stream(chat_id)
        return
    track = q.pop(0)
    current[chat_id] = track
    path = track["path"]
    try:
        # If already in VC, change; else join
        await change_stream(chat_id, path)
    except Exception:
        await start_stream(chat_id, path)

# ---- Handlers: Music ----

@app.on_message(filters.command("play") & filters.group)
async def cmd_play(_, msg: Message):
    if not msg.from_user:
        return
    chat_id = msg.chat.id
    query = msg.text.split(maxsplit=1)[1] if len(msg.command) > 1 else None
    if not query:
        return await msg.reply_text("Usage: /play <YouTube URL or search query>")
    if len(queues.get(chat_id, [])) >= MAX_QUEUE_LENGTH:
        return await msg.reply_text("Queue is full. Try again later.")

    m = await msg.reply_text("🔎 Searching / downloading...")
    try:
        path, title, duration = await asyncio.to_thread(ytdl_extract, query)
    except Exception as e:
        return await m.edit_text(f"❌ Failed: {e}")

    track = {"path": path, "title": title, "duration": duration, "requester": msg.from_user.id}
    queues.setdefault(chat_id, []).append(track)

    if current.get(chat_id) is None:
        await play_next(chat_id)
        await m.edit_text(f"▶️ Now playing: **{human(track)}**")
    else:
        await m.edit_text(f"➕ Queued: **{human(track)}**")

@app.on_message(filters.command("skip") & filters.group)
async def cmd_skip(_, msg: Message):
    chat_id = msg.chat.id
    if not await is_admin(chat_id, msg.from_user.id):
        return await msg.reply_text("Admins only.")
    await msg.reply_text("⏭ Skipping...")
    await play_next(chat_id)

@app.on_message(filters.command("pause") & filters.group)
async def cmd_pause(_, msg: Message):
    chat_id = msg.chat.id
    if not await is_admin(chat_id, msg.from_user.id):
        return await msg.reply_text("Admins only.")
    try:
        await pytg.pause_stream(chat_id)
        await msg.reply_text("⏸ Paused.")
    except Exception as e:
        await msg.reply_text(f"Failed: {e}")

@app.on_message(filters.command("resume") & filters.group)
async def cmd_resume(_, msg: Message):
    chat_id = msg.chat.id
    if not await is_admin(chat_id, msg.from_user.id):
        return await msg.reply_text("Admins only.")
    try:
        await pytg.resume_stream(chat_id)
        await msg.reply_text("▶️ Resumed.")
    except Exception as e:
        await msg.reply_text(f"Failed: {e}")

@app.on_message(filters.command("stop") & filters.group)
async def cmd_stop(_, msg: Message):
    chat_id = msg.chat.id
    if not await is_admin(chat_id, msg.from_user.id):
        return await msg.reply_text("Admins only.")
    queues[chat_id] = []
    current[chat_id] = None
    await stop_stream(chat_id)
    try:
        for fn in os.listdir(DOWNLOAD_DIR):
            os.remove(os.path.join(DOWNLOAD_DIR, fn))
    except Exception:
        pass
    await msg.reply_text("⏹ Stopped and cleared queue.")

@app.on_message(filters.command("queue") & filters.group)
async def cmd_queue(_, msg: Message):
    chat_id = msg.chat.id
    q = queues.get(chat_id, [])
    if current.get(chat_id):
        now = f"Now: {human(current[chat_id])}"
    else:
        now = "Nothing is playing."
    if not q:
        return await msg.reply_text(f"{now}\nQueue is empty.")
    lines = [f"{i+1}. {human(t)}" for i, t in enumerate(q[:20])]
    await msg.reply_text(now + "\n\nUpcoming:\n" + "\n".join(lines))

@app.on_message(filters.command("now") & filters.group)
async def cmd_now(_, msg: Message):
    chat_id = msg.chat.id
    cur = current.get(chat_id)
    if not cur:
        return await msg.reply_text("Nothing is playing.")
    await msg.reply_text(f"🎧 {human(cur)}")

@pytg.on_stream_end()
async def on_end(_, update: Update):
    if isinstance(update, StreamAudioEnded):
        chat_id = update.chat_id
        await play_next(chat_id)

# ---- AFK ----

@app.on_message(filters.command("afk"))
async def cmd_afk(_, msg: Message):
    if not msg.from_user:
        return
    reason = msg.text.split(maxsplit=1)[1] if len(msg.command) > 1 else None
    afk[msg.from_user.id] = {"since": int(time.time()), "reason": reason or "AFK"}
    await msg.reply_text(f"🛌 {msg.from_user.mention} is now AFK: {afk[msg.from_user.id]['reason']}")

@app.on_message(filters.group | filters.private, group=1)
async def clear_afk_on_activity(_, msg: Message):
    if not msg.from_user:
        return
    uid = msg.from_user.id
    if uid in afk:
        data = afk.pop(uid)
        since = format_timespan(int(time.time()) - data["since"])
        if msg.chat.type != ChatType.PRIVATE:
            await msg.reply_text(f"✅ {msg.from_user.mention} is back (AFK {since}).")

@app.on_message(filters.group & (filters.reply | filters.mentioned), group=2)
async def notify_afk(_, msg: Message):
    target = None
    if msg.reply_to_message and msg.reply_to_message.from_user:
        target = msg.reply_to_message.from_user.id
    else:
        if msg.entities:
            for e in msg.entities:
                if e.type == "text_mention" and e.user:
                    target = e.user.id
                    break
    if target and target in afk:
        data = afk[target]
        since = format_timespan(int(time.time()) - data["since"])
        await msg.reply_text(f"🔔 That user is AFK: {data['reason']} (for {since}).")

# ---- GC Management ----

def is_elevated_member_status(status) -> bool:
    return status in (ChatMemberStatus.OWNER, ChatMemberStatus.ADMINISTRATOR)

async def require_admin_or_die(msg: Message) -> bool:
    if not msg.from_user:
        return False
    try:
        m = await app.get_chat_member(msg.chat.id, msg.from_user.id)
        if is_elevated_member_status(m.status) or is_sudo(msg.from_user.id):
            return True
    except Exception:
        pass
    await msg.reply_text("Admins only.")
    return False

@app.on_message(filters.command("mute") & filters.group)
async def cmd_mute(_, msg: Message):
    if not await require_admin_or_die(msg):
        return
    target = parse_target_user(msg)
    if not target:
        return await msg.reply_text("Reply or mention a user to mute. `/mute @user [minutes]`")
    mins = 0
    parts = msg.text.split()
    if len(parts) >= 3 and parts[2].isdigit():
        mins = int(parts[2])
    until_date = int(time.time()) + mins*60 if mins > 0 else None
    try:
        await app.restrict_chat_member(msg.chat.id, target, types.ChatPermissions(can_send_messages=False), until_date=until_date)
        await msg.reply_text(f"🔇 Muted user for {mins} minutes." if mins else "🔇 Muted user indefinitely.")
    except RPCError as e:
        await msg.reply_text(f"Failed: {e}")

@app.on_message(filters.command("unmute") & filters.group)
async def cmd_unmute(_, msg: Message):
    if not await require_admin_or_die(msg):
        return
    target = parse_target_user(msg)
    if not target:
        return await msg.reply_text("Reply or mention a user to unmute.")
    try:
        await app.restrict_chat_member(
            msg.chat.id,
            target,
            types.ChatPermissions(
                can_send_messages=True,
                can_send_media_messages=True,
                can_send_polls=True,
                can_add_web_page_previews=True
            )
        )
        await msg.reply_text("🔈 Unmuted user.")
    except RPCError as e:
        await msg.reply_text(f"Failed: {e}")

@app.on_message(filters.command("ban") & filters.group)
async def cmd_ban(_, msg: Message):
    if not await require_admin_or_die(msg):
        return
    target = parse_target_user(msg)
    if not target:
        return await msg.reply_text("Reply or mention a user to ban. `/ban @user [minutes] [reason]`")
    parts = msg.text.split()
    mins = int(parts[2]) if len(parts) >= 3 and parts[2].isdigit() else 0
    reason = " ".join(parts[3:]) if len(parts) >= 4 else None
    until_date = int(time.time()) + mins*60 if mins > 0 else None
    try:
        await app.ban_chat_member(msg.chat.id, target, until_date=until_date)
        await msg.reply_text(f"🚫 Banned user{' for ' + str(mins) + ' minutes' if mins else ''}.{(' Reason: ' + reason) if reason else ''}")
    except RPCError as e:
        await msg.reply_text(f"Failed: {e}")

@app.on_message(filters.command("kick") & filters.group)
async def cmd_kick(_, msg: Message):
    if not await require_admin_or_die(msg):
        return
    target = parse_target_user(msg)
    if not target:
        return await msg.reply_text("Reply or mention a user to kick.")
    try:
        await app.ban_chat_member(msg.chat.id, target)
        await app.unban_chat_member(msg.chat.id, target)
        await msg.reply_text("👢 Kicked user.")
    except RPCError as e:
        await msg.reply_text(f"Failed: {e}")

@app.on_message(filters.command("warn") & filters.group)
async def cmd_warn(_, msg: Message):
    if not await require_admin_or_die(msg):
        return
    target = parse_target_user(msg)
    if not target:
        return await msg.reply_text("Reply or mention a user to warn. `/warn @user [reason]`")
    reason = " ".join(msg.text.split()[2:]) if len(msg.text.split()) >= 3 else None
    warnings.setdefault(msg.chat.id, {})
    warnings[msg.chat.id][target] = warnings[msg.chat.id].get(target, 0) + 1
    await msg.reply_text(f"⚠️ Warned. Total warnings: {warnings[msg.chat.id][target]}{(' | ' + reason) if reason else ''}")

@app.on_message(filters.command("warns") & filters.group)
async def cmd_warns(_, msg: Message):
    target = parse_target_user(msg) or (msg.from_user.id if msg.from_user else None)
    if not target:
        return
    count = warnings.get(msg.chat.id, {}).get(target, 0)
    await msg.reply_text(f"⚠️ Warnings for user: {count}")

@app.on_message(filters.command("clearwarns") & filters.group)
async def cmd_clearwarns(_, msg: Message):
    if not await require_admin_or_die(msg):
        return
    target = parse_target_user(msg)
    if not target:
        return await msg.reply_text("Reply or mention a user to clear warns.")
    warnings.setdefault(msg.chat.id, {}).pop(target, None)
    await msg.reply_text("🧹 Cleared warnings.")

@app.on_message(filters.command("slowmode") & filters.group)
async def cmd_slowmode(_, msg: Message):
    if not await require_admin_or_die(msg):
        return
    secs = int(msg.text.split()[1]) if len(msg.text.split()) >= 2 and msg.text.split()[1].isdigit() else 0
    try:
        await app.set_slow_mode(msg.chat.id, secs)
        await msg.reply_text(f"🐢 Slowmode set to {secs} seconds." if secs else "🚀 Slowmode disabled.")
    except RPCError as e:
        await msg.reply_text(f"Failed: {e}")

@app.on_message(filters.command("purge") & filters.group)
async def cmd_purge(_, msg: Message):
    if not await require_admin_or_die(msg):
        return
    if not msg.reply_to_message:
        return await msg.reply_text("Reply to a message to start purging from there.")
    start_id = msg.reply_to_message.id
    end_id = msg.id
    await msg.reply_text("🧽 Purging...")
    try:
        for i in range(start_id, end_id + 1, 100):
            await app.delete_messages(msg.chat.id, list(range(i, min(i + 100, end_id + 1))))
    except RPCError as e:
        await msg.reply_text(f"Partial purge. Error: {e}")

@app.on_message(filters.command("pin") & filters.group)
async def cmd_pin(_, msg: Message):
    if not await require_admin_or_die(msg):
        return
    if not msg.reply_to_message:
        return await msg.reply_text("Reply to a message to pin.")
    try:
        await app.pin_chat_message(msg.chat.id, msg.reply_to_message.id, disable_notification=True)
        await msg.reply_text("📌 Pinned.")
    except RPCError as e:
        await msg.reply_text(f"Failed: {e}")

@app.on_message(filters.command("unpin") & filters.group)
async def cmd_unpin(_, msg: Message):
    if not await require_admin_or_die(msg):
        return
    try:
        await app.unpin_chat_message(msg.chat.id)
        await msg.reply_text("📍 Unpinned last message.")
    except RPCError as e:
        await msg.reply_text(f"Failed: {e}")

# ---- Lifecycle ----

@app.on_message(filters.command(["start", "help"]))
async def cmd_help(_, msg: Message):
    await msg.reply_text(
        "👋 **Advanced Music Bot**\n\n"
        "Music: /play, /skip, /pause, /resume, /stop, /queue, /now\n"
        "AFK: /afk [reason]\n"
        "GC: /mute, /unmute, /ban, /kick, /warn, /warns, /clearwarns, /slowmode, /purge (reply), /pin (reply), /unpin\n\n"
        "Add me to a group, start a voice chat, then /play a song!"
    )

async def main():
    await app.start()
    await pytg.start()
    print("Bot is up. Press Ctrl+C to stop.")
    try:
        await idle()
    finally:
        await pytg.stop()
        await app.stop()

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        pass
