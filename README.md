import os
import requests
import time
from datetime import datetime

INSTAGRAM_BUSINESS_ACCOUNT_ID = os.getenv("INSTAGRAM_BUSINESS_ACCOUNT_ID", "")
ACCESS_TOKEN = os.getenv("ACCESS_TOKEN", "")
TELEGRAM_LINK = os.getenv("TELEGRAM_LINK", "https://t.me/RozaKnigaBot")

API_VERSION = "v18.0"
TRIGGER_WORD = "ВЫЖИМКА"

class InstagramBot:
    def __init__(self):
        self.account_id = INSTAGRAM_BUSINESS_ACCOUNT_ID
        self.token = ACCESS_TOKEN
        self.processed = set()

    def get_recent_posts(self):
        url = f"https://graph.instagram.com/{API_VERSION}/{self.account_id}/ig_media"
        params = {"fields": "id,media_type,caption", "access_token": self.token, "limit": 5}
        try:
            r = requests.get(url, params=params, timeout=10)
            return r.json().get("data", []) if r.status_code == 200 else []
        except:
            return []

    def get_comments(self, media_id):
        url = f"https://graph.instagram.com/{API_VERSION}/{media_id}/comments"
        params = {"fields": "id,from,text,timestamp", "access_token": self.token, "limit": 100}
        try:
            r = requests.get(url, params=params, timeout=10)
            return r.json().get("data", []) if r.status_code == 200 else []
        except:
            return []

    def is_subscriber(self, user_id):
        try:
            url = f"https://graph.instagram.com/{API_VERSION}/{user_id}"
            params = {"fields": "username", "access_token": self.token}
            r = requests.get(url, params=params, timeout=10)
            return r.status_code == 200
        except:
            return False

    def send_dm(self, user_id):
        msg = f"""Привет! 🎁

Вот полная выжимка из этой книги — все инсайты + практика.

👇 Забрать:
{TELEGRAM_LINK}

Делись выжимкой 📚✨"""

        url = f"https://graph.instagram.com/{API_VERSION}/{self.account_id}/messages"
        data = {
            "recipient": {"id": user_id},
            "message": msg,
            "access_token": self.token
        }
        try:
            r = requests.post(url, json=data, timeout=10)
            return r.status_code in [200, 201]
        except:
            return False

    def run(self):
        print(f"🤖 Бот запущен: @roza.kniga")
        print(f"🎯 Триггер: {TRIGGER_WORD}")
        print(f"⏰ {datetime.now().strftime('%H:%M:%S')}\n")

        while True:
            try:
                posts = self.get_recent_posts()

                for post in posts:
                    comments = self.get_comments(post["id"])

                    for comment in comments:
                        cid = comment["id"]
                        if cid in self.processed:
                            continue

                        if TRIGGER_WORD in comment.get("text", "").upper():
                            self.processed.add(cid)
                            user_id = comment["from"]["id"]
                            username = comment["from"].get("username", "user")

                            print(f"✅ {username}: {TRIGGER_WORD}")

                            if self.is_subscriber(user_id):
                                if self.send_dm(user_id):
                                    print(f"📩 DM отправлено")
                                else:
                                    print(f"❌ Ошибка отправки")
                            else:
                                print(f"⚠️ Не подписан")

                time.sleep(30)

            except KeyboardInterrupt:
                print("\n🛑 Остановлено")
                break
            except Exception as e:
                print(f"❌ {str(e)}")
                time.sleep(60)

if __name__ == "__main__":
    if not INSTAGRAM_BUSINESS_ACCOUNT_ID or not ACCESS_TOKEN:
        print("❌ Ошибка: установи переменные:")
        print("   INSTAGRAM_BUSINESS_ACCOUNT_ID")
        print("   ACCESS_TOKEN")
        exit(1)

    bot = InstagramBot()
    bot.run()
