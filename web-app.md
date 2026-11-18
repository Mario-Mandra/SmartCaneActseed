# Web app (phone camera → COCO-SSD → TalkBack) — v0, 18 Nov

Repo layout: `index.html` + `manifest.webmanifest` in the root of a public GitHub repo, GitHub Pages on (Settings → Pages → Deploy from branch → main / root). URL: `https://<user>.github.io/<repo>/`. Add `?voce=1` for demos without TalkBack (speechSynthesis ro-RO).

Implements the design rules in `claude/roadmap-and-decisions.md`: one full-screen button, aria-live assertive, left/centre/right by box centre, "aproape" if box height > 50% of frame, cooldown 3.5 s per class, score ≥ 0.6, Wake Lock, frames never stored. Announces one object per cycle (near first, then largest box), only obstacle-relevant COCO classes.

Open-source to declare: TensorFlow.js 4.22.0, COCO-SSD 2.2.3 (Apache-2.0), loaded from jsDelivr.

## index.html
```html
<!DOCTYPE html>
<html lang="ro">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Baston – detectare obiecte</title>
  <link rel="manifest" href="manifest.webmanifest">
  <meta name="theme-color" content="#000000">
  <style>
    html, body { margin: 0; height: 100%; background: #000; color: #fff;
                 font-family: system-ui, sans-serif; }
    #main { position: fixed; inset: 0; width: 100%; height: 100%; border: 0;
            padding: 0; background: #000; color: #fff; cursor: pointer; }
    #main:focus-visible { outline: 6px solid #ff0; outline-offset: -6px; }
    video, canvas { position: absolute; inset: 0; width: 100%; height: 100%;
                    object-fit: cover; }
    #label { position: absolute; left: 0; right: 0; bottom: 8%; padding: 1rem;
             font-size: 2rem; font-weight: 700; text-align: center;
             background: rgba(0,0,0,.65); }
    .sr-only { position: absolute; width: 1px; height: 1px; overflow: hidden;
               clip: rect(0 0 0 0); white-space: nowrap; }
  </style>
</head>
<body>
  <button id="main" type="button">
    <video id="video" playsinline muted aria-hidden="true"></video>
    <canvas id="overlay" aria-hidden="true"></canvas>
    <span id="label">Pornește detectarea</span>
  </button>
  <div id="live" class="sr-only" aria-live="assertive" aria-atomic="true"></div>

  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.22.0/dist/tf.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd@2.2.3/dist/coco-ssd.min.js"></script>
  <script>
  // ---- Settings (tune here) ----
  const MIN_SCORE   = 0.6;    // confidence threshold
  const COOLDOWN_MS = 3500;   // same object is not repeated before this
  const LOOP_MS     = 300;    // pause between detections (battery)
  const NEAR_RATIO  = 0.5;    // box taller than half the frame = "aproape"
  // ?voce=1 -> also speak with speechSynthesis (for demos without TalkBack)
  const USE_VOICE = new URLSearchParams(location.search).get('voce') === '1';

  // Only obstacle-relevant COCO classes are announced, in Romanian.
  const RO = {
    'person': 'Persoană', 'bicycle': 'Bicicletă', 'car': 'Mașină',
    'motorcycle': 'Motocicletă', 'bus': 'Autobuz', 'truck': 'Camion',
    'traffic light': 'Semafor', 'fire hydrant': 'Hidrant',
    'stop sign': 'Indicator stop', 'parking meter': 'Parcometru',
    'bench': 'Bancă', 'chair': 'Scaun', 'couch': 'Canapea',
    'dining table': 'Masă', 'potted plant': 'Ghiveci', 'dog': 'Câine',
    'cat': 'Pisică', 'suitcase': 'Valiză', 'backpack': 'Rucsac',
    'umbrella': 'Umbrelă', 'bed': 'Pat', 'toilet': 'Toaletă', 'tv': 'Televizor'
  };

  const btn = document.getElementById('main');
  const label = document.getElementById('label');
  const live = document.getElementById('live');
  const video = document.getElementById('video');
  const canvas = document.getElementById('overlay');
  const ctx = canvas.getContext('2d');

  let model = null, stream = null, running = false, wakeLock = null;
  const lastSaid = {};   // class -> timestamp

  function announce(text) {
    // Clear first so the screen reader repeats identical messages.
    live.textContent = '';
    setTimeout(() => { live.textContent = text; }, 50);
    if (USE_VOICE && 'speechSynthesis' in window) {
      speechSynthesis.cancel();
      const u = new SpeechSynthesisUtterance(text);
      u.lang = 'ro-RO';
      u.rate = 1.1;
      speechSynthesis.speak(u);
    }
  }

  function setLabel(text) {
    label.textContent = text;
    btn.setAttribute('aria-label', text);
  }

  async function keepAwake() {
    try { wakeLock = await navigator.wakeLock.request('screen'); } catch (e) {}
  }
  document.addEventListener('visibilitychange', () => {
    if (running && document.visibilityState === 'visible') keepAwake();
  });

  function describe(det) {
    const [x, , w, h] = det.bbox;
    const cx = (x + w / 2) / video.videoWidth;
    const pos = cx < 1 / 3 ? 'în stânga' : cx > 2 / 3 ? 'în dreapta' : 'în față';
    const near = h / video.videoHeight > NEAR_RATIO;
    return { near, text: `${RO[det.class]}${near ? ' aproape' : ''}, ${pos}` };
  }

  function draw(dets) {
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.lineWidth = 4;
    ctx.font = '24px system-ui';
    for (const d of dets) {
      const [x, y, w, h] = d.bbox;
      ctx.strokeStyle = '#ff0';
      ctx.strokeRect(x, y, w, h);
      ctx.fillStyle = '#ff0';
      ctx.fillText(`${RO[d.class]} ${Math.round(d.score * 100)}%`, x + 4, y + 26);
    }
  }

  async function loop() {
    if (!running) return;
    try {
      const all = await model.detect(video, 10, MIN_SCORE);
      const dets = all.filter(d => RO[d.class]);
      draw(dets);

      // Pick ONE thing to say: near objects first, then the biggest box.
      const now = Date.now();
      const candidates = dets
        .filter(d => !lastSaid[d.class] || now - lastSaid[d.class] > COOLDOWN_MS)
        .map(d => ({ d, ...describe(d), area: d.bbox[2] * d.bbox[3] }))
        .sort((a, b) => (b.near - a.near) || (b.area - a.area));
      if (candidates.length) {
        lastSaid[candidates[0].d.class] = now;
        announce(candidates[0].text);
      }
    } catch (e) {
      console.error(e);
    }
    setTimeout(loop, LOOP_MS);
  }

  async function start() {
    setLabel('Se pornește camera…');
    announce('Se pornește camera');
    try {
      stream = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: 'environment', width: { ideal: 640 }, height: { ideal: 480 } },
        audio: false
      });
    } catch (e) {
      setLabel('Camera nu a putut fi pornită. Atinge pentru a încerca din nou.');
      announce('Camera nu a putut fi pornită');
      return;
    }
    video.srcObject = stream;
    await video.play();

    if (!model) {
      setLabel('Se încarcă modelul…');
      announce('Se încarcă modelul');
      model = await cocoSsd.load({ base: 'lite_mobilenet_v2' });
    }
    running = true;
    keepAwake();
    setLabel('Detectare pornită. Atinge pentru a opri.');
    announce('Detectare pornită');
    loop();
  }

  function stop() {
    running = false;
    if (stream) stream.getTracks().forEach(t => t.stop());
    stream = null;
    if (wakeLock) { wakeLock.release(); wakeLock = null; }
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    setLabel('Detectare oprită. Atinge pentru a porni.');
    announce('Detectare oprită');
  }

  let busy = false;
  btn.addEventListener('click', async () => {
    if (busy) return;
    busy = true;
    try { running ? stop() : await start(); } finally { busy = false; }
  });

  setLabel('Pornește detectarea');
  // Frames are only read from the live video in memory; nothing is stored or sent.
  </script>
</body>
</html>
```

## manifest.webmanifest
```json
{
  "name": "Baston – detectare obiecte",
  "short_name": "Baston",
  "lang": "ro",
  "start_url": ".",
  "display": "standalone",
  "background_color": "#000000",
  "theme_color": "#000000"
}
```
