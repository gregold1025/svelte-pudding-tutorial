<script>
	import lyricTranscription from "$data/lyric-transcription.json";
	import drumTranscription from "$data/drum-transcription.json";

	const words = lyricTranscription.lines.flatMap((line) => line.words);

	const syllableBuckets = words.reduce((acc, word) => {
		const count = word.nSyllables ?? 0;
		if (!acc[count]) acc[count] = [];
		acc[count].push(word);
		return acc;
	}, {});

	const syllableByWord = Object.entries(syllableBuckets)
		.map(([count, bucketWords]) => ({
			count: Number(count),
			words: bucketWords
		}))
		.sort((a, b) => a.count - b.count);

	const beats = drumTranscription.beats;

	const bars = [];
	for (let i = 0; i + 4 < beats.length; i += 4) {
		const start = beats[i];
		const end = beats[i + 4];
		const wordsInBar = words.filter(
			(word) => word.start >= start && word.start < end
		);
		const syllables = wordsInBar.reduce(
			(sum, word) => sum + (word.nSyllables ?? 0),
			0
		);
		bars.push({
			index: bars.length + 1,
			start,
			end,
			syllables,
			words: wordsInBar
		});
	}

	const maxSyllablesPerBar = Math.max(...bars.map((bar) => bar.syllables), 1);
	const maxWordsInBucket = Math.max(
		...syllableByWord.map((bucket) => bucket.words.length),
		1
	);

	const formatTime = (seconds) => seconds.toFixed(2) + "s";
</script>

<section class="graphic">
	<h1>Syllable density per bar</h1>
	<p>
		Using the beat grid from <code>drum-transcription.json</code>, each bar
		spans four beats. Words that start inside a bar are counted even if they end
		after the bar, while words that start outside but end inside are ignored.
	</p>
	<div class="bars">
		{#each bars as bar}
			<div class="bar">
				<div class="bar__label">
					Bar {bar.index} ({formatTime(bar.start)} – {formatTime(bar.end)})
				</div>
				<div class="bar__track">
					<div
						class="bar__fill"
						style={`width: ${(bar.syllables / maxSyllablesPerBar) * 100}%`}
					>
						<span>{bar.syllables} syllable{bar.syllables === 1 ? "" : "s"}</span
						>
					</div>
				</div>
				{#if bar.words.length}
					<ul class="bar__words">
						{#each bar.words as word}
							<li>{word.text.trim()} ({word.nSyllables})</li>
						{/each}
					</ul>
				{/if}
			</div>
		{/each}
	</div>
</section>

<section class="graphic">
	<h2>Words grouped by syllable count</h2>
	<p>
		Each column stacks the words with the same syllable count. Taller stacks
		show where longer or simpler words appear more often.
	</p>
	<div class="syllable-grid">
		{#each syllableByWord as bucket}
			<div class="syllable-grid__column">
				<div class="syllable-grid__label">
					{bucket.count} syllable{bucket.count === 1 ? "" : "s"}
				</div>
				<div class="syllable-grid__stack">
					{#each bucket.words as word}
						<div class="syllable-grid__cell">
							<span class="syllable-grid__word"
								>{word.text.trim() || "(silence)"}</span
							>
							<span class="syllable-grid__meta">{word.nSyllables} syll.</span>
						</div>
					{/each}
				</div>
				<div class="syllable-grid__count">
					{bucket.words.length} word{bucket.words.length === 1 ? "" : "s"}
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

	graphic {
		display: block;
	}

	section {
		padding: 2rem clamp(1rem, 4vw, 3rem);
		max-width: 960px;
		margin: 0 auto;
	}

	h1,
	h2 {
		margin: 0 0 0.5rem 0;
		font-weight: 700;
	}

	p {
		margin: 0 0 1.5rem 0;
		line-height: 1.5;
	}

	code {
		padding: 0.1rem 0.2rem;
		background: #efefef;
		border-radius: 4px;
		font-size: 0.9rem;
	}

	.bars {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.bar {
		background: #fff;
		border: 1px solid #e3e3e0;
		border-radius: 8px;
		padding: 1rem;
		box-shadow: 0 2px 6px rgba(0, 0, 0, 0.04);
	}

	.bar__label {
		font-weight: 600;
		margin-bottom: 0.5rem;
	}

	.bar__track {
		position: relative;
		height: 32px;
		background: #f0f0ec;
		border-radius: 6px;
		overflow: hidden;
	}

	.bar__fill {
		height: 100%;
		background: linear-gradient(90deg, #8cd2f3, #5fb3e6);
		color: #0b2b3f;
		display: flex;
		align-items: center;
		padding-left: 0.75rem;
		font-weight: 600;
		transition: width 150ms ease;
	}

	.bar__words {
		margin: 0.75rem 0 0 0;
		padding: 0;
		list-style: none;
		display: flex;
		flex-wrap: wrap;
		gap: 0.5rem;
		font-size: 0.95rem;
		color: #3d3d3d;
	}

	.bar__words li {
		background: #f5f5f2;
		border-radius: 4px;
		padding: 0.35rem 0.5rem;
		border: 1px solid #e5e5e0;
	}

	.syllable-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
		gap: 1rem;
	}

	.syllable-grid__column {
		background: #fff;
		border: 1px solid #e3e3e0;
		border-radius: 8px;
		padding: 0.75rem;
		box-shadow: 0 2px 6px rgba(0, 0, 0, 0.04);
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}

	.syllable-grid__label {
		font-weight: 700;
	}

	.syllable-grid__stack {
		border-radius: 6px;
		border: 1px dashed #c7c7c2;
		background: linear-gradient(180deg, #fafaf8, #f3f3f0);
		padding: 0.4rem;
		max-height: 360px;
		overflow: auto;
	}

	.syllable-grid__cell {
		background: #eef6ff;
		border: 1px solid #d6e7ff;
		border-radius: 6px;
		padding: 0.4rem 0.5rem;
		margin-bottom: 0.4rem;
		box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.7);
	}

	.syllable-grid__cell:last-child {
		margin-bottom: 0;
	}

	.syllable-grid__word {
		display: block;
		font-weight: 600;
	}

	.syllable-grid__meta {
		font-size: 0.85rem;
		color: #4f4f4f;
	}

	.syllable-grid__count {
		font-size: 0.9rem;
		color: #4f4f4f;
	}

	@media (max-width: 600px) {
		.bar__words {
			flex-direction: column;
		}
	}
</style>
