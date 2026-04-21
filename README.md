# AutoStream Conversational AI Agent
### ServiceHive Inflx Assignment — Social-to-Lead Agentic Workflow

---

## What This Is
A conversational AI agent for AutoStream, a fictional SaaS platform for video creators.
The agent understands user intent, answers product questions from a knowledge base,
detects high-intent leads, and captures them via a mock CRM tool.

---

## How to Run Locally

### 1. Install Ollama
```bash
curl -fsSL https://ollama.ai/install.sh | sh
ollama serve &
ollama pull llama3.2
```

### 2. Install Python Dependencies
```bash
pip install -r requirements.txt
```

### 3. Run the Agent
```bash
python agent.py
```

---

## Architecture Explanation

This agent is built using **LangGraph**, a stateful graph framework built on top of LangChain.
I chose LangGraph over AutoGen because it gives explicit, fine-grained control over conversation
flow and state transitions — which is critical for a lead capture workflow where the tool must
not fire prematurely.

The agent is structured as a single-node StateGraph with a conditional routing function.
Each conversation turn passes through the `chat_node`, which reads the full message history,
classifies intent, calls the LLM, and writes back to the shared state. The state is defined
as a TypedDict containing the full message history, detected intent, lead fields (name, email,
platform), a `lead_captured` boolean flag, and a turn counter.

The `lead_captured` flag is the key safety mechanism — it ensures `mock_lead_capture()` fires
exactly once and only after all three lead fields are confirmed. Memory is maintained across
turns by passing the full conversation history to the LLM on every call, giving the agent
genuine multi-turn context without an external vector store.

The RAG pipeline retrieves answers directly from a local `knowledge_base.json` file. Pricing
questions are matched via keyword detection and answered deterministically from the knowledge
base, ensuring accuracy. The LLM handles intent reasoning and lead collection, where
natural language understanding is actually needed.

---

## WhatsApp Deployment via Webhooks

To deploy this agent on WhatsApp using the Meta Cloud API:

1. **Register a Meta Developer App** at developers.facebook.com and enable the
   WhatsApp Business API for your app.

2. **Set up a webhook endpoint** using FastAPI or Flask:
```python
   @app.post("/webhook")
   async def webhook(request: Request):
       data = await request.json()
       phone = data["entry"][0]["changes"][0]["value"]["messages"][0]["from"]
       message = data["entry"][0]["changes"][0]["value"]["messages"][0]["text"]["body"]
       response = run_agent(phone, message)
       send_whatsapp_message(phone, response)
       return {"status": "ok"}
```

3. **Key per user by phone number** — store each user's LangGraph state in Redis
   or a lightweight database, keyed by their WhatsApp phone number. This ensures
   each conversation is independent and persistent across sessions.

4. **Send replies** via POST to:
   `https://graph.facebook.com/v18.0/{phone_number_id}/messages`
   with the agent's response as the message body.

5. **Deploy** the webhook on any cloud server (Railway, Render, AWS EC2) with a
   public HTTPS URL that Meta can reach.

This architecture scales naturally — each incoming WhatsApp message triggers the webhook,
loads that user's state, runs it through the LangGraph agent, and sends the reply back.
The agent logic requires zero changes for WhatsApp deployment.

---

## Knowledge Base
Stored in `knowledge_base.json`:
- Basic Plan: $29/month, 10 videos, 720p
- Pro Plan: $79/month, unlimited videos, 4K, AI captions, 24/7 support
- No refunds after 7 days

---

## Tech Stack
- **Language**: Python 3.9+
- **Framework**: LangChain + LangGraph
- **LLM**: Llama 3.2 via Ollama (local)
- **State Management**: LangGraph TypedDict StateGraph
- **RAG**: Local JSON knowledge base with keyword retrieval

---

## Evaluation Checklist
- ✅ Intent detection — greeting / inquiry / high_intent
- ✅ RAG knowledge retrieval from local JSON
- ✅ Lead capture tool fires only after all 3 fields collected
- ✅ State retained across 5-6 conversation turns
- ✅ LangGraph StateGraph with conditional routing
