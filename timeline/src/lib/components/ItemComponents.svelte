<script>
	// @ts-nocheck
	import { mobile } from "$lib/stores/window.js";
	import { fade, slide } from "svelte/transition";
	import { writable } from "svelte/store";
	import Text2Speech from "$lib/components/TextToSpeech.svelte";
	import Fullscreen from "svelte-fullscreen";
	import { format } from "date-fns";
	import supabase from "$lib/supabaseClient";
	import { loadingAction } from "svelte-legos";
	import { toast } from "@zerodevx/svelte-toast";
	import { mode } from "$lib/stores/store";

	let loading = true;
	let uploading = false;

	export let item = {
		id: 0,
		title: "",
		image: "",
		image_credit: "",
		body: "",
		start_date: "",
	};

	const full = writable(false);

	export let editList;
	export let addList;

	let formatted_date;

	// makes date readable
	function formatDate(date) {
		if (date.slice(5) == "01-01") {
			return date.slice(0, 4);
		}
		date = new Date(date + "T00:00:00");
		if (date == "Invalid Date") {
			return date;
		}
		return format(date, "MMMM d, yyyy");
	}

	formatted_date = formatDate(item.start_date);

	function autofill(event) {
		const input = event.target.value.replace(/\D/g, "");
		const year = input.slice(0, 4);
		const month = input.slice(4, 6);
		const day = input.slice(6, 8);

		let formattedDate = "";
		if (year) {
			formattedDate += year;
			if (month) {
				formattedDate += "-" + month;
				if (day) {
					formattedDate += "-" + day;
				}
			}
		}

		$mode === "edit"
			? (editList.start_date = formattedDate)
			: (addList.start_date = formattedDate);
	}

	let inputElement;
	function setCursorPositionToEnd() {
		if (inputElement) {
			const inputValueLength = inputElement.value.length;
			inputElement.setSelectionRange(inputValueLength, inputValueLength);
		}
	}

	function handleKeydown(event) {
		if (
			event.key === "ArrowLeft" ||
			event.key === "ArrowUp" ||
			event.key === "ArrowRight" ||
			event.key === "Home" ||
			event.key === "End"
		) {
			event.preventDefault();
			setCursorPositionToEnd();
		}
	}

	async function upload(e) {
		uploading = true;
		const file = e.target.files[0];
		if (file) {
			if (file.size > 4 * 1024 * 1024) {
				toast.push(`<b>Error:</b><br>File size should be less than 4MB`);
				return;
			} else {
				const { data, error } = await supabase.storage
					.from("images")
					.upload(file.name, file);
				if (error) {
					if (error.error === "Duplicate") {
						const imageUrl = supabase.storage
							.from("images")
							.getPublicUrl(file.name);
						if (editing) {
							editList.media = imageUrl.data["publicUrl"];
						} else if (adding) {
							addList.media = imageUrl.data["publicUrl"];
						}
					} else {
						toast.push(
							`<b>Error ${error.satusCode}: ${error.error}</b><br>${error.message}`
						);
					}
				} else {
					const imageUrl = supabase.storage
						.from("images")
						.getPublicUrl(file.name);
					if (editing) {
						editList.media = imageUrl.data["publicUrl"];
					} else if (adding) {
						addList.media = imageUrl.data["publicUrl"];
					}
				}
			}
		}
		uploading = false;
	}
</script>

{#key $mode}
	<section class="item-components">
		{#if item.image || $mode !== "default"}
			<div class="media-component">
				{#if $mode !== "default"}
					<p class={uploading ? "upload-notice red" : "upload-notice"}>
						<span
							style="font-size:var(--font-size-small)"
							class={uploading
								? "material-symbols-rounded i"
								: "material-symbols-rounded"}
							>{uploading ? "autorenew" : "cloud_upload"}</span
						>{uploading ? "Uploading..." : "Upload image"}
					</p>
				{/if}
				<div class="image-cont">
					{#if $mode !== "default"}
						<div class="edit-cont">
							<input
								type="file"
								class="image-edit upload"
								id="file_upload"
								on:change={upload} />
							<img
								class="image-edit"
								src={$mode === "add"
									? $mode === "add"
										? addList.media
										: ""
									: $mode === "edit"
									? editList.media
									: ""}
								alt={$mode === "add"
									? $mode === "add"
										? addList.title
										: ""
									: $mode === "edit"
									? editList.title
									: ""} />
							<div style="width:100%;text-align:center;">
								<p
									style="font-size:var(--font-size-small);align-content:center">
									<i
										>Paste image URL or drag and drop onto image section (4MB
										limit).</i>
								</p>
							</div>

							<div class="input-cont">
								<label for="media">Image URL</label>
								{#if $mode === "edit"}
									<input
										type="text"
										placeholder="https://example.com/image.jpg"
										bind:value={editList.media} />
								{:else if $mode === "add"}
									<input
										type="text"
										placeholder="https://example.com/image.jpg"
										bind:value={addList.media} />
								{/if}
							</div>
							<div class="input-cont">
								<label for="image_credit">Image source</label>
								{#if $mode === "edit"}
									<input
										type="text"
										placeholder="https://example.com"
										bind:value={editList.image_credit} />
								{:else if $mode === "add"}
									<input
										type="text"
										placeholder="https://example.com"
										bind:value={addList.image_credit} />
								{/if}
							</div>
						</div>
					{:else if item.image}
						{#if item.image.includes("youtube.com")}
							<iframe
								class="video"
								title="youtube video"
								src={item.image.replace("watch?v=", "embed/")}
								frameborder="0"
								allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
								allowfullscreen />
						{:else}
							<div class="image-placeholder" use:loadingAction={loading}>
								<Fullscreen let:onToggle let:onExit>
									<img
										on:load={() => (loading = false)}
										class="image"
										src={item.image}
										alt={item.title}
										on:keydown={(e) => {
											if (e.key === "Escape") {
												onExit();
												$full = false;
											}
										}}
										on:click={() => {
											if (!$mobile) {
												onToggle();
												$full = !$full;
											}
										}} />
								</Fullscreen>
							</div>
						{/if}
					{/if}
				</div>
			</div>
		{/if}
		<div class="text-component">
			{#if $mode !== "default"}
				<form>
					<div class="input-cont">
						<label for="title"
							>Title <span style="color:var(--color-theme-1)">*</span></label>
						{#if $mode === "edit"}
							<input
								type="text"
								placeholder="Title"
								bind:value={editList.title} />
						{:else if $mode === "add"}
							<input
								type="text"
								placeholder="Title"
								bind:value={addList.title} />
						{/if}
					</div>
					<div class="input-cont">
						<label for="start_date"
							>Date <span style="color:var(--color-theme-1)">*</span></label>
						{#if $mode === "edit"}
							<input
								type="text"
								placeholder="YYYY-MM-DD"
								bind:value={editList.start_date}
								on:input={autofill}
								bind:this={inputElement}
								on:click={setCursorPositionToEnd}
								on:focus={setCursorPositionToEnd}
								on:keydown={handleKeydown} />
						{:else if $mode === "add"}
							<input
								type="text"
								placeholder="YYYY-MM-DD"
								bind:value={addList.start_date}
								on:input={autofill}
								bind:this={inputElement}
								on:click={setCursorPositionToEnd}
								on:focus={setCursorPositionToEnd}
								on:keydown={handleKeydown} />
						{/if}
					</div>
					<div class="input-cont">
						<label for="body">Description</label>
						{#if $mode === "edit"}
							<textarea placeholder="Description" bind:value={editList.body} />
						{:else if $mode === "add"}
							<textarea placeholder="Description" bind:value={addList.body} />
						{/if}
					</div>
				</form>
			{:else}
				<h1 class="title">{item.title}</h1>
				<p class="date"><i>{formatted_date}</i></p>
				<hr />
				{#if item.body}
					<p class="desc">{item.body}</p>
				{/if}
				<div class="cont-tts">
					<div class="tts">
						<Text2Speech
							title={item.title}
							date={formatted_date}
							body={item.body} />
					</div>
					<div class="image_cred">
						<a href={item.image_credit} target="_blank" rel="noreferrer"
							>Image source</a>
					</div>
				</div>
			{/if}
		</div>
	</section>
	<gap />
{/key}

<style>
	:root {
		--vid-ratio: 0.5625;
	}

	hr {
		margin-top: 2rem;
		width: 66%;
		height: 1px;
		border: none;
		opacity: 0.33;
		background-color: var(--color-text);
	}

	p {
		font-family: var(--font-sans);
		font-size: var(--font-size-base);
		padding: 1em 0 1em 0;
		margin: 0.5rem 0;
	}

	.cont-tts {
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		width: 100%;
	}

	.image_cred {
		display: flex;
		align-items: center;
		font-size: var(--font-size-xsmall);
	}

	.image_cred a {
		color: var(--color-text);
		opacity: 0.5;
		transition: all 0.3s var(--curve);
	}

	.image_cred a:hover {
		opacity: 1;
		color: var(--color-theme-1);
		text-decoration: none;
	}

	.item-components {
		display: flex;
		flex-direction: row;
		flex-wrap: wrap;
		justify-content: center;
		align-items: center;
		margin-bottom: 3rem;
		gap: 1.5rem;
	}

	.text-component {
		flex: 1 2 50%;
		padding: 1rem 2rem;
		border-radius: 1.5rem;
		max-width: 60rem;
		border: var(--border);
		background: var(--color-text-card);
		filter: drop-shadow(5px 5px 7px 0 #00000020);
		text-align: justify;
	}

	.title {
		padding-top: 1rem;
		text-align: center;
	}

	.date {
		text-align: center;
		padding: 0;
		font-size: var(--font-size-medium);
		font-weight: 400;
	}

	.desc {
		text-align: justify;
		padding-top: 1rem;
	}

	.image-cont {
		position: relative;
		display: flex;
		flex-direction: column;
		align-items: center;
		border: var(--border);
		border-radius: 1.5rem;
		max-width: 100%;
	}

	.image-placeholder {
		display: flex;
	}

	.image {
		z-index: 2;
		cursor: pointer;
		min-height: 60vh;
		max-height: 50vh;
		max-width: 100%;
		object-position: center center;
		object-fit: contain;
		border-radius: 1.5rem;
		box-shadow: 5px 5px 7px 0 #00000020;
	}

	@media (max-width: 1000px) {
		.text-component {
			padding: 1rem 1.5rem;
		}
		.title {
			font-size: var(--font-size-large);
		}
		.date {
			font-size: var(--font-size-base);
		}
		.desc {
			font-size: var(--font-size-small);
		}
		.image {
			min-height: 0vh;
			max-height: 40vh;
		}
	}

	.video {
		z-index: 2;
		max-width: 50vw;
		width: 60vw;
		height: calc(60vw * var(--vid-ratio));
		object-position: center center;
		object-fit: cover;
		border-radius: 1.5rem;
		box-shadow: 1rem 0rem 7px 0 #00000030;
	}

	/* ---------------------- EDIT ---------------------- */

	.edit-cont {
		display: flex;
		width: 100%;
		flex: 1 1 49%;
		flex-flow: row wrap;
		align-items: center;
		justify-content: center;
		gap: 1rem;
		margin: 1rem;
		max-width: 60rem;
	}

	.input-cont {
		flex: 1 1 49%;
		display: flex;
		flex-flow: column wrap;
	}

	label {
		font-size: 1em;
		padding-left: 1rem;
		color: var(--color-text);
	}

	input::placeholder,
	textarea::placeholder {
		color: var(--color-text);
		opacity: 0.5;
	}

	input[type="text"],
	textarea {
		text-overflow: ellipsis;
		flex: 1 1 auto;
		text-align: left;
		margin: 0.5em 0;
		padding: 0.9em;
		border-radius: var(--font-size-xsmall) var(--font-size-xsmall) 0 0;
		outline: none;
		border: none;
		background: transparent;
		backdrop-filter: invert(0.1) sepia(0.1) saturate(0.1) brightness(1.1)
			contrast(1.1);
		font-size: var(--font-size-small);
		color: var(--color-text);
		font-family: var(--font-sans);
		box-shadow: inset 0 -4px 0 -1px var(--color-theme-1);
		transition: all 0.3s var(--curve);
	}

	textarea {
		height: clamp(10rem, 20vh, 30rem);
		resize: none;
	}

	input:focus,
	textarea:focus {
		box-shadow: inset 0 -6px 0 -1px var(--color-theme-1);
	}

	.image-edit {
		position: relative;
		width: clamp(5rem, 70vw, 60rem);
		height: clamp(10rem, 30vw, 30rem);
		border: 3px solid var(--color-theme-1);
		z-index: 2;
		object-position: center center;
		object-fit: contain;
		border-radius: var(--font-size-medium);
		background: var(--color-bg-2);
	}

	.upload {
		position: absolute;
		left: 0;
		top: 0;
		right: 0;
		margin: 0 auto;
		bottom: 0;
		opacity: 0;
		background: grey;
		z-index: 3;
	}

	.upload-notice {
		font-size: var(--font-size-small);
		margin-bottom: 0;
		padding-bottom: 0;
		justify-content: center;
		align-items: center;
		display: flex;
		gap: 0.5rem;
	}

	.i {
		animation: spin 1s linear infinite;
	}

	@keyframes spin {
		0% {
			transform: rotate(0deg);
		}
		100% {
			transform: rotate(360deg);
		}
	}

	.red {
		font-weight: 700;
		color: var(--color-theme-1);
	}
</style>
