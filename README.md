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
      --card-bg: rgba(255, 255, 255, 0.05);
      --text: var(--tg-theme-text-color, #f3f3f3);
      --btn: var(--tg-theme-button-color, #66c0f4);
      --discount: #a4d007;
    }
    body {
      margin: 0;
      padding: 16px;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: var(--bg);
      color: var(--text);
      box-sizing: border-box;
    }
    h2 {
      margin: 0 0 14px 0;
      text-align: center;
      font-size: 20px;
    }
    .search-input {
      width: 100%;
      box-sizing: border-box;
      padding: 12px 14px;
      border-radius: 12px;
      border: 1px solid rgba(255, 255, 255, 0.15);
      background: rgba(0, 0, 0, 0.25);
      color: #fff;
      font-size: 15px;
      outline: none;
      margin-bottom: 12px;
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
    }
    .cat-btn.active {
      background: var(--btn);
      color: #000;
      font-weight: 600;
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
    }
    .card img {
      width: 110px;
      height: 65px;
      object-fit: cover;
    }
    .info {
      padding: 8px 12px;
      display: flex;
      flex-direction: column;
      justify-content: space-between;
      flex: 1;
    }
    .title {
      font-size: 14px;
      font-weight: 600;
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
      font-size: 12px;
      opacity: 0.5;
    }
    .new-price {
      font-weight: bold;
      font-size: 14px;
      color: #a4d007;
    }
  </style>
</head>
<body>

  <h2>SteamSell 🎮</h2>
  <input type="text" id="search" class="search-input" placeholder="Поиск игры...">

  <div class="categories">
    <div class="cat-btn active" onclick="filterCat('all', this)">Все</div>
    <div class="cat-btn" onclick="filterCat('rpg', this)">RPG</div>
    <div class="cat-btn" onclick="filterCat('action', this)">Экшен</div>
    <div class="cat-btn" onclick="filterCat('strategy', this)">Стратегии</div>
    <div class="cat-btn" onclick="filterCat('indie', this)">Инди</div>
  </div>

  <div class="games-list" id="games"></div>

  <script>
    const tg = window.Telegram?.WebApp;
    if (tg) {
      tg.expand();
      tg.ready();
    }

    const items = [
      { id: 1, title: "Cyberpunk 2077", cat: "rpg", disc: "-50%", old: "2000 ₽", price: "1000 ₽", img: "https://cdn.akamai.steamstatic.com/steam/apps/1091500/header.jpg" },
      { id: 2, title: "The Witcher 3: Wild Hunt", cat: "rpg", disc: "-75%", old: "1500 ₽", price: "375 ₽", img: "https://cdn.akamai.steamstatic.com/steam/apps/292030/header.jpg" },
      { id: 3, title: "DOOM Eternal", cat: "action", disc: "-80%", old: "2800 ₽", price: "560 ₽", img: "https://cdn.akamai.steamstatic.com/steam/apps/782330/header.jpg" },
      { id: 4, title: "Civilization VI", cat: "strategy", disc: "-90%", old: "1999 ₽", price: "199 ₽", img: "https://cdn.akamai.steamstatic.com/steam/apps/289070/header.jpg" },
      { id: 5, title: "Hollow Knight", cat: "indie", disc: "-50%", old: "550 ₽", price: "275 ₽", img: "https://cdn.akamai.steamstatic.com/steam/apps/367520/header.jpg" }
    ];

    let currentCat = 'all';

    function showGames() {
      const q = document.getElementById("search").value.toLowerCase();
      const list = document.getElementById("games");
      list.innerHTML = "";

      const res = items.filter(g => {
        const byCat = currentCat === 'all' || g.cat === currentCat;
        const byQ = g.title.toLowerCase().includes(q);
        return byCat && byQ;
      });

      if (!res.length) {
        list.innerHTML = "<p style='text-align:center; opacity:0.6;'>Ничего не найдено</p>";
        return;
      }

      res.forEach(g => {
        const div = document.createElement("div");
        div.className = "card";
        div.innerHTML = `
          <img src="${g.img}">
          <div class="info">
            <div class="title">${g.title}</div>
            <div class="prices">
              <span class="badge">${g.disc}</span>
              <span class="old-price">${g.old}</span>
              <span class="new-price">${g.price}</span>
            </div>
          </div>
        `;
        list.appendChild(div);
      });
    }

    function filterCat(cat, btn) {
      currentCat = cat;
      document.querySelectorAll(".cat-btn").forEach(b => b.classList.remove("active"));
      btn.classList.add("active");
      showGames();
    }

    document.getElementById("search").addEventListener("input", showGames);
    showGames();
  </script>
</body>
</html>
