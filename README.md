<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>NFC Armor — Live Tag Scanner & Cloner Detector</title>
  <meta name="description" content="Tap any NFC tag → instantly see if it's cloned + stream live to laptop.">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css">
  <style>
    :root { --accent: #00ff9d; --bg: #0f0f1b; --card: #1a1a2e; --red: #ff0066; --green: #00ff9d; }
    body { margin:0; font-family: system-ui, sans-serif; background: var(--bg); color: white; text-align:center; padding: 2rem 1rem; }
    h1 { font-size: 2.8rem; margin:0; background: linear-gradient(90deg, #00ff9d, #00b8ff); -webkit-background-clip:text; -webkit-text-fill-color:transparent; }
    .scan-btn, .stream-btn { background:var(--accent); color:#000; font-weight:bold; font-size:1.8rem; padding:1.2rem 2.5rem; border:none; border-radius:20px; cursor:pointer; margin:1rem; box-shadow:0 10px 30px rgba(0,255,157,0.4); }
    .results { margin-top:3rem; padding:2rem; background:var(--card); border-radius:16px; display:none; max-width:90%; margin-left:auto; margin-right:auto; }
    #qr { margin:2rem 0; }
    pre { background:#000; padding:1rem; border-radius:8px; overflow-x:auto; white-space: pre-wrap; }
    .verdict { font-size:2.8rem; font-weight:bold; margin:1rem 0; }
  </style>
</head>
<body>

  <h1>NFC Armor</h1>
  <p>Tap any tag → instantly know if it's cloned.</p>

  <button id="start" class="scan-btn">TAP YOUR TAG NOW</button>
  <button id="stream" class="stream-btn" style="display:none">LIVE STREAM TO LAPTOP</button>

  <div id="results" class="results">
    <div id="verdict" class="verdict">Scanning…</div>
    <div id="qr"></div>
    <pre id="raw"></pre>
    <button onclick="copy('#raw')">Copy Raw</button>
    <button onclick="downloadJSON()">Download JSON</button>
  </div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
<script>
  let resultsJSON = "", roomCode = "", encryptKey = "";

  document.getElementById("start").addEventListener("click", async () => {
    if (!("NDEFReader" in window)) return alert("Android + Chrome only for now");
    try {
      const ndef = new NDEFReader();
      await ndef.scan();
      document.getElementById("results").style.display = "block";

      ndef.onreading = async ({ message, serialNumber: sn }) => {
        roomCode = Math.random().toString(36).substring(2,8).toUpperCase();
        encryptKey = crypto.getRandomValues(new Uint8Array(32));
        const data = {serialNumber: sn, records: message.records, timestamp: Date.now()};
        resultsJSON = JSON.stringify(data);

        // Clone detection
        const isClone = ["04","08","88","00"].some(p => sn.toUpperCase().startsWith(p));
        document.getElementById("verdict").textContent = isClone ? "CLONED TAG DETECTED" : "GENUINE TAG";
        document.getElementById("verdict").style.color = isClone ? "#ff0066" : "#00ff9d";

        // Encrypt & send
        const encrypted = await encrypt(resultsJSON, encryptKey);
        await fetch(`/.netlify/functions/relay?room=${roomCode}`, {
          method: "POST",
          body: JSON.stringify({data: encrypted, key: Array.from(encryptKey)})
        });

        // Show QR + code
        const url = `${location.origin}/live/?room=${roomCode}#${btoa(String.fromCharCode(...encryptKey))}`;
        new QRCode(document.getElementById("qr"), {text: url, width: 256, height: 256});
        document.getElementById("raw").textContent = `Room code: ${roomCode}\nScan QR on laptop → live view\n\n` + resultsJSON;
        document.getElementById("stream").style.display = "block";
        document.getElementById("stream").onclick = () => location.href = url;
      };
    } catch(e) { alert(e); }
  });

  async function encrypt(text, key) {
    const enc = new TextEncoder();
    const iv = crypto.getRandomValues(new Uint8Array(12));
    const encrypted = await crypto.subtle.encrypt(
      {name: "AES-GCM", iv},
      await crypto.subtle.importKey("raw", key, "AES-GCM", false, ["encrypt"]),
      enc.encode(text)
    );
    return {iv: Array.from(iv), data: Array.from(new Uint8Array(encrypted))};
  }

  function copy(sel) { navigator.clipboard.writeText(document.querySelector(sel).textContent); }
  function downloadJSON() {
    const a = document.createElement("a");
    a.href = URL.createObjectURL(new Blob([resultsJSON], {type:"application/json"}));
    a.download = "nfcarmor-scan.json";
    a.click();
  }
</script>
</body><!DOCTYPE html>
<html>
<head>
  <title>NFC Armor — Live View</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: system-ui; background:#0f0f1b; color:#00ff9d; text-align:center; padding:3rem; }
    pre { background:#000; color:#0f0; padding:2rem; margin:2rem auto; max-width:90%; border-radius:12px; }
  </style>
</head>
<body>
  <h1>NFC Armor — Live View</h1>
  <p>Waiting for phone to scan tag…</p>
  <pre id="out">Room: <span id="room"></span></pre>

<script>
  const params = new URLSearchParams(location.search);
  const room = params.get("room");
  const key = location.hash.substring(1);
  document.getElementById("room").textContent = room || "unknown";

  if (!room || !key) {
    document.getElementById("out").textContent = "Invalid link – open from phone QR code";
  } else {
    const evtSource = new EventSource(`/.netlify/functions/relay?room=${room}&watch=1`);
    evtSource.onmessage = async e => {
      const {data, key: keyArray} = JSON.parse(e.data);
      const decryptKey = Uint8Array.from(atob(key).split("").map(c=>c.charCodeAt(0)));
      const decrypted = await decrypt(data, decryptKey);
      document.getElementById("out").textContent = decrypted;
      evtSource.close();
    };
  }

  async function decrypt(payload, key) {
    const {iv, data} = payload;
    const encrypted = Uint8Array.from(data);
    const cryptoKey = await crypto.subtle.importKey("raw", key, "AES-GCM", false, ["decrypt"]);
    const decrypted = await crypto.subtle.decrypt(
      {name: "AES-GCM", iv: Uint8Array.from(iv)},
      cryptoKey,
      encrypted
    );
    return new TextDecoder().decode(decrypted);
  }
</script>
</body>
</html>
exports.handler = async (event) => {
  const room = event.queryStringParameters.room;
  const watch = event.queryStringParameters.watch;

  if (event.httpMethod === "POST") {
    const body = JSON.parse(event.body);
    // Store in memory only (gone when function instance dies)
    globalThis[room] = body;
    return { statusCode: 200, body: "ok" };
  }

  if (watch) {
    // Server-Sent Events – push when data arrives
    const send = (client) => {
      if (globalThis[room]) {
        client.write(`data: ${JSON.stringify(globalThis[room])}\n\n`);
        delete globalThis[room];
      }
    };
    return {
      statusCode: 200,
      headers: {"Content-Type": "text/event-stream", "Cache-Control": "no-cache"},
      body: ""  // Netlify will stream
    };
  }

  return { statusCode: 400, body: "bad request" };
};</html>
