<script>
  import { onMount, onDestroy } from 'svelte';

  // --- State ---------------------------------------------------------------
  let currentPosition = null;
  let errorMessage = null;
  let permissionState = 'unknown';
  let watching = false;

  // Target (pre-filled)
  let target = { lat: 42.05913363079434, lon: 19.51725368912721 };

  // UI helpers
  let watchId = null;
  let highAccuracy = true;

  // Derived
  let distanceMeters = null;
  let insideZone = false;
  let bearingDeg = null; // bearing to target
  let headingDeg = null; // device heading (0=N)
  let turnDeg = null;    // turn needed

  // Leaflet map refs
  let mapEl, L, map, userMarker, targetMarker, lineLayer;

  // Orientation listeners (so we can remove them)
  let usingAbs = false;
  const onDOA = (e) => handleOrientation(e, true);
  const onDO  = (e) => handleOrientation(e, false);

  // Reactive calculations
  $: distanceMeters = currentPosition
    ? haversineMeters(
        currentPosition.coords.latitude,
        currentPosition.coords.longitude,
        target.lat,
        target.lon
      )
    : null;

  $: insideZone = (distanceMeters ?? Infinity) <= 15;

  $: bearingDeg = currentPosition
    ? initialBearing(
        currentPosition.coords.latitude,
        currentPosition.coords.longitude,
        target.lat,
        target.lon
      )
    : null;

  $: turnDeg = (bearingDeg != null && headingDeg != null)
    ? ((bearingDeg - headingDeg + 360) % 360)
    : null;

  // --- Geolocation ---------------------------------------------------------
  function startWatching() {
    errorMessage = null;
    if (!('geolocation' in navigator)) {
      errorMessage = 'Geolocation is not supported in this browser.';
      return;
    }

    checkPermission();

    const opts = { enableHighAccuracy: highAccuracy, maximumAge: 0, timeout: 10000 };
    watching = true;
    watchId = navigator.geolocation.watchPosition(
      (pos) => {
        currentPosition = pos;
        // Fallback heading from motion (if deviceorientation not available)
        if (typeof pos.coords.heading === 'number' && !Number.isNaN(pos.coords.heading)) {
          headingDeg = (pos.coords.heading + 360) % 360; // already clockwise from North
        }
      },
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
      } else permissionState = 'unknown';
    } catch { permissionState = 'unknown'; }
  }

  function useCurrentAsTarget() {
    if (!currentPosition) return;
    target = { lat: currentPosition.coords.latitude, lon: currentPosition.coords.longitude };
    if (targetMarker) targetMarker.setLatLng([target.lat, target.lon]);
    if (map && currentPosition) fitBoundsToUserAndTarget();
  }

  // --- Device orientation (compass) ---------------------------------------
  function handleOrientation(e, absoluteEvent) {
    // iOS Safari: webkitCompassHeading (0..360, 0=N) — most reliable on iOS
    if (typeof e.webkitCompassHeading === 'number') {
      headingDeg = (e.webkitCompassHeading + 360) % 360;
      return;
    }

    // Prefer absolute if available (Android Chrome may fire deviceorientationabsolute)
    if (absoluteEvent && typeof e.alpha === 'number') {
      usingAbs = true;
      // Convert to compass: 0=N, clockwise positive
      headingDeg = (360 - e.alpha + 360) % 360;
      return;
    }

    // If not absolute, try regular deviceorientation with absolute flag
    if (!absoluteEvent && e.absolute === true && typeof e.alpha === 'number') {
      usingAbs = true;
      headingDeg = (360 - e.alpha + 360) % 360;
      return;
    }

    // Last resort: non-absolute alpha; better than nothing
    if (typeof e.alpha === 'number') {
      usingAbs = false;
      headingDeg = (360 - e.alpha + 360) % 360;
    }
  }

  function enableCompass() {
    // Remove any previous listeners to avoid duplicates
    window.removeEventListener('deviceorientationabsolute', onDOA, true);
    window.removeEventListener('deviceorientation', onDO, true);

    try {
      const D = window.DeviceOrientationEvent;
      const add = () => {
        // Listen to both; whichever yields good data first wins
        window.addEventListener('deviceorientationabsolute', onDOA, true);
        window.addEventListener('deviceorientation', onDO, true);
      };

      // iOS permission gate (must be triggered by user gesture)
      if (D && typeof D.requestPermission === 'function') {
        D.requestPermission().then((res) => { if (res === 'granted') add(); });
      } else {
        add();
      }
    } catch { /* ignore */ }
  }

  // --- Leaflet Map ---------------------------------------------------------
  async function ensureMap() {
    if (map) return;
    const mod = await import('leaflet'); // client-only
    L = mod.default ?? mod;

    map = L.map(mapEl, { center: [target.lat, target.lon], zoom: 15, scrollWheelZoom: true });

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { attribution: '© OpenStreetMap' })
      .addTo(map);

    targetMarker = L.circleMarker([target.lat, target.lon], {
      radius: 8, color: '#22c55e', fillColor: '#22c55e', fillOpacity: 0.9
    }).addTo(map).bindPopup('🎯 Target');

    if (currentPosition) {
      updateUserMarker();
      fitBoundsToUserAndTarget();
    }
  }

  function updateUserMarker() {
    if (!L || !map || !currentPosition) return;
    const here = [currentPosition.coords.latitude, currentPosition.coords.longitude];

    if (!userMarker) {
      userMarker = L.circleMarker(here, {
        radius: 8, color: '#38bdf8', fillColor: '#38bdf8', fillOpacity: 0.9
      }).addTo(map).bindPopup('📍 You');
    } else {
      userMarker.setLatLng(here);
    }

    if (lineLayer) lineLayer.remove();
    lineLayer = L.polyline([here, [target.lat, target.lon]], { color: '#94a3b8', weight: 2 }).addTo(map);
  }

  function fitBoundsToUserAndTarget() {
    if (!L || !map || !currentPosition) return;
    const here = [currentPosition.coords.latitude, currentPosition.coords.longitude];
    const dest = [target.lat, target.lon];
    map.fitBounds(L.latLngBounds([here, dest]), { padding: [24, 24] });
  }

  // Keep user marker in sync with position
  $: if (map && currentPosition) updateUserMarker();

  onMount(() => {
    checkPermission();
    ensureMap();
  });

  onDestroy(() => {
    stopWatching();
    window.removeEventListener('deviceorientationabsolute', onDOA, true);
    window.removeEventListener('deviceorientation', onDO, true);
  });

  // --- Utilities -----------------------------------------------------------
  function toFixed(n, digits = 2) { return (n == null) ? '—' : Number(n).toFixed(digits); }

  function humanizeGeoError(err) {
    switch (err.code) {
      case err.PERMISSION_DENIED: return 'Permission denied. Please allow location access.';
      case err.POSITION_UNAVAILABLE: return 'Position unavailable. Try moving to an open area or check your connection.';
      case err.TIMEOUT: return 'Location request timed out. Try again.';
      default: return 'An unknown geolocation error occurred.';
    }
  }

  // Haversine distance in meters
  function haversineMeters(lat1, lon1, lat2, lon2) {
    const toRad = (d) => (d * Math.PI) / 180;
    const R = 6371000;
    const dLat = toRad(lat2 - lat1);
    const dLon = toRad(lon2 - lon1);
    const a = Math.sin(dLat/2)**2 + Math.cos(toRad(lat1))*Math.cos(toRad(lat2))*Math.sin(dLon/2)**2;
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return R * c;
  }

  // Initial bearing (degrees from North, clockwise)
  function initialBearing(lat1, lon1, lat2, lon2) {
    const toRad = (d) => (d * Math.PI) / 180;
    const toDeg = (r) => (r * 180) / Math.PI;
    const φ1 = toRad(lat1), φ2 = toRad(lat2), Δλ = toRad(lon2 - lon1);
    const y = Math.sin(Δλ) * Math.cos(φ2);
    const x = Math.cos(φ1) * Math.sin(φ2) - Math.sin(φ1) * Math.cos(φ2) * Math.cos(Δλ);
    return (toDeg(Math.atan2(y, x)) + 360) % 360;
  }

  function degToCompass(d) {
    if (d == null) return '–';
    const dirs = ['N','NNE','NE','ENE','E','ESE','SE','SSE','S','SSW','SW','WSW','W','WNW','NW','NNW'];
    return dirs[Math.round(d / 22.5) % 16];
  }
</script>

<svelte:head>
  <!-- Leaflet CSS -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
</svelte:head>

<!-- Layout -->
<div class="min-h-screen text-white
            bg-[radial-gradient(1200px_600px_at_80%_-10%,rgba(99,102,241,.35),transparent_60%),radial-gradient(1000px_500px_at_-10%_20%,rgba(236,72,153,.35),transparent_60%),linear-gradient(180deg,#0f172a_0%,#111827_60%,#0b1220_100%)]">
  <div class="mx-auto max-w-5xl p-6 md:p-8 lg:p-10">
    <!-- Header -->
    <header class="mb-6 flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
      <div>
        <h1 class="text-2xl font-semibold tracking-tight text-transparent bg-clip-text bg-gradient-to-r from-white to-white/60">
          Geofence Proximity Tracker
        </h1>
        <p class="text-sm text-white/70">Map • Compass • 15 m zone</p>
      </div>
      <div class="flex items-center gap-2">
        {#if !watching}
          <button
            class="rounded-2xl px-4 py-2 text-sm font-semibold
                   bg-gradient-to-r from-violet-400 via-sky-400 to-emerald-400
                   text-slate-900 shadow-lg shadow-black/30 ring-1 ring-white/20
                   hover:brightness-110 transition"
            on:click={() => { startWatching(); ensureMap(); }}>
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
        <label class="flex items-center gap-2 text-xs text-white/80">
          <input type="checkbox" bind:checked={highAccuracy} class="h-4 w-4 rounded accent-sky-400" /> High accuracy
        </label>
      </div>
    </header>

    <!-- Map + Compass -->
    <div class="grid grid-cols-1 lg:grid-cols-3 gap-6 mb-6">
      <!-- Map -->
      <section class="lg:col-span-2 rounded-2xl bg-white/10 ring-1 ring-white/15 p-4 md:p-6 backdrop-blur-md shadow-xl shadow-black/20">
        <h2 class="mb-3 text-sm font-semibold tracking-wide text-white/90">Map</h2>
        <div bind:this={mapEl} class="h-[46vh] md:h-[56vh] w-full rounded-xl overflow-hidden"></div>
        <div class="mt-3 text-xs text-white/70">Blue = You · Green = Target · Grey = Path</div>
      </section>

      <!-- Compass -->
      <section class="rounded-2xl bg-white/10 ring-1 ring-white/15 p-4 md:p-6 backdrop-blur-md shadow-xl shadow-black/20">
        <h2 class="mb-3 text-sm font-semibold tracking-wide text-white/90">Compass</h2>
        <div class="flex items-center justify-center">
          <div class="relative h-48 w-48 md:h-56 md:w-56 rounded-full border-2 border-dashed border-white/30 shadow-2xl shadow-black/40">
            <div class="absolute left-1/2 top-1 -translate-x-1/2 text-xs text-white/70">N</div>
            <!-- Bearing arrow -->
            <div
              class="absolute left-1/2 top-1/2 h-20 w-0.5 -translate-x-1/2 -translate-y-full origin-bottom bg-gradient-to-b from-emerald-300 to-emerald-600"
              style={`transform: translate(-50%, -100%) rotate(${bearingDeg ?? 0}deg);`}>
            </div>
            <!-- Heading needle -->
            <div
              class="absolute left-1/2 top-1/2 h-24 w-[3px] -translate-x-1/2 -translate-y-full origin-bottom bg-gradient-to-b from-sky-300 to-sky-600"
              style={`transform: translate(-50%, -100%) rotate(${headingDeg ?? 0}deg);`}>
            </div>
          </div>
        </div>
        <div class="mt-4 grid grid-cols-2 gap-3 text-sm">
          <div class="rounded-xl border border-white/15 bg-white/5 p-3">
            <div class="text-xs text-white/70">Bearing to target</div>
            <div class="mt-1 font-semibold">{bearingDeg == null ? '—' : `${Math.round(bearingDeg)}° (${degToCompass(bearingDeg)})`}</div>
          </div>
          <div class="rounded-2xl border border-white/15 bg-white/5 p-3">
            <div class="text-xs text-white/70">Turn</div>
            <div class="mt-1 font-semibold">
              {turnDeg == null ? '—' : `${Math.round(turnDeg)}° ${turnDeg <= 180 ? 'right →' : 'left ←'}`}
            </div>
          </div>
        </div>
        <div class="mt-4 flex flex-wrap gap-2">
          <button class="rounded-xl px-4 py-2 text-sm font-semibold bg-white/10 ring-1 ring-white/20 hover:bg-white/15 transition" on:click={enableCompass}>
            Enable/Refresh Compass
          </button>
          <span class="text-xs text-white/70">
            {headingDeg == null
              ? 'Tap “Enable/Refresh Compass” and move the device slightly. On iOS, grant motion access.'
              : usingAbs
                ? 'Compass: absolute mode'
                : 'Compass: relative/fallback mode'}
          </span>
        </div>
      </section>
    </div>

    <!-- Target inputs + Live status -->
    <section class="grid grid-cols-1 lg:grid-cols-2 gap-6">
      <div class="rounded-2xl bg-white/10 ring-1 ring-white/15 p-4 md:p-6 backdrop-blur-md shadow-xl shadow-black/20">
        <h2 class="mb-3 text-sm font-semibold tracking-wide text-white/90">Target</h2>
        <div class="grid grid-cols-1 gap-3 sm:grid-cols-3">
          <label class="flex flex-col sm:col-span-1">
            <span class="text-xs text-white/70">Latitude</span>
            <input class="mt-1 rounded-xl border border-white/20 bg-white/10 px-3 py-2 text-sm text-white placeholder-white/50 focus:outline-none focus:ring-2 focus:ring-sky-400"
                   type="number" step="any" bind:value={target.lat} />
          </label>
          <label class="flex flex-col sm:col-span-1">
            <span class="text-xs text-white/70">Longitude</span>
            <input class="mt-1 rounded-xl border border-white/20 bg-white/10 px-3 py-2 text-sm text-white placeholder-white/50 focus:outline-none focus:ring-2 focus:ring-sky-400"
                   type="number" step="any" bind:value={target.lon} />
          </label>
          <div class="flex items-end sm:col-span-1">
            <button class="w-full rounded-xl bg-gradient-to-r from-cyan-400 to-lime-400 px-3 py-2 text-sm font-semibold text-slate-900 ring-1 ring-white/25 hover:brightness-110 disabled:opacity-50"
                    on:click={useCurrentAsTarget}
                    disabled={!currentPosition}>
              Use my current position
            </button>
          </div>
        </div>
      </div>

      <div class="rounded-2xl bg-white/10 ring-1 ring-white/15 p-4 md:p-6 backdrop-blur-md shadow-xl shadow-black/20">
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
      </div>
    </section>

    <!-- Permissions & errors -->
    {#if permissionState === 'denied'}
      <div class="mt-6 rounded-xl border border-rose-300/50 bg-rose-500/20 p-3">
        Location permission is denied. Enable it in your browser settings, then click <em>Start</em>.
      </div>
    {/if}
    {#if errorMessage}
      <div class="mt-4 rounded-xl border border-amber-300/50 bg-amber-400/20 p-3">{errorMessage}</div>
    {/if}

    <!-- Footer -->
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
