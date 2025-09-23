<script>
  import { onMount, onDestroy } from 'svelte';

  // --- State (JS, no TS/runes) --------------------------------------------
  let currentPosition = null;
  let errorMessage = null;
  let permissionState = 'unknown'; // 'prompt' | 'granted' | 'denied' | 'unknown'
  let watching = false;

  // Target (pre-filled)
  let target = { lat: 42.05913363079434, lon: 19.51725368912721 };

  // UI helpers
  let watchId = null;
  let highAccuracy = true;

  // Derived
  let distanceMeters = null;
  let insideZone = false;

  $: distanceMeters = currentPosition
    ? haversineMeters(
        currentPosition.coords.latitude,
        currentPosition.coords.longitude,
        target.lat,
        target.lon
      )
    : null;

  $: insideZone = (distanceMeters ?? Infinity) <= 15;

  // --- Geolocation logic ---------------------------------------------------
  function startWatching() {
    errorMessage = null;
    if (!('geolocation' in navigator)) {
      errorMessage = 'Geolocation is not supported in this browser.';
      return;
    }

    checkPermission();

    const opts = {
      enableHighAccuracy: highAccuracy,
      maximumAge: 0,
      timeout: 10000
    };

    watching = true;
    watchId = navigator.geolocation.watchPosition(
      (pos) => { currentPosition = pos; },
      (err) => { errorMessage = humanizeGeoError(err); watching = false; },
      opts
    );
  }

  function stopWatching() {
    if (watchId !== null) {
      navigator.geolocation.clearWatch(watchId);
      watchId = null;
    }
    watching = false;
  }

  async function checkPermission() {
    try {
      const status = await navigator.permissions?.query?.({ name: 'geolocation' });
      if (status) {
        permissionState = status.state;
        status.onchange = () => (permissionState = status.state);
      } else {
        permissionState = 'unknown';
      }
    } catch {
      permissionState = 'unknown';
    }
  }

  function useCurrentAsTarget() {
    if (!currentPosition) return;
    target = {
      lat: currentPosition.coords.latitude,
      lon: currentPosition.coords.longitude
    };
  }

  onDestroy(() => stopWatching());
  onMount(() => { checkPermission(); });

  // --- Utilities -----------------------------------------------------------
  function toFixed(n, digits = 2) {
    if (n === null || n === undefined) return '—';
    return Number(n).toFixed(digits);
  }

  function humanizeGeoError(err) {
    switch (err.code) {
      case err.PERMISSION_DENIED:
        return 'Permission denied. Please allow location access.';
      case err.POSITION_UNAVAILABLE:
        return 'Position unavailable. Try moving to an open area or check your connection.';
      case err.TIMEOUT:
        return 'Location request timed out. Try again.';
      default:
        return 'An unknown geolocation error occurred.';
    }
  }

  function haversineMeters(lat1, lon1, lat2, lon2) {
    const toRad = (d) => (d * Math.PI) / 180;
    const R = 6371000;
    const dLat = toRad(lat2 - lat1);
    const dLon = toRad(lon2 - lon1);
    const a =
      Math.sin(dLat / 2) ** 2 +
      Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) *
      Math.sin(dLon / 2) ** 2;
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return R * c;
  }
</script>

<!-- Layout -->
<div class="min-h-screen text-white
            bg-[radial-gradient(1200px_600px_at_80%_-10%,rgba(99,102,241,.35),transparent_60%),radial-gradient(1000px_500px_at_-10%_20%,rgba(236,72,153,.35),transparent_60%),linear-gradient(180deg,#0f172a_0%,#111827_60%,#0b1220_100%)]">
  <div class="mx-auto max-w-3xl p-6 md:p-8 lg:p-10">
    <!-- Header -->
    <header class="mb-6 flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
      <div>
        <h1 class="text-2xl font-semibold tracking-tight text-transparent bg-clip-text bg-gradient-to-r from-white to-white/60">
          Geofence Proximity Tracker
        </h1>
        <p class="text-sm text-white/70">Realtime location vs. target (15 m radius)</p>
      </div>
      <div class="flex items-center gap-2">
        {#if !watching}
          <button
            class="rounded-2xl px-4 py-2 text-sm font-semibold
                   bg-gradient-to-r from-violet-400 via-sky-400 to-emerald-400
                   text-slate-900 shadow-lg shadow-black/30 ring-1 ring-white/20
                   hover:brightness-110 transition">
            Start
          </button>
        {:else}
          <button
            class="rounded-2xl px-4 py-2 text-sm font-semibold
                   text-white bg-white/10 ring-1 ring-white/20
                   hover:bg-white/15 transition"
            on:click={stopWatching}>
            Stop
          </button>
        {/if}
        <!-- bind click to start -->
        {#if !watching}
          <span class="hidden">{/* keep DOM consistent */}</span>
        {/if}
        <div class="hidden">{/* spacer */}</div>
        <label class="flex items-center gap-2 text-xs text-white/80">
          <input type="checkbox" bind:checked={highAccuracy} class="h-4 w-4 rounded accent-sky-400" /> High accuracy
        </label>
      </div>
    </header>

    <!-- Wire Start button -->
    <script>
      // attach after DOM render
      $: if (!watching) {
        // delegate click binding here without changing logic above
        const btn = document?.querySelector('button.rounded-2xl.bg-gradient-to-r');
        btn && (btn.onclick = startWatching);
      }
    </script>

    <!-- Permissions & errors -->
    {#if permissionState === 'denied'}
      <div class="mb-4 rounded-xl border border-rose-300/50 bg-rose-500/20 p-3">
        Location permission is denied. Enable it in your browser settings, then click <em>Start</em>.
      </div>
    {/if}
    {#if errorMessage}
      <div class="mb-4 rounded-xl border border-amber-300/50 bg-amber-400/20 p-3">{errorMessage}</div>
    {/if}

    <!-- Target inputs -->
    <section class="mb-6 rounded-2xl bg-white/10 ring-1 ring-white/15 p-4 md:p-6 backdrop-blur-md shadow-xl shadow-black/20">
      <h2 class="mb-3 text-sm font-semibold tracking-wide text-white/90">Target location (provided by professor)</h2>
      <div class="grid grid-cols-1 gap-3 sm:grid-cols-3">
        <label class="flex flex-col">
          <span class="text-xs text-white/70">Latitude</span>
          <input class="mt-1 rounded-xl border border-white/20 bg-white/10 px-3 py-2 text-sm text-white placeholder-white/50 focus:outline-none focus:ring-2 focus:ring-sky-400"
                 type="number" step="any" bind:value={target.lat} />
        </label>
        <label class="flex flex-col">
          <span class="text-xs text-white/70">Longitude</span>
          <input class="mt-1 rounded-xl border border-white/20 bg-white/10 px-3 py-2 text-sm text-white placeholder-white/50 focus:outline-none focus:ring-2 focus:ring-sky-400"
                 type="number" step="any" bind:value={target.lon} />
        </label>
        <div class="flex items-end">
          <button class="w-full rounded-xl bg-gradient-to-r from-cyan-400 to-lime-400 px-3 py-2 text-sm font-semibold text-slate-900 ring-1 ring-white/25 hover:brightness-110 disabled:opacity-50"
                  on:click={useCurrentAsTarget}
                  disabled={!currentPosition}>
            Use my current position
          </button>
        </div>
      </div>
    </section>

    <!-- Live readout -->
    <section class="mb-6 rounded-2xl bg-white/10 ring-1 ring-white/15 p-4 md:p-6 backdrop-blur-md shadow-xl shadow-black/20">
      <h2 class="mb-3 text-sm font-semibold tracking-wide text-white/90">Live status</h2>
      <div class="grid grid-cols-1 gap-4 sm:grid-cols-3">
        <div class="rounded-xl border border-white/15 bg-white/5 p-3">
          <div class="text-xs text-white/70">Current position</div>
          <div class="mt-1 text-sm">
            Lat {currentPosition ? toFixed(currentPosition.coords.latitude, 6) : '—'}<br />
            Lon {currentPosition ? toFixed(currentPosition.coords.longitude, 6) : '—'}
          </div>
        </div>
        <div class="rounded-xl border border-white/15 bg-white/5 p-3">
          <div class="text-xs text-white/70">Target position</div>
          <div class="mt-1 text-sm">
            Lat {toFixed(target.lat, 6)}<br />
            Lon {toFixed(target.lon, 6)}
          </div>
        </div>
        <div class="rounded-xl border border-white/15 bg-white/5 p-3">
          <div class="text-xs text-white/70">Distance to target</div>
          <div class="mt-1 text-xl font-semibold">{distanceMeters === null ? '—' : `${Math.round(distanceMeters)} m`}</div>
        </div>
      </div>
    </section>

    <!-- Indicator / Zone visualization -->
    <section class="rounded-2xl bg-white/10 ring-1 ring-white/15 p-6 backdrop-blur-md shadow-xl shadow-black/20">
      <div class="mb-4 flex items-center justify-between">
        <h2 class="text-sm font-semibold tracking-wide text-white/90">15 m Zone</h2>
        <span class="inline-flex items-center gap-2 rounded-full px-3 py-1 text-sm"
              class:bg-green-500/20={insideZone}
              class:text-green-100={insideZone}
              class:bg-rose-500/20={!insideZone}
              class:text-rose-100={!insideZone}>
          <span class="h-2 w-2 rounded-full"
                class:bg-green-400={insideZone}
                class:bg-rose-400={!insideZone}></span>
          {insideZone ? 'Inside (≤ 15 m)' : 'Outside (> 15 m)'}
        </span>
      </div>

      <!-- Symbolic visualization -->
      <div class="mx-auto flex max-w-sm items-center justify-center">
        <div class="relative h-64 w-64 rounded-full border-2 border-dashed border-white/30 shadow-2xl shadow-black/40">
          <!-- Zone fill -->
          <div class="absolute inset-0 rounded-full opacity-30"
               class:bg-gradient-to-br={insideZone}
               class:from-green-300={insideZone}
               class:to-green-600={insideZone}
               class:bg-rose-500={!insideZone}></div>

          <!-- Target dot -->
          <div class="absolute left-1/2 top-1/2 h-3 w-3 -translate-x-1/2 -translate-y-1/2 rounded-full bg-white" title="Target"></div>

          <!-- User dot -->
          <div class="absolute left-1/2 top-1/2 h-4 w-4 -translate-x-1/2 -translate-y-1/2 rounded-full ring-4"
               class:bg-green-400={insideZone}
               class:bg-rose-400={!insideZone}
               class:ring-green-200/40={insideZone}
               class:ring-rose-200/40={!insideZone}
               title="You"></div>
        </div>
      </div>

      <p class="mt-4 text-center text-xs text-white/80">
        Visualization is symbolic (not a map). Color reflects whether your live location is within 15 meters of the target.
      </p>
    </section>

    <!-- Footer tips -->
    <footer class="mt-6 text-center text-xs text-white/80">
      {#if permissionState === 'prompt'}
        Tip: Click <strong>Start</strong> and allow location access when prompted.
      {/if}
      {#if watching && !currentPosition}
        <div class="mt-1">Waiting for first location fix…</div>
      {/if}
    </footer>
  </div>
</div>
