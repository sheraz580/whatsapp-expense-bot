from flask import Flask, request, jsonify
import openai

app = Flask(__name__)

# Configure OpenAI API
openai.api_key = "YOUR_OPENAI_API_KEY"

# Sample user data store (replace with database later)
users = {}

HBL_ACCOUNT_DETAILS = "HBL Account: 53857000040203\nAccount Title: Sheraz Ali\nPlease send a screenshot after payment."

@app.route('/webhook', methods=['POST'])
def whatsapp_webhook():
    data = request.get_json()
    sender = data['from']
    message = data['message']

    if sender not in users:
        users[sender] = {'expenses': [], 'subscribed': False, 'pending_payment': False}

    if message.lower() == "subscribe":
        users[sender]['pending_payment'] = True
        return jsonify({"reply": f"To subscribe, transfer the payment to:\n{HBL_ACCOUNT_DETAILS}"})
    
    if users[sender]['pending_payment'] and "screenshot" in message.lower():
        users[sender]['subscribed'] = True
        users[sender]['pending_payment'] = False
        return jsonify({"reply": "Payment received! You are now subscribed."})
    
    if users[sender]['subscribed']:
        if "spent" in message:
            amount, category = extract_expense(message)
            users[sender]['expenses'].append((amount, category))
            return jsonify({"reply": f"Recorded {amount} in {category}"})
        elif "total" in message:
            total = sum(exp[0] for exp in users[sender]['expenses'])
            return jsonify({"reply": f"Total spent this month: {total}"})
        elif "budget" in message:
            return create_budget(sender)
    else:
        return jsonify({"reply": "Please subscribe to use this feature. Reply 'subscribe' to proceed."})


def extract_expense(message):
    words = message.split()
    amount = int(words[1])
    category = words[-1]
    return amount, category


def create_budget(user):
    total = sum(exp[0] for exp in users[user]['expenses'])
    budget = total * 1.2  # Suggest a 20% increase as budget
    return jsonify({"reply": f"Based on your spending, your suggested budget is {budget}."})

if __name__ == '__main__':
    app.run(port=5000)
