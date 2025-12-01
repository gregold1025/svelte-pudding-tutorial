<script>
        import lyricTranscription from "$data/lyric-transcription.json";
        import drumTranscription from "$data/drum-transcription.json";

        const words = lyricTranscription.lines.flatMap((line) => line.words);
        const beats = drumTranscription.beats;

        const beatsPerBar = 4;
        const fallbackBeatInterval = beats.length > 1
                ? Math.max(beats[beats.length - 1] - beats[beats.length - 2], 0.25)
                : 0.5;

        const bars = [];
        for (let i = 0; i < beats.length; i += beatsPerBar) {
                const start = beats[i];
                if (start === undefined) break;

                const end =
                        beats[i + beatsPerBar] ?? beats[beats.length - 1] + fallbackBeatInterval;

                const wordsInBar = words.filter((word) => word.start >= start && word.start < end);
                const syllables = wordsInBar.reduce((sum, word) => sum + (word.nSyllables ?? 0), 0);
                bars.push({
                        index: bars.length + 1,
                        start,
                        end,
                        syllables,
                        words: wordsInBar
                });
        }

        const groupSize = 8;
        const colors = [
                "#4c7ef3",
                "#e46386",
                "#5bbf8a",
                "#f2a950",
                "#9b7ad3",
                "#4fb1c9",
                "#f27f3d",
                "#6fb872"
        ];

        const groups = [];
        for (let i = 0; i < bars.length; i += groupSize) {
                const slice = bars.slice(i, i + groupSize);
                const total = slice.reduce((sum, bar) => sum + bar.syllables, 0);
                const avg = slice.length ? total / slice.length : 0;
                groups.push({
                        index: groups.length,
                        bars: slice,
                        avg,
                        color: colors[groups.length % colors.length]
                });
        }

        const maxSyllablesPerBar = Math.max(...bars.map((bar) => bar.syllables), 1);

        const syllableBuckets = Array.from({ length: 6 }, (_, i) => ({
                count: i + 1,
                words: []
        }));

        words.forEach((word) => {
                const n = Math.min(word.nSyllables ?? 0, 6);
                if (n > 0) {
                        syllableBuckets[n - 1].words.push(word);
                }
        });

        const yTickCount = 5;
        const yTickStep = Math.max(1, Math.ceil(maxSyllablesPerBar / (yTickCount - 1)));
        const yTicks = Array.from({ length: yTickCount }, (_, i) => i * yTickStep);

        const formatTime = (seconds) => seconds.toFixed(2) + "s";

        let hoveredBar = null;
        const onHover = (bar) => {
                hoveredBar = bar;
        };
        const clearHover = () => {
                hoveredBar = null;
        };
</script>

<section class="graphic">
        <div class="header">
                <div>
                        <p class="eyebrow">Syllable density</p>
                        <h1>Bars and 8-bar averages</h1>
                        <p>
                                Each vertical bar shows syllables that <strong>start</strong> within a four-beat bar.
                                Hover any bar to view the words and timings inside that bar. Every eight bars share a
                                color, and a wider translucent bar behind them shows the eight-bar average.
                        </p>
                </div>
                <div class="legend">
                        <span class="legend__swatch legend__swatch--bar"></span>
                        <span>Single bar syllables</span>
                        <span class="legend__swatch legend__swatch--avg"></span>
                        <span>8-bar average</span>
                </div>
        </div>

        <div class="chart">
                <div class="chart__y-axis">
                        {#each yTicks as tick}
                                <div class="chart__y-tick" style={`bottom: ${(tick / maxSyllablesPerBar) * 100}%`}>
                                        <span>{tick}</span>
                                        <div class="chart__y-line"></div>
                                </div>
                        {/each}
                </div>

                <div class="chart__plot" on:mouseleave={clearHover}>
                        {#each groups as group}
                                <div
                                        class="chart__avg"
                                        style={`left: ${(group.index * groupSize * 100) / bars.length}%; width: ${(group.bars.length * 100) / bars.length}%; height: ${(group.avg / maxSyllablesPerBar) * 100}%; background-color: ${group.color};`}
                                >
                                        <span>Average</span>
                                </div>
                        {/each}

                        <div class="chart__bars">
                                {#each bars as bar}
                                        <div class="chart__cell">
                                                <div
                                                        class="chart__bar"
                                                        style={`height: ${(bar.syllables / maxSyllablesPerBar) * 100}%; background-color: ${colors[Math.floor((bar.index - 1) / groupSize) % colors.length]};`}
                                                        on:mouseenter={() => onHover(bar)}
                                                ></div>
                                        </div>
                                {/each}
                        </div>

                        {#if hoveredBar}
                                <div class="tooltip">
                                        <div class="tooltip__title">Bar {hoveredBar.index}</div>
                                        <div class="tooltip__row">
                                                <span>Window</span>
                                                <span>{formatTime(hoveredBar.start)} – {formatTime(hoveredBar.end)}</span>
                                        </div>
                                        <div class="tooltip__row">
                                                <span>Syllables</span>
                                                <span>{hoveredBar.syllables}</span>
                                        </div>
                                        <div class="tooltip__row">
                                                <span>Words</span>
                                                <span>{hoveredBar.words.length}</span>
                                        </div>
                                        {#if hoveredBar.words.length}
                                                <div class="tooltip__words">
                                                        {#each hoveredBar.words as word}
                                                                <div class="tooltip__word">
                                                                        <span class="tooltip__word-text">{word.text.trim() || "(silence)"}</span>
                                                                        <span class="tooltip__word-meta">{word.nSyllables} syll.</span>
                                                                </div>
                                                        {/each}
                                                </div>
                                        {/if}
                                </div>
                        {/if}
                </div>
        </div>
</section>

<section class="graphic">
        <div class="header">
                <div>
                        <p class="eyebrow">Word shapes</p>
                        <h2>Words grouped by syllable count</h2>
                        <p>Columns stack every word that has 1–6 syllables. Hover to see the word and its syllables.</p>
                </div>
        </div>

        <div class="syllable-grid">
                {#each syllableBuckets as bucket}
                        <div class="syllable-grid__col">
                                <div class="syllable-grid__title">{bucket.count} syllable{bucket.count > 1 ? "s" : ""}</div>
                                <div class="syllable-grid__stack">
                                        {#if bucket.words.length === 0}
                                                <div class="syllable-grid__cell syllable-grid__cell--empty">No words</div>
                                        {:else}
                                                {#each bucket.words as word}
                                                        <div class="syllable-grid__cell" title={`${word.text.trim()} — ${word.nSyllables} syllables`}>
                                                                <span class="syllable-grid__word">{word.text.trim() || "(silence)"}</span>
                                                        </div>
                                                {/each}
                                        {/if}
                                </div>
                        </div>
                {/each}
        </div>
</section>

<style>
        :global(body) {
                font-family:
                        "Atlas Grotesk",
                        system-ui,
                        -apple-system,
                        BlinkMacSystemFont,
                        "Segoe UI",
                        sans-serif;
                color: #0e0e0e;
                background: #f7f7f5;
                margin: 0;
        }

        section {
                padding: 2.5rem clamp(1rem, 4vw, 3rem);
                max-width: 1200px;
                margin: 0 auto;
        }

        h1 {
                margin: 0.15rem 0 0.5rem 0;
                font-weight: 700;
        }

        h2 {
                margin: 0.25rem 0 0.35rem 0;
                font-weight: 700;
        }

        p {
                margin: 0.2rem 0 0;
                line-height: 1.55;
        }

        .eyebrow {
                text-transform: uppercase;
                letter-spacing: 0.08em;
                font-size: 0.82rem;
                color: #5f6368;
                margin: 0;
                font-weight: 700;
        }

        .header {
                display: flex;
                justify-content: space-between;
                align-items: flex-start;
                gap: 1rem;
                flex-wrap: wrap;
        }

        .legend {
                display: grid;
                grid-template-columns: auto 1fr;
                gap: 0.35rem 0.4rem;
                align-items: center;
                background: #ffffff;
                border: 1px solid #e3e3e0;
                border-radius: 8px;
                padding: 0.75rem 0.9rem;
                box-shadow: 0 2px 6px rgba(0, 0, 0, 0.04);
        }

        .legend__swatch {
                display: inline-block;
                width: 18px;
                height: 10px;
                border-radius: 3px;
                background: #4c7ef3;
        }

        .legend__swatch--avg {
                opacity: 0.35;
        }

        .chart {
                position: relative;
                margin-top: 2rem;
                background: #fff;
                border: 1px solid #e3e3e0;
                border-radius: 10px;
                padding: 1.4rem 1.2rem 2.2rem 1.2rem;
                overflow: hidden;
                box-shadow: 0 10px 25px rgba(0, 0, 0, 0.04);
        }

        .chart__y-axis {
                position: absolute;
                left: 0.8rem;
                top: 1.2rem;
                bottom: 1.8rem;
                width: 60px;
        }

        .chart__y-tick {
                position: absolute;
                left: 0;
                width: 100%;
                display: flex;
                align-items: center;
                gap: 0.4rem;
                font-size: 0.9rem;
                color: #4b4d52;
        }

        .chart__y-line {
                flex: 1;
                height: 1px;
                background: #e7e7e2;
        }

        .chart__plot {
                margin-left: 70px;
                position: relative;
                height: 360px;
        }

        .chart__bars {
                position: absolute;
                inset: 0;
                bottom: 0;
                display: grid;
                grid-template-columns: repeat(auto-fit, minmax(12px, 1fr));
                align-items: end;
                gap: 6px;
        }

        .chart__avg {
                position: absolute;
                bottom: 0;
                opacity: 0.28;
                border-radius: 8px;
                display: flex;
                align-items: flex-end;
                justify-content: center;
                color: #0f1010;
                font-weight: 800;
                font-size: 0.85rem;
                padding-bottom: 0.5rem;
        }

        .chart__avg span {
                background: rgba(255, 255, 255, 0.5);
                padding: 0.2rem 0.4rem;
                border-radius: 4px;
                text-transform: uppercase;
                letter-spacing: 0.05em;
        }

        .chart__cell {
                display: flex;
                align-items: flex-end;
        }

        .chart__bar {
                width: 100%;
                min-width: 14px;
                border-radius: 6px 6px 2px 2px;
                position: relative;
                display: block;
                box-shadow: 0 4px 10px rgba(0, 0, 0, 0.08);
                transition: transform 120ms ease, box-shadow 120ms ease;
        }

        .chart__bar:hover {
                transform: translateY(-4px);
                box-shadow: 0 8px 18px rgba(0, 0, 0, 0.12);
        }

        .tooltip {
                position: absolute;
                right: 1rem;
                top: 1rem;
                background: #0f0f10;
                color: #f8f8f6;
                padding: 0.9rem 1rem;
                border-radius: 10px;
                width: min(340px, 90vw);
                box-shadow: 0 10px 25px rgba(0, 0, 0, 0.25);
                z-index: 2;
        }

        .tooltip__title {
                font-weight: 800;
                margin-bottom: 0.45rem;
                letter-spacing: 0.01em;
        }

        .tooltip__row {
                display: flex;
                justify-content: space-between;
                font-size: 0.92rem;
                margin-bottom: 0.25rem;
                gap: 0.4rem;
        }

        .tooltip__row:last-child {
                margin-bottom: 0.5rem;
        }

        .tooltip__words {
                max-height: 180px;
                overflow: auto;
                border-radius: 8px;
                background: rgba(255, 255, 255, 0.04);
                padding: 0.5rem 0.55rem;
                border: 1px solid rgba(255, 255, 255, 0.08);
        }

        .tooltip__word {
                display: flex;
                justify-content: space-between;
                gap: 0.5rem;
                font-size: 0.9rem;
                padding: 0.25rem 0;
                border-bottom: 1px solid rgba(255, 255, 255, 0.05);
        }

        .tooltip__word:last-child {
                border-bottom: none;
        }

        .tooltip__word-text {
                font-weight: 600;
        }

        .tooltip__word-meta {
                color: #c3c3c3;
                font-variant-numeric: tabular-nums;
        }

        .syllable-grid {
                margin-top: 1.5rem;
                display: grid;
                grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
                gap: 1rem;
        }

        .syllable-grid__col {
                background: #fff;
                border: 1px solid #e3e3e0;
                border-radius: 10px;
                padding: 0.9rem;
                box-shadow: 0 8px 20px rgba(0, 0, 0, 0.04);
        }

        .syllable-grid__title {
                font-weight: 700;
                margin-bottom: 0.6rem;
                color: #2f3135;
        }

        .syllable-grid__stack {
                display: flex;
                flex-direction: column;
                gap: 0.4rem;
                max-height: 320px;
                overflow: auto;
        }

        .syllable-grid__cell {
                border: 1px solid #e6e6e2;
                border-radius: 8px;
                padding: 0.45rem 0.55rem;
                background: linear-gradient(135deg, #f8f8f6, #ffffff);
                font-weight: 600;
                color: #1f2227;
                box-shadow: 0 3px 8px rgba(0, 0, 0, 0.05);
        }

        .syllable-grid__cell--empty {
                color: #6b6d70;
                font-weight: 500;
                text-align: center;
        }

        .syllable-grid__word {
                display: block;
                white-space: nowrap;
                overflow: hidden;
                text-overflow: ellipsis;
        }

        @media (max-width: 900px) {
                .chart {
                        padding: 1.1rem 0.8rem 2rem 0.8rem;
                }

                .chart__y-axis {
                        left: 0.5rem;
                        width: 52px;
                }

                .chart__plot {
                        margin-left: 60px;
                        height: 320px;
                }
        }

        @media (max-width: 640px) {
                section {
                        padding: 2rem 0.75rem;
                }

                .chart__bars {
                        gap: 4px;
                }

                .chart__avg span {
                        font-size: 0.78rem;
                }
        }
</style>
