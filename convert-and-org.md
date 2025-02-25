## Prompt
here is my project structure: 

```
📦doug
 ┣ 📂app
 ┃ ┣ 📂characters
 ┃ ┃ ┣ 📜c3po.character.json
 ┃ ┃ ┣ 📜doug.character.json
 ┃ ┃ ┗ 📜shaw.character.json
 ┃ ┣ 📂clients
 ┃ ┃ ┗ 📜command_line.py
 ┃ ┣ 📂components
 ┃ ┃ ┣ 📜__init__.py
 ┃ ┃ ┗ 📜start.py
 ┃ ┣ 📂extras
 ┃ ┃ ┣ 📜__init__.py
 ┃ ┃ ┣ 📜models.py
 ┃ ┃ ┗ 📜utils.py
 ┃ ┗ 📜app.py
 ┣ 📂bash
 ┃ ┗ 📜start.sh
 ┣ 📜.env
 ┣ 📜.env.example
 ┣ 📜.gitignore
 ┣ 📜Dockerfile
 ┣ 📜README.md
 ┣ 📜docker-compose.yml
 ┣ 📜poetry.lock
 ┗ 📜pyproject.toml
```
 i've converted my project from running with main.py to using guinicorn to serve flask.
 before I was communicating and printing responses from an llm to the command line, but
 now that its running on gunicorn id like to set it up so I can use an api end point.
 at the same time i'd like to modify my existing code so I can still communicate through
 the command line if I accessed it through ssh or through the docker container directly.

can you help me do this?

here is my new app.py:

```
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello, World!"

if __name__ == "__main__":
    app.run()
```

and here is my old main.py:
```
import requests
from dotenv import load_dotenv
import os
from app.extras.utils import get_model
import json

# Load environment variables from .env file
load_dotenv()

# Get the API key from environment variables
api_key = os.getenv('OPEN_ROUTER')

# Get model by name key
model_name = get_model("deepseekv3")

# Initialize conversation history (context) as a list
conversation_history = []

def load_character_context():
    with open('characters/c3po.character.json', 'r') as f:
        character = json.load(f)
    
    # Create context from name and bio
    context = f"You are {character['name']}. "
    context += " ".join(character['bio'])
    context += " ".join(character['lore'])
    context += " ".join(character['knowledge'])
    #context += " ".join(character['messageExamples'])
    #context += " ".join(character['postExamples'])
    context += " ".join(character['style'])
    context += " ".join(character['adjectives'])

    
    # Add the initial context as a system message
    conversation_history.append({
        "role": "system",
        "content": context
    })

def get_chat_response(message_content):
    url = "https://openrouter.ai/api/v1/chat/completions"
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }

    conversation_history.append({"role": "user", "content": message_content})

    payload = {
        "model": model_name,
        "messages": conversation_history,
        "max_tokens": 100
    }

    response = requests.post(url, headers=headers, json=payload)
    response_data = response.json()
    response_content = response_data['choices'][0]['message']['content']
    
    conversation_history.append({"role": "assistant", "content": response_content})
    print("\nChatbot:", response_content)

def main():
    # Load character context before starting chat
    load_character_context()
    print("Chatbot is ready! Type 'exit' to quit.")
    
    while True:
        user_message = input("\nYou: ")
        if user_message.lower() == "exit":
            print("Goodbye!")
            break
        get_chat_response(user_message)

if __name__ == "__main__":
    main()
```

## Response

You need to refactor your project to support both an API endpoint (for Gunicorn) and a command-line interface (for direct interaction). Here's a step-by-step approach:

Plan:
1. Refactor app.py:
   - Convert it into a proper Flask app with an API endpoint for chat.
   - Move the chatbot logic from main.py into a reusable module.
2. Create chatbot.py (new module):
   - Extract chatbot logic into a separate module.
   - Allow reuse for both API and CLI.
  

3. Update command_line.py:

    - Modify it to use chatbot.py while maintaining CLI functionality.


### 1. Refactor app.py for API
```
from flask import Flask, request, jsonify
from app.extras.utils import get_model
from app.chatbot import Chatbot

app = Flask(__name__)
chatbot = Chatbot("characters/c3po.character.json")  # Load chatbot with character context

@app.route("/chat", methods=["POST"])
def chat():
    data = request.get_json()
    user_message = data.get("message")
    
    if not user_message:
        return jsonify({"error": "No message provided"}), 400

    response = chatbot.get_chat_response(user_message)
    return jsonify({"response": response})

if __name__ == "__main__":
    app.run()
```

### 2. Create chatbot.py
   
Move the chatbot logic here so both the Flask app and CLI can use it.
```
import requests
import json
import os
from dotenv import load_dotenv
from app.extras.utils import get_model

class Chatbot:
    def __init__(self, character_file):
        load_dotenv()
        self.api_key = os.getenv("OPEN_ROUTER")
        self.model_name = get_model("deepseekv3")
        self.conversation_history = []
        self.load_character_context(character_file)

    def load_character_context(self, character_file):
        with open(character_file, "r") as f:
            character = json.load(f)
        
        context = f"You are {character['name']}. "
        context += " ".join(character.get('bio', []))
        context += " ".join(character.get('lore', []))
        context += " ".join(character.get('knowledge', []))
        context += " ".join(character.get('style', []))
        context += " ".join(character.get('adjectives', []))

        self.conversation_history.append({"role": "system", "content": context})

    def get_chat_response(self, message_content):
        url = "https://openrouter.ai/api/v1/chat/completions"
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json"
        }

        self.conversation_history.append({"role": "user", "content": message_content})

        payload = {
            "model": self.model_name,
            "messages": self.conversation_history,
            "max_tokens": 100
        }

        response = requests.post(url, headers=headers, json=payload)
        response_data = response.json()
        response_content = response_data['choices'][0]['message']['content']

        self.conversation_history.append({"role": "assistant", "content": response_content})
        return response_content
```
### 3. Update command_line.py
Modify it to use Chatbot while keeping CLI interaction.
```
from app.chatbot import Chatbot

def main():
    chatbot = Chatbot("characters/c3po.character.json")
    print("Chatbot is ready! Type 'exit' to quit.")

    while True:
        user_message = input("\nYou: ")
        if user_message.lower() == "exit":
            print("Goodbye!")
            break
        
        response = chatbot.get_chat_response(user_message)
        print("\nChatbot:", response)

if __name__ == "__main__":
    main()
```

## How It Works
✅ Flask API
- Run via Gunicorn (gunicorn -w 4 -b 0.0.0.0:8000 app:app).
- Accepts JSON requests at /chat, responding with chatbot replies.
  
✅ Command-line Chat
- Run python -m app.clients.command_line for direct interaction.

✅ Modular Design
- chatbot.py is now reusable across both Flask and CLI.


This setup keeps everything clean and scalable. 🚀 Let me know if you need tweaks!







