from flask import Flask, request, jsonify
import threading
import requests
import time
import json
from datetime import datetime

# Memuat konfigurasi dari file JSON
with open('config.json', 'r') as config_file:
    config = json.load(config_file)

# Mengambil variabel penting dari konfigurasi
TOKEN = config['TOKEN']
CHANNEL_IDS = config['CHANNEL_IDS']
LOG_WEBHOOK_URL = config['LOG_WEBHOOK_URL']
LOG_FILE = config.get('LOG_FILE', 'logs.json')
EMOJIS = config['EMOJIS']
EMBED_IMAGE = config['EMBED_IMAGE']
DISCORD_LINK = config['DISCORD_LINK']

# Emoji yang akan digunakan dalam embed/log
TITLE_EMOJI = EMOJIS['TITLE_EMOJI']
INFO_EMOJI = EMOJIS['INFO_EMOJI']
STATUS_EMOJI = EMOJIS['STATUS_EMOJI']
ACCOUNT_EMOJI = EMOJIS['ACCOUNT_EMOJI']
UPTIME_EMOJI = EMOJIS['UPTIME_EMOJI']
CHANNEL_EMOJI = EMOJIS['CHANNEL_EMOJI']
MESSAGE_EMOJI = EMOJIS['MESSAGE_EMOJI']
DISCORD_EMOJI = EMOJIS['DISCORD_EMOJI']
ARROW_EMOJI = EMOJIS['ARROW_EMOJI']

# Merekam waktu mulai untuk menghitung uptime
start_time = datetime.now()

# Menghitung total pesan yang sudah terkirim
message_counts = 0

def format_uptime(start_time):
    """Menghitung dan memformat uptime sejak bot mulai berjalan."""
    uptime_duration = datetime.now() - start_time
    days = uptime_duration.days
    hours, remainder = divmod(uptime_duration.seconds, 3600)
    minutes, seconds = divmod(remainder, 60)
    return f"{days}d {hours}h {minutes}m {seconds}s"

def save_log_to_file(log_entry):
    """Menyimpan log ke file JSON."""
    try:
        # Membaca log yang sudah ada
        with open(LOG_FILE, 'r') as file:
            logs = json.load(file)
    except FileNotFoundError:
        logs = []

    # Menambahkan log baru
    logs.append(log_entry)

    # Menyimpan kembali ke file
    with open(LOG_FILE, 'w') as file:
        json.dump(logs, file, indent=4)

def get_bot_user_id():
    """Secara otomatis mengambil User ID bot menggunakan API Discord."""
    url = "https://discord.com/api/v10/users/@me"
    headers = {'Authorization': TOKEN}
    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        bot_data = response.json()
        return bot_data['id']
    else:
        print(f"Gagal Mengambil User ID Bot: {response.status_code} - {response.text}")
        return None

def send_message(content, channel_id):
    """Mengirim pesan ke channel Discord tertentu menggunakan API Discord."""
    global message_counts
    url = f'https://discord.com/api/v10/channels/{channel_id}/messages'
    headers = {
        'Authorization': TOKEN,
        'Content-Type': 'application/json',
    }
    json_data = {'content': content}
    response = requests.post(url, headers=headers, json=json_data)

    if response.status_code == 200:
        message_counts += 1
        log_message(f"Pesan Berhasil Terkirim Ke <#{channel_id}>.", channel_id=channel_id, status="Success")
    elif response.status_code == 401:
        print("Unauthorized: Periksa Token Dan Izin.")
        log_message(f"Unauthorized: Gagal Mengirim Pesan Ke <#{channel_id}>.", channel_id=channel_id, status="Failed")
    elif response.status_code == 429:
        retry_after = response.json().get('retry_after', 0) / 1000
        print(f"Rate Limit: Menunggu {retry_after:.2f} Detik.")
        time.sleep(retry_after)
        send_message(content, channel_id)
    else:
        log_message(f"Gagal Mengirim Pesan: {response.status_code} - {response.text}", channel_id=channel_id, status="Gagal Terkirim")

def log_message(log_content, channel_id, status):
    """Mencatat aktivitas pengiriman pesan ke webhook Discord dan file log."""
    current_time = datetime.now().strftime('[ %d-%m-%Y ]   %H:%M:%S')
    color = 15158332 if status == "Success" else 3066993
    log_entry = {
        "timestamp": current_time,
        "status": status,
        "channel_id": channel_id,
        "message_count": message_counts
    }
    save_log_to_file(log_entry)

    log_embed = {
        "title": f"**{TITLE_EMOJI} 𝗔𝘂𝘁𝗼 𝗣𝗼𝘀𝘁 𝗗𝗶𝘀𝗰𝗼𝗿𝗱 {TITLE_EMOJI}**",
        "color": color,
        "fields": [
            {
                "name": f" **{STATUS_EMOJI} 𝗦𝘁𝗮𝘁𝘂𝘀 𝗟𝗼𝗴**",
                "value": f" **{ARROW_EMOJI}   {status}**",
                "inline": True
            },
            {
                "name": f" **{UPTIME_EMOJI} 𝗨𝗽 𝗧𝗶𝗺𝗲**",
                "value": f" **{ARROW_EMOJI}   {current_time}**",
                "inline": True
            },
            {
                "name": f" **{MESSAGE_EMOJI} 𝗧𝗼𝘁𝗮𝗹 𝗠𝗲𝘀𝘀𝗮𝗴𝗲**",
                "value": f" **{ARROW_EMOJI}   {message_counts}**",
                "inline": True
            },
            {
                "name": f"**{CHANNEL_EMOJI} 𝗖𝗵𝗮𝗻𝗻𝗲𝗹 𝗧𝗮𝗿𝗴𝗲𝘁**",
                "value": f" **{ARROW_EMOJI}   <#{channel_id}>**",
                "inline": True
            },
            {
                "name": f"**{DISCORD_EMOJI} 𝗟𝗶𝗻𝗸 𝗗𝗶𝘀𝗰𝗼𝗿𝗱 𝗦𝗲𝗿𝘃𝗲𝗿**",
                "value": f" **{ARROW_EMOJI}   {DISCORD_LINK}**",
                "inline": False
            }
        ],
        "footer": {
            "text": f"𝗔𝘂𝘁𝗼 𝗣𝗼𝘀𝘁 𝗗𝗶𝘀𝗰𝗼𝗿𝗱",
            "icon_url": EMBED_IMAGE['SMALL_IMAGE']
        },
        "image": {
            "url": EMBED_IMAGE['BIG_IMAGE']
        }
    }

    log_data = {
        "embeds": [log_embed],
        "username": "𝗕𝗮𝗯𝘂 𝗟𝗼𝗴"
    }

    response = requests.post(LOG_WEBHOOK_URL, json=log_data)
    if response.status_code != 204:
        print(f"Log Gagal Di Kirim: {response.status_code} - {response.text}")

# Flask server untuk Web Hosting
app = Flask(__name__)
bot_running = False
threads = []

@app.route("/")
def home():
    return jsonify({"message": "Bot is running!", "status": "OK"})

@app.route("/start", methods=["POST"])
def start_bot():
    global bot_running
    if bot_running:
        return jsonify({"status": "error", "message": "Bot sudah berjalan."})
    bot_running = True
    mode = config["MODE"]
    threading.Thread(target=run_multi_mode if mode == "MULTI" else run_single_mode).start()
    return jsonify({"status": "success", "message": f"Bot dimulai dalam mode {mode}."})

@app.route("/stop", methods=["POST"])
def stop_bot():
    global bot_running
    bot_running = False
    return jsonify({"status": "success", "message": "Bot dihentikan."})

@app.route("/status", methods=["GET"])
def status():
    return jsonify({"status": "running" if bot_running else "stopped", "messages_sent": message_counts})

if __name__ == "__main__":
    threading.Thread(target=lambda: app.run(host="0.0.0.0", port=5000)).start()
    print("Server berjalan di http://localhost:5000/")
