import os
import asyncio
from pyrogram import Client, filters
from pyrogram.types import Message
from yt_dlp import YoutubeDL

API_ID = "24717197"
API_HASH = "c059312004327a36c54bce90967a2ce4"
BOT_TOKEN = "8403984026:AAGT_lvVR3OWhK2RGipt0HRS1sDUyACffQI"

app = Client("musicbot", api_id=API_ID, api_hash=API_HASH, bot_token=BOT_TOKEN)

music_queue = {}  # Dictionary to hold queues per chat

def search_youtube(query):
    with YoutubeDL({'format': 'bestaudio', 'noplaylist': 'True', 'quiet': True}) as ydl:
        try:
            info = ydl.extract_info(f"ytsearch:{query}", download=False)['entries'][0]
            return {
                'title': info['title'],
                'url': info['webpage_url'],
                'duration': info.get('duration'),
                'id': info['id'],
                'audio_url': info['url'],
                'thumbnail': info.get('thumbnail')
            }
        except Exception as e:
            return None

@app.on_message(filters.command("start"))
async def start(client, message: Message):
    await message.reply("ðŸ‘‹ Welcome to Advanced Telegram Music Bot!\nUse /play <song name or URL> to play music.")

@app.on_message(filters.command("play"))
async def play(client, message: Message):
    chat_id = message.chat.id
    if len(message.command) < 2:
        await message.reply("Please provide a song name or URL after /play.")
        return
    query = " ".join(message.command[1:])
    msg = await message.reply("ðŸ”Ž Searching for music...")
    track = search_youtube(query)
    if not track:
        await msg.edit("âŒ Song not found.")
        return

    # Enqueue or play immediately
    if chat_id not in music_queue:
        music_queue[chat_id] = []
    music_queue[chat_id].append(track)
    if len(music_queue[chat_id]) == 1:
        await play_next(client, message, chat_id)
        await msg.delete()
    else:
        await msg.edit(f"âœ… Added to queue: **{track['title']}**")

async def play_next(client, message, chat_id):
    if chat_id not in music_queue or not music_queue[chat_id]:
        return
    track = music_queue[chat_id][0]
    audio = await download_audio(track['url'])
    if audio:
        await message.reply_audio(audio, title=track['title'], caption=f"[{track['title']}]({track['url']})", thumb=track['thumbnail'])
        os.remove(audio)
    else:
        await message.reply(f"âŒ Failed to download: {track['title']}")
    # Remove from queue and play next if available
    music_queue[chat_id].pop(0)
    if music_queue[chat_id]:
        await play_next(client, message, chat_id)

async def download_audio(url):
    out = "temp.%(ext)s"
    ydl_opts = {
        "format": "bestaudio/best",
        "outtmpl": out,
        "noplaylist": True,
        "quiet": True,
        "postprocessors": [{
            "key": "FFmpegExtractAudio",
            "preferredcodec": "mp3",
            "preferredquality": "192",
        }],
    }
    try:
        with YoutubeDL(ydl_opts) as ydl:
            info_dict = ydl.extract_info(url, download=True)
            file_path = ydl.prepare_filename(info_dict)
            audio_file = file_path.rsplit(".", 1)[0] + ".mp3"
            return audio_file
    except Exception as e:
        print("Download error:", e)
        return None

@app.on_message(filters.command("queue"))
async def queue(client, message: Message):
    chat_id = message.chat.id
    if chat_id not in music_queue or not music_queue[chat_id]:
        await message.reply("ðŸ”ˆ The queue is empty!")
        return
    txt = "**Current Queue:**\n"
    for i, track in enumerate(music_queue[chat_id], 1):
        txt += f"{i}. [{track['title']}]({track['url']})\n"
    await message.reply(txt, disable_web_page_preview=True)

@app.on_message(filters.command("skip"))
async def skip(client, message: Message):
    chat_id = message.chat.id
    if chat_id not in music_queue or len(music_queue[chat_id]) < 2:
        await message.reply("â­ï¸ No next song to skip to.")
        return
    # Remove current song and play next
    music_queue[chat_id].pop(0)
    await play_next(client, message, chat_id)

if __name__ == "__main__":
    app.run()
