/* Hiillos — service worker.
   Network-first: with signal you always get the newest version, without it you
   get the last copy that loaded. Your logged data lives in localStorage and is
   never touched by any of this. */

var CACHE = "hiillos-v1";
var SHELL = ["./", "./index.html"];

self.addEventListener("install", function(e){
  // Precache the app shell now. Without this, the very first page load isn't
  // cached (the worker only takes control afterwards) and the app would be
  // blank the first time it's opened offline.
  e.waitUntil(
    caches.open(CACHE)
      .then(function(c){ return c.addAll(SHELL); })
      .catch(function(){ /* a missing path shouldn't block install */ })
      .then(function(){ return self.skipWaiting(); })
  );
});

self.addEventListener("activate", function(e){
  e.waitUntil(
    caches.keys().then(function(keys){
      return Promise.all(keys.filter(function(k){ return k !== CACHE; })
                             .map(function(k){ return caches.delete(k); }));
    }).then(function(){ return self.clients.claim(); })
  );
});

self.addEventListener("fetch", function(e){
  var req = e.request;
  if(req.method !== "GET") return;
  if(new URL(req.url).origin !== self.location.origin) return;

  e.respondWith(
    fetch(req).then(function(res){
      if(res && res.status === 200 && res.type === "basic"){
        var copy = res.clone();
        caches.open(CACHE).then(function(c){ c.put(req, copy); });
      }
      return res;
    }).catch(function(){
      return caches.match(req).then(function(hit){
        if(hit) return hit;
        // Nothing cached for this exact URL. For a page load, fall back to the
        // app shell so opening the app offline still works.
        if(req.mode === "navigate"){
          return caches.match("./index.html").then(function(shell){
            return shell || caches.match("./").then(function(root){
              return root || new Response(
                "<h1>Offline</h1><p>Open this once with a connection and it will work offline afterwards.</p>",
                {status:200, headers:{"Content-Type":"text/html; charset=utf-8"}});
            });
          });
        }
        return new Response("Offline", {status:503, statusText:"Offline"});
      });
    })
  );
});
