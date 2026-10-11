<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>SteamSell</title>
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
  <style>
    /* Настройки минималистичного стиля */
    :root {
      --bg: var(--tg-theme-bg-color, #ffffff);
      --text: var(--tg-theme-text-color, #000000);
      --hint: var(--tg-theme-hint-color, #8e8e93);
      --btn-bg: var(--tg-theme-button-color, #007aff);
      --btn-text: var(--tg-theme-button-text-color, #ffffff);
      --secondary-bg: var(--tg-theme-secondary-bg-color, #f2f2f7);
      --discount-bg: #ff3b30;
      --free-color: #34c759;
    }
    
    * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; margin: 0; padding: 0; }
    
    body {
      font-family: -apple-system, BlinkMacSystemFont, "SF Pro Text", "Segoe UI", Roboto, sans-serif;
      background: var(--bg);
      color: var(--text);
      padding: 16px;
    }

    /* Шапка */
    header { text-align: center; margin-bottom: 20px; }
    header h1 { font-size: 22px; font-weight: 700; margin-bottom: 4px; letter-spacing: -0.5px; }
    header p { font-size: 11px; color: var(--hint); text-transform: uppercase; letter-spacing: 0.5px; }

    /* Поиск */
    .search-box {
      width: 100%;
      padding: 12px 16px;
      border-radius: 12px;
      border: none;
      background: var(--secondary-bg);
      color: var(--text);
      font-size: 16px;
      outline: none;
      margin-bottom: 16px;
    }
    .search-box::placeholder { color: var(--hint); }

    /* Категории */
    .categories {
      display: flex;
      gap: 8px;
      overflow-x: auto;
      padding-bottom: 8px;
      margin-bottom: 12px;
      scrollbar-width: none;
    }
    .categories::-webkit-scrollbar { display: none; }
    
    .cat-btn {
      padding: 8px 16px;
      border-radius: 20px;
      font-size: 14px;
      font-weight: 500;
      background: var(--secondary-bg);
      color: var(--text);
      white-space: nowrap;
      cursor: pointer;
      transition: 0.2s ease;
    }
    .cat-btn.active {
      background: var(--btn-bg);
      color: var(--btn-text);
    }

    /* Список игр (Минимализм) */
    .games-list { display: flex; flex-direction: column; gap: 16px; }
    .status-msg { text-align: center; padding: 30px 10px; color: var(--hint); font-size: 14px; }
    
    .card {
      display: flex;
      gap: 14px;
      align-items: center;
      cursor: pointer;
    }
    .card:active { opacity: 0.7; }
    
    .card img {
      width: 130px;
      height: 73px;
      border-radius: 8px;
      object-fit: cover;
      background: var(--secondary-bg);
      flex-shrink: 0;
      box-shadow: 0 2px 8px rgba(0,0,0,0.05);
    }
    
    .info {
      display: flex;
      flex-direction: column;
      justify-content: center;
      flex: 1;
      min-width: 0;
    }
    .title {
      font-size: 15px;
      font-weight: 600;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
      margin-bottom: 6px;
    }
    
    /* Цены */
    .prices { display: flex; align-items: center; gap: 8px; font-size: 14px; }
    .discount-badge {
      background: var(--discount-bg);
      color: #fff;
      font-weight: 700;
      font-size: 12px;
      padding: 2px 6px;
      border-radius: 6px;
    }
    .old-price { text-decoration: line-through; color: var(--hint); font-size: 13px; }
    .new-price { font-weight: 700; color: var(--text); }
    .free-price { font-weight: 600; color: var(--free-color); }
    .no-price { font-size: 13px; color: var(--hint); font-weight: 500; }
  </style>
</head>
<body>

  <header>
    <h1>SteamSell 🎮</h1>
    <p>Регион: РФ • Нажмите для перехода</p>
  </header>

  <input type="text" id="search" class="search-box" placeholder="Поиск игры...">

  <div class="categories">
    <div class="cat-btn active" onclick="filterCat('specials', this)">🔥 Акции</div>
    <div class="cat-btn" onclick="filterCat('top_sellers', this)">🏆 Топ продаж</div>
    <div class="cat-btn" onclick="filterCat('new_releases', this)">✨ Новинки</div>
    <div class="cat-btn" onclick="filterCat('all', this)">Все</div>
  </div>

  <div class="games-list" id="games">
    <div class="status-msg">Ищем лучшие предложения... ⏳</div>
  </div>

  <script>
    const tg = window.Telegram?.WebApp;
    if (tg) {
      tg.expand();
      tg.ready();
    }

    let allGames = [];
    let currentCategory = 'specials';

    // Функция форматирования цен
    function formatPrice(game) {
      // Если игра бесплатная
      if (game.is_free || game.final_price === 0) {
        return `<span class="free-price">Бесплатно</span>`;
      }
      // Если цены вообще нет в базе (еще не вышла и т.д.)
      if (game.final_price === undefined || game.final_price === null) {
        return `<span class="no-price">Нет цены</span>`;
      }
      
      const finalRub = (game.final_price / 100).toLocaleString('ru-RU') + " ₽";
      
      // Если есть скидка
      if (game.discount_percent && game.discount_percent > 0) {
        const oldRub = (game.original_price / 100).toLocaleString('ru-RU') + " ₽";
        return `
          <span class="discount-badge">-${game.discount_percent}%</span>
          <span class="old-price">${oldRub}</span>
          <span class="new-price">${finalRub}</span>
        `;
      }
      
      // Обычная цена без скидки
      return `<span class="new-price">${finalRub}</span>`;
    }

    function openSteamApp(appId) {
      const url = `https://store.steampowered.com/app/${appId}/`;
      if (tg && tg.openLink) { tg.openLink(url); } 
      else { window.open(url, '_blank'); }
    }

    async function fetchSteamDiscounts() {
      const steamApiUrl = "https://store.steampowered.com/api/featuredcategories/?cc=ru&l=russian";
      const proxyUrl = `https://api.allorigins.win/raw?url=${encodeURIComponent(steamApiUrl)}`;

      try {
        const response = await fetch(proxyUrl);
        const data = await response.json();
        const parsed = [];
        const seenIds = new Set();

        function addItems(items, categoryType) {
          if (!items) return;
          items.forEach(game => {
            if (!game || !game.id || seenIds.has(game.id)) return;
            seenIds.add(game.id);
            game.custom_category = categoryType; // сохраняем категорию
            parsed.push(game);
          });
        }

        if (data.specials?.items) addItems(data.specials.items, 'specials');
        if (data.top_sellers?.items) addItems(data.top_sellers.items, 'top_sellers');
        if (data.new_releases?.items) addItems(data.new_releases.items, 'new_releases');

        allGames = parsed;
        renderGames();
      } catch (err) {
        document.getElementById("games").innerHTML = `<div class="status-msg">Ошибка загрузки. Попробуйте позже.</div>`;
      }
    }

    function renderGames() {
      const query = document.getElementById("search").value.toLowerCase().trim();
      const list = document.getElementById("games");
      list.innerHTML = "";

      const filtered = allGames.filter(g => {
        const matchCat = (currentCategory === 'all') || (g.custom_category === currentCategory);
        const matchSearch = g.name.toLowerCase().includes(query);
        return matchCat && matchSearch;
      });

      if (!filtered.length) {
        list.innerHTML = "<div class='status-msg'>Ничего не найдено</div>";
        return;
      }

      filtered.forEach(game => {
        const card = document.createElement("div");
        card.className = "card";
        card.onclick = () => openSteamApp(game.id);

        const imgUrl = game.header_image || game.large_capsule_image || `https://cdn.akamai.steamstatic.com/steam/apps/${game.id}/header.jpg`;
        const priceHtml = formatPrice(game);

        card.innerHTML = `
          <img src="${imgUrl}" alt="${game.name}">
          <div class="info">
            <div class="title">${game.name}</div>
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
    fetchSteamDiscounts();
  </script>
</body>
</html>
