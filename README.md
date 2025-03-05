# wolking_app
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ウォーキングコース作成アプリ</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.js"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.css" />
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            font-family: 'Helvetica Neue', Arial, sans-serif;
        }
        body {
            background-color: #f5f7fa;
            color: #333;
            line-height: 1.6;
        }
        .container {
            max-width: 100%;
            padding: 16px;
        }
        header {
            text-align: center;
            margin-bottom: 20px;
        }
        h1 {
            font-size: 24px;
            color: #2c3e50;
            margin-bottom: 10px;
        }
        .description {
            font-size: 14px;
            color: #7f8c8d;
            margin-bottom: 20px;
        }
        .input-container {
            background-color: #fff;
            padding: 16px;
            border-radius: 12px;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
            margin-bottom: 20px;
        }
        .input-group {
            margin-bottom: 15px;
        }
        label {
            display: block;
            font-weight: 600;
            margin-bottom: 8px;
            color: #34495e;
        }
        input {
            width: 100%;
            padding: 12px;
            border: 1px solid #ddd;
            border-radius: 8px;
            font-size: 16px;
        }
        button {
            width: 100%;
            padding: 14px;
            background-color: #3498db;
            color: white;
            border: none;
            border-radius: 8px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
            transition: background-color 0.3s;
        }
        button:hover {
            background-color: #2980b9;
        }
        #map {
            height: 400px;
            border-radius: 12px;
            margin-bottom: 20px;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
        }
        .saved-routes {
            background-color: #fff;
            padding: 16px;
            border-radius: 12px;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
        }
        .saved-routes h2 {
            font-size: 18px;
            margin-bottom: 10px;
            color: #2c3e50;
        }
        .route-list {
            list-style: none;
        }
        .route-item {
            padding: 12px;
            border-bottom: 1px solid #eee;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .route-info {
            font-size: 14px;
        }
        .route-distance {
            font-weight: 600;
            color: #3498db;
        }
        .route-date {
            font-size: 12px;
            color: #95a5a6;
        }
        .no-routes {
            text-align: center;
            color: #95a5a6;
            padding: 20px 0;
        }
        .actions {
            margin-top: 15px;
            display: flex;
            gap: 10px;
        }
        .action-btn {
            padding: 8px 12px;
            font-size: 14px;
        }
        .save-btn {
            background-color: #2ecc71;
        }
        .save-btn:hover {
            background-color: #27ae60;
        }
        .loading {
            text-align: center;
            padding: 20px;
            color: #7f8c8d;
        }
        .error {
            color: #e74c3c;
            margin-top: 5px;
            font-size: 14px;
        }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <h1>ウォーキングコース作成アプリ</h1>
            <p class="description">希望する距離を入力するだけで、現在地からのウォーキングコースを作成します</p>
        </header>

        <div class="input-container">
            <div class="input-group">
                <label for="distance">希望する距離 (km)</label>
                <input type="number" id="distance" min="0.5" max="20" step="0.1" value="3">
            </div>
            <div id="location-status"></div>
            <button id="generate-btn">コースを作成</button>
        </div>

        <div id="map"></div>

        <div class="actions">
            <button id="save-route-btn" class="action-btn save-btn" style="display: none;">お気に入りに保存</button>
        </div>

        <div class="saved-routes">
            <h2>保存したコース</h2>
            <ul id="route-list" class="route-list">
                <li class="no-routes">保存したコースはありません</li>
            </ul>
        </div>
    </div>

    <script>
        // アプリケーションの状態
        const appState = {
            map: null,
            userLocation: null,
            currentRoute: null,
            savedRoutes: [],
            routePolyline: null
        };

        // LocalStorageからコースの読み込み
        function loadSavedRoutes() {
            const savedRoutes = localStorage.getItem('walkingRoutes');
            if (savedRoutes) {
                appState.savedRoutes = JSON.parse(savedRoutes);
                updateRouteList();
            }
        }

        // LocalStorageにコースを保存
        function saveRoutes() {
            localStorage.setItem('walkingRoutes', JSON.stringify(appState.savedRoutes));
        }

        // コースリストの更新
        function updateRouteList() {
            const routeList = document.getElementById('route-list');
            
            if (appState.savedRoutes.length === 0) {
                routeList.innerHTML = '<li class="no-routes">保存したコースはありません</li>';
                return;
            }
            
            routeList.innerHTML = '';
            appState.savedRoutes.forEach((route, index) => {
                const date = new Date(route.date);
                const formattedDate = `${date.getFullYear()}/${date.getMonth() + 1}/${date.getDate()}`;
                
                const li = document.createElement('li');
                li.className = 'route-item';
                li.innerHTML = `
                    <div class="route-info">
                        <div>コース ${index + 1} <span class="route-distance">${route.distance}km</span></div>
                        <div class="route-date">${formattedDate}</div>
                    </div>
                    <button class="show-route" data-index="${index}">表示</button>
                `;
                routeList.appendChild(li);
            });
            
            // 保存したコースを表示するイベントリスナー
            document.querySelectorAll('.show-route').forEach(btn => {
                btn.addEventListener('click', function() {
                    const index = parseInt(this.getAttribute('data-index'));
                    showSavedRoute(index);
                });
            });
        }

        // マップの初期化
        function initMap(lat, lng) {
            // 既存のマップがあれば削除
            if (appState.map) {
                appState.map.remove();
            }
            
            const mapContainer = document.getElementById('map');
            appState.map = L.map(mapContainer).setView([lat, lng], 15);
            
            // OpenStreetMapを利用（無料）
            L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
            }).addTo(appState.map);
            
            // 現在地のマーカー
            L.marker([lat, lng]).addTo(appState.map)
                .bindPopup('現在地')
                .openPopup();
        }

        // コースの生成（シンプルな円形ルート）
        function generateRoute(lat, lng, distance) {
            // 距離をメートルに変換
            const radiusInMeters = distance * 1000 / 2;
            
            // 円形のルートを作成（実際のアプリではAPIを使ってより自然なルートを作成する）
            const points = [];
            const steps = 20;
            
            for (let i = 0; i <= steps; i++) {
                const angle = (i / steps) * 2 * Math.PI;
                
                // 緯度経度をオフセット
                // 緯度1度は約111km、経度1度は緯度によって異なる (約111km * cos(lat))
                const latOffset = (radiusInMeters / 111000) * Math.sin(angle);
                const lngOffset = (radiusInMeters / (111000 * Math.cos(lat * Math.PI / 180))) * Math.cos(angle);
                
                points.push([lat + latOffset, lng + lngOffset]);
            }
            
            // 始点終点を現在地に
            points[0] = [lat, lng];
            points[points.length - 1] = [lat, lng];
            
            // 既存のルートがあれば削除
            if (appState.routePolyline) {
                appState.map.removeLayer(appState.routePolyline);
            }
            
            // 新しいルートを描画
            appState.routePolyline = L.polyline(points, {color: '#3498db', weight: 5}).addTo(appState.map);
            appState.map.fitBounds(appState.routePolyline.getBounds());
            
            // 現在のルートを保存
            appState.currentRoute = {
                points: points,
                distance: distance,
                date: new Date().toISOString()
            };
            
            // 保存ボタンを表示
            document.getElementById('save-route-btn').style.display = 'block';
        }

        // 保存したコースを表示
        function showSavedRoute(index) {
            const route = appState.savedRoutes[index];
            
            // 既存のルートがあれば削除
            if (appState.routePolyline) {
                appState.map.removeLayer(appState.routePolyline);
            }
            
            // 保存されたルートを描画
            appState.routePolyline = L.polyline(route.points, {color: '#3498db', weight: 5}).addTo(appState.map);
            appState.map.fitBounds(appState.routePolyline.getBounds());
            
            // 現在のルートを更新
            appState.currentRoute = route;
            
            // 保存ボタンを非表示（既に保存済み）
            document.getElementById('save-route-btn').style.display = 'none';
        }

        // ユーザーの現在地を取得
        function getUserLocation() {
            const locationStatus = document.getElementById('location-status');
            locationStatus.textContent = '位置情報を取得中...';
            
            if (navigator.geolocation) {
                navigator.geolocation.getCurrentPosition(
                    (position) => {
                        // 成功
                        const lat = position.coords.latitude;
                        const lng = position.coords.longitude;
                        
                        appState.userLocation = {lat, lng};
                        locationStatus.textContent = '位置情報の取得に成功しました';
                        
                        // マップの初期化
                        initMap(lat, lng);
                    },
                    (error) => {
                        // エラー
                        console.error('位置情報の取得に失敗しました:', error);
                        locationStatus.innerHTML = '<span class="error">位置情報の取得に失敗しました。位置情報の許可を確認してください。</span>';
                        
                        // デフォルトの位置（東京）
                        appState.userLocation = {lat: 35.6895, lng: 139.6917};
                        initMap(35.6895, 139.6917);
                    }
                );
            } else {
                locationStatus.innerHTML = '<span class="error">お使いのブラウザは位置情報をサポートしていません。</span>';
                
                // デフォルトの位置（東京）
                appState.userLocation = {lat: 35.6895, lng: 139.6917};
                initMap(35.6895, 139.6917);
            }
        }

        // アプリの初期化
        function init() {
            // 保存されたコースを読み込み
            loadSavedRoutes();
            
            // 位置情報の取得
            getUserLocation();
            
            // イベントリスナーを設定
            document.getElementById('generate-btn').addEventListener('click', () => {
                const distanceInput = document.getElementById('distance');
                const distance = parseFloat(distanceInput.value);
                
                if (isNaN(distance) || distance <= 0) {
                    alert('有効な距離を入力してください');
                    return;
                }
                
                if (appState.userLocation) {
                    generateRoute(appState.userLocation.lat, appState.userLocation.lng, distance);
                } else {
                    alert('位置情報が取得できていません');
                }
            });
            
            document.getElementById('save-route-btn').addEventListener('click', () => {
                if (appState.currentRoute) {
                    appState.savedRoutes.push(appState.currentRoute);
                    saveRoutes();
                    updateRouteList();
                    document.getElementById('save-route-btn').style.display = 'none';
                    alert('コースを保存しました');
                }
            });
        }

        // ページロード時にアプリを初期化
        window.addEventListener('load', init);
    </script>
</body>
</html>
