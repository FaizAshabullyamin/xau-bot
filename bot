import os, json
import requests
import pandas as pd
import yfinance as yf

TICKER = os.getenv("TICKER", "XAUUSD=X")   # fallback: GC=F
OFFSET = float(os.getenv("OFFSET") or 0)   # laras beza harga vs broker MT5
PIP = 0.10                                  # 10 pip = $1
MIN_SL, MAX_SL = 30, 80                     # had SL (pip)
TP_R = (1, 2, 4)                            # TP1/2/3 = 1R/2R/4R (30/60/120 pip jika SL 30)
STATE = "state.json"
TOKEN = os.getenv("TELEGRAM_TOKEN")
CHAT = os.getenv("TELEGRAM_CHAT_ID")


def tg(msg):
    print(msg)
    if not TOKEN or not CHAT:
        return
    try:
        requests.post(f"https://api.telegram.org/bot{TOKEN}/sendMessage",
                      data={"chat_id": CHAT, "text": msg}, timeout=20)
    except Exception as e:
        print("telegram error", e)


def load():
    try:
        with open(STATE) as f:
            return json.load(f)
    except Exception:
        return {"open": [], "closed": [], "last_bar": ""}


def save(s):
    with open(STATE, "w") as f:
        json.dump(s, f, indent=1)


def fetch(interval, period):
    try:
        df = yf.Ticker(TICKER).history(period=period, interval=interval)
    except Exception as e:
        print("fetch error", e)
        return None
    if df is None or df.empty:
        return None
    df = df[["Open", "High", "Low", "Close"]].dropna()
    idx = df.index
    df.index = idx.tz_localize("UTC") if idx.tz is None else idx.tz_convert("UTC")
    return df + OFFSET


def pips(t, price):
    return round((price - t["entry"]) * t["d"] / PIP)


def stats(s):
    c = s["closed"]
    w = sum(1 for x in c if x["res"] == "PROFIT")
    l = sum(1 for x in c if x["res"] == "LOSS")
    tot = sum(x["pips"] for x in c)
    wr = round(100 * w / len(c)) if c else 0
    return f"Rekod: {len(c)} trade | {w} profit | {l} loss | win {wr}% | jumlah {tot:+d} pip"


def close(s, t, price, why):
    t["close"] = round(price, 2)
    t["pips"] = pips(t, price)
    t["res"] = "PROFIT" if t["pips"] > 0 else "LOSS" if t["pips"] < 0 else "BE"
    s["open"].remove(t)
    s["closed"].append(t)
    tg(f"{'✅' if t['res']=='PROFIT' else '❌' if t['res']=='LOSS' else '➖'} XAUUSD {t['side']} DITUTUP ({why})\n"
       f"Entry {t['entry']:,.2f} → Tutup {t['close']:,.2f}\n"
       f"Hasil: {t['pips']:+d} pip ({t['res']}) | peak {t['peak']:+d}\n{stats(s)}")


def manage(s, m5):
    for t in list(s["open"]):
        d = t["d"]
        since = pd.Timestamp(t["checked"])
        for ts, b in m5[m5.index > since].iterrows():
            t["checked"] = ts.isoformat()
            best = b.High if d > 0 else b.Low
            t["peak"] = max(t["peak"], pips(t, best))
            hit = (lambda lv: b.High >= lv) if d > 0 else (lambda lv: b.Low <= lv)
            if (b.Low <= t["sl"]) if d > 0 else (b.High >= t["sl"]):
                why = "SL" if t["stage"] == 0 else "BE" if t["stage"] == 1 else "SL naik (TP1)"
                close(s, t, t["sl"], why)
                break
            if t["stage"] == 0 and hit(t["tp1"]):
                t["stage"] = 1
                t["sl"] = t["entry"]
                tg(f"🎯 XAUUSD {t['side']} — TP1 HIT ({t['tp1']:,.2f})\nSL dialih ke Break Even {t['entry']:,.2f}")
            if t["stage"] == 1 and hit(t["tp2"]):
                t["stage"] = 2
                t["sl"] = t["tp1"]
                tg(f"🎯🎯 XAUUSD {t['side']} — TP2 HIT ({t['tp2']:,.2f})\nSL dinaikkan ke TP1 {t['tp1']:,.2f}")
            if t["stage"] >= 1 and hit(t["tp3"]):
                close(s, t, t["tp3"], "TP3")
                break


def signal(m15):
    d = m15.iloc[:-1]  # candle tertutup sahaja
    if len(d) < 30:
        return None
    ha_c = (d.Open + d.High + d.Low + d.Close) / 4
    ho = [(d.Open.iloc[0] + d.Close.iloc[0]) / 2]
    for i in range(1, len(d)):
        ho.append((ho[-1] + ha_c.iloc[i - 1]) / 2)
    ha_o = pd.Series(ho, index=d.index)
    ha_h = pd.concat([d.High, ha_o, ha_c], axis=1).max(axis=1)
    ha_l = pd.concat([d.Low, ha_o, ha_c], axis=1).min(axis=1)
    dl = d.Close.diff()
    up = dl.clip(lower=0).ewm(alpha=1 / 14, adjust=False).mean()
    dn = (-dl.clip(upper=0)).ewm(alpha=1 / 14, adjust=False).mean()
    rsi = 100 - 100 / (1 + up / dn.replace(0, 1e-9))
    bull = ha_c > ha_o
    mom = d.Close.iloc[-1] - d.Close.iloc[-4]
    r = float(rsi.iloc[-1])
    bar = d.index[-1].isoformat()
    if bull.iloc[-1] and not bull.iloc[-2] and r > 50 and mom > 0:
        return "BUY", 1, float(ha_l.iloc[-1]), r, bar, "HA Bull Candle"
    if (not bull.iloc[-1]) and bull.iloc[-2] and r < 50 and mom < 0:
        return "SELL", -1, float(ha_h.iloc[-1]), r, bar, "HA Bear Candle"
    return None


def main():
    s = load()
    if os.getenv("TEST"):
        tg("✅ Bot XAUUSD aktif. Alert signal akan dihantar ke sini.")
    m5, m15 = fetch("5m", "2d"), fetch("15m", "5d")
    if m5 is None or m15 is None:
        print("tiada data (pasaran tutup / API gagal)")
        return
    manage(s, m5)
    if not s["open"]:
        sg = signal(m15)
        age = (pd.Timestamp.now(tz="UTC") - m15.index[-2]).total_seconds() / 60 if len(m15) > 1 else 999
        if sg and sg[4] != s.get("last_bar") and age < 45:
            side, d, ref, r, bar, why = sg
            entry = float(m5.Close.iloc[-1])
            risk = abs(entry - ref) + 0.5
            risk = min(max(risk, MIN_SL * PIP), MAX_SL * PIP)
            t = {"side": side, "d": d, "entry": round(entry, 2), "sl": round(entry - d * risk, 2),
                 "tp1": round(entry + d * risk * TP_R[0], 2), "tp2": round(entry + d * risk * TP_R[1], 2),
                 "tp3": round(entry + d * risk * TP_R[2], 2), "stage": 0, "peak": 0,
                 "sent": pd.Timestamp.now(tz="UTC").strftime("%d %b %H:%M UTC"),
                 "checked": m5.index[-1].isoformat()}
            s["open"].append(t)
            s["last_bar"] = bar
            tg(f"🥇 XAUUSD · {side}\n\nEntry: {t['entry']:,.2f}\n"
               f"SL: {t['sl']:,.2f} (-{round(risk/PIP)} pip)\n"
               f"TP1: {t['tp1']:,.2f} (+{round(risk/PIP*TP_R[0])})\n"
               f"TP2: {t['tp2']:,.2f} (+{round(risk/PIP*TP_R[1])})\n"
               f"TP3: {t['tp3']:,.2f} (+{round(risk/PIP*TP_R[2])})\n\n"
               f"Sebab: {why} · Momentum {side} · RSI {r:.0f}\n"
               f"Hantar: {t['sent']}")
    save(s)


if __name__ == "__main__":
    main()
