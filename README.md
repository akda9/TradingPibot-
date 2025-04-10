# TradingPibot-
import ccxt import pandas as pd import time import requests from ta.trend import EMAIndicator from ta.momentum import RSIIndicator

=== CONFIGURACIÓN OKX ===

API_KEY = 'TU_OKX_API_KEY' API_SECRET = 'TU_OKX_SECRET_KEY' API_PASSPHRASE = 'TU_OKX_PASSPHRASE'

=== TELEGRAM CONFIG ===

TELEGRAM_TOKEN = 'TU_TELEGRAM_TOKEN' TELEGRAM_CHAT_ID = 'TU_TELEGRAM_CHAT_ID'

def send_telegram(message): url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage" data = {'chat_id': TELEGRAM_CHAT_ID, 'text': message} try: requests.post(url, data=data) except: print("[Telegram Error]")

=== PARAMETROS DE TRADING ===

pair = 'PI/USDT' timeframes = ['5m', '15m'] amount_usdt = 5  # Puedes hacer versiones para $1, $10, $50 también profit_target = 0.03 stop_loss_limit = 0.02

=== CONEXIÓN A OKX ===

exchange = ccxt.okx({ 'apiKey': API_KEY, 'secret': API_SECRET, 'password': API_PASSPHRASE, 'enableRateLimit': True, 'options': { 'defaultType': 'spot', } })

positions = {}

def fetch_data(symbol, timeframe): ohlcv = exchange.fetch_ohlcv(symbol, timeframe, limit=100) df = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume']) df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms') return df

def analyze(df): df['ema7'] = EMAIndicator(df['close'], window=7).ema_indicator() df['ema25'] = EMAIndicator(df['close'], window=25).ema_indicator() df['ema99'] = EMAIndicator(df['close'], window=99).ema_indicator() df['rsi'] = RSIIndicator(df['close'], window=14).rsi() last = df.iloc[-1] if (last['close'] < last['ema7'] and last['close'] < last['ema25'] and last['close'] < last['ema99'] and last['rsi'] < 30): return True, last['close'] return False, last['close']

def place_order(symbol, entry_price): amount = round(amount_usdt / entry_price, 6) try: exchange.create_market_buy_order(symbol, amount) tp = entry_price * (1 + profit_target) sl = entry_price * (1 - stop_loss_limit) send_telegram(f"Compra ejecutada en {symbol} a {entry_price}") return {'entry': entry_price, 'tp': tp, 'sl': sl, 'amount': amount} except Exception as e: print(f"[ERROR] Compra fallida: {e}") send_telegram(f"[ERROR] Compra fallida en {symbol}: {e}") return None

def log_trade(tf, symbol, entry, tp, sl): with open('trades_log.csv', 'a') as f: f.write(f"{tf},{symbol},{entry},{tp},{sl},{pd.Timestamp.now()}\n")

while True: try: for tf in timeframes: key = f"{pair}_{tf}" if key in positions: price = exchange.fetch_ticker(pair)['last'] pos = positions[key] if price >= pos['tp']: exchange.create_market_sell_order(pair, pos['amount']) send_telegram(f"Take Profit ejecutado en {pair} {tf} a {price}") del positions[key] elif price <= pos['sl']: exchange.create_market_sell_order(pair, pos['amount']) send_telegram(f"Stop Loss ejecutado en {pair} {tf} a {price}") del positions[key] continue

df = fetch_data(pair, tf)
        signal, last_price = analyze(df)
        if signal:
            send_telegram(f"Señal de compra detectada en {pair} {tf} a {last_price}")
            order = place_order(pair, last_price)
            if order:
                positions[key] = order
                log_trade(tf, pair, last_price, order['tp'], order['sl'])

    time.sleep(60)

except Exception as e:
    print(f"[ERROR GENERAL] {e}")
    send_telegram(f"[ERROR GENERAL] {e}")
    time.sleep(10)

