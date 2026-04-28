<script lang="ts">
  import { onMount } from "svelte";
  import { page } from "$app/stores";

  onMount(() => {
    console.log("Page data:", $page);
  });

  // svelte-ignore non_reactive_update
  let url = "";
  // svelte-ignore non_reactive_update
  let shortenedUrl = "";
  // svelte-ignore non_reactive_update
  let stats: { totalClicks: any; createdAt: any; lastAccesedAt: any } | null =
    null;
  // svelte-ignore non_reactive_update
  let error = "";

  const API = "http://localhost:8080/api";

  async function shortenUrl() {
    error = "";
    stats = null;
    // Placeholder for URL shortening logic

    try {
      const res = await fetch("${API}/shorten", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ originalUrl: url }),
      });

      if (!res.ok) throw new Error("Failed to shorten URL");

      const data = await res.json();
      shortenedUrl = data.shortenedUrl;
    } catch (e) {
      error = "Please enter a valid URL.";
    }
  }

  async function loadStats() {
    try {
      const res = await fetch(`${API}/stats/${shortenedUrl}`);
      if (!res.ok) throw new Error("Failed to load stats");
      stats = await res.json();
    } catch (e) {
      error = "Failed to load statistics.";
    }
  }

  function copy() {
    navigator.clipboard.writeText('${API.replace("/api", "")}/${shortenedUrl}');
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

  {#if shortenedUrl}
    <div class="result">
      <p>
        Short URL: <a
          href={"http://localhost:8080/${shortenedUrl}"}
          target="_blank">/{shortenedUrl}</a
        >
      </p>
      <div class="buttons">
        <button onclick={copy}>Copy to Clipboard</button>
        <button onclick={loadStats}>Load Stats</button>
      </div>
    </div>
  {/if}

  {#if stats}
    <div class="stats">
      <h2>Statistics</h2>
      <p>Total Clicks: {stats.totalClicks}</p>
      <p>Created: {stats.createdAt}</p>
      <p>Last Access: {stats.lastAccesedAt ?? "-"}</p>
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
