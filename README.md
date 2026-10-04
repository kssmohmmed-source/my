"""
Webhook Server for IQ Option & TradingView Integration
"""

import os
from flask import Flask, request, jsonify
from iqoptionapi.stable_api import IQ_Option

app = Flask(__name__)

# -----------------------------
# 1. إعدادات الحساب والأمان
# -----------------------------
IQ_EMAIL = os.environ.get("IQ_EMAIL", "your_email@example.com")
IQ_PASSWORD = os.environ.get("IQ_PASSWORD", "your_password")

# كلمة سر للتحقق من أن الطلب قادم فعلاً من حسابك على TradingView
WEBHOOK_SECRET_TOKEN = os.environ.get("WEBHOOK_SECRET", "MY_SECRET_PASSCODE_123")

# تفعيل الحساب التجريبي افتراضياً للحماية والتجربة
ACCOUNT_TYPE = "PRACTICE"  # غيّرها إلى "REAL" فقط عند الجاهزية التامة

# -----------------------------
# 2. تهيئة الاتصال بـ IQ Option
# -----------------------------
print("جاري الاتصال بـ IQ Option...")
api = IQ_Option(IQ_EMAIL, IQ_PASSWORD)
connected, reason = api.connect()

if connected:
    print("تم الاتصال بـ IQ Option بنجاح.")
    api.change_balance(ACCOUNT_TYPE)
    print(f"الوضع النشط: {ACCOUNT_TYPE} | الرصيد: {api.get_balance()}")
else:
    print(f"فشل الاتصال: {reason}")


# -----------------------------
# 3. مسار استقبال الـ Webhook
# -----------------------------
@app.route("/webhook", methods=["POST"])
def webhook():
    if not request.is_json:
        return jsonify({"status": "error", "message": "Expected JSON data"}), 400

    data = request.get_json()

    # أ) التحقق الأمني من الرمز السري
    secret = data.get("secret")
    if secret != WEBHOOK_SECRET_TOKEN:
        return jsonify({"status": "unauthorized", "message": "Invalid secret token"}), 401

    # ب) استخراج بيانات الصفقة من رسالة التنبيه
    ticker = data.get("ticker")              # مثال: EURUSD
    action = data.get("action", "").lower()  # call / buy أو put / sell
    amount = float(data.get("amount", 1))    # قيمة الصفقة بالدولار (الافتراضي 1$)
    duration = int(data.get("duration", 1))  # مدة الشمعة/الصفقة بالدقائق

    # تحويل الأوامر إلى call / put المتوافقة مع IQ Option
    direction = "call" if action in ["buy", "call", "long"] else "put" if action in ["sell", "put", "short"] else None

    if not ticker or not direction:
        return jsonify({"status": "error", "message": "Invalid ticker or action"}), 400

    # ج) التحقق من بقاء الاتصال فعالاً
    if not api.check_connect():
        print("إعادة الاتصال بالمنصة...")
        api.connect()

    # د) إرسال أمر الصفقة (Binary Options)
    print(f"تنفيذ صفقة: {direction.upper()} على {ticker} بمبلغ {amount}$ لمدة {duration} دقيقة...")
    status, order_id = api.buy(amount, ticker, direction, duration)

    if status:
        print(f"تم فتح الصفقة بنجاح! رقم الأمر: {order_id}")
        return jsonify({
            "status": "success",
            "order_id": order_id,
            "pair": ticker,
            "direction": direction,
            "amount": amount,
            "duration": duration
        }), 200
    else:
        print("فشل تنفيذ الصفقة على IQ Option.")
        return jsonify({"status": "failed", "message": "Order rejected by broker"}), 500


if __name__ == "__main__":
    # تشغيل السيرفر على المنفذ 5000
    app.run(host="0.0.0.0", port=5000, debug=False)
# my