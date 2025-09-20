#!/usr/bin/env python3
import os, sys, requests
from bs4 import BeautifulSoup
from datetime import datetime

CHARTINK_URL = os.environ.get("CHARTINK_URL", "")
BOT_TOKEN = os.environ.get("TG_BOT_TOKEN", "")
CHAT_ID = os.environ.get("TG_CHAT_ID", "")
UA = "Mozilla/5.0"

if not (CHARTINK_URL and BOT_TOKEN and CHAT_ID):
    print("Please set CHARTINK_URL, TG_BOT_TOKEN and TG_CHAT_ID as repo secrets or env vars.")
    sys.exit(1)

def fetch_chartink(url):
    r = requests.get(url, headers={"User-Agent": UA}, timeout=20)
    r.raise_for_status()
    return r.text

def parse_table(html):
    soup = BeautifulSoup(html, "html.parser")
    table = soup.find("table")
    if not table:
        return []
    rows = []
    for tr in table.find_all("tr"):
        cols = [td.get_text(strip=True) for td in tr.find_all(["td","th"])]
        if cols:
            rows.append(cols)
    return rows

def build_message(rows):
    now = datetime.now().strftime("%Y-%m-%d %H:%M IST")
    header = f"📣 Pivot Gap-up Alert — Snapshot at {now}\n\n"
    if not rows or len(rows) <= 1:
        return header + "_No matches found._"
    data = rows[1:101]
    lines = []
    for cols in data:
        symbol = cols[0]
        extra = (" " + " ".join(cols[1:3])) if len(cols) > 1 else ""
        lines.append(f"{symbol}{extra}")
    return header + "\n".join(lines)

def send_telegram(text):
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    payload = {"chat_id": CHAT_ID, "text": text, "parse_mode": "Markdown", "disable_web_page_preview": True}
    r = requests.post(url, data=payload, timeout=20)
    r.raise_for_status()
    return r.json()

def main():
    html = fetch_chartink(CHARTINK_URL)
    rows = parse_table(html)
    msg = build_message(rows)
    send_telegram(msg)
    print("✅ Sent to Telegram")

if __name__ == "__main__":
    main()
