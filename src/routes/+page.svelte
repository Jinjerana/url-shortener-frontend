<script lang="ts">

  let url = $state("");
  let shortCode = $state("");
  
  let totalClicks = $state(0);
  let createdAt = $state("");
  let lastAccessedAt = $state<string | null>(null);
  let showStats = $state(false);

    //let stats: { totalClicks: number; createdAt: string; lastAccessedAt: string | null } | null = $state(null);
  
  let error = $state("");

  const API = "https://url-shortener-api-ncfg.onrender.com/api";

  async function shortenUrl() {
    error = "";
    //stats = null;
    showStats = false;
    shortCode = "";
    // Placeholder for URL shortening logic

    try {
      const res = await fetch(`${API}/shorten`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ originalUrl: url }),
      });

      if (!res.ok) throw new Error("Failed to shorten URL");

      const data = await res.json();
      shortCode = data.shortCode;
    } catch (e) {
      error = "Please enter a valid URL.";
    }
  }

  async function loadStats() {
    console.log("Loading stats for", shortCode);
    try {
      const res = await fetch(`${API}/stats/${shortCode}`);
      console.log("Stats response:", res);

      if (!res.ok) throw new Error("Failed to load stats");
      const data = await res.json();
      console.log("Stats data:", data);

      totalClicks = data.totalClicks;
      createdAt = new Date(data.createdAt).toLocaleString();
      lastAccessedAt = data.lastAccessedAt ? new Date(data.lastAccessedAt).toLocaleString() : null;
      showStats = true;
      //stats = null;
      // stats = {
      //   totalClicks: data.totalClicks,
      //   createdAt: new Date(data.createdAt).toLocaleString(),
      //   lastAccessedAt: data.lastAccessedAt ? new Date(data.lastAccessedAt).toLocaleString() : null,
      // };

    } catch (e) {
      
      console.log("Error loading stats:", e);
      error = "Failed to load statistics.";
    }
  }

  function copy() {
    navigator.clipboard.writeText(`http://localhost:8080/${shortCode}`);
  }
</script>

<main>
  <h1>Make it short</h1>

  <p>Put a long Link hier</p>

  <div class="card">
    <input bind:value={url} placeholder="Enter URL" />
  </div>

  <p>
    <button onclick={shortenUrl}>Shorten</button>
  </p>

  {#if error}
    <p style="color: red;">{error}</p>
  {/if}

  {#if shortCode}
    <div class="result">
      <p>
        Short URL: <a
          href={`http://localhost:8080/${shortCode}`}
          target="_blank">http://localhost:8080/{shortCode}</a
        >
      </p>
      <div class="buttons">
        <button onclick={copy}>Copy to Clipboard</button>
        <button onclick={loadStats}>Load Stats</button>
      </div>
    </div>
  {/if}

  {#if showStats}
    <div class="stats">
      <h2>Statistics</h2>
      <p>Total Clicks: {totalClicks}</p>
      <p>Created: {createdAt}</p>
      <p>Last Access: {lastAccessedAt ?? "-"}</p>
    </div>
  {/if}
</main>

<style>
  main {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    background: #0f172a;
    color: #fff;
    font-family: system-ui;
    flex-direction: column;
  }

  h1 {
    text-align: center;
    margin-bottom: 20px;
  }

  .card {
    background: #1e293b;
    padding: 30px;
    border-radius: 16px;
    width: 350px;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  }

  input {
    width: 100%;
    padding: 10px;
    border: none;
    border-radius: 8px;
    margin-bottom: 10px;
  }

  button {
    width: 100%;
    padding: 10px;
    border: none;
    border-radius: 8px;
    background: #3b82f6;
    color: #fff;
    cursor: pointer;
    margin-top: 5px;
    transition: background 0.3s ease;
  }

  button:hover {
    background: #2563eb;
  }

  .buttons {
    display: flex;
    gap: 10px;
  }

  .result {
    margin-top: 15px;
  }

  .stats {
    margin-top: 15px;
    font-size: 0.9rem;
    color: #cbd5f5;
  }
</style>
