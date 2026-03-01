
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import WebSocket from '@tauri-apps/plugin-websocket';

  // --- CONFIGURATION ---
  const IS_VERTICAL = true; // Set to true for stacked layout, false for side-by-side
  // ---------------------

  const DEV_FUEL_RED = 'FUEL_SENSOR_RED';
  const DEV_FUEL_BLUE = 'FUEL_SENSOR_BLUE';
  const CMD_GET_COUNTS = 'GET_INTERNAL_COUNTS';
  const EVT_FUEL_DETECTED = 'FUEL_DETECTED';
  const EVT_INTERNAL_COUNTS = 'INTERNAL_COUNT';

  let ws: WebSocket | null = null;
  let poll_interval: number;

  let red_score = $state(0);
  let blue_score = $state(0);

  /**
Dynamically adjusts the cqmin size based on digit count.
1-2 digits = 55cqmin (large)
3 digits = 38cqmin (smaller to prevent overflow)
   */
  function getFontSize(score: number) {
    const len = score.toString().length;
    if (len >= 3) return '38cqmin';
    if (len === 2) return '50cqmin';
    return '55cqmin';
  }

  const blueFontSize = $derived(getFontSize(blue_score));
  const redFontSize = $derived(getFontSize(red_score));

  function send_ws(device_id: string, command: string) {
    if (!ws) return;
    ws.send(JSON.stringify({ type: 'device_command', device_id, command, payload: {} }));
  }

  onMount(async () => {
    try {
      ws = await WebSocket.connect('ws://10.0.100.5:8000/connect');
      ws.addListener((msg) => {
        try {
          const parsed = JSON.parse(msg.data as string);
          const event = parsed.data?.event;
          if (event === EVT_FUEL_DETECTED) {
            if (parsed.device_id === DEV_FUEL_RED) red_score += parsed.data.count;
            if (parsed.device_id === DEV_FUEL_BLUE) blue_score += parsed.data.count;
          } else if (event === EVT_INTERNAL_COUNTS) {
            if (parsed.device_id === DEV_FUEL_RED) red_score = parsed.data.count;
            if (parsed.device_id === DEV_FUEL_BLUE) blue_score = parsed.data.count;
          }
        } catch (e) {}
      });

      poll_interval = window.setInterval(() => {
        send_ws(DEV_FUEL_RED, CMD_GET_COUNTS);
        send_ws(DEV_FUEL_BLUE, CMD_GET_COUNTS);
      }, 3000);
    } catch (err) {
      console.error("Connection failed", err);
    }
  });

  onDestroy(() => {
    if (poll_interval) clearInterval(poll_interval);
    if (ws) ws.disconnect();
  });
</script>

<main class="scoreboard" class:layout-vertical={IS_VERTICAL}>
  <div class="team-box blue-box">
    <div class="label">BLUE ALLIANCE</div>
    <div class="score" style="font-size: {blueFontSize}">{blue_score}</div>
  </div>

  <div class="team-box red-box">
    <div class="label">RED ALLIANCE</div>
    <div class="score" style="font-size: {redFontSize}">{red_score}</div>
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    font-family: system-ui, -apple-system, sans-serif;
    background-color: #0f172a;
    height: 100vh;
    width: 100vw;
    overflow: hidden;
  }

  .scoreboard {
    display: flex;
    flex-direction: row; /* Default: side-by-side */
    height: 100vh;
    width: 100vw;
    padding: 2vw;
    gap: 2vw;
    box-sizing: border-box;
  }

  /* When the constant is true, this class is applied */
  .layout-vertical {
    flex-direction: column;
  }

  .team-box {
    flex: 1;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    border-radius: 3vw;
    container-type: size;
    color: white;
    box-shadow: 0 10px 25px rgba(0,0,0,0.5);
    transition: all 0.3s ease; /* Smooth transition if layout changes */
  }

  .blue-box { background-color: #3b82f6; }
  .red-box { background-color: #ef4444; }

  .label {
    font-size: 6cqmin;
    font-weight: 700;
    letter-spacing: 0.1em;
    opacity: 0.9;
    margin-bottom: 2cqmin;
  }

  .score {
    /* Base style; font-size is overridden by inline style from getFontSize() */
    font-weight: 800;
    line-height: 1;
    font-variant-numeric: tabular-nums;
    text-shadow: 0 1cqmin 2cqmin rgba(0,0,0,0.2);
    transition: font-size 0.2s ease-out; /* Prevents jarring jumps when score increases */
  }
</style>

