<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>SteamSell</title>
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
  <style>
    :root {
      --bg: var(--tg-theme-bg-color, #101822);
      --card: var(--tg-theme-secondary-bg-color, #1a2430);
      --text: var(--tg-theme-text-color, #ffffff);
      --hint: var(--tg-theme-hint-color, #8b9baa);
      --accent: var(--tg-theme-button-color, #0088cc);
      --discount-bg: #4c6b22;
      --discount-text: #beee11;
      --free-color: #2ecc71;
    }

    * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; margin: 0; padding: 0; }

    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: var(--bg);
      color: var(--text);
      padding: 14px;
    }

    header { text-align: center; margin-bottom: 14px; }
    header h1 { font-size: 20px; font-weight: 700; margin-bottom: 4px; }
    header p { font-size: 11px; color: var(--hint); }

    .search-box {
      width: 100%;
      padding: 12px 14px;
      border-radius: 12px;
      border: 1px solid rgba(255, 255, 255, 0.12);
      background: rgba(0, 0, 0, 0.35);
      color: var(--text);
      font-size: 15px;
      outline: none;
      margin-bottom: 12px;
    }
    .search-box:focus { border-color: var(--accent); }

    .categories {
      display: flex;
      gap: 8px;
      overflow-x: auto;
      padding-bottom: 8px;
      margin-bottom: 14px;
      scrollbar-width: none;
    }
    .categories::-webkit-scrollbar { display: none; }

    .cat-btn {
      padding: 7px 14px;
      border-radius: 20px;
      font-size: 13px;
      font-weight: 500;
      background: var(--card);
      color: var(--text);
      white-space: nowrap;
      cursor: pointer;
      border: 1px solid rgba(255, 255, 255, 0.05);
    }
    .cat-btn.active {
      background: var(--accent);
      color: #fff;
      font-weight: 600;
      border-color: transparent;
    }

    .games-list { display: flex; flex-direction: column; gap: 10px; }
    .status-msg { text-align: center; padding: 30px 10px; color: var(--hint); font-size: 14px; line-height: 1.5; }

    .card {
      display: flex;
      background: var(--card);
      border-radius: 10px;
      overflow: hidden;
      border: 1px solid rgba(255, 255, 255, 0.06);
      cursor: pointer;
      text-decoration: none;
      color: inherit;
    }
    .card:active { opacity: 0.8; }

    .card img {
      width: 120px;
      height: 65px;
      object-fit: cover;
      background: #000;
      flex-shrink: 0;
    }

    .info {
      padding: 8px 12px;
      display: flex;
      flex-direction: column;
      justify-content: center;
      flex: 1;
      min-width: 0;
      gap: 4px;
    }

    .title {
      font-size: 13.5px;
      font-weight: 600;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }

    .prices { display: flex; align-items: center; gap: 8px; font-size: 13px; }
    .discount-badge {
      background: var(--discount-bg);
      color: var(--discount-text);
      font-weight: 700;
      font-size: 11px;
      padding: 2px 6px;
      border-radius: 4px;
    }
    .old-price { text-decoration: line-through; color: var(--hint); font-size: 12px; }
    .new-price { font-weight: 700; color: #a4d007; }
    .free-price { font-weight: 600; color: var(--free-color); }
    .no-price { font-size: 12px; color: var(--hint); }
  </style>
</head>
<body>

  <header>
    <h1>SteamSell 🎮</h1>
    <p>Официальные скидки Steam (Россия, ₽) • Кликните для перехода</p>
  </header>

  <input type="text" id="search" class="search-box" placeholder="Поиск по названию игры...">

  <div class="categories">
    <div class="cat-btn active" onclick="switchCategory('discounts', this)">🔥 Только скидки</div>
    <div class="cat-btn" onclick="switchCategory('all', this)">🌐 Все игры</div>
  </div>

  <div class="games-list" id="games">
    <div class="status-msg">Загрузка актуальных цен Steam... ⏳</div>
  </div>

  <script>
    const tg = window.Telegram?.WebApp;
    if (tg) {
      tg.expand();
      tg.ready();
    }

    let allGames = [];
    let currentCategory = 'discounts';

    function openSteamApp(appId) {
      const url = `https://store.steampowered.com/app/${appId}/`;
      if (tg && tg.openLink) {
        tg.openLink(url);
      } else {
        window.open(url, '_blank');
      }
    }

    function formatPrice(game) {
      if (game.is_free || game.price === 0) {
        return `<span class="free-price">Бесплатно</span>`;
      }
      if (game.disc) {
        return `
          <span class="discount-badge">${game.disc}</span>
          <span class="old-price">${game.old}</span>
          <span class="new-price">${game.price}</span>
        `;
      }
      if (game.price) {
        return `<span class="new-price" style="color: #fff;">${game.price}</span>`;
      }
      return `<span class="no-price">Нет цены</span>`;
    }

    // Загрузка данных через резервные зеркала Steam API
    async function loadDiscounts() {
      const list = document.getElementById("games");
      
      const proxies = [
        "https://corsproxy.io/?url=" + encodeURIComponent("https://store.steampowered.com/api/featuredcategories/?cc=ru&l=russian"),
        "https://api.codetabs.com/v1/proxy?quest=" + encodeURIComponent("https://store.steampowered.com/api/featuredcategories/?cc=ru&l=russian")
      ];

      let rawData = null;

      for (let url of proxies) {
        try {
          const res = await fetch(url, { cache: "no-store" });
          if (res.ok) {
            rawData = await res.json();
            if (rawData && rawData.specials) break;
          }
        } catch (e) {
          continue;
        }
      }

      if (!rawData) {
        list.innerHTML = `
          <div class="status-msg">
            ⚠️ Ограничение сети Telegram WebApp.<br>
            Используйте поиск в боте или перезапустите приложение.
          </div>
        `;
        return;
      }

      const parsed = [];
      const seen = new Set();

      function extractGames(items, isSpecial) {
        if (!items) return;
        items.forEach(g => {
          if (!g || !g.id || seen.has(g.id)) return;
          seen.add(g.id);

          const hasDisc = Boolean(g.discount_percent && g.discount_percent > 0);
          parsed.push({
            id: g.id,
            name: g.name,
            isSpecial: isSpecial,
            disc: hasDisc ? `-${g.discount_percent}%` : null,
            old: hasDisc && g.original_price ? (g.original_price / 100).toLocaleString('ru-RU') + " ₽" : null,
            price: g.final_price !== undefined ? (g.final_price === 0 ? "Бесплатно" : (g.final_price / 100).toLocaleString('ru-RU') + " ₽") : null,
            img: g.header_image || `https://cdn.akamai.steamstatic.com/steam/apps/${g.id}/header.jpg`
          });
        });
      }

      extractGames(rawData.specials?.items, true);
      extractGames(rawData.top_sellers?.items, false);
      extractGames(rawData.new_releases?.items, false);

      allGames = parsed;
      render();
    }

    function render() {
      const q = document.getElementById("search").value.toLowerCase().trim();
      const list = document.getElementById("games");
      list.innerHTML = "";

      const filtered = allGames.filter(g => {
        const matchCat = (currentCategory === 'all') || (currentCategory === 'discounts' && g.disc);
        const matchQ = g.name.toLowerCase().includes(q);
        return matchCat && matchQ;
      });

      if (!filtered.length) {
        list.innerHTML = `<div class="status-msg">Игры не найдены</div>`;
        return;
      }

      filtered.forEach(g => {
        const card = document.createElement("div");
        card.className = "card";
        card.onclick = () => openSteamApp(g.id);

        card.innerHTML = `
          <img src="${g.img}" alt="${g.name}" loading="lazy">
          <div class="info">
            <div class="title" title="${g.name}">${g.name}</div>
            <div class="prices">${formatPrice(g)}</div>
          </div>
        `;
        list.appendChild(card);
      });
    }

    function switchCategory(cat, btn) {
      currentCategory = cat;
      document.querySelectorAll(".cat-btn").forEach(b => b.classList.remove("active"));
      btn.classList.add("active");
      render();
    }

    document.getElementById("search").addEventListener("input", render);
    loadDiscounts();
  </script>
</body>
</html>
