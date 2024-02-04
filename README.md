from flask import Flask, request
from twilio.twiml.messaging_response import MessagingResponse
import random

app = Flask(__name__)

@app.route("/whatsapp", methods=['POST'])
def whatsapp():
    incoming_message = request.values.get('Body', '').lower()

    if 'anime' in incoming_message:
        response_message = "¡Bienvenido al mundo del anime! Aquí encontrarás información emocionante sobre tus series favoritas."

    elif 'ruleta' in incoming_message:
        result = random.choice(['rojo', 'negro'])
        response_message = f"La ruleta ha girado y el resultado es: {result}."

    else:
        response_message = "¡Hola! Soy tu asistente virtual. Puedo proporcionarte información sobre anime y jugar a la ruleta. Solo menciona 'anime' o 'ruleta' para comenzar."

    resp = MessagingResponse()
    resp.message(response_message)

    return str(resp)

if __name__ == "__main__":
    app.run(debug=True)
