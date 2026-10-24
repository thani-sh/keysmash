<script lang="ts">
	import { toPillImage } from "#lib/images.js";
	import { clearImage, removeWord, setEmoji, setImage, type Pack, type Word } from "#lib/packs.js";

	/**
	 * One word in a pack, as the parent edits it: its letters, its emoji, its
	 * picture, and the controls that change them. Everything it does goes back
	 * through `change` as new packs; it holds no data of its own.
	 */
	let {
		packs,
		packId,
		entry,
		change,
		onError
	}: {
		packs: Pack[];
		packId: string;
		entry: Word;
		change: (next: Pack[]) => void;
		onError: (message: string) => void;
	} = $props();

	function setTheEmoji(event: Event): void {
		change(setEmoji(packs, packId, entry.word, (event.currentTarget as HTMLInputElement).value));
	}

	async function giveItAPicture(event: Event): Promise<void> {
		const chosen = (event.currentTarget as HTMLInputElement).files?.[0];
		if (chosen === undefined) return;
		try {
			change(setImage(packs, packId, entry.word, await toPillImage(chosen)));
		} catch {
			onError("That file is not a picture the app can read.");
		}
	}
</script>

<div class="word">
	<span class="name">{entry.word}</span>
	<input type="text" class="emoji" value={entry.emoji ?? ""} placeholder="emoji" aria-label={`Emoji for ${entry.word}`} onchange={setTheEmoji} />
	{#if entry.image !== undefined}<img class="thumb" src={entry.image} alt="" />{/if}
	<label class="picture">Picture<input type="file" accept="image/*" aria-label={`Picture for ${entry.word}`} onchange={giveItAPicture} /></label>
	{#if entry.image !== undefined}<button type="button" class="unset" onclick={() => change(clearImage(packs, packId, entry.word))}>No picture</button>{/if}
	<button type="button" class="remove" onclick={() => change(removeWord(packs, packId, entry.word))}>Remove</button>
</div>
