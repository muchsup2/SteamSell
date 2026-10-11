<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>SteamSell</title>
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
  <style>
    :root {
      --bg: var(--tg-theme-bg-color, #0e141b);
      --card: var(--tg-theme-secondary-bg-color, #1b2838);
      --text: var(--tg-theme-text-color, #ffffff);
      --hint: var(--tg-theme-hint-color, #8f98a0);
      --accent: var(--tg-theme-button-color, #66c0f4);
      --discount-bg: #4c6b22;
      --discount-text: #beee11;
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
      border-radius: 10px;
      border: 1px solid rgba(255, 255, 255, 0.1);
      background: rgba(0, 0, 0, 0.25);
      color: var(--text);
      font-size: 15px;
      outline: none;
      margin-bottom: 12px;
    }

    .categories {
      display: flex;
      gap: 8px;
      overflow-x: auto;
      padding-bottom: 6px;
      margin-bottom: 14px;
      scrollbar-width: none;
    }
    .categories::-webkit-scrollbar { display: none; }

    .cat-btn {
      padding: 7px 14px;
      border-radius: 18px;
      font-size: 13px;
      font-weight: 500;
      background: var(--card);
      color: var(--text);
      white-space: nowrap;
      cursor: pointer;
    }
    .cat-btn.active {
      background: var(--accent);
      color: #000;
      font-weight: 600;
    }

    .games-list { display: flex; flex-direction: column; gap: 10px; }
    .status-msg { text-align: center; padding: 30px 10px; color: var(--hint); font-size: 14px; }

    .card {
      display: flex;
      background: var(--card);
      border-radius: 10px;
      overflow: hidden;
      border: 1px solid rgba(255, 255, 255, 0.05);
      cursor: pointer;
      text-decoration: none;
      color: inherit;
    }
    .card:active { opacity: 0.8; }

    .card img {
      width: 120px;
      height: 65px;
      object-fit: cover;
      background: #111;
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
      font-size: 12px;
      padding: 2px 6px;
      border-radius: 4px;
    }
    .old-price { text-decoration: line-through; color: var(--hint); font-size: 12px; }
    .new-price { font-weight: 700; color: #a4d007; }
  </style>
</head>
<body>

  <header>
    <h1>SteamSell 🎮</h1>
    <p>Регион: РФ • Нажмите для перехода в Steam</p>
  </header>

  <input type="text" id="search" class="search-box" placeholder="Поиск по названию игры...">

  <div class="categories">
    <div class="cat-btn active" onclick="filterCat('all', this)">Все игры</div>
    <div class="cat-btn" onclick="filterCat('action', this)">Экшен</div>
    <div class="cat-btn" onclick="filterCat('rpg', this)">RPG</div>
    <div class="cat-btn" onclick="filterCat('strategy', this)">Стратегии</div>
    <div class="cat-btn" onclick="filterCat('indie', this)">Инди</div>
  </div>

  <div class="games-list" id="games">
    <div class="status-msg">Загрузка скидок Steam... ⏳</div>
  </div>

  <script>
    const tg = window.Telegram?.WebApp;
    if (tg) {
      tg.expand();
      tg.ready();
    }

    let allGames = [];
    let currentCategory = 'all';

    function openSteamApp(appId) {
      const url = `https://store.steampowered.com/app/${appId}/`;
      if (tg && tg.openLink) {
        tg.openLink(url);
      } else {
        window.open(url, '_blank');
      }
    }

    // Резервный надежный список на случай падения внешних прокси
    const fallbackGames = [
      { id: 292030, name: "The Witcher 3: Wild Hunt", cat: "rpg", disc: "-75%", old: "1 500 ₽", price: "375 ₽", img: "https://cdn.akamai.steamstatic.com/steam/apps/292030/header.jpg" },
      { id: 1091500, name: "Cyberpunk 2077", cat: "rpg", disc: "-50%", old: "2 000 ₽", price: "1 000 ₽", img: "https://cdn.akamai.steamstatic.com/steam/apps/1091500/header.jpg" },
      { id: 782330, name: "DOOM Eternal", cat: "action", disc: "-80%", old: "2 800 ₽", price: "560 ₽", img: "https://cdn.akamai.steamstatic.com/steam/apps/782330/header.jpg" },
      { id: 289070, name: "Civilization VI", cat: "strategy", disc: "-90%", old: "1 999 ₽", price: "199 ₽", img: "https://cdn.akamai.steamstatic.com/steam/apps/289070/header.jpg" },
      { id: 367520, name: "Hollow Knight", cat: "indie", disc: "-50%", old: "550 ₽", price: "275 ₽", img: "https://cdn.akamai.steamstatic.com/steam/apps/367520/header.jpg" },
      { id: 1172470, name: "Apex Legends", cat: "action", disc: null, old: null, price: "Бесплатно", img: "https://cdn.akamai.steamstatic.com/steam/apps/1172470/header.jpg" }
    ];

    async function loadGames() {
      // Используем быстрый сервис-прокси с тайм-аутом
      const target = "https://store.steampowered.com/api/featuredcategories/?cc=ru&l=russian";
      const proxy = `https://corsproxy.io/?${encodeURIComponent(target)}`;

      try {
        const controller = new AbortController();
        const timer = setTimeout(() => controller.abort(), 4000); // 4 сек лимит

        const res = await fetch(proxy, { signal: controller.signal });
        clearTimeout(timer);

        if (!res.ok) throw new Error("bad network");
        const data = await res.json();

        const parsed = [];
        const seen = new Set();

        const addItems = (items, defaultCat) => {
          if (!items) return;
          items.forEach(g => {
            if (!g || !g.id || seen.has(g.id)) return;
            seen.add(g.id);

            const hasDisc = Boolean(g.discount_percent && g.discount_percent > 0);
            parsed.push({
              id: g.id,
              name: g.name,
              cat: defaultCat,
              disc: hasDisc ? `-${g.discount_percent}%` : null,
              old: hasDisc && g.original_price ? (g.original_price / 100).toLocaleString('ru-RU') + " ₽" : null,
              price: g.final_price !== undefined ? (g.final_price === 0 ? "Бесплатно" : (g.final_price / 100).toLocaleString('ru-RU') + " ₽") : "Нет цены",
              img: g.header_image || g.large_capsule_image || `https://cdn.akamai.steamstatic.com/steam/apps/${g.id}/header.jpg`
            });
          });
        };

        if (data.specials?.items) addItems(data.specials.items, 'action');
        if (data.top_sellers?.items) addItems(data.top_sellers.items, 'rpg');
        if (data.new_releases?.items) addItems(data.new_releases.items, 'indie');

        allGames = parsed.length ? parsed : fallbackGames;
      } catch (e) {
        // Если прокси заблокирован или недоступен — мгновенно отдаем резервный каталог
        allGames = fallbackGames;
      }

      renderGames();
    }

    function renderGames() {
      const q = document.getElementById("search").value.toLowerCase().trim();
      const list = document.getElementById("games");
      list.innerHTML = "";

      const filtered = allGames.filter(g => {
        const matchCat = (currentCategory === 'all') || (g.cat === currentCategory);
        const matchQ = g.name.toLowerCase().includes(q);
        return matchCat && matchQ;
      });

      if (!filtered.length) {
        list.innerHTML = "<div class='status-msg'>Игры не найдены</div>";
        return;
      }

      filtered.forEach(g => {
        const card = document.createElement("div");
        card.className = "card";
        card.onclick = () => openSteamApp(g.id);

        let priceHtml = '';
        if (g.disc) {
          priceHtml = `
            <span class="discount-badge">${g.disc}</span>
            <span class="old-price">${g.old}</span>
            <span class="new-price">${g.price}</span>
          `;
        } else {
          priceHtml = `<span class="new-price" style="color: #fff;">${g.price}</span>`;
        }

        card.innerHTML = `
          <img src="${g.img}" alt="${g.name}" loading="lazy">
          <div class="info">
            <div class="title">${g.name}</div>
            <div class="prices">${priceHtml}</div>
          </div>
        `;
        list.appendChild(card);
      });
    }

    function filterCat(cat, btn) {
      currentCategory = cat;
      document.querySelectorAll(".cat-btn").forEach(b => b.classList.remove("active"));
      btn.classList.add("active");
      renderGames();
    }

    document.getElementById("search").addEventListener("input", renderGames);
    loadGames();
  </script>
</body>
</html>
