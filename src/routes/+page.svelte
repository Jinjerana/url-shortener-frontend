<script lang="ts">
  import { tick } from 'svelte';
  import * as QRCode from 'qrcode';
  import { onMount } from 'svelte';

  // Typ-Definition für unsere Historie
  interface HistoryItem {
    shortCode: string;
    longUrl: string;
    date: string;
  }

  // Svelte 5 Runes - Jetzt ist history global für die ganze Datei sichtbar!
  let url = $state("");
  let shortCode = $state("");
  let error = $state("");
  let notFoundError = $state(false);
  
  let totalClicks = $state(0);
  let createdAt = $state("");
  let lastAccessedAt = $state<string | null>(null);
  let showStats = $state(false);

  let copied = $state(false);
  let canvasElement = $state<HTMLCanvasElement | null>(null);
  
  // Unsere Historie als reaktiver Zustand
  let historyList = $state<HistoryItem[]>([]);

  const API = "https://url-shortener-api-ncfg.onrender.com/api";
  const LIVE_URL_BASE = "https://url-shortener-api-ncfg.onrender.com";

  onMount(() => {
    // 1. Checken, ob wir wegen eines 404-Fehlers vom Backend umgeleitet wurden
    const urlParams = new URLSearchParams(window.location.search);
    if (urlParams.get('error') === 'notfound') {
      notFoundError = true;
      error = "The short code you are looking for does not exist or was entered incorrectly.";
    }

    // 2. Historie sauber aus dem LocalStorage laden
    const saved = localStorage.getItem("zaplink_history");
    if (saved) {
      try {
        historyList = JSON.parse(saved);
      } catch (e) {
        console.error("Failed to load history.", e);
      }
    }
  });

  // Funktion zum Speichern eines neuen Links in der Historie
  function saveToHistory(code: string, originalUrl: string) {
    const newItem: HistoryItem = {
      shortCode: code,
      longUrl: originalUrl,
      date: new Date().toLocaleDateString()
    };
    
    // Neuen Eintrag nach oben packen, Duplikate filtern, max. 5 Einträge behalten
    historyList = [newItem, ...historyList.filter(item => item.shortCode !== code)].slice(0, 5);
    localStorage.setItem("zaplink_history", JSON.stringify(historyList));
  }

  // Wenn man in der Historie auf einen Link klickt
  function selectFromHistory(item: HistoryItem) {
    shortCode = item.shortCode;
    url = item.longUrl;
    error = "";
    showStats = false;
  }

  async function shortenUrl() {
    error = "";
    showStats = false;
    shortCode = "";
    copied = false;

    if (!url.trim()) {
      error = "Please enter a URL first.";
      return;
    }

    try {
      const res = await fetch(`${API}/shorten`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ originalUrl: url }),
      });

      if (!res.ok) throw new Error();

      const data = await res.json();
      shortCode = data.shortCode;
      
      // HIER SPEICHERN WIR DEN ERFOLG IN DIE HISTORIE!
      saveToHistory(data.shortCode, url);
    } catch (e) {
      error = "Please enter a valid URL (e.g., https://example.com).";
    }
  }

  async function loadStats() {
    error = "";

    try {
      const res = await fetch(`${API}/stats/${shortCode}`);
      if (!res.ok) {
        showStats = false;
        error = "Statistics not available for this link.";
        return;
      }

      const data = await res.json();

      if (!data || data.totalClicks === undefined) {
        throw new Error("Invalid data format");
      }

      totalClicks = data.totalClicks;
      createdAt = new Date(data.createdAt).toLocaleString();
      lastAccessedAt = data.lastAccessedAt ? new Date(data.lastAccessedAt).toLocaleString() : null;
      showStats = true;
    } catch (e) {
      console.error("Failed to load statistics.", e);
      showStats = false;
      error = "Failed to load statistics. Please check the short code.";
    }
  }

  function copyToClipboard() {
    navigator.clipboard.writeText(`${LIVE_URL_BASE}/${shortCode}`);
    copied = true;
    setTimeout(() => (copied = false), 2000);
  }

  async function generateQR() {
    if (!shortCode || !canvasElement) return;
    await tick();
    try {
      await QRCode.toCanvas(canvasElement, `${LIVE_URL_BASE}/${shortCode}`, {
        width: 180,
        margin: 1,
        color: {
          dark: '#0f172a',
          light: '#ffffff'
        }
      });
    } catch (err) {
      console.error("QR Code Error:", err);
    }
  }

  function downloadQR() {
    if (!canvasElement) return;
    const link = document.createElement('a');
    link.download = `qr-${shortCode}.png`;
    link.href = canvasElement.toDataURL();
    link.click();
  }

  // Svelte 5 Reactivity Trigger
  $effect(() => {
    if (shortCode) {
      generateQR();
    }
  });
</script>

<div class="app-container">
  <header class="main-header">
    <h1>ZapLink <span class="version-badge">v1</span></h1>
    <p class="subtitle">Shorten your links, generate dynamic QR codes and track analytics.</p>
  </header>

  <main class="content-box">
    <section class="input-section">
      <div class="input-flex">
        <input 
          type="url" 
          bind:value={url} 
          placeholder="Paste your long URL here (https://...)" 
        />
        <button onclick={shortenUrl} class="btn-primary">Shorten Link</button>
      </div>

      {#if error}
        <p class="error-message">{error}</p>

      {#if notFoundError}
       <div class="error-page-card animate-fade-in">
        <div class="error-icon">🔍</div>
        <h2>Link not found!</h2>
        <p class="error-description">
          The short code you are looking for does not exist or was entered incorrectly. 
        </p>
        
        <div class="error-actions">
          <button onclick={() => { notFoundError = false; url = ""; shortCode = ""; }} class="btn-retry">
            🔄 Create New Link
          </button>
          
          {#if historyList.length > 0}
            <p class="error-hint">Or select a working link from your recent history below!</p>
          {/if}
        </div>
      </div>
      {/if}
    </section>

    {#if shortCode && !error}
      <div class="dashboard-layout">
        
        <div class="panel-card">
          <h3>Your Shortened Link</h3>
          <div class="url-display">
            <a href={`${LIVE_URL_BASE}/${shortCode}`} target="_blank" rel="noreferrer">
              {LIVE_URL_BASE.replace('https://', '')}/{shortCode}
            </a>
          </div>
          
          <div class="action-row">
            <button onclick={copyToClipboard} class="btn-secondary">
              {copied ? 'Copied! ✓' : 'Copy Link'}
            </button>
            <button onclick={loadStats} class="btn-secondary">
              {showStats ? 'Update Stats' : 'Track Clicks'}
            </button>
          </div>

          {#if showStats}
            <div class="analytics-box">
              <p><strong>Total Clicks:</strong> {totalClicks}</p>
              <p><strong>Created:</strong> {createdAt}</p>
              <p><strong>Last Access:</strong> {lastAccessedAt ?? "Never"}</p>
            </div>
          {/if}
        </div>

        <div class="panel-card qr-panel">
          <h3>Dynamic QR Code</h3>
          <div class="canvas-holder">
            <canvas bind:this={canvasElement}></canvas>
          </div>
          <button onclick={downloadQR} class="btn-download">
            📥 Download QR Code (.png)
          </button>
        </div>

      </div>
    {/if}

    {#if historyList.length > 0 && !shortCode}
      <section class="history-section" style="margin-top: 30px;">
        <h3 style="color: #94a3b8; font-size: 1rem; text-transform: uppercase; margin-bottom: 10px;">Recent Links</h3>
        <div style="display: flex; flex-direction: column; gap: 8px;">
          {#each historyList as item}
            <button onclick={() => selectFromHistory(item)} style="background: #111827; border: 1px solid #1f2937; color: white; padding: 12px; border-radius: 8px; display: flex; justify-content: space-between; text-align: left; cursor: pointer; width: 100%;">
              <div>
                <div style="color: #60a5fa; font-weight: bold;">{LIVE_URL_BASE.replace('https://', '')}/{item.shortCode}</div>
                <div style="color: #64748b; font-size: 0.8rem; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 400px;">{item.longUrl}</div>
              </div>
              <div style="color: #475569; font-size: 0.8rem;">{item.date}</div>
            </button>
          {/each}
        </div>
      </section>
    {/if}
  </main>
</div>

<style>
  :global(body) {
    background-color: #0b0f19;
    margin: 0;
    font-family: system-ui, -apple-system, sans-serif;
  }

  .app-container {
    max-width: 850px;
    margin: 0 auto;
    padding: 50px 20px;
    color: #f8fafc;
  }

  .main-header {
    text-align: center;
    margin-bottom: 35px;
  }

  h1 {
    font-size: 2.4rem;
    font-weight: 800;
    margin-bottom: 8px;
    color: #3b82f6;
  }

  .version-badge {
    font-size: 0.8rem;
    padding: 3px 8px;
    background: #1e293b;
    border-radius: 12px;
    color: #94a3b8;
    margin-left: 5px;
  }

  .subtitle {
    color: #94a3b8;
    margin: 0;
  }

  .input-section {
    background: #111827;
    border: 1px solid #1f2937;
    padding: 20px;
    border-radius: 12px;
    margin-bottom: 25px;
  }

  .input-flex {
    display: flex;
    gap: 10px;
  }

  input {
    flex: 1;
    background: #1f2937;
    border: 1px solid #374151;
    padding: 12px;
    border-radius: 8px;
    color: #fff;
    font-size: 1rem;
  }

  input:focus {
    outline: none;
    border-color: #3b82f6;
  }

  .btn-primary {
    background: #3b82f6;
    color: white;
    border: none;
    padding: 12px 24px;
    font-weight: 600;
    border-radius: 8px;
    cursor: pointer;
  }

  .btn-primary:hover {
    background: #2563eb;
  }

  .dashboard-layout {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
  }

  .panel-card {
    background: #111827;
    border: 1px solid #1f2937;
    border-radius: 12px;
    padding: 20px;
  }

  .url-display {
    background: #1f2937;
    padding: 12px;
    border-radius: 8px;
    text-align: center;
    margin-bottom: 15px;
  }

  .url-display a {
    color: #60a5fa;
    text-decoration: none;
    font-weight: bold;
  }

  .action-row {
    display: flex;
    gap: 10px;
  }

  .btn-secondary {
    flex: 1;
    background: #1e293b;
    border: 1px solid #334155;
    color: #cbd5e1;
    padding: 10px;
    border-radius: 6px;
    cursor: pointer;
  }

  .btn-secondary:hover {
    background: #334155;
  }

  .analytics-box {
    margin-top: 15px;
    padding-top: 15px;
    border-top: 1px solid #1f2937;
    font-size: 0.9rem;
    color: #94a3b8;
  }

  .qr-panel {
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .canvas-holder {
    background: white;
    padding: 10px;
    border-radius: 8px;
    margin-bottom: 15px;
    display: inline-flex;
  }

  .btn-download {
    background: transparent;
    border: none;
    color: #94a3b8;
    cursor: pointer;
    font-size: 0.9rem;
  }

  .btn-download:hover {
    color: #60a5fa;
  }

  .error-page-card {
    background: linear-gradient(135deg, #1e1b4b 0%, #111827 100%);
    border: 1px solid #ef4444; /* Ein schicker roter Rahmen für die Warnung */
    padding: 35px;
    border-radius: 16px;
    text-align: center;
    max-width: 500px;
    width: 100%;
    box-shadow: 0 10px 30px rgba(239, 68, 68, 0.1);
    margin: 20px auto;
  }

  .error-icon {
    font-size: 3rem;
    margin-bottom: 15px;
    animation: pulse 2s infinite;
  }

  .error-page-card h2 {
    color: #f87171;
    font-size: 1.5rem;
    margin-top: 0;
    margin-bottom: 10px;
  }

  .error-description {
    color: #94a3b8;
    font-size: 0.95rem;
    line-height: 1.5;
    margin-bottom: 25px;
  }

  .btn-retry {
    background: #ef4444;
    color: white;
    border: none;
    padding: 12px 24px;
    font-weight: 600;
    border-radius: 8px;
    cursor: pointer;
    transition: background 0.2s;
    width: auto; /* Verhindert, dass der Button die ganze Breite einnimmt */
    display: inline-block;
  }

  .btn-retry:hover {
    background: #dc2626;
  }

  .error-hint {
    font-size: 0.85rem;
    color: #64748b;
    margin-top: 20px;
    margin-bottom: 0;
  }

  /* Eine kleine, geschmeidige Einblend-Animation */
  .animate-fade-in {
    animation: fadeIn 0.4s ease-out;
  }

  @keyframes fadeIn {
    from { opacity: 0; transform: translateY(10px); }
    to { opacity: 1; transform: translateY(0); }
  }

  @keyframes pulse {
    0% { transform: scale(1); }
    50% { transform: scale(1.1); }
    100% { transform: scale(1); }
  }
</style>