import base from './worker-hybrid.js';

// v0.8.3: uploaded screenshot order is detected automatically before stitching.
const AUTO_ORDER_PATCH = String.raw`<script id="__factorAutoOrderV083">
(function () {
  'use strict';

  if (window.__factorAutoOrderV083) return;
  window.__factorAutoOrderV083 = true;

  const originalStitch = stitch;

  function sleepFrame() {
    return new Promise(function (resolve) {
      setTimeout(resolve, 0);
    });
  }

  function detailedDirectedOverlap(a, b, top, bottom) {
    const w = Math.min(a.width, b.width);
    const bodyH = Math.min(bottom - top, a.height - top, b.height - top);
    if (w < 80 || bodyH < 80) return { h: 0, score: 999, valid: false };

    const dataA = a.getContext('2d', { willReadFrequently: true }).getImageData(0, top, w, bodyH).data;
    const dataB = b.getContext('2d', { willReadFrequently: true }).getImageData(0, top, w, bodyH).data;
    const minH = Math.min(28, Math.max(10, Math.floor(bodyH * 0.07)));
    const maxH = Math.min(Math.floor(bodyH * 0.82), 720);
    let bestH = 0;
    let bestScore = 999;

    // Low-resolution ordering pass. Accurate overlap is recalculated after ordering.
    for (let h = minH; h <= maxH; h += 8) {
      const score = overlapScore(dataA, dataB, w, bodyH, h);
      if (score < bestScore) {
        bestH = h;
        bestScore = score;
      }
    }

    for (let h = Math.max(minH, bestH - 10); h <= Math.min(maxH, bestH + 10); h += 2) {
      const score = overlapScore(dataA, dataB, w, bodyH, h);
      if (score < bestScore) {
        bestH = h;
        bestScore = score;
      }
    }

    return {
      h: bestScore < 34 ? bestH : 0,
      score: bestScore,
      valid: bestScore < 34 && bestH > 0
    };
  }

  function edgeWeight(edge) {
    if (!edge || !edge.valid) return -10000000 - Math.min(999, edge ? edge.score : 999) * 100;
    return 1000000 + (34 - edge.score) * 10000 + Math.min(1000, edge.h);
  }

  function bestFullPath(edges) {
    const n = edges.length;
    if (n <= 1) return n ? [0] : [];

    // The UI normally uses 2-8 screenshots. This exact DP remains lightweight up to 12.
    if (n <= 12) {
      const size = 1 << n;
      const dp = Array.from({ length: size }, function () {
        const row = new Float64Array(n);
        row.fill(-Infinity);
        return row;
      });
      const parent = Array.from({ length: size }, function () {
        const row = new Int16Array(n);
        row.fill(-1);
        return row;
      });

      for (let i = 0; i < n; i++) dp[1 << i][i] = 0;

      for (let mask = 1; mask < size; mask++) {
        for (let last = 0; last < n; last++) {
          const current = dp[mask][last];
          if (!Number.isFinite(current)) continue;
          for (let next = 0; next < n; next++) {
            if (mask & (1 << next)) continue;
            const nextMask = mask | (1 << next);
            const value = current + edgeWeight(edges[last][next]);
            if (value > dp[nextMask][next]) {
              dp[nextMask][next] = value;
              parent[nextMask][next] = last;
            }
          }
        }
      }

      const full = size - 1;
      let last = 0;
      for (let i = 1; i < n; i++) {
        if (dp[full][i] > dp[full][last]) last = i;
      }

      const path = [];
      let mask = full;
      while (last >= 0) {
        path.push(last);
        const prev = parent[mask][last];
        mask &= ~(1 << last);
        last = prev;
      }
      return path.reverse();
    }

    // Safety fallback for an unusually large selection.
    let start = 0;
    let weakestIncoming = Infinity;
    for (let i = 0; i < n; i++) {
      let incoming = 0;
      for (let j = 0; j < n; j++) {
        if (i !== j && edges[j][i] && edges[j][i].valid) incoming++;
      }
      if (incoming < weakestIncoming) {
        weakestIncoming = incoming;
        start = i;
      }
    }

    const path = [start];
    const unused = new Set(Array.from({ length: n }, function (_, i) { return i; }));
    unused.delete(start);
    while (unused.size) {
      const last = path[path.length - 1];
      let next = null;
      let best = -Infinity;
      unused.forEach(function (candidate) {
        const value = edgeWeight(edges[last][candidate]);
        if (value > best) {
          best = value;
          next = candidate;
        }
      });
      path.push(next);
      unused.delete(next);
    }
    return path;
  }

  async function determineImageOrder() {
    const n = loaded.length;
    if (n <= 1) return { path: n ? [0] : [], edges: [], reliable: true };

    const sourceWidth = Math.min(1200, Math.min.apply(null, loaded.map(function (img) {
      return img.naturalWidth;
    })));
    const orderWidth = Math.min(620, sourceWidth);
    const scaleBack = sourceWidth / orderWidth;
    const canvases = loaded.map(function (img) {
      return scaledCanvas(img, orderWidth);
    });
    const viewport = detectScrollViewport(canvases);
    const top = viewport.top || 0;
    const bottom = viewport.bottom || canvases[0].height;
    const edges = Array.from({ length: n }, function () {
      return Array(n).fill(null);
    });

    let completed = 0;
    const total = n * (n - 1);
    for (let from = 0; from < n; from++) {
      for (let to = 0; to < n; to++) {
        if (from === to) continue;
        const edge = detailedDirectedOverlap(canvases[from], canvases[to], top, bottom);
        edge.h = Math.round(edge.h * scaleBack);
        edges[from][to] = edge;
        completed++;
        $('#stitchStatus').textContent = '画像の上下順を判定中 ' + completed + '/' + total;
        await sleepFrame();
      }
    }

    const path = bestFullPath(edges);
    let validEdges = 0;
    for (let i = 0; i < path.length - 1; i++) {
      if (edges[path[i]][path[i + 1]] && edges[path[i]][path[i + 1]].valid) validEdges++;
    }

    return {
      path: path,
      edges: edges,
      reliable: validEdges === Math.max(0, n - 1),
      validEdges: validEdges
    };
  }

  stitch = async function (auto) {
    if (auto === false || loaded.length <= 1) {
      return originalStitch(auto);
    }

    if (!loaded.length) {
      alert('画像を選択してください。');
      return;
    }

    $('#stitchStatus').textContent = '画像の選択順に関係なく、上下順を判定しています。';

    try {
      const order = await determineImageOrder();
      if (!order.path.length) return originalStitch(true);

      if (order.reliable) {
        const oldFiles = files.slice();
        const oldLoaded = loaded.slice();
        files = order.path.map(function (index) { return oldFiles[index]; });
        loaded = order.path.map(function (index) { return oldLoaded[index]; });
        renderThumbs();
      } else {
        $('#stitchStatus').textContent = '一部の重なりが弱いため、最も可能性の高い順で連結します。';
        const oldFiles = files.slice();
        const oldLoaded = loaded.slice();
        files = order.path.map(function (index) { return oldFiles[index]; });
        loaded = order.path.map(function (index) { return oldLoaded[index]; });
        renderThumbs();
      }

      // Recalculate accurate overlaps only for the selected top-to-bottom path.
      const targetW = Math.min(1200, Math.min.apply(null, loaded.map(function (img) {
        return img.naturalWidth;
      })));
      const orderedCanvases = loaded.map(function (img) {
        return scaledCanvas(img, targetW);
      });
      stitchViewport = detectScrollViewport(orderedCanvases);
      overlaps = [];
      for (let i = 0; i < orderedCanvases.length - 1; i++) {
        overlaps.push(await detectOverlap(
          orderedCanvases[i],
          orderedCanvases[i + 1],
          stitchViewport.top,
          stitchViewport.bottom
        ));
        $('#stitchStatus').textContent = '並べ替え完了・重なりを精密検出中 ' + (i + 1) + '/' + (orderedCanvases.length - 1);
        await sleepFrame();
      }

      await originalStitch(false);
      const finalText = $('#stitchStatus').textContent;
      $('#stitchStatus').textContent = '画像順を自動判定しました。' + finalText;
    } catch (error) {
      console.error('auto order failed', error);
      $('#stitchStatus').textContent = '自動並べ替えに失敗したため、選択順のまま連結します。';
      await originalStitch(true);
    }
  };
})();
</script>`;

export default {
  async fetch(request, env, ctx) {
    const response = await base.fetch(request, env, ctx);
    const url = new URL(request.url);
    const contentType = response.headers.get('content-type') || '';

    if (request.method === 'GET' && url.pathname === '/' && contentType.includes('text/html')) {
      let html = await response.text();
      html = html.includes('</body>')
        ? html.replace('</body>', AUTO_ORDER_PATCH + '</body>')
        : html + AUTO_ORDER_PATCH;

      const headers = new Headers(response.headers);
      headers.delete('content-length');
      headers.delete('etag');
      headers.set('cache-control', 'no-store');
      return new Response(html, {
        status: response.status,
        statusText: response.statusText,
        headers
      });
    }

    return response;
  }
};
