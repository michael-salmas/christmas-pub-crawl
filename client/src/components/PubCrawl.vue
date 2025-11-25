<template>
  <div class="pubcrawl-container">
    <div ref="emojiLayer" class="emoji-layer"></div>

    <header class="header">
      <h1 class="title">🎄 6 Pubs of Xmas 🎅</h1>
    </header>

    <div class="content">

      <transition name="card-fade" mode="out-in">
      <!-- Pre-crawl countdown -->
      <div v-if="!crawlStarted" class="pre-crawl-card">
        <h2>🎁 Pub Crawl Begins Soon!</h2>
        <p class="pre-countdown">
          Starts in: <strong>{{ preCountdown }}</strong>
        </p>
        <a
          v-if="firstPub?.mapUrl"
          :href="firstPub.mapUrl"
          target="_blank"
          rel="noopener"
          class="map-btn"
        >
          📍 Directions to {{ firstPub.name }}
        </a>
      </div>

        <!-- Main crawl -->
        <div
          v-else-if="currentPub"
          :key="currentPub.name"
          class="card pub-card"
        >
          <h2 class="pub-name">{{ currentPub.name }}</h2>

          <p class="time-range">
            ⏰ {{ formatTime(currentPub.start) }} → {{ formatTime(currentPub.end) }}
          </p>

          <a
            v-if="currentPub.mapUrl"
            :href="currentPub.mapUrl"
            target="_blank"
            rel="noopener"
            class="map-btn"
          >
            📍 Directions to this pub
          </a>

          <h3 class="rules-title">Rules for This Pub</h3>
          <ul class="rules">
            <li v-for="rule in currentPub.rules" :key="rule">
              🎄 {{ rule }}
            </li>
          </ul>

          <div class="next-section" v-if="nextPub">
            <p class="countdown">
              ⏳ Next pub in: <strong>{{ countdown }}</strong>
            </p>
            <a
              v-if="nextPub.mapUrl"
              :href="nextPub.mapUrl"
              target="_blank"
              rel="noopener"
              class="next-map-btn"
            >
              👉 Directions to {{ nextPub.name }}
            </a>
          </div>
        </div>

        <div v-else :key="'upcoming'" class="card upcoming-card">
          <h2>🎁 Crawl Hasn't Started</h2>
          <p>Get cozy — the first pub begins soon!</p>
        </div>
      </transition>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";

const pubs = ref([]);
const currentPub = ref(null);
const nextPub = ref(null);
const countdown = ref("00:00");
const preCountdown = ref("");
const crawlStarted = ref(false);
const emojiLayer = ref(null);
let lastPubName = null;
const firstPub = ref(null);
let preCrawlSnowRunning = false;

async function loadSchedule() {
  const res = await fetch("/schedule.json");
  pubs.value = await res.json();
  if (pubs.value.length) firstPub.value = pubs.value[0];

  updatePubStatus();
  setInterval(updatePubStatus, 1000);
}

function updatePubStatus() {
  const now = new Date();
  const active = pubs.value.find(
    (p) => new Date(p.start) <= now && now < new Date(p.end)
  );
  const upcoming = pubs.value.find((p) => new Date(p.start) > now);

  crawlStarted.value = !!active;

  currentPub.value = active || null;
  nextPub.value = upcoming || null;

  if (!crawlStarted.value && firstPub.value) {
    updatePreCountdown(now, firstPub.value.start);
  } else {
    updateCountdown(now, upcoming);
  }

  // Trigger short emoji fall when pub changes
  if (active && active.name !== lastPubName) {
    lastPubName = active.name;
    triggerEmojiFallOnce();
  }

  if (!crawlStarted.value && firstPub.value) {
    updatePreCountdown(now, firstPub.value.start);
    if (!preCrawlSnowRunning) {
      startCalmPreCrawlSnow();
      preCrawlSnowRunning = true;
    }
  } else {
    // crawl has started, stop snow
    stopCalmPreCrawlSnow();
  }
}

function updatePreCountdown(now, startTime) {
  const diff = new Date(startTime) - now;
  if (diff <= 0) {
    preCountdown.value = "Starting now!";
    return;
  }
  const days = Math.floor(diff / (1000 * 60 * 60 * 24));
  const hrs = Math.floor((diff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
  const mins = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60));
  const secs = Math.floor((diff % (1000 * 60 )) / (1000 ))

  preCountdown.value = `${days}d ${hrs}h ${mins}m ${secs}s`;
}

function updateCountdown(now, upcoming) {
  if (!upcoming) {
    countdown.value = "—";
    return;
  }
  const diff = new Date(upcoming.start) - now;
  if (diff <= 0) {
    countdown.value = "Starting now!";
    return;
  }

  const hrs = Math.floor(diff / 3600000);
  const mins = Math.floor((diff % 3600000) / 60000);
  const sec = Math.floor((diff % 60000) / 1000);

  countdown.value = `${hrs > 0 ? hrs + "h " : ""}${String(mins).padStart(2,"0")}m ${String(sec).padStart(2,"0")}s`;
}

function formatTime(t) {
  return new Date(t).toLocaleTimeString([], {
    hour: "2-digit",
    minute: "2-digit",
  });
}

/* --- STAGGERED FALLING EMOJIS WITH SWAY --- */
const EMOJI_LIST = ["🍺","❄️","❄️","❄️","❄️","❄️"];
function triggerEmojiFallOnce() {
  const layer = emojiLayer.value;
  for (let i = 0; i < 25; i++) {
    const delay = i * 100;
    setTimeout(() => {
      const span = document.createElement("span");
      span.className = "emoji";
      span.textContent = EMOJI_LIST[Math.floor(Math.random() * EMOJI_LIST.length)];
      span.style.left = Math.random() * 100 + "%";
      span.style.fontSize = 18 + Math.random() * 24 + "px";
      span.style.top = "-50px";
      span.style.opacity = "1";

      const swayAmplitude = 5 + Math.random() * 10;
      const duration = 4000;
      const startTime = performance.now();
      const startLeft = parseFloat(span.style.left);

      layer.appendChild(span);

      function fall(timestamp) {
        const elapsed = timestamp - startTime;
        const progress = Math.min(elapsed / duration, 1);
        span.style.top = progress * (window.innerHeight + 50) - 50 + "px";
        span.style.left = startLeft + swayAmplitude * Math.sin(progress * Math.PI * 4) + "%";
        span.style.opacity = 1 - progress;

        if (progress < 1) requestAnimationFrame(fall);
        else span.remove();
      }

      requestAnimationFrame(fall);
    }, delay);
  }
}
function startCalmPreCrawlSnow() {
  const layer = emojiLayer.value;
  if (!layer) return;

  function spawnEmoji() {
    const span = document.createElement("span");
    span.className = "emoji";
    span.textContent = EMOJI_LIST[Math.floor(Math.random() * EMOJI_LIST.length)];
    span.style.left = Math.random() * 100 + "%";
    span.style.fontSize = 16 + Math.random() * 20 + "px";
    span.style.top = "-50px";
    span.style.opacity = "1";

    const swayAmplitude = 10 + Math.random() * 10; // smaller sway for calm
    const swaySpeed = 0.001 + Math.random() * 0.001; // slower
    const duration = 8000 + Math.random() * 4000; // 8-12s fall duration
    const startTime = performance.now();
    const startLeft = parseFloat(span.style.left);

    layer.appendChild(span);

    function fall(timestamp) {
      const elapsed = timestamp - startTime;
      const progress = Math.min(elapsed / duration, 1);
      span.style.top = progress * (window.innerHeight + 50) - 50 + "px";
      span.style.left = startLeft + swayAmplitude * Math.sin(progress * Math.PI * 2) + "%";
      span.style.opacity = 1 - progress * 0.5; // fade slightly

      if (progress < 1) requestAnimationFrame(fall);
      else span.remove();
    }

    requestAnimationFrame(fall);
  }

  // Spawn a new emoji every 200ms
  preCrawlSnowInterval = setInterval(spawnEmoji, 200);
}

function stopCalmPreCrawlSnow() {
  if (preCrawlSnowInterval) {
    clearInterval(preCrawlSnowInterval);
    preCrawlSnowInterval = null;
  }
}

onMounted(() => {
  loadSchedule();
});
</script>
