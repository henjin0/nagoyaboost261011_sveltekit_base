<script>
    import "leaflet/dist/leaflet.css";

    let mapElement = $state();

    $effect(() => {
        if (!mapElement) return;

        let map;
        let destroyed = false;

        const initializeMap = async () => {
            // Leafletはブラウザ専用ライブラリのため、ここでimportする
            const L = (await import("leaflet")).default;

            if (destroyed) return;

            // 名古屋駅を中心に地図を表示
            map = L.map(mapElement).setView([35.1709, 136.8815], 15);

            // 地図タイル（OpenStreetMap）
            L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
                attribution: "&copy; OpenStreetMap contributors",
            }).addTo(map);

            // 地図をクリックしたときの処理
            map.on("click", async (event) => {
                const { lat, lng } = event.latlng;

                // まずクリック地点に仮のポップアップを表示
                const popup = L.popup()
                    .setLatLng([lat, lng])
                    .setContent("場所の情報を取得しています…")
                    .openOn(map);

                try {
                    // Nominatimを使って緯度・経度から地名を取得
                    const response = await fetch(
                        `https://nominatim.openstreetmap.org/reverse?format=jsonv2&lat=${lat}&lon=${lng}&accept-language=ja`
                    );

                    if (!response.ok) {
                        throw new Error("地名の取得に失敗しました");
                    }

                    const data = await response.json();

                    const address = data.address ?? {};

                    // 取得できた住所情報を表示
                    const displayName =
                        data.display_name ?? "地名を取得できませんでした";

                    const details = [
                        address.country,
                        address.state,
                        address.city ||
                            address.town ||
                            address.village ||
                            address.municipality,
                        address.city_district,
                        address.suburb,
                        address.neighbourhood,
                    ]
                        .filter(Boolean)
                        .filter(
                            (value, index, array) =>
                                array.indexOf(value) === index
                        )
                        .join(" ");

                    popup.setContent(`
                        <div class="popup-content">
                            <strong>${escapeHtml(
                                address.amenity ||
                                    address.building ||
                                    address.road ||
                                    "クリック地点"
                            )}</strong>
                            <p>${escapeHtml(details || displayName)}</p>
                            <small>
                                緯度: ${lat.toFixed(6)}<br>
                                経度: ${lng.toFixed(6)}
                            </small>
                        </div>
                    `);
                } catch (error) {
                    console.error(error);

                    popup.setContent(`
                        <div>
                            <strong>クリック地点</strong>
                            <p>地名を取得できませんでした。</p>
                            <small>
                                緯度: ${lat.toFixed(6)}<br>
                                経度: ${lng.toFixed(6)}
                            </small>
                        </div>
                    `);
                }
            });
        };

        // HTML文字列に入れる値をエスケープ
        const escapeHtml = (value) => {
            return String(value)
                .replaceAll("&", "&amp;")
                .replaceAll("<", "&lt;")
                .replaceAll(">", "&gt;")
                .replaceAll('"', "&quot;")
                .replaceAll("'", "&#039;");
        };

        initializeMap();

        // コンポーネント破棄時に地図を削除
        return () => {
            destroyed = true;

            if (map) {
                map.remove();
                map = undefined;
            }
        };
    });
</script>

<div bind:this={mapElement} class="map"></div>

<style>
    :global(html),
    :global(body) {
        margin: 0;
        width: 100%;
        height: 100%;
    }

    :global(body) {
        overflow: hidden;
    }

    :global(#svelte) {
        width: 100%;
        height: 100%;
    }

    .map {
        width: 100vw;
        height: 100vh;
    }

    :global(.popup-content) {
        line-height: 1.5;
    }

    :global(.popup-content strong) {
        font-size: 1.1rem;
    }

    :global(.popup-content p) {
        margin: 6px 0;
    }

    :global(.popup-content small) {
        color: #666;
    }
</style>