<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>SteamSell</title>
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
  <style>
    :root {
      --bg: var(--tg-theme-bg-color, #12161c);
      --card: var(--tg-theme-secondary-bg-color, #1a222d);
      --text: var(--tg-theme-text-color, #ffffff);
      --hint: var(--tg-theme-hint-color, #8395a7);
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
      background: rgba(0, 0, 0, 0.3);
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
    .status-msg { text-align: center; padding: 30px 10px; color: var(--hint); font-size: 14px; }

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
    <p>Официальные цены Steam (Россия, ₽) • Кликните для перехода</p>
  </header>

  <input type="text" id="search" class="search-box" placeholder="Поиск по всему Steam (название игры)...">

  <div class="categories">
    <div class="cat-btn active" onclick="switchCategory('discounts', this)">🔥 Только скидки</div>
    <div class="cat-btn" onclick="switchCategory('popular', this)">🏆 Популярные игры</div>
    <div class="cat-btn" onclick="switchCategory('all', this)">🌐 Все категории</div>
  </div>

  <div class="games-list" id="games">
    <div class="status-msg">Загрузка данных из каталога Steam... ⏳</div>
  </div>

  <script>
    const tg = window.Telegram?.WebApp;
    if (tg) {
      tg.expand();
      tg.ready();
    }

    let currentMode = 'discounts';
    let searchDebounceTimer = null;

    function openSteamApp(appId) {
      const url = `https://store.steampowered.com/app/${appId}/`;
      if (tg && tg.openLink) {
        tg.openLink(url);
      } else {
        window.open(url, '_blank');
      }
    }

    function formatPriceHtml(item) {
      // Бесплатная игра
      if (item.is_free || (item.price && item.price.final === 0)) {
        return `<span class="free-price">Бесплатно</span>`;
      }
      
      // Если есть объект с ценой
      if (item.price) {
        const p = item.price;
        if (p.discount_percent > 0) {
          const oldP = (p.initial / 100).toLocaleString('ru-RU') + " ₽";
          const newP = (p.final / 100).toLocaleString('ru-RU') + " ₽";
          return `
            <span class="discount-badge">-${p.discount_percent}%</span>
            <span class="old-price">${oldP}</span>
            <span class="new-price">${newP}</span>
          `;
        }
        return `<span class="new-price" style="color: #fff;">${(p.final / 100).toLocaleString('ru-RU')} ₽</span>`;
      }

      return `<span class="no-price">В корзину / Нет цены</span>`;
    }

    // Живой поиск по каталогу Steam (любая игра)
    async function searchLiveSteam(query, onlyDiscounts = false) {
      const list = document.getElementById("games");
      list.innerHTML = `<div class="status-msg">Поиск в базе Steam... ⏳</div>`;

      const steamSearchUrl = `https://store.steampowered.com/api/storesearch/?term=${encodeURIComponent(query)}&l=russian&cc=ru`;
      const proxy = `https://api.allorigins.win/raw?url=${encodeURIComponent(steamSearchUrl)}`;

      try {
        const res = await fetch(proxy);
        const data = await res.json();
        let items = data.items || [];

        if (onlyDiscounts) {
          items = items.filter(g => g.price && g.price.discount_percent > 0);
        }

        renderItems(items);
      } catch (err) {
        list.innerHTML = `<div class="status-msg">Не удалось загрузить результаты. Проверьте интернет.</div>`;
      }
    }

    // Загрузка стартовой витрины актуальных предложений
    async function loadFeatured() {
      const list = document.getElementById("games");
      list.innerHTML = `<div class="status-msg">Получение актуальных скидок Steam... ⏳</div>`;

      const steamUrl = "https://store.steampowered.com/api/featuredcategories/?cc=ru&l=russian";
      const proxy = `https://api.allorigins.win/raw?url=${encodeURIComponent(steamUrl)}`;

      try {
        const res = await fetch(proxy);
        const data = await res.json();
        let items = [];

        if (currentMode === 'discounts') {
          // Выбираем только игры со скидкой
          const specials = data.specials?.items || [];
          items = specials.map(g => ({
            id: g.id,
            name: g.name,
            tiny_image: g.header_image || `https://cdn.akamai.steamstatic.com/steam/apps/${g.id}/header.jpg`,
            price: {
              initial: g.original_price,
              final: g.final_price,
              discount_percent: g.discount_percent
            }
          }));
        } else {
          // Все популярные/лидеры
          const top = data.top_sellers?.items || [];
          items = top.map(g => ({
            id: g.id,
            name: g.name,
            tiny_image: g.header_image || `https://cdn.akamai.steamstatic.com/steam/apps/${g.id}/header.jpg`,
            price: {
              initial: g.original_price,
              final: g.final_price,
              discount_percent: g.discount_percent || 0
            }
          }));
        }

        renderItems(items);
      } catch (err) {
        // Запасной поиск
        searchLiveSteam("sale", currentMode === 'discounts');
      }
    }

    function renderItems(items) {
      const list = document.getElementById("games");
      list.innerHTML = "";

      if (!items || items.length === 0) {
        list.innerHTML = `<div class="status-msg">Ничего не найдено</div>`;
        return;
      }

      items.forEach(game => {
        const card = document.createElement("div");
        card.className = "card";
        card.onclick = () => openSteamApp(game.id);

        const imgUrl = game.tiny_image || `https://cdn.akamai.steamstatic.com/steam/apps/${game.id}/header.jpg`;
        const prices = formatPriceHtml(game);

        card.innerHTML = `
          <img src="${imgUrl}" alt="${game.name}" loading="lazy" onerror="this.src='https://store.steampowered.com/public/shared/images/header/globalheader_logo.png'">
          <div class="info">
            <div class="title" title="${game.name}">${game.name}</div>
            <div class="prices">${prices}</div>
          </div>
        `;
        list.appendChild(card);
      });
    }

    function switchCategory(mode, btn) {
      currentMode = mode;
      document.querySelectorAll(".cat-btn").forEach(b => b.classList.remove("active"));
      btn.classList.add("active");

      const query = document.getElementById("search").value.trim();
      if (query.length > 0) {
        searchLiveSteam(query, currentMode === 'discounts');
      } else {
        loadFeatured();
      }
    }

    // Слушатель строки поиска с задержкой 400мс
    document.getElementById("search").addEventListener("input", (e) => {
      const query = e.target.value.trim();
      clearTimeout(searchDebounceTimer);

      if (query.length === 0) {
        loadFeatured();
        return;
      }

      searchDebounceTimer = setTimeout(() => {
        searchLiveSteam(query, currentMode === 'discounts');
      }, 400);
    });

    // Первый запуск
    loadFeatured();
  </script>
</body>
</html>
