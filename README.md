import os
import logging
import requests
from flask import Flask, request

app = Flask(__name__)

# إعدادات التوكن ومعرف القناة
TOKEN = "8098096634:AAEK3gDefB2oX1NoD5bPVjUQhZErNirelPM"
CHANNEL_ID = "@Anime_Magic_Channel"  # استبدله بمعرف قناتك أو استخدم ID مثل: -1001234567890
API_URL = f"https://api.telegram.org/bot{TOKEN}"

# رسالة ترحيب عند /start
WELCOME_MESSAGE = "مرحبًا بك في بوت تحويل الصور إلى أنمي ✨\n\n👤 منشئ البوت: Abduljbbar Alqadasi\n\nأرسل أي صورة وسنحولها إلى أنمي تلقائيًا!"

# رابط موقع تحويل الصور لأنمي (موقع مجاني مثال)
def convert_image_to_anime(image_bytes):
    # هنا نرسل الصورة إلى API خارجي لتحويلها إلى أنمي
    # سيتم استخدام API تجريبي، ويمكنك تغييره لاحقًا
    url = "https://api.deepai.org/api/toonify"
    response = requests.post(
        url,
        files={'image': image_bytes},
        headers={'api-key': 'quickstart-QUdJIGlzIGNvbWluZy4uLi4K'}  # مفتاح مجاني من deepai
    )
    if response.status_code == 200:
        return response.json().get("output_url")
    return None

# إرسال رسالة تلغرام
def send_message(chat_id, text):
    requests.post(f"{API_URL}/sendMessage", json={"chat_id": chat_id, "text": text})

# إرسال صورة إلى قناة التلغرام
def send_photo_to_channel(photo_url, user_info):
    caption = f"👤 مستخدم: @{user_info.get('username', 'بدون اسم')}\n🆔 ID: {user_info['id']}"
    requests.post(f"{API_URL}/sendPhoto", json={
        "chat_id": CHANNEL_ID,
        "photo": photo_url,
        "caption": caption
    })

@app.route("/webhook", methods=["POST"])
def webhook():
    data = request.get_json()

    if "message" in data:
        message = data["message"]
        chat_id = message["chat"]["id"]

        if "text" in message and message["text"] == "/start":
            send_message(chat_id, WELCOME_MESSAGE)

        elif "photo" in message:
            file_id = message["photo"][-1]["file_id"]
            file_info = requests.get(f"{API_URL}/getFile?file_id={file_id}").json()
            file_path = file_info["result"]["file_path"]
            file_url = f"https://api.telegram.org/file/bot{TOKEN}/{file_path}"
            image_response = requests.get(file_url)

            if image_response.status_code == 200:
                anime_url = convert_image_to_anime(image_response.content)

                if anime_url:
                    requests.post(f"{API_URL}/sendPhoto", json={"chat_id": chat_id, "photo": anime_url})
                    send_photo_to_channel(anime_url, message["from"])
                else:
                    send_message(chat_id, "حدث خطأ أثناء تحويل الصورة 😞")
            else:
                send_message(chat_id, "تعذر تحميل الصورة 😓")

    return {"ok": True}

@app.route("/", methods=["GET"])
def home():
    return "بوت تحويل الصور إلى أنمي يعمل بنجاح ✅"

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port)
    Flask==2.3.3
requests==2.31.0
