from flask import Flask, request, jsonify
import requests

app = Flask(__name__)

@app.route("/")
def home():
    return {"status": "NSE API Bridge active"}

@app.route("/optiondata")
def option_data():
    symbol = request.args.get("symbol")
    if not symbol:
        return jsonify({"error": "symbol required"}), 400

    headers = {
        "User-Agent": "Mozilla/5.0",
        "Accept": "application/json",
        "Referer": "https://www.nseindia.com/option-chain",
    }

    session = requests.Session()
    session.get("https://www.nseindia.com", headers=headers)
    r = session.get(f"https://www.nseindia.com/api/option-chain-equities?symbol={symbol}", headers=headers)

    if r.status_code != 200:
        return jsonify({"error": f"Failed to fetch: {r.status_code}"}), 500

    data = r.json()
    cmp = data["records"].get("underlyingValue", None)
    ce_list = [x["CE"] for x in data["filtered"]["data"] if "CE" in x]
    if not ce_list:
        return jsonify({"error": "No CE found"})

    nearest = min(ce_list, key=lambda ce: abs(ce["strikePrice"] - cmp))
    ce_price = nearest.get("lastPrice", 0)
    oi_change = nearest.get("changeinOpenInterest", 0)

    lot_sizes = {"SAIL": 4750, "HUDCO": 3500, "BHEL": 10500, "IRFC": 6000, "NBCC": 9000, "PNB": 8000}
    lot = lot_sizes.get(symbol.upper(), 1000)
    lot_value = cmp * lot
    monthly_yield = (ce_price / lot_value) * 100
    annualized = monthly_yield * 12
    margin = round(lot_value * 0.2)

    return jsonify({
        "symbol": symbol,
        "cmp": cmp,
        "lotSize": lot,
        "lotValue": lot_value,
        "nearestStrike": nearest["strikePrice"],
        "cePrice": ce_price,
        "monthlyYield": monthly_yield,
        "annualized": annualized,
        "oiChange": oi_change,
        "marginEst": margin,
        "remarks": "OK"
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
