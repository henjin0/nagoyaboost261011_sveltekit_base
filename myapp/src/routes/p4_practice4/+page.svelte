<script>
    import { onMount } from "svelte";
    import Chart from "chart.js/auto";

    let canvas = $state();
    let chart;
    let weatherData = $state([]);
    let loading = $state(true);
    let error = $state("");

    const JMA_BASE_URL =
        "https://www.data.jma.go.jp/stats/etrn/view/daily_s1.php";

    const getTargetDates = () => {
        const today = new Date();
        today.setHours(0, 0, 0, 0);

        const dates = [];

        for (let i = 30; i >= 0; i--) {
            const date = new Date(today);
            date.setDate(today.getDate() - i);
            dates.push(date);
        }

        return dates;
    };

    const formatDate = (date) => {
        const year = date.getFullYear();
        const month = String(date.getMonth() + 1).padStart(2, "0");
        const day = String(date.getDate()).padStart(2, "0");

        return `${year}-${month}-${day}`;
    };

    const createJmaUrl = (year, month) => {
        const params = new URLSearchParams({
            block_no: "47636",
            day: "",
            month: String(month),
            prec_no: "51",
            view: "p1s",
            year: String(year),
        });

        return `${JMA_BASE_URL}?${params.toString()}`;
    };

    const parseNumber = (cell) => {
        const value = cell.textContent.trim();

        if (!value || value === "--") {
            return null;
        }

        const number = Number(value);

        return Number.isFinite(number) ? number : null;
    };

    const parseJmaData = (html, year, month) => {
        const parser = new DOMParser();
        const document = parser.parseFromString(html, "text/html");

        const tables = [...document.querySelectorAll("table")];

        const table = tables.find((table) =>
            table.textContent.includes("天気概況")
        );

        if (!table) {
            throw new Error(
                `${year}年${month}月の気象データ表が見つかりませんでした。`
            );
        }

        const rows = [...table.querySelectorAll("tr")];
        const result = [];

        for (const row of rows) {
            const cells = [...row.querySelectorAll("td")];

            if (cells.length < 9) {
                continue;
            }

            const dayText = cells[0].textContent.trim();

            if (!/^\d+$/.test(dayText)) {
                continue;
            }

            const day = Number(dayText);
            const date = new Date(year, month - 1, day);

            result.push({
                date: formatDate(date),
                label: `${month}/${day}`,
                averageTemperature: parseNumber(cells[6]),
                maximumTemperature: parseNumber(cells[7]),
                minimumTemperature: parseNumber(cells[8]),
                precipitation: parseNumber(cells[5]),
                daytimeWeather:
                    cells[cells.length - 2]?.textContent.trim() ?? "",
                nighttimeWeather:
                    cells[cells.length - 1]?.textContent.trim() ?? "",
            });
        }

        return result;
    };

    const loadWeatherData = async () => {
        loading = true;
        error = "";

        try {
            const targetDates = getTargetDates();

            const months = new Map();

            for (const date of targetDates) {
                const key = `${date.getFullYear()}-${date.getMonth() + 1}`;

                if (!months.has(key)) {
                    months.set(key, {
                        year: date.getFullYear(),
                        month: date.getMonth() + 1,
                    });
                }
            }

            const responses = await Promise.all(
                [...months.values()].map(async ({ year, month }) => {
                    const response = await fetch(
                        createJmaUrl(year, month)
                    );

                    if (!response.ok) {
                        throw new Error(
                            `気象庁データの取得に失敗しました: ${year}-${String(
                                month
                            ).padStart(2, "0")}`
                        );
                    }

                    const html = await response.text();

                    return parseJmaData(html, year, month);
                })
            );

            const targetDateSet = new Set(
                targetDates.map((date) => formatDate(date))
            );

            weatherData = responses
                .flat()
                .filter((data) => targetDateSet.has(data.date))
                .sort((a, b) => a.date.localeCompare(b.date));

            if (weatherData.length === 0) {
                throw new Error(
                    "対象期間の気象データが見つかりませんでした。"
                );
            }
        } catch (e) {
            console.error(e);

            error =
                e instanceof Error
                    ? e.message
                    : "気象データの取得に失敗しました。";
        } finally {
            loading = false;
        }
    };

    let labels = $derived(
        weatherData.map((data) => data.label)
    );

    let averageTemperatures = $derived(
        weatherData.map((data) => data.averageTemperature)
    );

    let maximumTemperatures = $derived(
        weatherData.map((data) => data.maximumTemperature)
    );

    let minimumTemperatures = $derived(
        weatherData.map((data) => data.minimumTemperature)
    );

    let precipitations = $derived(
        weatherData.map((data) => data.precipitation)
    );

    onMount(() => {
        loadWeatherData();

        return () => {
            if (chart) {
                chart.destroy();
                chart = undefined;
            }
        };
    });

    /*
     * weatherDataが取得されたらグラフを作成する。
     *
     * Chartインスタンス自体は$stateにしない。
     * Chart.jsはSvelteのリアクティブ状態として管理する必要がないため。
     */
    $effect(() => {
        if (!canvas || weatherData.length === 0) {
            return;
        }

        // すでにChartが存在する場合は新しく作らず更新する
        if (chart) {
            chart.data.labels = labels;

            chart.data.datasets[0].data = averageTemperatures;
            chart.data.datasets[1].data = maximumTemperatures;
            chart.data.datasets[2].data = minimumTemperatures;
            chart.data.datasets[3].data = precipitations;

            chart.update();

            return;
        }

        chart = new Chart(canvas, {
            data: {
                labels,

                datasets: [
                    {
                        type: "line",
                        label: "平均気温",
                        data: averageTemperatures,
                        borderWidth: 2,
                        tension: 0.3,
                        yAxisID: "temperature",
                    },
                    {
                        type: "line",
                        label: "最高気温",
                        data: maximumTemperatures,
                        borderWidth: 2,
                        tension: 0.3,
                        yAxisID: "temperature",
                    },
                    {
                        type: "line",
                        label: "最低気温",
                        data: minimumTemperatures,
                        borderWidth: 2,
                        tension: 0.3,
                        yAxisID: "temperature",
                    },
                    {
                        type: "bar",
                        label: "降水量",
                        data: precipitations,
                        borderWidth: 1,
                        yAxisID: "precipitation",
                    },
                ],
            },

            options: {
                responsive: true,
                maintainAspectRatio: false,

                interaction: {
                    mode: "index",
                    intersect: false,
                },

                plugins: {
                    title: {
                        display: true,
                        text: "名古屋の過去30日間の気象データ",
                    },

                    tooltip: {
                        callbacks: {
                            afterBody: (items) => {
                                const index = items[0]?.dataIndex;

                                if (index === undefined) {
                                    return [];
                                }

                                const data = weatherData[index];

                                return [
                                    "",
                                    `昼：${
                                        data.daytimeWeather || "データなし"
                                    }`,
                                    `夜：${
                                        data.nighttimeWeather || "データなし"
                                    }`,
                                ];
                            },
                        },
                    },
                },

                scales: {
                    x: {
                        title: {
                            display: true,
                            text: "日付",
                        },

                        ticks: {
                            maxTicksLimit: 15,
                        },
                    },

                    temperature: {
                        type: "linear",
                        position: "left",

                        title: {
                            display: true,
                            text: "気温（℃）",
                        },
                    },

                    precipitation: {
                        type: "linear",
                        position: "right",

                        title: {
                            display: true,
                            text: "降水量（mm）",
                        },

                        grid: {
                            drawOnChartArea: false,
                        },
                    },
                },
            },
        });
    });
</script>

<svelte:head>
    <title>名古屋の過去30日間の天気</title>
</svelte:head>

<div class="container">
    <h1>名古屋の過去30日間の天気</h1>

    <p class="description">
        気象庁の観測データをもとに、今日から30日前までの気温と降水量を表示しています。
    </p>

    {#if loading}
        <div class="status">
            気象庁から気象データを取得しています……
        </div>
    {:else if error}
        <div class="error">
            {error}
        </div>
    {:else}
        <div class="chart">
            <canvas bind:this={canvas}></canvas>
        </div>

        <div class="info">
            <p>
                グラフにマウスを合わせると、その日の天気概況を確認できます。
            </p>

            <p>
                データ：気象庁「過去の気象データ検索」
            </p>
        </div>
    {/if}
</div>

<style>
    .container {
        width: min(1100px, 100%);
        margin: 0 auto;
        padding: 24px;
        box-sizing: border-box;
    }

    h1 {
        margin: 0 0 8px;
        font-size: 28px;
    }

    .description {
        margin: 0 0 24px;
        color: #666;
    }

    .chart {
        position: relative;
        width: 100%;
        height: 500px;
    }

    .status {
        padding: 40px;
        text-align: center;
        color: #666;
    }

    .error {
        padding: 16px;
        border: 1px solid #e57373;
        border-radius: 8px;
        background: #ffebee;
        color: #c62828;
    }

    .info {
        margin-top: 16px;
        font-size: 13px;
        color: #666;
    }

    .info p {
        margin: 4px 0;
    }
</style>