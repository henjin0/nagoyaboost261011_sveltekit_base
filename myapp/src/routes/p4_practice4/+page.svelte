<script>
    import * as THREE from "three";
    import { OrbitControls } from "three/addons/controls/OrbitControls.js";
    import { onMount } from "svelte";

    let canvas = $state();
    let loading = $state(true);
    let error = $state("");
    let prefectureCount = $state(0);

    const modelStatus = $derived(
        loading
            ? "日本列島の地形データを読み込んでいます..."
            : error
                ? "読み込みに失敗しました"
                : `日本列島を3D化しました（${prefectureCount}都道府県）`,
    );

    // 日本の都道府県ポリゴン
    // Geolonia / prefecture-tiles で公開されているGeoJSON
    const GEOJSON_URL =
        "https://raw.githubusercontent.com/geolonia/prefecture-tiles/master/prefectures.geojson";

    onMount(() => {
        const scene = new THREE.Scene();

        // 背景
        scene.background = new THREE.Color(0xfff8dc);

        // 軸
        const axesHelper = new THREE.AxesHelper(2);
        scene.add(axesHelper);

        // カメラ
        const camera = new THREE.PerspectiveCamera(
            45,
            canvas.clientWidth / canvas.clientHeight,
            0.1,
            100,
        );

        camera.position.set(5.5, 5.5, 8);
        camera.updateProjectionMatrix();

        // ライト
        const directionalLight = new THREE.DirectionalLight(0xffffff, 2.2);
        directionalLight.position.set(-5, 10, 5);
        directionalLight.castShadow = true;
        scene.add(directionalLight);

        const ambientLight = new THREE.AmbientLight(0xffffff, 1.5);
        scene.add(ambientLight);

        // レンダラー
        const renderer = new THREE.WebGLRenderer({
            canvas,
            antialias: true,
        });

        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.setSize(canvas.clientWidth, canvas.clientHeight, false);

        // OrbitControls
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.08;
        controls.target.set(0, 0, 0);

        // 日本列島を格納するグループ
        const japanGroup = new THREE.Group();
        scene.add(japanGroup);

        // ------------------------------------------------------------
        // リングの面積を計算
        //
        // 正:
        //   CCW
        //
        // 負:
        //   CW
        //
        // GeoJSONのリングの向きに依存せず、
        // 外周をCCW、穴をCWに統一することで
        // ExtrudeGeometryの表裏反転を防ぐ。
        // ------------------------------------------------------------
        function signedArea(ring) {
            let area = 0;

            for (let i = 0; i < ring.length - 1; i++) {
                const [x1, y1] = ring[i];
                const [x2, y2] = ring[i + 1];

                area += x1 * y2 - x2 * y1;
            }

            return area / 2;
        }

        // ------------------------------------------------------------
        // 緯度経度 → Three.js座標
        //
        // X = 経度
        // Z = 緯度
        // Y = 高さ
        //
        // これにより、
        // 東西 = X
        // 南北 = Z
        // 上下 = Y
        //
        // という直感的な座標系になる。
        // ------------------------------------------------------------
        function convertRing(ring, centerLng, centerLat) {
            return ring.map(([lng, lat]) => {
                return new THREE.Vector2(
                    (lng - centerLng) * 1.0,
                    (lat - centerLat) * 1.0,
                );
            });
        }

        // ------------------------------------------------------------
        // Polygonを3D化
        // ------------------------------------------------------------
        function createPolygon(
            coordinates,
            centerLng,
            centerLat,
            material,
        ) {
            if (!coordinates || coordinates.length === 0) {
                return null;
            }

            const outer = coordinates[0];

            if (!outer || outer.length < 4) {
                return null;
            }

            // 外周をCCWに統一
            let outerRing = outer;

            if (signedArea(outerRing) < 0) {
                outerRing = [...outerRing].reverse();
            }

            const shape = new THREE.Shape();

            const outerPoints = convertRing(
                outerRing,
                centerLng,
                centerLat,
            );

            outerPoints.forEach((point, index) => {
                if (index === 0) {
                    shape.moveTo(point.x, point.y);
                } else {
                    shape.lineTo(point.x, point.y);
                }
            });

            // 穴を追加
            for (let i = 1; i < coordinates.length; i++) {
                const hole = coordinates[i];

                if (!hole || hole.length < 4) {
                    continue;
                }

                // 穴はCWに統一
                let holeRing = hole;

                if (signedArea(holeRing) > 0) {
                    holeRing = [...holeRing].reverse();
                }

                const holePath = new THREE.Path();

                const holePoints = convertRing(
                    holeRing,
                    centerLng,
                    centerLat,
                );

                holePoints.forEach((point, index) => {
                    if (index === 0) {
                        holePath.moveTo(point.x, point.y);
                    } else {
                        holePath.lineTo(point.x, point.y);
                    }
                });

                shape.holes.push(holePath);
            }

            // 押し出して3D化
            const geometry = new THREE.ExtrudeGeometry(shape, {
                depth: 0.12,
                bevelEnabled: false,
                curveSegments: 1,
                steps: 1,
            });

            // ShapeGeometry / ExtrudeGeometryはXY平面を基準に作られる。
            //
            // ここで -90度回転させることで、
            //
            // 元のX → Three.js X
            // 元のY → Three.js Z
            // Extrude方向 → Three.js Y
            //
            // となる。
            //
            // つまり、
            // 経度 = X
            // 緯度 = Z
            // 高さ = Y
            //
            // になり、日本列島が上下反転しない。
            geometry.rotateX(-Math.PI / 2);

            geometry.computeVertexNormals();

            const mesh = new THREE.Mesh(geometry, material);

            mesh.castShadow = true;
            mesh.receiveShadow = true;

            return mesh;
        }

        // ------------------------------------------------------------
        // GeoJSONのFeatureを3D化
        // ------------------------------------------------------------
        function createFeature(
            feature,
            centerLng,
            centerLat,
            material,
        ) {
            const geometry = feature.geometry;

            if (!geometry) {
                return null;
            }

            const group = new THREE.Group();

            if (geometry.type === "Polygon") {
                const mesh = createPolygon(
                    geometry.coordinates,
                    centerLng,
                    centerLat,
                    material,
                );

                if (mesh) {
                    group.add(mesh);
                }
            }

            if (geometry.type === "MultiPolygon") {
                for (const polygon of geometry.coordinates) {
                    const mesh = createPolygon(
                        polygon,
                        centerLng,
                        centerLat,
                        material,
                    );

                    if (mesh) {
                        group.add(mesh);
                    }
                }
            }

            return group.children.length > 0 ? group : null;
        }

        // ------------------------------------------------------------
        // GeoJSON読み込み
        // ------------------------------------------------------------
        async function loadJapan() {
            try {
                loading = true;
                error = "";

                const response = await fetch(GEOJSON_URL);

                if (!response.ok) {
                    throw new Error(
                        `GeoJSONの取得に失敗しました: ${response.status}`,
                    );
                }

                const geojson = await response.json();

                if (
                    !geojson ||
                    geojson.type !== "FeatureCollection" ||
                    !Array.isArray(geojson.features)
                ) {
                    throw new Error(
                        "GeoJSONの形式が正しくありません。",
                    );
                }

                // ----------------------------------------------------
                // 日本列島の中心位置
                //
                // 日本全体を扱うため、単純な緯度経度の中心を使用。
                // この中心を原点にすることで数値を小さくする。
                // ----------------------------------------------------
                const centerLng = 137.0;
                const centerLat = 36.0;

                // ----------------------------------------------------
                // マテリアル
                // ----------------------------------------------------
                const material = new THREE.MeshStandardMaterial({
                    color: 0x4f9d69,
                    roughness: 0.85,
                    metalness: 0.05,
                    side: THREE.FrontSide,
                });

                // 都道府県ごとに3D化
                let count = 0;

                for (const feature of geojson.features) {
                    const group = createFeature(
                        feature,
                        centerLng,
                        centerLat,
                        material,
                    );

                    if (group) {
                        japanGroup.add(group);
                        count++;
                    }
                }

                prefectureCount = count;

                // ----------------------------------------------------
                // 日本列島全体を少し縮小
                //
                // 元の緯度経度の差をそのまま使うと大きすぎるため、
                // 全体を適度なサイズにする。
                // ----------------------------------------------------
                japanGroup.scale.set(0.35, 0.35, 0.35);

                // ----------------------------------------------------
                // 日本列島の向きを確認
                //
                // 北海道 → 北側(+Z)
                // 九州 → 南側(-Z)
                // 東日本 → +X
                // 西日本 → -X
                //
                // とする。
                // ----------------------------------------------------
                japanGroup.rotation.set(0, 0, 0);

                // 日本列島の中心を少し下げる
                japanGroup.position.y = 0;

                // ----------------------------------------------------
                // カメラを日本列島が見える位置へ
                // ----------------------------------------------------
                camera.position.set(4.5, 5.5, 7.5);
                controls.target.set(0, 0, 0);
                controls.update();

                loading = false;
            } catch (e) {
                console.error(e);

                error =
                    e instanceof Error
                        ? e.message
                        : "日本列島の読み込みに失敗しました";

                loading = false;
            }
        }

        loadJapan();

        // ------------------------------------------------------------
        // リサイズ
        // ------------------------------------------------------------
        function resize() {
            const width = canvas.clientWidth;
            const height = canvas.clientHeight;

            if (width === 0 || height === 0) {
                return;
            }

            camera.aspect = width / height;
            camera.updateProjectionMatrix();

            renderer.setSize(width, height, false);
        }

        const resizeObserver = new ResizeObserver(resize);
        resizeObserver.observe(canvas);

        // ------------------------------------------------------------
        // アニメーション
        // ------------------------------------------------------------
        let animationId;

        function animate() {
            animationId = requestAnimationFrame(animate);

            controls.update();

            renderer.render(scene, camera);
        }

        animate();

        // ------------------------------------------------------------
        // Svelte / Three.jsの後片付け
        // ------------------------------------------------------------
        return () => {
            cancelAnimationFrame(animationId);

            resizeObserver.disconnect();

            controls.dispose();

            scene.traverse((object) => {
                if (object instanceof THREE.Mesh) {
                    object.geometry?.dispose();

                    if (Array.isArray(object.material)) {
                        object.material.forEach((material) => {
                            material.dispose();
                        });
                    } else {
                        object.material?.dispose();
                    }
                }
            });

            renderer.dispose();
        };
    });

    // $effectを使って状態変化を監視
    $effect(() => {
        if (error) {
            console.error("日本列島3Dモデル:", error);
        }
    });
</script>

<div class="container">
    <div class="status">
        {modelStatus}
    </div>

    <canvas
        bind:this={canvas}
        width="700"
        height="600"
    ></canvas>
</div>

<style>
    .container {
        width: 700px;
        max-width: 100%;
    }

    .status {
        min-height: 24px;
        margin-bottom: 8px;
        font-family: sans-serif;
        font-size: 14px;
        color: #333;
    }

    canvas {
        display: block;
        width: 700px;
        height: 600px;
        max-width: 100%;
        background: #fff8dc;
        border-radius: 8px;
    }
</style>