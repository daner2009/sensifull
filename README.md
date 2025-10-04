# app.py
"""
SensifyFF - Web todo en 1 archivo (Flask)
----------------------------------------
Instrucciones rápidas:
1) Crear entorno e instalar:
   python -m venv venv
   # Linux / mac:
   source venv/bin/activate
   # Windows:
   # venv\Scripts\activate
   pip install flask requests python-dotenv flask-cors

   (Opcional para IA premium)
   pip install openai

2) Crear archivo .env (ejemplo abajo) o exportar variables:
   FLASK_SECRET_KEY="cambia_esto"
   ADMIN_TOKEN="token_admin_seguro"
   DEFAULT_NEQUI_NUMBER="+573502113660"   # tu número Nequi (tu mamá)
   YOUTUBE_API_KEY=""      # opcional (mejora IA básica)
   OPENAI_API_KEY=""       # opcional (IA premium)
   OPENAI_MODEL="gpt-4o-mini" # opcional

3) Ejecutar:
   python app.py
   Abrir: http://localhost:5000

4) Para desplegar en Railway/Render: sube este archivo a un repo, configura variables de entorno en la plataforma, y pon el comando de inicio: `python app.py` o usar Gunicorn.

IMPORTANTE:
- Por seguridad usa cuenta adulta (tu mamá) para retirar pagos de AdSense/AdMob.
- Nequi en este proyecto es manual: usuario sube comprobante y admin aprueba.
"""
import os, json, time, sqlite3, logging, re, random
from datetime import datetime
from pathlib import Path
from flask import Flask, request, jsonify, render_template_string, redirect, url_for, send_from_directory, session
from flask_cors import CORS
from werkzeug.utils import secure_filename

# Optional OpenAI
try:
    import openai
except Exception:
    openai = None

# Load .env if present
try:
    from dotenv import load_dotenv
    load_dotenv()
except Exception:
    pass

# --- Config ---
APP_ROOT = Path(__file__).parent
UPLOADS = APP_ROOT / "uploads"
UPLOADS.mkdir(exist_ok=True)

DB_FILE = APP_ROOT / "sensifyff.db"
ALLOWED_EXT = {"png", "jpg", "jpeg", "webp"}

FLASK_SECRET_KEY = os.environ.get("FLASK_SECRET_KEY", "sensifyff-dev-secret")
ADMIN_TOKEN = os.environ.get("ADMIN_TOKEN", "admin-token-change-me")
DEFAULT_NEQUI_NUMBER = os.environ.get("DEFAULT_NEQUI_NUMBER", "+573502113660")
YOUTUBE_API_KEY = os.environ.get("YOUTUBE_API_KEY", "")
OPENAI_API_KEY = os.environ.get("OPENAI_API_KEY", "")
OPENAI_MODEL = os.environ.get("OPENAI_MODEL", "gpt-4o-mini")

if openai and OPENAI_API_KEY:
    openai.api_key = OPENAI_API_KEY

# Flask init
app = Flask(__name__)
app.secret_key = FLASK_SECRET_KEY
CORS(app)
app.config['MAX_CONTENT_LENGTH'] = 6 * 1024 * 1024

# Logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(message)s')

# --- DB helpers ---
def init_db():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute('''
      CREATE TABLE IF NOT EXISTS premium_users (
         email TEXT PRIMARY KEY,
         active INTEGER DEFAULT 0,
         source TEXT,
         nequi_file TEXT,
         created_at TEXT
      );
    ''')
    c.execute('''
      CREATE TABLE IF NOT EXISTS nequi_files (
         id INTEGER PRIMARY KEY AUTOINCREMENT,
         email TEXT,
         filename TEXT,
         uploaded_at TEXT,
         approved INTEGER DEFAULT 0
      );
    ''')
    conn.commit()
    conn.close()

def mark_premium(email, source="nequi_admin"):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    now = datetime.utcnow().isoformat()
    c.execute('INSERT OR REPLACE INTO premium_users (email, active, source, created_at) VALUES (?, ?, ?, ?)', (email, 1, source, now))
    conn.commit(); conn.close()
    logging.info("Usuario marcado como premium: %s", email)

def set_nequi_pending(email, filename):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    now = datetime.utcnow().isoformat()
    c.execute('INSERT INTO nequi_files (email, filename, uploaded_at, approved) VALUES (?, ?, ?, 0)', (email, filename, now))
    conn.commit(); conn.close()
    logging.info("Nequi comprobante subido para %s -> %s", email, filename)

def get_pending_nequi():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute('SELECT id, email, filename, uploaded_at, approved FROM nequi_files WHERE approved=0')
    rows = c.fetchall(); conn.close()
    return [{"id":r[0],"email":r[1],"filename":r[2],"uploaded_at":r[3],"approved":r[4]} for r in rows]

def approve_nequi(file_id):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute('SELECT email, filename FROM nequi_files WHERE id=?', (file_id,))
    row = c.fetchone()
    if not row:
        conn.close()
        return False
    email, fname = row
    c.execute('UPDATE nequi_files SET approved=1 WHERE id=?', (file_id,))
    c.execute('INSERT OR REPLACE INTO premium_users (email, active, source, nequi_file, created_at) VALUES (?, ?, ?, ?, ?)', (email, 1, "nequi_admin", fname, datetime.utcnow().isoformat()))
    conn.commit(); conn.close()
    logging.info("Aprobado comprobante nequi %s -> %s", file_id, email)
    return True

init_db()

# --- Device DB (modelos más usados) ---
DEVICE_DB = [
    {"brand":"Xiaomi","model":"Redmi A10","tier":"baja"},
    {"brand":"Xiaomi","model":"Redmi 9","tier":"media"},
    {"brand":"Xiaomi","model":"Poco X3","tier":"alta"},
    {"brand":"Samsung","model":"Galaxy A12","tier":"baja"},
    {"brand":"Samsung","model":"Galaxy A20","tier":"media"},
    {"brand":"Samsung","model":"Galaxy S21","tier":"alta"},
    {"brand":"Apple","model":"iPhone 7","tier":"media"},
    {"brand":"Apple","model":"iPhone 11","tier":"alta"},
    {"brand":"Motorola","model":"Moto G20","tier":"baja"},
    {"brand":"Motorola","model":"Moto G60","tier":"media"},
    {"brand":"Realme","model":"Realme 7","tier":"media"},
    {"brand":"Infinix","model":"Hot 10","tier":"baja"},
    {"brand":"Tecno","model":"Spark 6","tier":"baja"},
    {"brand":"Vivo","model":"Y20","tier":"baja"},
    {"brand":"Oppo","model":"A54","tier":"media"},
]

# --- Utilities & AI helpers ---
def allowed_file(filename):
    return '.' in filename and filename.rsplit('.',1)[1].lower() in ALLOWED_EXT

def heuristic_by_tier(tier):
    # produce values in range 0..200 (we'll display 0..200)
    if tier == "alta":
        return {
            "general": random.randint(120, 180),
            "punto_rojo": random.randint(110,170),
            "mira_2x": random.randint(100,160),
            "mira_4x": random.randint(90,150),
            "francotirador": random.randint(90,150),
            "camara_360": random.randint(120,200),
            "boton_disparo": random.randint(60,140),
            "dpi": random.choice([360,400,480])
        }
    if tier == "media":
        return {
            "general": random.randint(90, 150),
            "punto_rojo": random.randint(80,140),
            "mira_2x": random.randint(70,130),
            "mira_4x": random.randint(60,120),
            "francotirador": random.randint(60,120),
            "camara_360": random.randint(90,160),
            "boton_disparo": random.randint(50,120),
            "dpi": random.choice([320,360,400])
        }
    return {
        "general": random.randint(60,120),
        "punto_rojo": random.randint(50,110),
        "mira_2x": random.randint(45,100),
        "mira_4x": random.randint(40,95),
        "francotirador": random.randint(40,95),
        "camara_360": random.randint(60,140),
        "boton_disparo": random.randint(40,100),
        "dpi": random.choice([300,320,360])
    }

def youtube_search_titles(q, maxr=6):
    if not YOUTUBE_API_KEY:
        return []
    try:
        url = "https://www.googleapis.com/youtube/v3/search"
        r = requests.get(url, params={"part":"snippet","q":q,"key":YOUTUBE_API_KEY,"maxResults":maxr,"type":"video"}, timeout=8)
        r.raise_for_status()
        items = r.json().get("items",[])
        out = []
        for it in items:
            sn = it.get("snippet",{})
            vid = it.get("id",{}).get("videoId")
            out.append({"title":sn.get("title"), "description":sn.get("description"), "link": ("https://youtube.com/watch?v="+vid) if vid else None})
        return out
    except Exception as e:
        logging.warning("YouTube API error: %s", e)
        return []

def extract_sensitivity_from_texts(items):
    cand = []
    pattern = re.compile(r"(\b(?:sensibilidad|sensi|general|dpi|red dot|punto rojo|mira)\b).{0,30}(\d{1,4})", re.I)
    for it in items:
        text = (it.get("title","") + " " + it.get("description",""))[:1200]
        for m in pattern.finditer(text):
            num = m.group(2)
            try:
                n = int(num)
                if 0 <= n <= 2000:
                    cand.append(n)
            except: pass
    if not cand:
        return None
    cand_sorted = sorted(cand)
    mid = cand_sorted[len(cand_sorted)//2]
    return {
        "general": max(0,min(200, mid)),
        "punto_rojo": max(0,min(200, mid-10)),
        "mira_2x": max(0,min(200, mid-15)),
        "mira_4x": max(0,min(200, mid-25)),
        "francotirador": max(0,min(200, mid-20)),
        "camara_360": max(0,min(200, mid+20)),
        "boton_disparo": max(0,min(200, mid-40)),
        "dpi": 400 if mid >= 150 else 320
    }

def ai_basic_recommendation(q, device=None):
    hits = youtube_search_titles(q + (" " + device if device else ""), maxr=6)
    extracted = extract_sensitivity_from_texts(hits)
    if extracted:
        return {"source":"youtube", "hits":hits, "suggestion": extracted}
    # fallback heuristics
    d = next((x for x in DEVICE_DB if device and x["model"].lower()==(device or "").lower()), None)
    tier = d["tier"] if d else "media"
    return {"source":"heuristic", "hits":[], "suggestion": heuristic_by_tier(tier)}

def ai_premium_generate(q, device=None, email=None):
    hits = youtube_search_titles(q + (" " + device if device else ""), maxr=8)
    if openai and OPENAI_API_KEY:
        prompt = f"Eres experto en Free Fire 2025. Con las fuentes listadas genera un JSON con 'values' (general,punto_rojo,mira_2x,mira_4x,francotirador,camara_360,boton_disparo,dpi), 'steps' (paso a paso), 'tips' y 'sources'. Fuentes:\n"
        for h in hits[:6]:
            prompt += f"- {h.get('title')} {h.get('link')}\n"
        try:
            resp = openai.ChatCompletion.create(
                model=OPENAI_MODEL,
                messages=[{"role":"user","content":prompt}],
                max_tokens=800, temperature=0.2
            )
            text = resp['choices'][0]['message']['content']
            try:
                parsed = json.loads(text)
                return {"ok":True, "parsed":parsed, "hits":hits}
            except:
                return {"ok":True, "raw": text, "hits":hits}
        except Exception as e:
            return {"error": str(e), "hits": hits}
    # fallback
    base = ai_basic_recommendation(q, device)["suggestion"]
    steps = [
        "Abre Free Fire -> Ajustes -> Sensibilidad.",
        f"Coloca General = {base['general']}.",
        f"Punto rojo = {base['punto_rojo']}.",
        f"Mira 2x = {base['mira_2x']}.",
        f"Mira 4x = {base['mira_4x']}.",
        f"Francotirador = {base['francotirador']}.",
        f"Botón cámara 360 sugerido = {base['camara_360']}.",
        f"Botón disparo = {base['boton_disparo']}.",
        f"Prueba DPI sugerido = {base['dpi']}.",
        "Haz pruebas en modo entrenamiento y ajusta +/- 5 hasta sentir control."
    ]
    tips = ["Si sientes lag reduce gráficos y baja 2-5 puntos la mira.", "Cierra apps en segundo plano."]
    return {"ok":True, "parsed": {"values": base, "steps": steps, "tips": tips, "sources": [h.get('link') for h in hits]}, "hits":hits}

# --- HTML template (oscuro gamer, en español) ---
INDEX_HTML = r"""
<!doctype html>
<html lang="es">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>SensifyFF — Generador IA Sensibilidad</title>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;500;700;900&display=swap" rel="stylesheet">
<style>
:root{--bg1:#06060a;--bg2:#12101b;--accent1:#ff3b3b;--accent2:#7c3cff;--glass:rgba(255,255,255,0.03);--muted:rgba(255,255,255,0.7)}
*{box-sizing:border-box}body{margin:0;font-family:Inter,system-ui,Arial;background:linear-gradient(135deg,var(--bg1),var(--bg2));color:#f3f6ff;padding:22px}
.wrap{max-width:1200px;margin:0 auto;display:grid;grid-template-columns:1fr 380px;gap:18px}
header{grid-column:1/-1;display:flex;align-items:center;gap:14px;padding:14px;border-radius:12px;background:linear-gradient(90deg,rgba(255,255,255,0.02),transparent);border:1px solid rgba(255,255,255,0.03)}
.logo{width:80px;height:80px;border-radius:12px;background:linear-gradient(90deg,var(--accent1),var(--accent2));display:flex;align-items:center;justify-content:center;font-weight:900;color:#111;font-size:28px;box-shadow:0 12px 30px rgba(0,0,0,0.6)}
.panel{background:var(--glass);padding:16px;border-radius:12px;border:1px solid rgba(255,255,255,0.02)}
.card{background:var(--glass);padding:12px;border-radius:12px;border:1px solid rgba(255,255,255,0.02)}
.controls{display:flex;gap:8px;flex-wrap:wrap}
input,select,textarea,button{border-radius:10px;padding:10px;border:none;font-size:14px;outline:none}
input,select,textarea{background:rgba(255,255,255,0.02);color:#fff;border:1px solid rgba(255,255,255,0.03)}
.btn{background:linear-gradient(90deg,var(--accent1),var(--accent2));color:#111;padding:10px 14px;border-radius:10px;cursor:pointer}
.btn.ghost{background:transparent;border:1px solid rgba(255,255,255,0.06);color:var(--muted)}
.sens-box{margin-top:12px;padding:14px;border-radius:10px;background:linear-gradient(180deg,rgba(255,255,255,0.02),transparent);border:1px solid rgba(255,255,255,0.02)}
.sens-grid{display:grid;grid-template-columns:repeat(2,1fr);gap:8px}
.small{font-size:13px;color:var(--muted)}
.ai-card{margin-top:12px;padding:12px;border-radius:10px;background:linear-gradient(90deg,rgba(255,255,255,0.02),transparent);border:1px solid rgba(255,255,255,0.02)}
#aiChat{height:220px;overflow:auto;padding:8px;border-radius:8px;background:rgba(0,0,0,0.12)}
.ad-box{margin-top:12px;padding:10px;border-radius:8px;background:rgba(0,0,0,0.18);text-align:center}
footer{grid-column:1/-1;text-align:center;color:var(--muted);font-size:13px;padding-top:8px}
@media (max-width:980px){.wrap{grid-template-columns:1fr}}
</style>
</head>
<body>
<div class="wrap">
  <header>
    <div class="logo">S🔥</div>
    <div>
      <h1 style="margin:0">SensifyFF</h1>
      <div class="small">Generador IA de Sensibilidades • Básico & Premium</div>
    </div>
    <div style="margin-left:auto;text-align:right">
      <div class="small">Nequi (Colombia): <strong id="nequiNumber">{{ nequi_number }}</strong></div>
      <div class="small">Admin token: usa header X-ADMIN-TOKEN para endpoints admin</div>
    </div>
  </header>

  <main class="panel">
    <h2>🎮 Generador IA de Sensibilidades</h2>
    <div class="controls" style="margin-top:8px">
      <input id="deviceInput" placeholder="Modelo (ej: Redmi 9)" style="width:300px">
      <select id="modeSelect">
        <option value="clasico">Clásico</option>
        <option value="ranked">Ranked</option>
        <option value="duelo">Duelo 1v1</option>
      </select>
      <button class="btn" onclick="generateBasic()">Generar (Básico)</button>
      <button class="btn" onclick="generatePremium()">Generar (Premium)</button>
    </div>

    <div id="resultBox" class="sens-box">
      <div style="display:flex;justify-content:space-between;align-items:center">
        <div><strong>Resultados</strong></div>
        <div class="small">Versión: <span id="versionTag">Gratis</span></div>
      </div>

      <div id="sensGrid" class="sens-grid" style="margin-top:12px"></div>

      <div class="ai-card">
        <h4>Chat / Consejos</h4>
        <div id="aiChat" class="small">Consejos y pasos aparecerán aquí.</div>
        <div style="display:flex;gap:8px;margin-top:8px">
          <input id="aiInput" placeholder="Pregunta a la IA (Premium)" style="flex:1">
          <button class="btn ghost" onclick="sendChat()">Enviar</button>
        </div>
      </div>
    </div>

    <div style="margin-top:14px" class="ai-card">
      <h4>Pago Premium — Nequi (Colombia)</h4>
      <p class="small">Paga al número indicado y sube el comprobante. El admin revisa y activa.</p>
      <div style="display:flex;gap:8px;align-items:center">
        <button class="btn" onclick="openWhatsApp()">Enviar comprobante por WhatsApp</button>
        <div style="flex:1">
          <input id="nequiEmail" placeholder="Tu correo para activar (ej: correo@ejemplo.com)" style="width:100%;margin-top:6px">
          <input id="nequiFile" type="file" style="margin-top:8px">
          <div style="display:flex;gap:8px;margin-top:8px">
            <button class="btn ghost" onclick="uploadNequi()">Subir comprobante</button>
            <div id="nequiMsg" class="small"></div>
          </div>
        </div>
      </div>
    </div>

  </main>

  <aside class="card">
    <h4>Guía rápida</h4>
    <p class="small">Detecta tu dispositivo o búscalo manualmente. La versión básica muestra recomendaciones; la premium (si configuras OpenAI) da pasos y chat interactivo.</p>

    <div style="margin-top:12px">
      <button class="btn ghost" onclick="detectMyDevice()">Detectar mi dispositivo</button>
      <div id="detected" class="small" style="margin-top:8px"></div>
    </div>

    <div class="ad-box">
      <!-- Lugar para AdSense: pega tu ca-pub-xxxx aquí -->
      <div class="small">Anuncio (AdSense) - pega tu código cuando tengas cuenta</div>
    </div>

    <div style="margin-top:12px">
      <h4 class="small">Admin</h4>
      <p class="small">Verifica comprobantes: GET /admin/pending-nequi (header X-ADMIN-TOKEN)</p>
      <p class="small">Aprobar: POST /admin/approve-nequi { "id": 123 } con X-ADMIN-TOKEN</p>
    </div>
  </aside>

  <footer>© SensifyFF — Hecho para probar • Usa cuenta adulta para pagos y AdSense</footer>
</div>

<script>
function setSensGrid(vals){
  const box = document.getElementById("sensGrid"); box.innerHTML="";
  const items=[["General","general"],["Punto Rojo","punto_rojo"],["Mira 2x","mira_2x"],["Mira 4x","mira_4x"],["Francotirador","francotirador"],["Cam 360","camara_360"],["Botón disparo","boton_disparo"],["DPI","dpi"]];
  items.forEach(([label,key])=>{
    const val = vals && vals[key] !== undefined ? vals[key] : "-";
    const el = document.createElement("div");
    el.style.background="rgba(0,0,0,0.12)";
    el.style.padding="10px";
    el.style.borderRadius="8px";
    el.innerHTML = `<b>${label}</b><div style="margin-top:6px;font-size:16px">${val}</div>`;
    box.appendChild(el);
  });
}

async function generateBasic(){
  const device = document.getElementById("deviceInput").value;
  document.getElementById("versionTag").innerText = "Básico";
  const q = "mejores sensibilidades free fire 2025";
  const res = await fetch("/ai/basic?q="+encodeURIComponent(q)+"&device="+encodeURIComponent(device));
  const j = await res.json();
  if(j.suggestion){ setSensGrid(j.suggestion); document.getElementById("aiChat").innerText = "Recomendación básica • Fuente: " + (j.source || "heurística"); }
  else { setSensGrid(null); document.getElementById("aiChat").innerText = "No se obtuvo sugerencia."; }
}

async function generatePremium(){
  const device = document.getElementById("deviceInput").value;
  document.getElementById("versionTag").innerText = "Premium";
  const body = { query: "mejores sensibilidades free fire 2025 "+(device||""), device: device, email: localStorage.getItem("sensifyff_email") || null };
  const r = await fetch("/ai/premium", { method:"POST", headers:{"Content-Type":"application/json"}, body: JSON.stringify(body) });
  const j = await r.json();
  if(j.ok && j.parsed && j.parsed.values){
    setSensGrid(j.parsed.values);
    document.getElementById("aiChat").innerText = (j.parsed.steps || []).slice(0,6).join("\n");
  } else if(j.parsed && j.parsed.values){
    setSensGrid(j.parsed.values);
    document.getElementById("aiChat").innerText = "Premium (fallback de IA).";
  } else {
    document.getElementById("aiChat").innerText = JSON.stringify(j);
  }
}

async function sendChat(){
  const q = document.getElementById("aiInput").value.trim();
  if(!q) return;
  document.getElementById("aiChat").innerText = "Enviando pregunta a IA...";
  const body = { query: q, device: document.getElementById("deviceInput").value || null, email: localStorage.getItem("sensifyff_email") || null };
  const r = await fetch("/ai/premium", { method:"POST", headers:{"Content-Type":"application/json"}, body: JSON.stringify(body) });
  const j = await r.json();
  if(j.ok && j.parsed){
    const s = j.parsed.steps || j.parsed.raw || j.parsed;
    document.getElementById("aiChat").innerText = Array.isArray(s) ? s.join("\n") : (typeof s === "string" ? s : JSON.stringify(s));
  } else {
    document.getElementById("aiChat").innerText = JSON.stringify(j);
  }
  document.getElementById("aiInput").value = "";
}

function openWhatsApp(){
  const num = "{{ nequi_number }}".replace(/\+/g,''); const msg = encodeURIComponent("Hola, ya realicé el pago. Adjunto comprobante. Correo: ");
  window.open("https://wa.me/" + num.replace(/\D/g,'') + "?text=" + msg, "_blank");
}

async function uploadNequi(){
  const email = document.getElementById("nequiEmail").value.trim();
  const file = document.getElementById("nequiFile").files[0];
  if(!email){ alert("Escribe tu correo para asociarlo a la activación"); return; }
  if(!file){ alert("Selecciona el comprobante"); return; }
  const fd = new FormData(); fd.append("email", email); fd.append("file", file);
  const r = await fetch("/nequi/upload", { method:"POST", body: fd });
  const j = await r.json();
  if(j.ok){ document.getElementById("nequiMsg").innerText = "Comprobante subido. Espera verificación manual (admin)."; localStorage.setItem("sensifyff_email", email); }
  else { document.getElementById("nequiMsg").innerText = "Error: " + (j.error || JSON.stringify(j)); }
}

async function detectMyDevice(){
  const r = await fetch("/detect-device"); const j = await r.json();
  if(j.detected){ document.getElementById("detected").innerText = j.detected.brand + " " + j.detected.model; document.getElementById("deviceInput").value = j.detected.model; }
  else document.getElementById("detected").innerText = "No detectado automáticamente";
}

setSensGrid(null);
</script>
</body>
</html>
"""

# --- Routes ---
@app.route("/")
def index():
    return render_template_string(INDEX_HTML, nequi_number=DEFAULT_NEQUI_NUMBER)

@app.route("/ai/basic")
def ai_basic_route():
    q = request.args.get("q","mejores sensibilidades free fire 2025")
    device = request.args.get("device")
    res = ai_basic_recommendation(q, device)
    return jsonify(res)

@app.route("/ai/premium", methods=["POST"])
def ai_premium_route():
    data = request.get_json() or {}
    q = data.get("query","mejores sensibilidades free fire 2025")
    device = data.get("device")
    email = data.get("email")
    res = ai_premium_generate(q, device, email)
    return jsonify(res)

# Detect device (heurística con user-agent)
@app.route("/detect-device")
def detect_device():
    ua = request.headers.get("User-Agent","").lower()
    for d in DEVICE_DB:
        key = d["model"].split()[0].lower()
        if key and key in ua:
            return jsonify({"detected": d, "ua": ua})
    for d in DEVICE_DB:
        if d["brand"].lower() in ua:
            return jsonify({"detected": d, "ua": ua})
    return jsonify({"detected": None, "ua": ua})

# Nequi upload
@app.route("/nequi/upload", methods=["POST"])
def nequi_upload():
    email = request.form.get("email")
    file = request.files.get("file")
    if not email or not file:
        return jsonify({"error":"email y archivo requeridos"}), 400
    if not allowed_file(file.filename):
        return jsonify({"error":"tipo de archivo no permitido"}), 400
    filename = secure_filename(f"{email.replace('@','_')}_{int(time.time())}_{file.filename}")
    path = UPLOADS / filename
    file.save(str(path))
    set_nequi_pending(email, filename)
    return jsonify({"ok":True, "file": filename})

# Admin: list pending
@app.route("/admin/pending-nequi")
def admin_pending():
    token = request.headers.get("X-ADMIN-TOKEN","")
    if token != ADMIN_TOKEN:
        return jsonify({"error":"no autorizado"}), 403
    return jsonify(get_pending_nequi())

# Admin: approve
@app.route("/admin/approve-nequi", methods=["POST"])
def admin_approve():
    token = request.headers.get("X-ADMIN-TOKEN","")
    if token != ADMIN_TOKEN:
        return jsonify({"error":"no autorizado"}), 403
    data = request.get_json() or {}
    file_id = data.get("id")
    if not file_id:
        return jsonify({"error":"id requerido"}), 400
    ok = approve_nequi(file_id)
    return jsonify({"ok": ok})

# Serve uploads (admin)
@app.route("/uploads/<path:filename>")
def serve_upload(filename):
    token = request.headers.get("X-ADMIN-TOKEN","")
    if token != ADMIN_TOKEN:
        return jsonify({"error":"no autorizado"}), 403
    filepath = UPLOADS / filename
    if not filepath.exists():
        return jsonify({"error":"no encontrado"}), 404
    return send_from_directory(str(UPLOADS), filename)

# Simple health
@app.route("/health")
def health():
    return jsonify({"ok": True, "time": datetime.utcnow().isoformat()})

# Run
if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    logging.info("Iniciando SensifyFF en puerto %s", port)
    app.run(host="0.0.0.0", port=port, debug=True)
