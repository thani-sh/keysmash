<script lang="ts">
	import { addWord, CUSTOM_PACK_ID, type Pack } from "#lib/packs.js";
	import WordRow from "./WordRow.svelte";

	/**
	 * The settings page: the language the app marks, and the parent's own words
	 * with the emoji or the picture each is drawn with.
	 *
	 * It holds no data. It draws the packs it is handed and gives every change
	 * back through `applyPacks`; storing them and redrawing the child's screen
	 * belong to the caller. The look is the app's own: one large text size,
	 * black on white (`src/app.css`, DESIGN.md).
	 */

	/** How long a Restore waits for its second tap before it stands down. */
	const RESTORE_ARM_MS = 5000;

	let {
		packs,
		language,
		hidden = false,
		onClose,
		onLanguage,
		applyPacks,
		restore
	}: {
		packs: Pack[];
		language: string;
		hidden?: boolean;
		onClose: () => void;
		onLanguage: (id: string) => void;
		applyPacks: (next: Pack[]) => Promise<string | null>;
		restore: () => Promise<void>;
	} = $props();

	/** The languages to choose between: every pack but the parent's own. */
	const languages = $derived(packs.filter((pack) => pack.id !== CUSTOM_PACK_ID));
	/** The parent's own words, marked whatever language is chosen. */
	const mine = $derived(packs.find((pack) => pack.id === CUSTOM_PACK_ID));
	/** What the parent has typed into the add field, so a refusal keeps it there. */
	let draft = $state("");
	/** Why an add was refused, or a picture refused. */
	let error = $state<string | null>(null);
	/** Whether a Restore is waiting for its second tap. */
	let armed = $state(false);
	/** Why the last Restore failed, when it did. */
	let restoreError = $state<string | null>(null);
	let armTimer: ReturnType<typeof setTimeout> | undefined;

	/** Applies a change to the packs; a device that will not keep it is told here. */
	async function change(next: Pack[]): Promise<void> {
		const refused = await applyPacks(next);
		if (refused !== null) error = refused;
	}

	/** Adds the parent's word to My words, or says why it will not go there. */
	async function add(typed: string): Promise<void> {
		const result = addWord(packs, CUSTOM_PACK_ID, typed);
		if (result.error !== null) {
			error = result.error;
			return;
		}
		error = null;
		draft = "";
		await change(result.packs);
	}

	/** Puts the built-in words back, on the second tap of a two-tap Restore. */
	async function onRestore(): Promise<void> {
		if (!armed) {
			armed = true;
			clearTimeout(armTimer);
			armTimer = setTimeout(() => {
				armed = false;
			}, RESTORE_ARM_MS);
			return;
		}
		armed = false;
		clearTimeout(armTimer);
		try {
			await restore();
			restoreError = null;
		} catch {
			restoreError = "The app could not put the built-in words back.";
		}
	}
</script>

<section id="settings" {hidden}>
	<header>
		<h1>Settings</h1>
		<button type="button" onclick={onClose}>Close</button>
	</header>

	<section class="section language">
		<h2>Language</h2>
		{#each languages as pack (pack.id)}
			<label class="choice">
				<input
					type="radio"
					name="language"
					value={pack.id}
					checked={language === pack.id}
					onchange={() => onLanguage(pack.id)}
				/>
				<span class="name">{pack.name}</span>
				<span class="count">({pack.words.length})</span>
			</label>
		{/each}
	</section>

	{#if mine !== undefined}
		<section class="section">
			<h2>My words <span class="count">({mine.words.length})</span></h2>
			<div class="words">
				{#each mine.words as entry (entry.word)}
					<WordRow
						{packs}
						packId={CUSTOM_PACK_ID}
						{entry}
						change={(next) => void change(next)}
						onError={(message) => (error = message)}
					/>
				{/each}
				<div class="add">
					<input
						type="text"
						class="new"
						placeholder="Add a word"
						aria-label="Add a word to My words"
						value={draft}
						oninput={(event) => (draft = event.currentTarget.value)}
						onkeydown={(event) => {
							if (event.key !== "Enter") return;
							event.preventDefault();
							void add(draft);
						}}
					/>
					<button type="button" class="submit" onclick={() => void add(draft)}>Add</button>
				</div>
				{#if error !== null}
					<p class="error">{error}</p>
				{/if}
			</div>
		</section>
	{/if}

	<div class="restore">
		<button type="button" class="restore-button" onclick={onRestore}>
			{armed ? "Tap again to restore" : "Restore built-in words"}
		</button>
		<p class="note">
			{restoreError ?? "Restoring puts the built-in words back and takes your words, emojis and pictures off this device."}
		</p>
	</div>

	<p class="hint">Hold the left and right Shift keys for three seconds to open this page while the app is full screen.</p>
</section>
