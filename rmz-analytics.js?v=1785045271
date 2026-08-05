/**
 * RMZ Analytics - First-party analytics tracking script
 * Lightweight (~4KB gzipped), auto-loaded on storefronts
 *
 * Tracks: Pageviews, SPA navigation, time on page, scroll depth,
 *         outbound clicks, and custom events via window.rmzAnalytics.track()
 *
 * Hooks into RMZPixel for e-commerce event tracking.
 */
(function() {
  'use strict';

  var ENDPOINT = '/api/storefront/analytics/collect';
  var BATCH_ENDPOINT = '/api/storefront/analytics/collect/batch';
  var RATE_LIMIT = 120; // max events per minute (increased for batching)
  var BATCH_FLUSH_INTERVAL = 5000; // flush batch every 5 seconds
  var MAX_BATCH_SIZE = 20; // max events before forced flush

  var storeEl = document.querySelector('script[data-store]');
  var storeId = storeEl ? storeEl.getAttribute('data-store') : null;

  if (!storeId) return;

  // Respect DNT
  if (navigator.doNotTrack === '1' || window.doNotTrack === '1') return;

  var sessionId = getSessionId();
  var visitorId = getVisitorId();
  var eventCount = 0;
  var eventCountResetTime = Date.now();
  var currentPath = location.pathname;

  // ==================== Event Batching Queue ====================

  var eventQueue = [];
  var batchTimer = null;

  var MAX_QUEUE_SIZE = 200; // Hard cap to prevent unbounded memory growth

  function queueEvent(data) {
    if (isRateLimited()) return;
    if (eventQueue.length >= MAX_QUEUE_SIZE) return; // Backpressure: drop if queue is full
    eventCount++;

    data.store_id = storeId;
    data.session_id = sessionId;
    data.visitor_id = visitorId;
    data.url = location.href;
    data.path = location.pathname;
    data.referrer = document.referrer || '';
    data.screen_width = screen.width;
    data.timestamp = new Date().toISOString();

    eventQueue.push(data);

    // Flush immediately if queue is full
    if (eventQueue.length >= MAX_BATCH_SIZE) {
      flushQueue();
    } else if (!batchTimer) {
      batchTimer = setTimeout(flushQueue, BATCH_FLUSH_INTERVAL);
    }
  }

  var retryQueue = [];
  var retryTimer = null;
  var MAX_RETRY_QUEUE = 100;

  function flushQueue() {
    if (batchTimer) {
      clearTimeout(batchTimer);
      batchTimer = null;
    }

    if (eventQueue.length === 0) return;

    var events = eventQueue.splice(0);
    var payload = JSON.stringify({ events: events, store_id: storeId });

    // sendBeacon is fire-and-forget (no status code), use XHR when possible for retry
    if (document.visibilityState === 'hidden' && navigator.sendBeacon) {
      // Page is hiding — must use sendBeacon (XHR gets cancelled)
      var sent = navigator.sendBeacon(BATCH_ENDPOINT, new Blob([payload], { type: 'application/json' }));
      if (!sent) {
        // sendBeacon failed (browser rejected) — push to retry queue
        for (var k = 0; k < events.length && retryQueue.length < MAX_RETRY_QUEUE; k++) {
          retryQueue.push(events[k]);
        }
      }
    } else {
      var xhr = new XMLHttpRequest();
      xhr.open('POST', BATCH_ENDPOINT, true);
      xhr.setRequestHeader('Content-Type', 'application/json');
      xhr.timeout = 10000; // 10s timeout
      xhr.onreadystatechange = function() {
        if (xhr.readyState === 4) {
          if (xhr.status === 429 || xhr.status >= 500) {
            // Rate limited or server error — re-queue for retry
            if (retryQueue.length < MAX_RETRY_QUEUE) {
              for (var i = 0; i < events.length && retryQueue.length < MAX_RETRY_QUEUE; i++) {
                retryQueue.push(events[i]);
              }
              scheduleRetry();
            }
          }
        }
      };
      xhr.ontimeout = function() {
        // Timeout — re-queue for retry
        if (retryQueue.length < MAX_RETRY_QUEUE) {
          for (var j = 0; j < events.length && retryQueue.length < MAX_RETRY_QUEUE; j++) {
            retryQueue.push(events[j]);
          }
          scheduleRetry();
        }
      };
      xhr.send(payload);
    }
  }

  function scheduleRetry() {
    if (retryTimer || retryQueue.length === 0) return;
    // Retry after 30 seconds
    retryTimer = setTimeout(function() {
      retryTimer = null;
      if (retryQueue.length === 0) return;
      var events = retryQueue.splice(0, MAX_BATCH_SIZE);
      var payload = JSON.stringify({ events: events, store_id: storeId });
      var xhr = new XMLHttpRequest();
      xhr.open('POST', BATCH_ENDPOINT, true);
      xhr.setRequestHeader('Content-Type', 'application/json');
      xhr.onreadystatechange = function() {
        if (xhr.readyState === 4 && xhr.status === 429) {
          // Still rate limited — drop to avoid infinite retry loop
          retryQueue = [];
        } else if (xhr.readyState === 4 && retryQueue.length > 0) {
          scheduleRetry();
        }
      };
      xhr.send(payload);
    }, 30000);
  }

  // ==================== Utilities ====================

  function generateId() {
    try {
      var arr = new Uint8Array(16);
      crypto.getRandomValues(arr);
      return Array.from(arr, function(b) { return b.toString(16).padStart(2, '0'); }).join('');
    } catch (e) {
      // Fallback for older browsers or non-HTTPS contexts
      var id = '';
      for (var i = 0; i < 32; i++) {
        id += Math.floor(Math.random() * 16).toString(16);
      }
      return id;
    }
  }

  function getSessionId() {
    var key = 'rmz_sid';
    var sid = sessionStorage.getItem(key);
    if (!sid) {
      sid = generateId();
      sessionStorage.setItem(key, sid);
    }
    return sid;
  }

  function getVisitorId() {
    var key = 'rmz_vid';
    var vid = localStorage.getItem(key);
    if (!vid) {
      vid = generateId();
      localStorage.setItem(key, vid);
    }
    return vid;
  }

  function isRateLimited() {
    var now = Date.now();
    if (now - eventCountResetTime > 60000) {
      eventCount = 0;
      eventCountResetTime = now;
    }
    return eventCount >= RATE_LIMIT;
  }

  /**
   * Queue an event for batched sending (default for most events).
   * Use sendImmediate() for page-unload events that can't wait.
   */
  function send(data) {
    queueEvent(data);
  }

  /**
   * Send immediately via sendBeacon — for visibilitychange/unload events
   * that need to fire before the page goes away.
   */
  function sendImmediate(data) {
    if (isRateLimited()) return;
    eventCount++;

    data.store_id = storeId;
    data.session_id = sessionId;
    data.visitor_id = visitorId;
    data.url = location.href;
    data.path = location.pathname;
    data.referrer = document.referrer || '';
    data.screen_width = screen.width;
    data.timestamp = new Date().toISOString();

    // Flush any queued events first, then send this one
    flushQueue();

    var payload = JSON.stringify(data);
    if (navigator.sendBeacon) {
      var sent = navigator.sendBeacon(ENDPOINT, new Blob([payload], { type: 'application/json' }));
      if (!sent) {
        // Fallback to XHR if sendBeacon was rejected
        var xhr = new XMLHttpRequest();
        xhr.open('POST', ENDPOINT, true);
        xhr.setRequestHeader('Content-Type', 'application/json');
        xhr.send(payload);
      }
    } else {
      var xhr2 = new XMLHttpRequest();
      xhr2.open('POST', ENDPOINT, true);
      xhr2.setRequestHeader('Content-Type', 'application/json');
      xhr2.send(payload);
    }
  }

  // ==================== Pageview Tracking ====================

  function trackPageview() {
    currentPath = location.pathname;

    send({
      type: 'pageview',
      title: document.title,
    });
  }

  // ==================== SPA Navigation (History API) ====================

  var originalPushState = history.pushState;
  var originalReplaceState = history.replaceState;

  history.pushState = function() {
    originalPushState.apply(this, arguments);
    onNavigation();
  };

  history.replaceState = function() {
    originalReplaceState.apply(this, arguments);
    onNavigation();
  };

  window.addEventListener('popstate', onNavigation);

  function onNavigation() {
    if (location.pathname !== currentPath) {
      trackPageview();
    }
  }

  // Flush queued events when page is hiding
  document.addEventListener('visibilitychange', function() {
    if (document.visibilityState === 'hidden') {
      flushQueue();
    }
  });

  // ==================== Outbound Link Clicks ====================

  document.addEventListener('click', function(e) {
    var link = e.target.closest('a');
    if (!link || !link.href) return;

    try {
      var url = new URL(link.href);
      if (url.hostname !== location.hostname) {
        send({
          type: 'outbound_click',
          destination: link.href,
          text: (link.textContent || '').trim().substring(0, 100),
        });
      }
    } catch (err) {
      // Invalid URL, skip
    }
  }, true);

  // ==================== Custom Event API ====================

  window.rmzAnalytics = window.rmzAnalytics || {};

  /**
   * Track a custom event
   * @param {string} eventName - Event name (e.g., 'add_to_cart', 'search')
   * @param {object} properties - Optional event properties
   */
  window.rmzAnalytics.track = function(eventName, properties) {
    if (!eventName || typeof eventName !== 'string') return;

    send({
      type: 'event',
      event_name: eventName.substring(0, 100),
      properties: properties || {},
    });
  };

  // ==================== RMZPixel Hook ====================

  // Hook RMZPixel.purchase() to fire first-party analytics.
  // Purchase is called from theme success pages and does NOT go through
  // rmz-core-ui's pixelTrackingService, so we must hook it here.
  //
  // All other e-commerce events (add_to_cart, initiate_checkout, search,
  // lead, complete_registration, view_content) are now tracked directly
  // by rmz-core-ui's pixelTrackingService → window.rmzAnalytics.track().
  function hookRMZPixel() {
    if (!window.RMZPixel) return;

    // Hook viewContent — called from theme product page templates, not rmz-core-ui
    var origViewContent = window.RMZPixel.viewContent;
    window.RMZPixel.viewContent = function(product) {
      origViewContent.call(window.RMZPixel, product);
      window.rmzAnalytics.track('view_content', {
        product_id: product.id,
        product_name: product.name,
        price: product.price,
        currency: product.currency || 'SAR',
        category: product.category || '',
      });
    };

    // Hook purchase — called from theme success page templates, not rmz-core-ui
    var origPurchase = window.RMZPixel.purchase;
    window.RMZPixel.purchase = function(data) {
      origPurchase.call(window.RMZPixel, data);
      window.rmzAnalytics.track('purchase', {
        order_id: data.orderId,
        value: data.value || 0,
        currency: data.currency || 'SAR',
        num_products: data.products ? data.products.length : 0,
      });
    };
  }

  // ==================== Auto-Detect E-Commerce Events ====================

  /**
   * Automatically track e-commerce events that RMZPixel doesn't cover
   * by observing URL patterns, form submissions, and DOM interactions.
   */
  function trackAutoEvents() {
    var path = location.pathname;

    // Track cart view
    if (path === '/cart' || path === '/cart/') {
      send({ type: 'event', event_name: 'view_cart', properties: {} });
    }

    // Note: initiate_checkout and search are tracked by rmz-core-ui's
    // pixelTrackingService directly — no auto-detection needed here.

    // Track auth events — hook into RMZ.showAuth if available
    if (window.RMZ && window.RMZ.showAuth) {
      var origShowAuth = window.RMZ.showAuth;
      window.RMZ.showAuth = function() {
        send({ type: 'event', event_name: 'auth_modal_opened', properties: {} });
        return origShowAuth.apply(this, arguments);
      };
    }

    // Track login/register form submissions
    document.addEventListener('submit', function(e) {
      var form = e.target;
      var action = (form.getAttribute('action') || '').toLowerCase();

      if (action.indexOf('create-auth') !== -1 || action.indexOf('init-auth') !== -1) {
        send({ type: 'event', event_name: 'login_attempt', properties: {} });
      }

      if (action.indexOf('verify-auth') !== -1) {
        send({ type: 'event', event_name: 'auth_verify', properties: {} });
      }
    }, true);

    // Track coupon usage
    document.addEventListener('submit', function(e) {
      var form = e.target;
      var action = (form.getAttribute('action') || '').toLowerCase();

      if (action.indexOf('coupon') !== -1) {
        var couponInput = form.querySelector('input[name="code"], input[name="coupon"]');
        var couponCode = couponInput ? couponInput.value : '';
        send({
          type: 'event',
          event_name: 'apply_coupon',
          properties: { coupon_code: couponCode },
        });
      }
    }, true);
  }

  // ==================== Initialize ====================

  // Track initial pageview
  trackPageview();

  // Auto-detect e-commerce events
  trackAutoEvents();

  // Hook RMZPixel (may load after this script)
  if (window.RMZPixel) {
    hookRMZPixel();
  } else {
    // Wait for RMZPixel to load
    var checkInterval = setInterval(function() {
      if (window.RMZPixel) {
        hookRMZPixel();
        clearInterval(checkInterval);
      }
    }, 100);
    // Give up after 10 seconds
    setTimeout(function() { clearInterval(checkInterval); }, 10000);
  }

})();
