<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>ATC Comm & Coord — Demo</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <style>
    body { font-family: system-ui, -apple-system, "Segoe UI", Roboto, Arial; margin: 1rem; }
    .container { max-width: 920px; margin: auto; }
    textarea { width:100%; height:80px; }
    .log { border:1px solid #ddd; padding:12px; height:300px; overflow:auto; background:#f9f9f9; }
    .row { display:flex; gap:8px; margin-top:8px; }
    button { padding:8px 12px; border-radius:6px; cursor:pointer; }
    .small { font-size:0.9rem; color:#666; }
  </style>
</head>
<body>
  <div class="container">
    <h1>AI-Powered Comm & Coordination (Demo)</h1>
    <p class="small">Simple demo UI. Backend REST endpoints: <code>/api/message</code>, <code>/api/flight</code>, <code>/api/staff</code>, <code>/api/notifications</code></p>

    <label><strong>Operator message / request</strong></label>
    <textarea id="msgText" placeholder="e.g. Tower reports overflow at stand B, possible delay..."></textarea>
    <div class="row">
      <button id="sendBtn">Send message</button>
      <button id="flightBtn">Ingest flight (sample)</button>
      <button id="staffBtn">Ingest staff (sample)</button>
      <button id="pollBtn">Poll notifications</button>
    </div>

    <h3>Activity log</h3>
    <div id="log" class="log"></div>
  </div>

  <script>
    const apiBase = location.hostname === 'localhost' ? 'http://localhost:5000' : '';

    function log(x) {
      const box = document.getElementById('log');
      const el = document.createElement('div');
      el.style.marginBottom = '8px';
      el.textContent = `[${new Date().toLocaleTimeString()}] ${x}`;
      box.prepend(el);
    }

    async function postMessage() {
      const text = document.getElementById('msgText').value.trim();
      if (!text) { alert('Type a message'); return; }
      const payload = { from: 'ops@airport', to: 'system', text: text, context: { location: 'TWR' } };
      const res = await fetch(apiBase + '/api/message', {
        method: 'POST',
        headers: {'Content-Type':'application/json'},
        body: JSON.stringify(payload)
      });
      const data = await res.json();
      if (res.ok) {
        log('Sent message: ' + text);
        log('AI reply: ' + (data.reply && data.reply.text ? data.reply.text : JSON.stringify(data.reply)));
      } else {
        log('Error: ' + JSON.stringify(data));
      }
    }

    async function ingestFlight() {
      const payload = {
        flight_id: 'AI' + Math.floor(100 + Math.random()*900),
        status: 'arriving',
        eta: new Date(Date.now() + 15*60000).toISOString(),
        slot: 'A' + Math.floor(1 + Math.random()*40),
        notes: 'simulated'
      };
      const res = await fetch(apiBase + '/api/flight', {
        method: 'POST',
        headers: {'Content-Type':'application/json'},
        body: JSON.stringify(payload)
      });
      const data = await res.json();
      if(res.ok) {
        log('Ingested flight: ' + payload.flight_id + ' slot=' + payload.slot);
      } else {
        log('Flight ingest error: ' + JSON.stringify(data));
      }
    }

    async function ingestStaff() {
      const payload = {
        staff_id: 'CTR' + Math.floor(1 + Math.random()*99),
        role: 'controller',
        available_from: new Date().toISOString()
      };
      const res = await fetch(apiBase + '/api/staff', {
        method: 'POST',
        headers: {'Content-Type':'application/json'},
        body: JSON.stringify(payload)
      });
      const data = await res.json();
      if(res.ok) {
        log('Ingested staff: ' + payload.staff_id);
      } else {
        log('Staff ingest error: ' + JSON.stringify(data));
      }
    }

    async function pollNotifications() {
      const res = await fetch(apiBase + '/api/notifications?max=10');
      const data = await res.json();
      if(res.ok) {
        if((data.notifications||[]).length === 0) log('No notifications');
        (data.notifications || []).forEach(n => {
          log('Notification: ' + JSON.stringify(n));
        });
      } else {
        log('Poll error: ' + JSON.stringify(data));
      }
    }

    document.getElementById('sendBtn').addEventListener('click', postMessage);
    document.getElementById('flightBtn').addEventListener('click', ingestFlight);
    document.getElementById('staffBtn').addEventListener('click', ingestStaff);
    document.getElementById('pollBtn').addEventListener('click', pollNotifications);

    // optional: poll every 8 seconds to show live updates (toggle if you like)
    setInterval(async () => {
      try { await pollNotifications(); } catch(e) {}
    }, 8000);
  </script>
</body>
# app.py
import os
import time
import queue
import threading
from flask import Flask, request, jsonify
from flask_cors import CORS

# Optional OpenAI integration (if you set OPENAI_API_KEY env var)
USE_OPENAI = bool(os.environ.get("OPENAI_API_KEY"))
if USE_OPENAI:
    import openai
    openai.api_key = os.environ["OPENAI_API_KEY"]

app = Flask(__name__)
CORS(app)

# Simple in-memory stores (replace with DB for production)
FLIGHTS = {}   # flight_id -> info
STAFF = {}     # staff_id -> info
MESSAGES = []  # list of messages (dict)

# A simple thread-safe queue used to simulate notifications to frontend (polling or SSE could read)
notify_queue = queue.Queue()

def ai_generate_response(prompt: str) -> str:
    """
    Simple AI stub:
    - If OPENAI_API_KEY is provided, calls OpenAI's ChatCompletion (gpt-4/3.5) to get a response.
    - Otherwise returns a deterministic heuristic/mock response.
    """
    if USE_OPENAI:
        # Minimal call; adapt model & parameters per your quotas and safety rules
        resp = openai.ChatCompletion.create(
            model="gpt-4o-mini" if "gpt-4o-mini" in openai.Model.list() else "gpt-3.5-turbo",
            messages=[{"role":"user","content": prompt}],
            max_tokens=300,
            temperature=0.2
        )
        # navigate response
        return resp["choices"][0]["message"]["content"].strip()
    else:
        # Mock logic for offline demo: short rules-based reply
        prompt_low = prompt.lower()
        if "delay" in prompt_low:
            return "Detected potential delay. Recommend reassigning runway slot and notifying airline Ops."
        if "staff" in prompt_low or "controller" in prompt_low:
            return "Staffing suggestion: request one additional controller for the next 2-hour peak window."
        # default echo
        return f"[AI-mock] Received: {prompt[:200]}"

@app.route("/api/health", methods=["GET"])
def health():
    return jsonify({"status":"ok", "time": time.time()})

@app.route("/api/flight", methods=["POST"])
def ingest_flight():
    """
    Example payload:
    {
      "flight_id": "AI123",
      "status": "arriving",
      "eta": "2025-10-28T14:30:00Z",
      "slot": "A12",
      "notes": "delayed 15m"
    }
    """
    data = request.get_json()
    if not data or "flight_id" not in data:
        return jsonify({"error":"flight_id required"}), 400
    FLIGHTS[data["flight_id"]] = data
    notify_queue.put({"type":"flight_update","flight_id": data["flight_id"], "data": data})
    return jsonify({"ok": True, "flight": data})

@app.route("/api/staff", methods=["POST"])
def ingest_staff():
    """
    Example payload:
    {
      "staff_id": "CTR001",
      "role": "controller",
      "available_from": "2025-10-28T08:00:00Z"
    }
    """
    data = request.get_json()
    if not data or "staff_id" not in data:
        return jsonify({"error":"staff_id required"}), 400
    STAFF[data["staff_id"]] = data
    notify_queue.put({"type":"staff_update","staff_id": data["staff_id"], "data": data})
    return jsonify({"ok": True, "staff": data})

@app.route("/api/message", methods=["POST"])
def post_message():
    """
    Operator posts a message to the system, e.g.
    { "from": "ops@airport", "to": "airline_ops", "text": "We may need 1 extra tower controller", "context": {...} }
    The backend runs AI to propose an action or reply.
    """
    payload = request.get_json() or {}
    text = payload.get("text","")
    if not text:
        return jsonify({"error":"text required"}), 400

    # store message
    msg = {
        "id": len(MESSAGES)+1,
        "from": payload.get("from","operator"),
        "to": payload.get("to","system"),
        "text": text,
        "timestamp": time.time(),
        "context": payload.get("context", {})
    }
    MESSAGES.append(msg)

    # run AI (synchronously here; for heavy loads, run async + job queue)
    prompt = f"Operator message: {text}\nContext: {msg['context']}\nProvide short actionable recommendation (1-2 lines)."
    ai_reply = ai_generate_response(prompt)
    # create reply object
    reply = {
        "id": f"r{msg['id']}",
        "in_reply_to": msg["id"],
        "text": ai_reply,
        "timestamp": time.time()
    }

    # store and notify
    MESSAGES.append(reply)
    notify_queue.put({"type":"message", "message": reply, "original": msg})
    return jsonify({"ok": True, "message": msg, "reply": reply})

@app.route("/api/messages", methods=["GET"])
def get_messages():
    # basic pagination
    since = float(request.args.get("since", 0))
    results = [m for m in MESSAGES if m.get("timestamp",0) > since]
    return jsonify({"messages": results})

@app.route("/api/notifications", methods=["GET"])
def poll_notifications():
    """
    Simple long-polling endpoint: caller can call frequently to get queued notifications.
    Returns up to N notifications currently waiting.
    """
    max_items = int(request.args.get("max", 10))
    items = []
    for _ in range(max_items):
        try:
            item = notify_queue.get_nowait()
            items.append(item)
        except queue.Empty:
            break
    return jsonify({"notifications": items})

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
# ATC Comm & Coordination — Demo

## Setup (local)
1. Create a virtualenv:
   python -m venv venv
   source venv/bin/activate   # Windows: venv\Scripts\activate

2. Install:
   pip install -r requirements.txt

3. (Optional) Set OpenAI key to enable real AI responses:
   export OPENAI_API_KEY="sk-..."
   # Windows (Powershell): $env:OPENAI_API_KEY="sk-..."

4. Run backend:
   python backend/app.py
   # or from repo root if file is at root

5. Open `frontend/index.html` in your browser (or serve it from a static server). 
   If you open the file directly, ensure the backend is at http://localhost:5000 (CORS enabled).

</html>

