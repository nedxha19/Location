<script>
  import { onMount } from "svelte";

  let batteryLevel = null;
  let isCharging = null;

  async function getBatteryStatus() {
    if ("getBattery" in navigator) {
      const battery = await navigator.getBattery();

      // Set initial values
      batteryLevel = Math.round(battery.level * 100);
      isCharging = battery.charging;

      // Update on change
      battery.addEventListener("levelchange", () => {
        batteryLevel = Math.round(battery.level * 100);
      });
      battery.addEventListener("chargingchange", () => {
        isCharging = battery.charging;
      });
    } else {
      batteryLevel = -1; // means not supported
      isCharging = null;
    }
  }

  onMount(() => {
    getBatteryStatus();
  });
</script>

<main class="min-h-screen flex flex-col items-center justify-center bg-gray-100 p-6">
  <div class="bg-white shadow-lg rounded-2xl p-6 w-full max-w-md text-center">
    <h1 class="text-2xl font-bold text-gray-800 mb-4">🔋 Battery Management</h1>
    <hr class="mb-4">

    {#if batteryLevel === -1}
      <p class="text-red-500">Battery API not supported on this device.</p>
    {:else if batteryLevel !== null && isCharging !== null}
      <p class="text-lg text-gray-700 mb-4">
        Charging Level: <span class="font-semibold">{batteryLevel}%</span><br>
        Is Charging: 
        <span class="font-semibold {isCharging ? 'text-green-600' : 'text-gray-600'}">
          {isCharging ? "Yes" : "No"}
        </span>
      </p>

      <!-- Battery Bar -->
      <div class="w-full bg-gray-200 rounded-full h-6 overflow-hidden">
        <div 
          class="h-6 rounded-full transition-all duration-500"
          class:bg-green-500={batteryLevel > 60}
          class:bg-yellow-400={batteryLevel > 20 && batteryLevel <= 60}
          class:bg-red-500={batteryLevel <= 20}
          style="width: {batteryLevel}%"
        ></div>
      </div>
    {:else}
      <p class="text-gray-500">Fetching battery status...</p>
    {/if}
  </div>
</main>
