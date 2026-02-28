// ══════════════════════════════════════════════
// FESTUS SACCO — Service Worker v1
// ══════════════════════════════════════════════
const CACHE_NAME   = 'festus-sacco-v1';
const STATIC_CACHE = 'festus-static-v1';

// Core app shell — cached on install
const APP_SHELL = [
  '/festus-v7.html',
  '/manifest.json',
];

// CDN resources to cache on first fetch
const CDN_ORIGINS = [
  'https://fonts.googleapis.com',
  'https://fonts.gstatic.com',
  'https://cdnjs.cloudflare.com',
  'https://www.gstatic.com',   // Firebase SDKs
];

// ── Install: cache app shell ───────────────────
self.addEventListener('install', e => {
  e.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(APP_SHELL))
      .then(() => self.skipWaiting())
  );
});

// ── Activate: clean up old caches ─────────────
self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys.filter(k => k !== CACHE_NAME && k !== STATIC_CACHE)
            .map(k => caches.delete(k))
      )
    ).then(() => self.clients.claim())
  );
});

// ── Fetch: cache-first for app, network-first for Firebase ──
self.addEventListener('fetch', e => {
  const url = new URL(e.request.url);

  // Always go to network for Firebase Firestore API calls
  if (url.hostname.includes('firestore.googleapis.com') ||
      url.hostname.includes('firebase') ||
      url.pathname.includes('/v1/projects/')) {
    e.respondWith(fetch(e.request).catch(() => new Response('', {status: 503})));
    return;
  }

  // Network-first for the main HTML (so updates deploy instantly)
  if (url.pathname.endsWith('.html') || url.pathname === '/') {
    e.respondWith(
      fetch(e.request)
        .then(res => {
          const clone = res.clone();
          caches.open(CACHE_NAME).then(cache => cache.put(e.request, clone));
          return res;
        })
        .catch(() => caches.match(e.request))
    );
    return;
  }

  // Cache-first for CDN resources (fonts, chart.js, etc.)
  if (CDN_ORIGINS.some(o => e.request.url.startsWith(o))) {
    e.respondWith(
      caches.match(e.request).then(cached => {
        if (cached) return cached;
        return fetch(e.request).then(res => {
          const clone = res.clone();
          caches.open(STATIC_CACHE).then(cache => cache.put(e.request, clone));
          return res;
        });
      })
    );
    return;
  }

  // Default: try network, fall back to cache
  e.respondWith(
    fetch(e.request).catch(() => caches.match(e.request))
  );
});

// ── Background sync: flush offline queue when back online ──
self.addEventListener('sync', e => {
  if (e.tag === 'sync-trips') {
    // The app handles this itself via flushQueue()
    // This is a hook for future background sync enhancement
    console.log('[SW] Background sync triggered');
  }
});

// ── Push notifications (future use) ───────────
self.addEventListener('push', e => {
  if (!e.data) return;
  const data = e.data.json();
  self.registration.showNotification(data.title || 'FESTUS SACCO', {
    body: data.body || '',
    icon: '/icons/icon-192.png',
    badge: '/icons/badge-72.png',
    data: data,
  });
});
