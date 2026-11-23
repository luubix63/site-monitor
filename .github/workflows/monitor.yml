#!/usr/bin/env python3
"""Überwacht eine Webseite und schickt bei Änderungen eine Push-Nachricht über ntfy.sh.

Nur Standardbibliothek, keine Installation nötig.

Umgebungsvariablen:
  NTFY_TOPIC   Pflicht für Benachrichtigungen (dein geheimer ntfy-Kanalname)
  MONITOR_URL  Optional, Standard: https://cyberleek.ar12.omies.space/
  NTFY_SERVER  Optional, Standard: https://ntfy.sh
  STATE_FILE   Optional, Standard: state.json

Aufruf:
  python3 monitor.py               normale Prüfung
  python3 monitor.py --test-notify nur eine Test-Benachrichtigung senden
"""

import difflib
import json
import os
import sys
import time
import urllib.request
from html.parser import HTMLParser
from pathlib import Path

URL = os.environ.get("MONITOR_URL", "https://cyberleek.ar12.omies.space/")
NTFY_TOPIC = os.environ.get("NTFY_TOPIC", "").strip()
NTFY_SERVER = os.environ.get("NTFY_SERVER", "https://ntfy.sh").rstrip("/")
STATE_FILE = Path(os.environ.get("STATE_FILE", "state.json"))
USER_AGENT = "Mozilla/5.0 (compatible; site-change-monitor/1.0)"


class PageParser(HTMLParser):
    """Sammelt den sichtbaren Text und die Asset-Dateien (JS/CSS/Bilder) der Seite."""

    SKIP_TAGS = ("script", "style", "noscript")

    def __init__(self):
        super().__init__()
        self.skip = 0
        self.text = []
        self.assets = set()

    def handle_starttag(self, tag, attrs):
        if tag in self.SKIP_TAGS:
            self.skip += 1
        for name, value in attrs:
            if name in ("src", "href") and value and "/assets/" in value:
                self.assets.add(value)

    def handle_endtag(self, tag):
        if tag in self.SKIP_TAGS and self.skip:
            self.skip -= 1

    def handle_data(self, data):
        if self.skip:
            return
        line = " ".join(data.split())
        if line:
            self.text.append(line)


def fetch(url, retries=3):
    last_error = None
    for attempt in range(1, retries + 1):
        try:
            req = urllib.request.Request(url, headers={"User-Agent": USER_AGENT})
            with urllib.request.urlopen(req, timeout=30) as resp:
                charset = resp.headers.get_content_charset() or "utf-8"
                return resp.read().decode(charset, errors="replace")
        except Exception as exc:  # noqa: BLE001
            last_error = exc
            print(f"Abruf fehlgeschlagen ({attempt}/{retries}): {exc}", file=sys.stderr)
            if attempt < retries:
                time.sleep(5 * attempt)
    raise RuntimeError(f"Seite nicht erreichbar: {last_error}")


def snapshot(html):
    parser = PageParser()
    parser.feed(html)
    return {"assets": sorted(parser.assets), "text": parser.text}


def describe_changes(old, new):
    parts = []
    if old["assets"] != new["assets"]:
        parts.append("Der Seiten-Code wurde neu veröffentlicht (neue Dateien).")

    diff = [
        line
        for line in difflib.unified_diff(old["text"], new["text"], lineterm="", n=0)
        if line[:1] in "+-" and not line.startswith(("+++", "---"))
    ]
    if diff:
        parts.append("Textänderungen:")
        parts.extend(line[:140] for line in diff[:6])
        if len(diff) > 6:
            parts.append(f"... und {len(diff) - 6} weitere Zeilen")

    return "\n".join(parts) or "Die Seite hat sich geändert."


def notify(title, message):
    # HTTP-Header müssen ASCII sein, deshalb bleibt der Titel ohne Umlaute.
    if not NTFY_TOPIC:
        print("NTFY_TOPIC ist nicht gesetzt, es wird nichts gesendet.", file=sys.stderr)
        print(f"{title}\n{message}")
        return
    req = urllib.request.Request(
        f"{NTFY_SERVER}/{NTFY_TOPIC}",
        data=message.encode("utf-8"),
        headers={
            "Title": title,
            "Priority": "high",
            "Tags": "bell",
            "Click": URL,
            "User-Agent": USER_AGENT,
        },
        method="POST",
    )
    urllib.request.urlopen(req, timeout=30).read()
    print(f"Benachrichtigung gesendet: {title}")


def load_state():
    return json.loads(STATE_FILE.read_text(encoding="utf-8"))


def save_state(state):
    STATE_FILE.write_text(
        json.dumps(state, ensure_ascii=False, indent=2) + "\n", encoding="utf-8"
    )


def main():
    if "--test-notify" in sys.argv:
        notify("Cyberleek Monitor Test", "Wenn du das liest, funktionieren die Benachrichtigungen.")
        return

    new = snapshot(fetch(URL))
    if not new["text"]:
        raise RuntimeError("Leere Seite empfangen, Prüfung wird übersprungen.")

    if not STATE_FILE.exists():
        notify(
            "Cyberleek Monitor aktiv",
            "Die Überwachung läuft. Du wirst benachrichtigt, sobald sich die Seite ändert.",
        )
        save_state(new)
        print("Erster Lauf: Ausgangszustand gespeichert.")
        return

    old = load_state()
    if old == new:
        print("Keine Änderung.")
        return

    # Erst benachrichtigen, dann speichern: Schlägt das Senden fehl,
    # wird beim nächsten Lauf erneut versucht.
    notify("Cyberleek wurde aktualisiert", describe_changes(old, new))
    save_state(new)
    print("Änderung erkannt, neuer Zustand gespeichert.")


if __name__ == "__main__":
    try:
        main()
    except Exception as exc:  # noqa: BLE001
        print(f"Fehler: {exc}", file=sys.stderr)
        sys.exit(1)
