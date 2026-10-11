<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>SteamSell</title>
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
  <style>
    :root {
      --bg: var(--tg-theme-bg-color, #171a21);
      --card-bg: rgba(255, 255, 255, 0.06);
      --card-hover: rgba(255, 255, 255, 0.12);
      --text: var(--tg-theme-text-color, #f3f3f3);
      --btn: var(--tg-theme-button-color, #66c0f4);
      --discount: #a4d007;
    }
    * {
      box-sizing: border-box;
      -webkit-tap-highlight-color: transparent;
    }
    body {
      margin: 0;
      padding: 16px;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: var(--bg);
      color: var(--text);
    }
    header {
      text-align: center;
      margin-bottom: 14px;
    }
    h2 {
      margin: 0 0 4px 0;
      font-size: 20px;
    }
    .region-badge {
      font-size: 11px;
      opacity: 0.6;
      text-transform: uppercase;
      letter-spacing: 0.5px;
    }
    .search-input {
      width: 100%;
      padding: 12px 14px;
      border-radius: 12px;
      border: 1px solid rgba(255, 255, 255, 0.15);
      background: rgba(0, 0, 0, 0.35);
      color: #fff;
      font-size: 15px;
      outline: none;
      margin-bottom: 12px;
    }
    .search-input:focus {
      border-color: var(--btn);
    }
    .categories {
      display: flex;
      gap: 8px;
      overflow-x: auto;
      padding-bottom: 10px;
      margin-bottom: 14px;
      scrollbar-width: none;
    }
    .categories::-webkit-scrollbar { display: none; }
    .cat-btn {
      background: rgba(255, 255, 255, 0.1);
      padding: 8px 14px;
      border-radius: 20px;
      font-size: 13px;
      white-space: nowrap;
      cursor: pointer;
      user-select: none;
      transition: background 0.2s;
    }
    .cat-btn.active {
      background: var(--btn);
      color: #000;
      font-weight: 600;
    }
    .status-msg {
      text-align: center;
      padding: 30px 10px;
      opacity: 0.7;
      font-size: 14px;
    }
    .games-list {
      display: flex;
      flex-direction: column;
      gap: 10px;
    }
    .card {
      display: flex;
      background: var(--card-bg);
      border-radius: 10px;
      overflow: hidden;
      border: 1px solid rgba(255, 255, 255, 0.08);
      cursor: pointer;
      text-decoration: none;
      color: inherit;
      transition: transform 0.15s, background 0.15s;
    }
    .card:active {
      transform: scale(0.98);
      background: var(--card-hover);
    }
    .card img {
      width: 120px;
      height: 70px;
      object-fit: cover;
      background: #111;
      flex-shrink: 0;
    }
    .info {
      padding: 8px 12px;
      display: flex;
      flex-direction: column;
      justify-content: space-between;
      flex: 1;
      min-width: 0;
    }
    .title {
      font-size: 13.5px;
      font-weight: 600;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }
    .prices {
      display: flex;
      align-items: center;
      gap: 8px;
      margin-top: 4px;
    }
    .badge {
      background: var(--discount);
      color: #000;
      font-weight: bold;
      font-size: 11px;
      padding: 2px 6px;
      border-radius: 4px;
    }
    .old-price {
      text-decoration: line-through;
      font-size: 11.5px;
      opacity: 0.5;
    }
    .new-price {
      font-weight: bold;
      font-size: 13.5px;
      color: #a4d007;
    }
    .free-price {
      font-weight: bold;
      font-size: 13px;
      color: #66c0f4;
    }
  </style>
</head>
<body>

  <header>
    <h2>SteamSell 🎮</h2>
    <div class="region-badge">Регион: Россия (RUB ₽) • Нажмите на игру для перехода в Steam</div>
  </header>

  <input type="text" id="search" class="search-input" placeholder="Поиск по названию игры...">

  <div class="categories">
    <div class="cat-btn active" onclick="filterCat('specials', this)">🔥 Скидки и Акции</div>
    <div class="cat-btn" onclick="filterCat('top_sellers', this)">🏆 Лидеры продаж</div>
    <div class="cat-btn" onclick="filterCat('new_releases', this)">✨ Популярные новинки</div>
    <div class="cat-btn" onclick="filterCat('all', this)">Все категории</div>
  </div>

  <div class="games-list" id="games">
    <div class="status-msg">Загрузка актуальных скидок Steam... ⏳</div>
  </div>

  <script>
    const tg = window.Telegram?.WebApp;
    if (tg) {
      tg.expand();
      tg.ready();
    }

    let allGames = [];
    let currentCategory = 'specials';

    // Форматирование копеек в рубли
    function formatRub(cents) {
      if (cents === 0) return "Бесплатно";
      return (cents / 100).toLocaleString('ru-RU', { maximumFractionDigits: 0 }) + " ₽";
    }

    // Открытие игры в Steam
    function openSteamApp(appId) {
      const url = `https://store.steampowered.com/app/${appId}/`;
      if (tg && tg.openLink) {
        tg.openLink(url);
      } else {
        window.open(url, '_blank');
      }
    }

    // Загрузка реальных данных из Steam Store API через надежный прокси
    async function fetchSteamDiscounts() {
      const steamApiUrl = "https://store.steampowered.com/api/featuredcategories/?cc=ru&l=russian";
      const proxyUrl = `https://api.allorigins.win/raw?url=${encodeURIComponent(steamApiUrl)}`;

      try {
        const response = await fetch(proxyUrl);
        if (!response.ok) throw new Error("Ошибка соединения");
        const data = await response.json();

        const parsed = [];
        const seenIds = new Set();

        function addItems(items, categoryType) {
          if (!items || !items.length) return;
          items.forEach(game => {
            if (!game || !game.id || seenIds.has(game.id)) return;
            seenIds.add(game.id);

            parsed.push({
              id: game.id,
              title: game.name,
              category: categoryType,
              discount: game.discount_percent ? `-${game.discount_percent}%` : null,
              old: game.original_price ? formatRub(game.original_price) : null,
              price: game.final_price !== undefined ? formatRub(game.final_price) : "Бесплатно",
              hasDiscount: Boolean(game.discount_percent && game.discount_percent > 0),
              img: game.header_image || game.large_capsule_image || `https://cdn.akamai.steamstatic.com/steam/apps/${game.id}/header.jpg`
            });
          });
        }

        // Парсим различные разделы Steam
        if (data.specials && data.specials.items) {
          addItems(data.specials.items, 'specials');
        }
        if (data.top_sellers && data.top_sellers.items) {
          addItems(data.top_sellers.items, 'top_sellers');
        }
        if (data.new_releases && data.new_releases.items) {
          addItems(data.new_releases.items, 'new_releases');
        }

        allGames = parsed;
        renderGames();
      } catch (err) {
        console.error(err);
        document.getElementById("games").innerHTML = `
          <div class="status-msg" style="color: #ff7675;">
            Не удалось загрузить данные из Steam.<br>
            Проверьте интернет-соединение или обновите страницу.
          </div>
        `;
      }
    }

    function renderGames() {
      const query = document.getElementById("search").value.toLowerCase().trim();
      const list = document.getElementById("games");
      list.innerHTML = "";

      const filtered = allGames.filter(g => {
        const matchesCategory = (currentCategory === 'all') || (g.category === currentCategory);
        const matchesSearch = g.title.toLowerCase().includes(query);
        return matchesCategory && matchesSearch;
      });

      if (!filtered.length) {
        list.innerHTML = "<div class='status-msg'>Игры не найдены</div>";
        return;
      }

      filtered.forEach(game => {
        const card = document.createElement("div");
        card.className = "card";
        card.onclick = () => openSteamApp(game.id);

        let priceHtml = '';
        if (game.hasDiscount) {
          priceHtml = `
            <span class="badge">${game.discount}</span>
            <span class="old-price">${game.old}</span>
            <span class="new-price">${game.price}</span>
          `;
        } else {
          priceHtml = `<span class="new-price" style="color: #fff;">${game.price}</span>`;
        }

        card.innerHTML = `
          <img src="${game.img}" alt="${game.title}" loading="lazy" onerror="this.src='https://store.steampowered.com/public/shared/images/header/globalheader_logo.png'">
          <div class="info">
            <div class="title" title="${game.title}">${game.title}</div>
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

    // Запуск загрузки реальных данных
    fetchSteamDiscounts();
  </script>
</body>
</html>
