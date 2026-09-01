<script>
    import { onMount } from "svelte";
    import Chart from "chart.js/auto";

    let canvas = $state();
    let chart;

    let condition = $state("普通");
    let temperature = $state(36.5);
    let records = $state([]);

    const conditions = ["とても良い", "良い", "普通", "悪い", "とても悪い"];

    const chartLabels = $derived(
        records.map((record) => record.time)
    );

    const chartData = $derived(
        records.map((record) => record.temperature)
    );

    function currentTime() {
        const now = new Date();

        return now.toLocaleTimeString("ja-JP", {
            hour: "2-digit",
            minute: "2-digit",
            second: "2-digit",
        });
    }

    function addRecord() {
        const now = new Date();

        records = [
            ...records,
            {
                id: now.getTime(),
                time: currentTime(),
                condition,
                temperature: Number(temperature),
            },
        ];
    }

    function deleteRecord(id) {
        records = records.filter((record) => record.id !== id);
    }

    onMount(() => {
        chart = new Chart(canvas, {
            type: "line",
            data: {
                labels: chartLabels,
                datasets: [
                    {
                        label: "体温（℃）",
                        data: chartData,
                        borderColor: "#ef4444",
                        backgroundColor: "rgba(239, 68, 68, 0.15)",
                        borderWidth: 2,
                        tension: 0.3,
                        fill: true,
                        pointRadius: 4,
                        pointBackgroundColor: "#ef4444",
                    },
                ],
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    y: {
                        title: {
                            display: true,
                            text: "体温（℃）",
                        },
                        suggestedMin: 35,
                        suggestedMax: 39,
                    },
                    x: {
                        title: {
                            display: true,
                            text: "記録時刻",
                        },
                    },
                },
            },
        });
    });

    $effect(() => {
        if (!chart) return;

        chart.data.labels = chartLabels;
        chart.data.datasets[0].data = chartData;
        chart.update();
    });
</script>

<svelte:head>
    <title>体調・体温記録</title>
</svelte:head>

<div class="page">
    <header>
        <h1>体調・体温記録</h1>
        <p>現在の体調と体温を記録できます。</p>
    </header>

    <main>
        <section class="input-section">
            <h2>体調を記録</h2>

            <div class="form">
                <div class="field">
                    <label for="condition">体調</label>
                    <select id="condition" bind:value={condition}>
                        {#each conditions as item}
                            <option value={item}>{item}</option>
                        {/each}
                    </select>
                </div>

                <div class="field">
                    <label for="temperature">体温（℃）</label>
                    <input
                        id="temperature"
                        type="number"
                        min="30"
                        max="45"
                        step="0.1"
                        bind:value={temperature}
                    />
                </div>

                <button type="button" onclick={addRecord}>
                    現在時刻で記録する
                </button>
            </div>
        </section>

        <section class="chart-section">
            <h2>体温の推移</h2>

            <div class="chart">
                <canvas bind:this={canvas}></canvas>

                {#if records.length === 0}
                    <div class="empty-chart">
                        体温を記録するとグラフが表示されます
                    </div>
                {/if}
            </div>
        </section>

        <section class="records-section">
            <h2>記録一覧</h2>

            {#if records.length === 0}
                <p class="empty">まだ記録がありません。</p>
            {:else}
                <div class="records">
                    {#each records as record}
                        <article class="record">
                            <div class="record-time">
                                {record.time}
                            </div>

                            <div class="record-condition">
                                <span>体調</span>
                                <strong>{record.condition}</strong>
                            </div>

                            <div class="record-temperature">
                                <span>体温</span>
                                <strong>{record.temperature.toFixed(1)}℃</strong>
                            </div>

                            <button
                                type="button"
                                class="delete-button"
                                onclick={() => deleteRecord(record.id)}
                            >
                                削除
                            </button>
                        </article>
                    {/each}
                </div>
            {/if}
        </section>
    </main>
</div>

<style>
    :global(*) {
        box-sizing: border-box;
    }

    :global(body) {
        margin: 0;
        background: #f5f7fa;
        color: #1f2937;
        font-family:
            -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    }

    .page {
        width: 100%;
        min-height: 100vh;
    }

    header {
        width: 100%;
        padding: 32px clamp(20px, 5vw, 80px);
        background: #2563eb;
        color: white;
    }

    header h1 {
        margin: 0 0 8px;
        font-size: clamp(24px, 4vw, 36px);
    }

    header p {
        margin: 0;
        opacity: 0.9;
    }

    main {
        width: 100%;
        padding: 24px clamp(20px, 5vw, 80px) 60px;
    }

    section {
        width: 100%;
        margin-bottom: 24px;
        padding: 24px;
        background: white;
        border-radius: 12px;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
    }

    h2 {
        margin: 0 0 20px;
        font-size: 20px;
    }

    .form {
        display: grid;
        grid-template-columns: 1fr 1fr auto;
        gap: 16px;
        align-items: end;
    }

    .field {
        display: flex;
        flex-direction: column;
        gap: 8px;
    }

    .field label {
        font-weight: 600;
        font-size: 14px;
    }

    select,
    input {
        width: 100%;
        min-height: 44px;
        padding: 8px 12px;
        border: 1px solid #d1d5db;
        border-radius: 8px;
        background: white;
        font-size: 16px;
    }

    button {
        min-height: 44px;
        padding: 8px 20px;
        border: 0;
        border-radius: 8px;
        background: #2563eb;
        color: white;
        font-size: 15px;
        font-weight: 600;
        cursor: pointer;
        white-space: nowrap;
    }

    button:hover {
        background: #1d4ed8;
    }

    .chart {
        position: relative;
        width: 100%;
        height: 400px;
    }

    .empty-chart {
        position: absolute;
        inset: 0;
        display: flex;
        align-items: center;
        justify-content: center;
        color: #9ca3af;
        pointer-events: none;
    }

    .records {
        display: flex;
        flex-direction: column;
        gap: 12px;
    }

    .record {
        display: grid;
        grid-template-columns: 1fr 1fr 1fr auto;
        gap: 20px;
        align-items: center;
        padding: 16px;
        border: 1px solid #e5e7eb;
        border-radius: 10px;
    }

    .record-time {
        font-size: 18px;
        font-weight: 700;
    }

    .record-condition,
    .record-temperature {
        display: flex;
        flex-direction: column;
        gap: 4px;
    }

    .record-condition span,
    .record-temperature span {
        color: #6b7280;
        font-size: 12px;
    }

    .record-temperature strong {
        color: #dc2626;
        font-size: 18px;
    }

    .delete-button {
        background: #f3f4f6;
        color: #4b5563;
    }

    .delete-button:hover {
        background: #e5e7eb;
    }

    .empty {
        margin: 0;
        color: #9ca3af;
        text-align: center;
    }

    @media (max-width: 700px) {
        main {
            padding: 16px 12px 40px;
        }

        header {
            padding: 24px 16px;
        }

        section {
            padding: 16px;
        }

        .form {
            grid-template-columns: 1fr;
        }

        .record {
            grid-template-columns: 1fr 1fr;
        }

        .delete-button {
            width: 100%;
        }

        .chart {
            height: 300px;
        }
    }
</style>
