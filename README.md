
---

# 5 — Sandbox Behavior Dashboard (simulation, Node + simple UI)  
**What:** Defensive research dashboard that ingests simulated sandbox JSON outputs, clusters by behavior, and serves a basic HTML UI showing clusters. **Important:** simulation only — no handling of real malware binaries.

**File:** `src/sandbox_dashboard_server.js`

```javascript
#!/usr/bin/env node
/**
 * Sandbox Behavior Dashboard (simulation)
 *
 * - Uses express to serve a small UI.
 * - Ingests simulated JSON sample files from `samples/` and clusters by behavior fingerprint.
 *
 * Usage:
 *   npm install express
 *   node src/sandbox_dashboard_server.js
 *
 * NOTE: Defensive research only — do NOT use real malware samples here.
 */

const express = require('express');
const fs = require('fs');
const path = require('path');

const SAMPLE_DIR = path.join(__dirname, '..', 'samples');

function ensureSamples(){
  if (!fs.existsSync(SAMPLE_DIR)) fs.mkdirSync(SAMPLE_DIR, { recursive: true });
  const list = fs.readdirSync(SAMPLE_DIR).filter(f => f.endsWith('.json'));
  if (list.length === 0){
    // create some simulated samples
    const s = [
      { id: 's1', behaviors: ['file_write','registry_set','network_connect'] },
      { id: 's2', behaviors: ['file_write','network_connect'] },
      { id: 's3', behaviors: ['dns_query','network_connect'] },
      { id: 's4', behaviors: ['file_delete','persistence'] }
    ];
    s.forEach(x => fs.writeFileSync(path.join(SAMPLE_DIR, x.id + '.json'), JSON.stringify(x, null, 2)));
  }
}

function loadSamples(){
  const files = fs.readdirSync(SAMPLE_DIR).filter(f => f.endsWith('.json'));
  return files.map(f => JSON.parse(fs.readFileSync(path.join(SAMPLE_DIR, f), 'utf8')));
}

function fingerprint(sample){
  return sample.behaviors ? sample.behaviors.slice().sort().join('|') : '';
}

function cluster(samples){
  const map = {};
  for (const s of samples){
    const fp = fingerprint(s);
    map[fp] = map[fp] || [];
    map[fp].push(s);
  }
  return map;
}

const app = express();
app.use(express.json());

app.get('/', (req, res) => {
  res.type('html').send(`<!doctype html><html><head><meta charset="utf-8"><title>Sandbox Dashboard</title></head><body>
  <h3>Sandbox Dashboard (Simulation)</h3>
  <div id="content">Loading...</div>
  <script>
    async function load(){ const r = await fetch('/api/cluster'); const j = await r.json();
      let out = '';
      for (const key of Object.keys(j)) {
        out += '<h4>Cluster: ' + key + ' (' + j[key].length + ')</h4><pre>' + JSON.stringify(j[key], null, 2) + '</pre>';
      }
      document.getElementById('content').innerHTML = out;
    }
    load();
  </script>
  </body></html>`);
});

app.get('/api/cluster', (req, res) => {
  ensureSamples();
  const s = loadSamples();
  res.json(cluster(s));
});

const PORT = process.env.PORT || 3100;
app.listen(PORT, () => console.log('Sandbox dashboard (simulation) at http://localhost:' + PORT));

