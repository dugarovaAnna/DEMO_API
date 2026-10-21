# demo_api_single.py
import os
import time
import re
import requests
from datetime import datetime
from fastapi import FastAPI
from dotenv import load_dotenv

load_dotenv()

# --- Конфигурация ---
VK_TOKEN = os.getenv("VK_ACCESS_TOKEN")
COMMUNITIES = [-12345678, -87654321]  #ID сообществ
API_VERSION = "5.131"
POSTS_PER_REQUEST = 20
MAX_POSTS_PER_COMMUNITY = 30
REQUEST_DELAY = 0.4

# --- Вспомогательные функции ---
def extract_time(text: str):
    match = re.search(r"\b(?:[01]?\d|2[0-3]):[0-5]\d\b", text)
    return match.group() if match else None

def extract_date_text(text: str):
    match = re.search(r"\b\d{1,2}\.\d{1,2}\.(?:\d{2}|\d{4})\b", text)
    return match.group() if match else None

def fetch_posts_from_community(owner_id: int):
    all_items = []
    offset = 0
    while len(all_items) < MAX_POSTS_PER_COMMUNITY:
        params = {
            "owner_id": owner_id,
            "count": POSTS_PER_REQUEST,
            "offset": offset,
            "access_token": VK_TOKEN,
            "v": API_VERSION,
        }
        try:
            resp = requests.get("https://api.vk.com/method/wall.get", params=params, timeout=10)
            resp.raise_for_status()
            data = resp.json()
        except Exception as e:
            print(f"Ошибка сети: {e}")
            break

        if "error" in data:
            print(f"VK API error: {data['error']}")
            break

        items = data.get("response", {}).get("items", [])
        if not items:
            break
        all_items.extend(items)
        offset += POSTS_PER_REQUEST
        if len(items) < POSTS_PER_REQUEST:
            break
    return all_items[:MAX_POSTS_PER_COMMUNITY]

def parse_post(post: dict):
    text = post.get("text", "")
    pub_date = datetime.fromtimestamp(post["date"]).strftime("%Y-%m-%d")
    time_str = extract_time(text)
    date_str = extract_date_text(text) or pub_date
    #простой поиск места
    place_str = None
    for kw in ["место", "зал", "аудитория", "корпус"]:
        pos = text.lower().find(kw)
        if pos != -1:
            rest = text[pos + len(kw):].strip()
            place_str = rest.split(".")[0].split(",")[0].strip()
            if place_str:
                break

    link = f"https://vk.com/wall{post['owner_id']}_{post['id']}"

    return {
        "date": date_str,
        "time": time_str,
        "place": place_str,
        "description": text[:300],
        "link": link,
        "source_community_id": post["owner_id"],
        "post_id": post["id"],
    }

# --- FastAPI ---
app = FastAPI()

@app.get("/events")
def get_events():
    if not VK_TOKEN:
        return {"error": "VK_ACCESS_TOKEN не найден"}

    all_posts = []
    for owner_id in COMMUNITIES:
        posts = fetch_posts_from_community(owner_id)
        all_posts.extend(posts)
        time.sleep(REQUEST_DELAY)

    events = [parse_post(p) for p in all_posts]
    return events
