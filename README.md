# 📘 Dialogflow CX Flask Server – Complete Documentation  
---

## 🧩 1. Overview

This backend service acts as a middleware between any frontend client (web, mobile, chatbot UI) and **Google Dialogflow CX**.

Its responsibilities:

- Authenticate using Google Service Account (JSON key)
- Receive user messages from frontend via `/chat` endpoint
- Forward the **JWT token** (Authorization header) directly to Dialogflow CX
- Send the user input to Dialogflow CX
- Return the agent response with:
  - Detected intent  
  - Parameters  
  - Current page  
  - Session ID  
- Provide health monitoring through `/health`

This document explains **setup**, **workflow**, **architecture**, and **code structure**.

---

# 🏗 2. Architecture Diagram
```
Frontend Client (React, Angular, etc)
|
| POST /chat
| Authorization: Bearer <JWT>
v
| Flask Backend Middleware |
| - Validates JWT |
| - Creates Session ID |
| - Sends user message to Dialogflow CX |
| - Forwards Authorization header to Tools |
java
Copy code
    |
    | Google Service Account Auth (JSON key)
    v
| Dialogflow CX Agent |
| - Detect intent |
| - Execute pages/flows |
| - Call Tools / Webhooks |
markdown
Copy code
    |
    v
Backend returns formatted response JSON
```


---

# ⚙️ 3. Setup Instructions

Follow these steps to set up and run the Flask server.

### **Step 1: Install Dependencies**
```bash
pip install flask python-dotenv google-cloud-dialogflow-cx
Step 2: Add Your Google Service Account JSON
Place your Dialogflow CX service account key at:

C:\Users\AGL\Desktop\Kola_lg\dkey.json
Or update the path in the script.

Step 3: Set the GOOGLE_APPLICATION_CREDENTIALS
pgsql
Copy code
set GOOGLE_APPLICATION_CREDENTIALS=C:\Users\AGL\Desktop\Kola_lg\dkey.json
Step 4: Update Project Details
python
Copy code
PROJECT_ID = "adglobal-adh-project"
LOCATION = "global"
AGENT_ID = "cc748dc6-64b5-4cd6-9906-3e97918d2318"
LANGUAGE_CODE = "en"
Step 5: Run the Flask server
bash
Copy code
python app.py
Server will start on:

cpp
Copy code
http://127.0.0.1:5001
🧠 4. Code Walkthrough (Line-by-Line)
Below is the complete annotated code.

📌 4.1 Imports & Setup
python
Copy code
import os
import uuid
from flask import Flask, request, jsonify
from dotenv import load_dotenv
from google.cloud import dialogflowcx_v3beta1 as dialogflow
from google.api_core.exceptions import InvalidArgument
import json
What this does
os → environment variable handling

uuid → generate unique session IDs

flask → API server

dotenv → load .env variables

dialogflowcx → core Dialogflow CX SDK

📌 4.2 Load Environment
python
Copy code
load_dotenv()
app = Flask(__name__)
📌 4.3 Configuration
python
Copy code
PROJECT_ID = "adglobal-adh-project"
LOCATION = "global"
AGENT_ID = "cc748dc6-64b5-4cd6-9906-3e97918d2318"
LANGUAGE_CODE = "en"
📌 4.4 Authentication Setup
python
Copy code
CREDENTIALS_PATH = r"C:\Users\AGL\Desktop\Kola_lg\dkey.json"
if not os.path.exists(CREDENTIALS_PATH):
    print("CRITICAL ERROR: Credentials file not found")
    exit()

os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = CREDENTIALS_PATH
Purpose
Check if the credential file exists

Exit if missing

Export GOOGLE_APPLICATION_CREDENTIALS for Dialogflow SDK

🔗 5. Dialogflow CX Integration
The key function:

📌 5.1 detect_intent_with_jwt()
python
Copy code
def detect_intent_with_jwt(session_id: str, user_text: str, auth_header: str):
    session_path = (
        f"projects/{PROJECT_ID}/locations/{LOCATION}/agents/{AGENT_ID}/sessions/{session_id}"
    )

    client_options = None
    if LOCATION != "global":
        api_endpoint = f"{LOCATION}-dialogflow.googleapis.com:443"
        client_options = {"api_endpoint": api_endpoint}

    try:
        sessions_client = dialogflow.SessionsClient(client_options=client_options)

        query_input = dialogflow.QueryInput(
            text=dialogflow.TextInput(text=user_text),
            language_code=LANGUAGE_CODE,
        )

        # Forward JWT token
        query_params = dialogflow.QueryParameters(
            headers={'Authorization': auth_header}
        )

        request_payload = dialogflow.DetectIntentRequest(
            session=session_path,
            query_input=query_input,
            query_params=query_params,
        )

        response = sessions_client.detect_intent(request=request_payload)

    except InvalidArgument as e:
        return None, f"Configuration error: {str(e)}"
    except Exception as e:
        return None, f"Unexpected error: {str(e)}"

    # Extract response messages
    agent_responses = [
        text
        for msg in response.query_result.response_messages
        if msg.text
        for text in msg.text.text
    ]

    return response, " ".join(agent_responses)
🧱 6. Format Dialogflow Response
python
Copy code
def format_response(response_obj, agent_reply, session_id):
    parameters = {}
    if response_obj and response_obj.query_result.parameters:
        for key, value in response_obj.query_result.parameters.items():
            parameters[key] = value

    return {
        "agent_response": agent_reply,
        "session_id": session_id,
        "intent": response_obj.query_result.intent.display_name,
        "parameters": parameters,
        "current_page": response_obj.query_result.current_page.display_name,
    }
This makes the output clean and frontend-friendly.

🌐 7. API Endpoints
📌 7.1 /chat (POST)
Expected Input
json
Copy code
{
  "message": "hello",
  "session_id": "optional"
}
Required Header
makefile
Copy code
Authorization: Bearer <JWT>
Logic
Validate request

Validate Authorization header

Generate new session if not provided

Send request to Dialogflow

Return formatted response

Possible Errors
Error	Meaning
401 Missing token	JWT not provided
400 Missing message	message field missing
500 DF error	Wrong agent/project or credentials

📌 7.2 /health (GET)
json
Copy code
{ "status": "healthy" }
Used for:

Uptime checks

Cloud Run

Kubernetes probes

🔍 8. Sample Request & Response
Request
bash
Copy code
curl -X POST http://localhost:5001/chat \
-H "Authorization: Bearer xyz123" \
-H "Content-Type: application/json" \
-d '{"message": "Hi"}'
Response
json
Copy code
{
  "agent_response": "Hello! How can I help?",
  "session_id": "7894f8ab-bc22-4b7e-93a9-8a30037b18ea",
  "intent": "Welcome Intent",
  "parameters": {},
  "current_page": "Start Page"
}
🛠 9. Troubleshooting
❌ InvalidArgument: Agent not found
✔ Check PROJECT_ID, AGENT_ID, LOCATION

❌ Webhook/Tools not receiving JWT
✔ Ensure this code exists:

python
Copy code
query_params = dialogflow.QueryParameters(
    headers={'Authorization': auth_header}
)
❌ Permission denied
✔ Ensure service account has:

Dialogflow API Admin

Service Account Token Creator

📦 10. Deployment Guide
Local Development
bash
Copy code
python app.py
Docker (optional)
I can provide the Dockerfile if needed.

Cloud Run / Kubernetes
Add health checks for /health.

🎯 Final Notes
This backend uses best practices for Dialogflow CX:

Stateless sessions

JWT passthrough

Clean API design

Separated formatting logic

✅ Need more?
Tell me if you want:

DOCX version

PDF version

README.md optimized for GitHub

Architecture diagram (PNG)

Postman Collection

Dockerfile
```







