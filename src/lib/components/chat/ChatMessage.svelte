<script lang="ts">
	import { marked } from "marked";
	import type { Message } from "$lib/types/Message";
	import { afterUpdate, createEventDispatcher } from "svelte";
	import { deepestChild } from "$lib/utils/deepestChild";
	import { page } from "$app/stores";

	import CodeBlock from "../CodeBlock.svelte";
	import IconLoading from "../icons/IconLoading.svelte";
	import CarbonRotate360 from "~icons/carbon/rotate-360";
	import CarbonDownload from "~icons/carbon/download";
	import { PUBLIC_SEP_TOKEN } from "$lib/constants/publicSepToken";
	import type { Model } from "$lib/types/Model";

	function sanitizeMd(md: string) {
		let ret = md
			.replace(/<\|[a-z]*$/, "")
			.replace(/<\|[a-z]+\|$/, "")
			.replace(/<$/, "")
			.replaceAll(PUBLIC_SEP_TOKEN, " ")
			.replaceAll(/<\|[a-z]+\|>/g, " ")
			.replaceAll(/<br\s?\/?>/gi, "\n")
			.replaceAll("<", "&lt;")
			.trim();

		for (const stop of [...(model.parameters?.stop ?? []), "<|endoftext|>"]) {
			if (ret.endsWith(stop)) {
				ret = ret.slice(0, -stop.length).trim();
			}
		}

		return ret;
	}
	function unsanitizeMd(md: string) {
		return md.replaceAll("&lt;", "<");
	}

	export let model: Model;
	export let message: Message;
	export let loading = false;
	export let readOnly = false;

	const dispatch = createEventDispatcher<{ retry: void }>();

	let contentEl: HTMLElement;
	let loadingEl: IconLoading;
	let pendingTimeout: ReturnType<typeof setTimeout>;

	const renderer = new marked.Renderer();

	// For code blocks with simple backticks
	renderer.codespan = (code) => {
		// Unsanitize double-sanitized code
		return `<code>${code.replaceAll("&amp;", "&")}</code>`;
	};

	const options: marked.MarkedOptions = {
		...marked.getDefaults(),
		gfm: true,
		breaks: true,
		renderer,
	};

	$: tokens = marked.lexer(sanitizeMd(message.content));

	afterUpdate(() => {
		loadingEl?.$destroy();
		clearTimeout(pendingTimeout);

		// Add loading animation to the last message if update takes more than 600ms
		if (loading) {
			pendingTimeout = setTimeout(() => {
				if (contentEl) {
					loadingEl = new IconLoading({
						target: deepestChild(contentEl),
						props: { classNames: "loading inline ml-2" },
					});
				}
			}, 600);
		}
	});

	$: downloadLink =
		message.from === "user" ? `${$page.url.pathname}/message/${message.id}/prompt` : undefined;
</script>

{#if message.from === "assistant"}
	<div class="flex items-start justify-start gap-4 leading-relaxed">
		<img
			alt=""
			src="https://huggingface.co/avatars/2edb18bd0206c16b433841a47f53fa8e.svg"
			class="mt-5 h-3 w-3 flex-none select-none rounded-full shadow-lg"
		/>
		<div
			class="relative min-h-[calc(2rem+theme(spacing[3.5])*2)] min-w-[100px] rounded-2xl border border-gray-100 bg-gradient-to-br from-gray-50 px-5 py-3.5 text-gray-600 prose-pre:my-2 dark:border-gray-800 dark:from-gray-800/40 dark:text-gray-300"
		>
			{#if !message.content}
				<IconLoading classNames="absolute inset-0 m-auto" />
			{/if}
			<div
				class="prose max-w-none dark:prose-invert max-sm:prose-sm prose-headings:font-semibold prose-h1:text-lg prose-h2:text-base prose-h3:text-base prose-pre:bg-gray-800 dark:prose-pre:bg-gray-900"
				bind:this={contentEl}
			>
				{#each tokens as token}
					{#if token.type === "code"}
						<CodeBlock lang={token.lang} code={unsanitizeMd(token.text)} />
					{:else}
						<!-- eslint-disable-next-line svelte/no-at-html-tags -->
						{@html marked(token.raw, options)}
					{/if}
				{/each}
			</div>
		</div>
	</div>
{/if}
{#if message.from === "user"}
	<div class="group relative flex items-start justify-start gap-4 max-sm:text-sm">
		<div class="mt-5 h-3 w-3 flex-none rounded-full" />
		<div class="whitespace-break-spaces rounded-2xl px-5 py-3.5 text-gray-500 dark:text-gray-400">
			{message.content.trim()}
		</div>
		{#if !loading}
			<div class="absolute right-0 top-3.5 flex gap-2 lg:-right-2">
				{#if downloadLink}
					<a
						class="rounded-lg border border-gray-100 p-1 text-xs text-gray-400 group-hover:block hover:text-gray-500 dark:border-gray-800 dark:text-gray-400 dark:hover:text-gray-300 md:hidden"
						title="Download prompt and parameters"
						type="button"
						target="_blank"
						href={downloadLink}
					>
						<CarbonDownload />
					</a>
				{/if}
				{#if !readOnly}
					<button
						class="cursor-pointer rounded-lg border border-gray-100 p-1 text-xs text-gray-400 group-hover:block hover:text-gray-500 dark:border-gray-800 dark:text-gray-400 dark:hover:text-gray-300 md:hidden lg:-right-2"
						title="Retry"
						type="button"
						on:click={() => dispatch("retry")}
					>
						<CarbonRotate360 />
					</button>
				{/if}
			</div>
		{/if}
	</div>
{/if}
