<!doctype html>
<html lang="de">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>OTTO Market Auftragsscan</title>
  <script src="https://unpkg.com/@zxing/browser@latest"></script>
</head>
<body style="font-family:Arial,sans-serif;margin:16px;">
  <h1>📦 OTTO Market Auftragsscan</h1>
  <p>
    Kamera-Scan: Halte das Retourenlabel oder den Barcode vor die Kamera.<br>
    Alternativ kannst du unten die Auftragsnummer eingeben (Format <code>cbn4...</code>).
  </p>

  <video id="video" muted playsinline style="width:100%;max-width:500px;background:#000;border-radius:8px;"></video>
  <div style="margin-top:10px;">
    <button id="start">📷 Scan starten</button>
    <button id="stop">⏹️ Stop</button>
  </div>
  <p id="status">Status: bereit.</p>

  <input id="manual" type="text" placeholder="z. B. cbn4abc123..." style="width:70%;padding:8px;">
  <button id="go">Öffnen</button>

  <script>
    function buildOrderUrl(orderId) {
      return `https://portal.otto.market/orders#/details/${encodeURIComponent(orderId)}/articles`;
    }

    function extractOrderId(text) {
      if (!text) return null;
      const m = String(text).trim().match(/\\b(cbn4[a-z0-9_-]+)\\b/i);
      return m ? m[1] : null;
    }

    function goToOrder(orderId, source="") {
      const url = buildOrderUrl(orderId);
      document.getElementById('status').innerHTML = `✔️ Auftrag erkannt (${source}): ${orderId}`;
      window.location.href = url;
    }

    const statusEl = document.getElementById('status');
    const manualInput = document.getElementById('manual');
    const goBtn = document.getElementById('go');
    const startBtn = document.getElementById('start');
    const stopBtn = document.getElementById('stop');
    const videoEl = document.getElementById('video');
    let codeReader;
    let running = false;

    async function startScanner() {
      try {
        const { BrowserMultiFormatReader } = ZXingBrowser;
        codeReader = new BrowserMultiFormatReader();
        const cams = await ZXingBrowser.BrowserCodeReader.listVideoInputDevices();
        if (!cams.length) { statusEl.innerHTML = 'Keine Kamera gefunden'; return; }
        let deviceId = cams[0].deviceId;
        const back = cams.find(c => (c.label || '').toLowerCase().includes('back'));
        if (back) deviceId = back.deviceId;
        running = true;
        statusEl.textContent = 'Kamera aktiv – bitte auf Label zielen…';
        await codeReader.decodeFromVideoDevice(deviceId, videoEl, (result, err) => {
          if (!running) return;
          if (result) {
            const text = result.getText();
            const orderId = extractOrderId(text);
            if (orderId) { stopScanner(); goToOrder(orderId, 'Scan'); }
          }
        });
      } catch (e) { statusEl.innerHTML = 'Fehler: ' + e.message; }
    }

    function stopScanner() {
      running = false;
      try { codeReader && codeReader.reset(); } catch(e){}
      statusEl.textContent = 'Scanner gestoppt.';
    }

    startBtn.addEventListener('click', startScanner);
    stopBtn.addEventListener('click', stopScanner);
    goBtn.addEventListener('click', () => {
      const v = manualInput.value.trim();
      const id = extractOrderId(v);
      if (id) goToOrder(id, 'Manuell');
      else statusEl.innerHTML = 'Ungültig – bitte cbn4-Code eingeben.';
    });
  </script>
</body>
</html>