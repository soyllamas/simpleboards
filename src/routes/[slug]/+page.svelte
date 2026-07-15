<script lang="ts">
	import { onDestroy, onMount } from "svelte";
	import { cubicIn, cubicOut } from "svelte/easing";
	import { prefersReducedMotion } from "svelte/motion";
	import { fly, scale } from "svelte/transition";
	import type { Task } from "$lib/domain/entity/task";
	import Markdoc from "@markdoc/markdoc";
	import { browser } from "$app/environment";
	import { goto } from "$app/navigation";
	import { resolve } from "$app/paths";
	import equal from "fast-deep-equal";
	import { createTask, updateTask, deleteTask, reorderTasks, updateExpiration } from "./board.remote";
	import EmojiPicker from "$lib/components/EmojiPicker.svelte";
	import { detectExactMatch, resolveNodeOffset, type TriggerState } from "$lib/domain/useCase/emoji";
	import { getTaskUpdatedAtTime, isTaskVisibleToday } from "$lib/domain/useCase/taskVisibility";
	import {
		boardTaskMaxCount,
		normalizeTaskTitle,
		taskTitleMaxLength
	} from "$lib/domain/useCase/boardLimits";
	import ChevronDown from "@lucide/svelte/icons/chevron-down";
	import Menu from "@lucide/svelte/icons/menu";
	import Plus from "@lucide/svelte/icons/plus";
	import Settings2 from "@lucide/svelte/icons/settings-2";
	import X from "@lucide/svelte/icons/x";

	let unsubscribe: any;
	let refreshToday: ReturnType<typeof setInterval>;

	type Column = {
		id: Task["status"];
		name: string;
		tasks: Task[];
		instance: HTMLDivElement;
	};

	type ExpirationValue = "30" | "90" | "never";

	type ExpirationOption = {
		value: ExpirationValue;
		name: string;
	};

	type ThemeOption = {
		value: "system" | "light" | "dark";
		name: string;
	};

	const expirationOptions: ExpirationOption[] = [
		{
			value: "30",
			name: "30 days of inactivity",
		},
		{
			value: "90",
			name: "90 days of inactivity",
		},
		{
			value: "never",
			name: "Do not delete",
		},
	];

	const themeOptions: ThemeOption[] = [
		{
			value: "system",
			name: "System",
		},
		{
			value: "light",
			name: "Light",
		},
		{
			value: "dark",
			name: "Dark",
		},
	];
	let { data } = $props();
	let menuItems = $state<string[]>([]);
	let currentBoardId = $state<string>();
	let selectedExpiration = $state<ExpirationValue | undefined>();
	let hasMounted = $state(false);
	let expirationSetting = $derived(selectedExpiration ?? parseExpiration(data.expiration));

	onMount(async () => {
		hasMounted = true;
		nextTaskId = await _generateId();

		// Request focus if tasks are empty
		addTaskInput?.focus();

		// Retrieve recent boards
		let stored = localStorage.getItem("recent");
		if (stored) {
			const recentBoards = JSON.parse(stored) ?? [];
			const isNewBoard = !recentBoards.includes(data.boardId);
			if (isNewBoard) {
				recentBoards.unshift(data.boardId);
			}
			menuItems = recentBoards;
		} else {
			menuItems = [data.boardId];
		}
		stored = JSON.stringify($state.snapshot(menuItems));
		localStorage.setItem("recent", stored);

		refreshToday = setInterval(() => {
			today = new Date();
		}, 60_000);
	});

	onDestroy(() => {
		unsubscribe?.();
		clearInterval(refreshToday);
	});

	// svelte-ignore state_referenced_locally
	let tasks = $state(data.tasks ?? []);
	let today = $state(new Date());
	let filteredTasks = $derived(tasks.filter((task) => isTaskVisibleToday(task, today)));

	let nextTaskId = "";

	$effect(() => {
		if (currentBoardId !== data.boardId) {
			currentBoardId = data.boardId;
			selectedExpiration = undefined;
		}

		tasks = data.tasks ?? [];
		listenToChanges(data.boardId);
	});

	let visibleTasks = $derived(hasMounted ? filteredTasks : data.visibleTasks);
	let todo = $derived(visibleTasks.filter((task) => task.status === "todo"));
	let doing = $derived(visibleTasks.filter((task) => task.status === "doing"));
	let done = $derived(visibleTasks.filter((task) => task.status === "done"));

	let addTask = $state(false);
	let addTaskInput = $state<HTMLDivElement>();
	let emojiPicker = $state<ReturnType<typeof EmojiPicker>>();
	let activeEditableElement = $state<HTMLElement | undefined>();
	let menuOpen = $state(false);
	let settingsOpen = $state(false);

	const columns = $derived([
		{
			id: "todo",
			name: "To-do",
			tasks: todo,
		} as Column,
		{
			id: "doing",
			name: "Doing",
			tasks: doing,
		} as Column,
		{
			id: "done",
			name: "Done",
			tasks: done,
		} as Column,
	]);

	async function listenToChanges(boardId: string) {
		if (!browser) return;

		const { db } = await import("$lib/client/firebase");
		const { doc, onSnapshot } = await import("firebase/firestore");

		unsubscribe?.();
		unsubscribe = onSnapshot(doc(db, "boards", boardId), (snapshot) => {
			if (!snapshot.exists()) return;

			const remoteTasks = snapshot.get("tasks") ?? [];
			const remoteExpiration = parseExpiration(snapshot.get("expiration"));
			const taskSnapshot = $state.snapshot(tasks) as Task[];
			const localTasks = taskSnapshot.map(toComparableTask);
			const comparableRemoteTasks = remoteTasks.map(toComparableTask);

			if (expirationSetting !== remoteExpiration) {
				selectedExpiration = remoteExpiration;
			}

			if (!equal(localTasks, comparableRemoteTasks)) {
				tasks = remoteTasks;
			}
		});
	}

	function toComparableTask(task: Task) {
		return {
			id: task.id,
			status: task.status,
			title: task.title,
			updatedAt: getTaskUpdatedAtTime(task)
		};
	}

	function onDrag(event: DragEvent, task: Task) {
		event.dataTransfer?.setData("text/plain", task.id);
	}

	async function onDrop(event: DragEvent, status: Task["status"]) {
		event.preventDefault();

		const taskId = event.dataTransfer?.getData("text/plain");
		const task = tasks.find((task) => task.id === taskId);
		if (!task) return;

		let underTask = null;
		let topZone = Number.MAX_VALUE;
		let bottomZone = 0;

		for (let t of visibleTasks.filter((task) => task.status == status)) {
			const { top, height } = t.instance.getBoundingClientRect();
			const topBound = top - 6;
			const bottomBound = top + height + 6;

			if (topBound < event.clientY && bottomBound > event.clientY) {
				underTask = t;
			}

			if (topBound < topZone) topZone = top;
			if (bottomBound > bottomZone) bottomZone = bottomBound;
		}

		if (underTask == undefined) {
			if (topZone > event.clientY) underTask = tasks[0];
			if (bottomZone < event.clientY) underTask = tasks[tasks.length - 1];
		}

		const underIndex = tasks.indexOf(underTask!);
		const _tasks = tasks.filter((task) => task.id !== taskId);

		const snapshot = $state.snapshot(tasks) as any;
		const oldStatus = task.status;

		task.status = status;
		if (oldStatus !== status) task.updatedAt = new Date().toISOString();
		tasks = _tasks.slice(0, underIndex).concat(task, _tasks.slice(underIndex));

		try {
			const normalizedTasks: Array<{ id: string; status: "todo" | "doing" | "done"; title: string }> = $state
				.snapshot(tasks)
				.map(({ id, status, title }: any) => ({
					id,
					status,
					title,
				}));
			await reorderTasks({
				boardId: data.boardId,
				tasks: normalizedTasks,
			});
		} catch (error) {
			task.status = oldStatus;
			tasks = snapshot as any;
			console.error("Failed to reorder tasks:", error);
		}
	}

	async function onKeyDownUpdateTask(event: KeyboardEvent, task: Task) {
		if (emojiPicker?.handleKeyDown(event)) return;

		const isEnter = event.key === "Enter" && !event.shiftKey;
		const isEscape = event.key === "Escape";

		if (isEscape) {
			task.instance.blur();
		}

		if (isEnter) {
			await _updateTask(task);
		}
	}

	async function _updateTask(task: Task) {
		task.instance.blur();

		const snapshot = $state.snapshot(tasks) as any;
		const oldTitle = task.title;
		const oldUpdatedAt = task.updatedAt;

		if (_isEmpty(task.title)) {
			tasks = tasks.filter((item) => item.id !== task.id);

			try {
				await deleteTask({
					boardId: data.boardId,
					taskId: task.id,
				});
			} catch (error) {
				tasks = snapshot as any;
				console.error("Failed to delete task:", error);
			}
		} else {
			task.title = normalizeTaskTitle(task.title);
			task.updatedAt = new Date().toISOString();
			tasks = tasks.map((t) =>
				t.id === task.id ? { ...t, title: normalizeTaskTitle(t.title), updatedAt: task.updatedAt } : t
			) as any;

			try {
				await updateTask({
					boardId: data.boardId,
					task: {
						id: task.id,
						status: task.status as "todo" | "doing" | "done",
						title: task.title,
					},
				});
			} catch (error) {
				task.title = oldTitle;
				task.updatedAt = oldUpdatedAt;
				tasks = snapshot as any;
				console.error("Failed to update task:", error);
			}
		}
	}

	function onKeyDownCreateTask(event: KeyboardEvent) {
		if (emojiPicker?.handleKeyDown(event)) return;

		const isEnter = event.key === "Enter" && !event.shiftKey;
		const isEscape = event.key === "Escape";
		const title = addTaskInput!.innerText;

		if (isEscape) {
			addTaskInput!.innerText = "";
			addTask = false;
		}

		if ((isEnter && _isEmpty(title)) || (isEnter && tasks.length === 0)) {
			event.preventDefault();
		}

		if (isEnter) {
			_createTask(title);
			addTaskInput!.innerText = "";
			addTask = false;
		}
	}

	async function _createTask(title: string) {
		const taskTitle = normalizeTaskTitle(title);
		if (_isEmpty(taskTitle) || tasks.length >= boardTaskMaxCount) return;

		const taskId = _getId();
		const task = {
			id: taskId,
			status: "todo",
			title: taskTitle,
			updatedAt: new Date().toISOString(),
			editable: false,
		} as Task;

		const snapshot = $state.snapshot(tasks) as any;
		tasks.unshift(task);

		try {
			await createTask({
				boardId: data.boardId,
				task: {
					id: task.id,
					status: task.status as "todo" | "doing" | "done",
					title: task.title,
				},
			});
		} catch (error) {
			tasks = snapshot as any;
			console.error("Failed to create task:", error);
		}
	}

	async function onExpirationChange(event: Event) {
		const oldExpiration = expirationSetting;
		const expiration = (event.currentTarget as HTMLSelectElement).value as ExpirationValue;

		selectedExpiration = expiration;

		try {
			await updateExpiration({
				boardId: data.boardId,
				expiration,
			});
		} catch (error) {
			selectedExpiration = oldExpiration;
			console.error("Failed to update expiration:", error);
		}
	}

	function onCreateTaskPressed() {
		addTask = true;
		setTimeout(() => {
			addTaskInput?.focus();
			_setCursorAtEnd();
		});
	}

	async function _generateId() {
		if (!browser) return "";

		const { db } = await import("$lib/client/firebase");
		const { collection, doc } = await import("firebase/firestore");
		const boardsRef = collection(db, `boards`);

		return doc(boardsRef).id;
	}

	function _getId() {
		_generateId().then((id) => (nextTaskId = id));
		return nextTaskId;
	}

	function _setCursorAtEnd(target: HTMLElement = addTaskInput!) {
		const selection = window.getSelection();
		const range = document.createRange();
		range.selectNodeContents(target);
		range.collapse(false);
		selection!.removeAllRanges();
		selection!.addRange(range);
	}

	function toHtml(source: string) {
		const ast = Markdoc.parse(source);
		const content = Markdoc.transform(ast);
		return Markdoc.renderers.html(content);
	}

	function removeRecentBoard(id: string) {
		let stored = localStorage.getItem("recent") ?? "";
		let boardIds = (JSON.parse(stored) ?? []) as string[];
		boardIds = boardIds.filter((boardId) => boardId != id);
		menuItems = boardIds;
		if (id === data.boardId && menuItems.length > 0) goto(resolve(`/${menuItems[0]}`));
		stored = JSON.stringify(boardIds);
		localStorage.setItem("recent", stored);
	}

	function copyToClipboard() {
		navigator.clipboard.writeText(data.link);
	}

	function toggleMenu() {
		menuOpen = !menuOpen;
		settingsOpen = false;
	}

	function toggleSettings() {
		settingsOpen = !settingsOpen;
		menuOpen = false;
	}

	function closePanels() {
		menuOpen = false;
		settingsOpen = false;
	}

	function closePanelsOnOutsideClick(event: MouseEvent) {
		if (!menuOpen && !settingsOpen) return;
		if (!(event.target instanceof Element)) return;
		if (event.target.closest("[data-panel], [data-panel-trigger]")) return;

		closePanels();
	}

	function closePanelsOnPageScroll() {
		if (!menuOpen && !settingsOpen) return;
		if (isDesktopViewport()) return;

		closePanels();
	}

	function openMenuOnDesktop() {
		if (!isDesktopViewport()) return;
		menuOpen = true;
		settingsOpen = false;
	}

	function openSettingsOnDesktop() {
		if (!isDesktopViewport()) return;
		settingsOpen = true;
		menuOpen = false;
	}

	function closeMenuOnDesktop() {
		if (isDesktopViewport()) menuOpen = false;
	}

	function closeSettingsOnDesktop() {
		if (isDesktopViewport()) settingsOpen = false;
	}

	function isDesktopViewport() {
		return browser && window.matchMedia("(min-width: 1024px)").matches;
	}

	function onTaskClicked(event: MouseEvent, task: Task) {
		// @ts-ignore
		const isATag = event.target?.tagName.toLowerCase() === "a";
		if (isATag) {
			// @ts-ignore
			event.target.setAttribute("target", "_blank");
			event.stopPropagation();
		} else {
			task.editable = true;
			setTimeout(() => {
				task.instance.focus();
				let selection = window.getSelection();
				selection?.selectAllChildren(task.instance);
				selection?.collapseToEnd();
			});
		}
	}

	function insertEmoji(target: HTMLElement, emoji: string, triggerOffset: number, cursorOffset: number) {
		const text = target.innerText;
		const before = text.slice(0, triggerOffset);
		const after = text.slice(cursorOffset);
		target.innerText = before + emoji + after;

		// Position cursor after the inserted emoji
		const newCursorPos = triggerOffset + emoji.length;
		const resolved = resolveNodeOffset(target, newCursorPos);
		if (resolved) {
			const selection = window.getSelection();
			const range = document.createRange();
			range.setStart(resolved.node, resolved.offset);
			range.collapse(true);
			selection!.removeAllRanges();
			selection!.addRange(range);
		}

		// Sync Svelte's bind:innerText
		target.dispatchEvent(new Event("input", { bubbles: true }));
	}

	function onEmojiSelect(emoji: string, triggerState: TriggerState) {
		if (activeEditableElement) {
			insertEmoji(activeEditableElement, emoji, triggerState.triggerOffset, triggerState.cursorOffset);
		}
	}

	function onEditableInput(event: Event) {
		const target = event.currentTarget as HTMLElement;
		if (target.innerText.length > taskTitleMaxLength) {
			target.innerText = target.innerText.slice(0, taskTitleMaxLength);
			_setCursorAtEnd(target);
		}

		const match = detectExactMatch(target);
		if (match) {
			insertEmoji(target, match.emoji, match.triggerOffset, match.cursorOffset);
		}
	}

	function onPaste(event: ClipboardEvent) {
		const url = event.clipboardData?.getData("text/plain") ?? "";
		if (!isUrl(url)) return;

		const selection = window.getSelection();
		const selectedText = selection?.toString() ?? "";
		if (!selectedText) return;

		event.preventDefault();
		const anchor = `[${selectedText}](${url})`;
		document.execCommand("insertText", false, anchor);
	}

	function isUrl(text: string) {
		try {
			const url = new URL(text);
			return url.protocol === "http:" || url.protocol === "https:";
		} catch {
			return false;
		}
	}

	function _isEmpty(value: string) {
		return value.trim().length === 0;
	}

	function parseExpiration(value: unknown): ExpirationValue {
		if (value === "30" || value === "90" || value === "never") return value;
		return "30";
	}

</script>

<script:head>
	<title>#{data.boardId}</title>
</script:head>

<svelte:window onclick={closePanelsOnOutsideClick} onscroll={closePanelsOnPageScroll} />

{#snippet menuButton(classes: string)}
	<button
		type="button"
		class={`relative z-40 grid size-10 shrink-0 items-center justify-center rounded-xl bg-white text-slate-950 ring-1 ring-slate-950/10 transition-[background-color,box-shadow,scale] duration-200 active:scale-[0.96] hover:bg-slate-50 focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600 dark:bg-slate-900 dark:text-slate-100 dark:ring-white/10 dark:hover:bg-slate-800 ${classes}`}
		aria-label="Open boards menu"
		aria-expanded={menuOpen}
		data-panel-trigger
		onclick={toggleMenu}
		onmouseenter={openMenuOnDesktop}
		onfocus={openMenuOnDesktop}
	>
		<span
			class="absolute top-1/2 left-1/2 size-[max(100%,3rem)] -translate-1/2 pointer-fine:hidden"
			aria-hidden="true"
		></span>
		<Menu class="size-5" aria-hidden="true" />
	</button>
{/snippet}

{#snippet settingsButton(classes: string)}
	<button
		type="button"
		class={`relative z-40 grid size-10 shrink-0 items-center justify-center rounded-xl bg-white text-slate-950 ring-1 ring-slate-950/10 transition-[background-color,box-shadow,scale] duration-200 active:scale-[0.96] hover:bg-slate-50 focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600 dark:bg-slate-900 dark:text-slate-100 dark:ring-white/10 dark:hover:bg-slate-800 ${classes}`}
		aria-label="Board settings"
		aria-expanded={settingsOpen}
		data-panel-trigger
		onclick={toggleSettings}
		onmouseenter={openSettingsOnDesktop}
		onfocus={openSettingsOnDesktop}
	>
		<span
			class="absolute top-1/2 left-1/2 size-[max(100%,3rem)] -translate-1/2 pointer-fine:hidden"
			aria-hidden="true"
		></span>
		<Settings2 class="size-5" aria-hidden="true" />
	</button>
{/snippet}

{#snippet boardTitle(classes: string)}
	<h1
		class={`min-w-0 cursor-pointer truncate text-2xl font-semibold tracking-tight text-balance text-slate-950 hover:underline dark:text-slate-50 ${classes}`}
		onclick={() => copyToClipboard()}
		role="none"
	>
		#{data.boardId}
	</h1>
{/snippet}

{#snippet menuPanelContent(isMobile: boolean)}
	<a
		class={[
			"mb-6 block rounded-2xl transition-colors duration-200 hover:bg-slate-100 focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600 dark:hover:bg-slate-800",
			isMobile ? "px-3 py-3" : "px-[10px] py-2"
		]}
		href={resolve("/")}
	>
		<h3
			class={[
				"text-slate-950 dark:text-slate-50",
				isMobile ? "text-2xl font-semibold tracking-tight" : "text-xl font-semibold"
			]}
		>
			SimpleBoard
		</h3>
		<p
			class={[
				"mt-0.5 font-normal text-slate-500 dark:text-slate-400",
				isMobile ? "text-sm/5" : "text-[11px]"
			]}
		>
			Kanban for minimalists
		</p>
	</a>
	<p
		class={[
			"mb-2 font-medium text-slate-500 dark:text-slate-400",
			isMobile ? "px-3 text-sm/5" : "px-[10px] text-[12px]"
		]}
	>
		Recent
	</p>
	{#each menuItems as menuItem (menuItem)}
		<div
			class={[
				"group/item mb-1 flex items-center text-slate-700 transition-colors duration-200 dark:text-slate-200",
				isMobile ? "gap-2 rounded-2xl px-3 py-2 text-base/7" : "rounded-xl px-[10px] py-[6px] text-sm",
				menuItem === data.boardId && "bg-slate-100 dark:bg-slate-800"
			]}
		>
			<a class="flex min-h-10 w-full items-center align-middle group-hover/item:underline" href={resolve(`/${menuItem}`)}>
				#{menuItem}
			</a>
			<button
				type="button"
				class={[
					"cursor-pointer text-slate-400 transition-[background-color,color,scale] duration-200 active:scale-[0.96] focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600 dark:text-slate-500 dark:hover:text-slate-300",
					isMobile
						? "relative grid size-10 shrink-0 place-items-center rounded-xl hover:bg-slate-100 dark:hover:bg-slate-800"
						: "hidden size-10 shrink-0 place-items-center rounded-xl hover:bg-slate-100 group-hover/item:grid dark:hover:bg-slate-800"
				]}
				aria-label={`Remove ${menuItem} from recent boards`}
				onclick={() => removeRecentBoard(menuItem)}
			>
				<X class="size-5" aria-hidden="true" />
			</button>
		</div>
	{/each}
{/snippet}

{#snippet settingsPanelContent(expirationId: string, themeId: string)}
	<h2 class="text-lg/7 font-semibold text-slate-950 sm:text-base/6 dark:text-slate-50">Settings</h2>
	<div class="mt-5">
		<label for={expirationId} class="text-base/6 font-medium text-slate-950 sm:text-sm/6 dark:text-slate-100">
			Auto-delete after
		</label>
		<div class="mt-2 inline-grid w-full grid-cols-[1fr_--spacing(8)]">
			<select
				id={expirationId}
				name="expiration"
				value={expirationSetting}
				onchange={(event) => onExpirationChange(event)}
				class="col-span-full row-start-1 min-h-10 appearance-none rounded-xl bg-white py-2 pr-8 pl-3 text-base/7 text-slate-950 ring-1 ring-slate-950/10 transition-shadow duration-200 focus-visible:outline-2 focus-visible:-outline-offset-1 focus-visible:outline-blue-600 sm:text-sm/6 dark:bg-slate-900 dark:text-slate-100 dark:ring-white/10"
			>
				{#each expirationOptions as option (option.value)}
					<option value={option.value}>{option.name}</option>
				{/each}
			</select>
			<ChevronDown class="pointer-events-none col-start-2 row-start-1 size-4 place-self-center text-slate-700 dark:text-slate-300" aria-hidden="true" />
		</div>
	</div>
	<div class="mt-5">
		<label for={themeId} class="text-base/6 font-medium text-slate-950 sm:text-sm/6 dark:text-slate-100">
			Theme
		</label>
		<div class="mt-2 inline-grid w-full grid-cols-[1fr_--spacing(8)]">
			<select
				id={themeId}
				name="theme"
				value="system"
				disabled
				class="col-span-full row-start-1 min-h-10 appearance-none rounded-xl bg-slate-50 py-2 pr-8 pl-3 text-base/7 text-slate-500 ring-1 ring-slate-950/10 sm:text-sm/6 dark:bg-slate-800 dark:text-slate-400 dark:ring-white/10"
			>
				{#each themeOptions as option (option.value)}
					<option value={option.value}>{option.name}</option>
				{/each}
			</select>
			<ChevronDown class="pointer-events-none col-start-2 row-start-1 size-4 place-self-center text-slate-700 dark:text-slate-300" aria-hidden="true" />
		</div>
	</div>
{/snippet}

<header
	class="mx-auto flex max-w-[960px] items-center gap-3 px-4 pt-4 sm:pt-6 lg:block lg:max-w-[min(960px,calc(100%-8rem))] lg:px-0 lg:pt-0"
>
	{@render menuButton("lg:fixed lg:top-6 lg:left-6")}
	{@render boardTitle("grow text-center lg:my-8 lg:block lg:text-left")}
	{@render settingsButton("lg:fixed lg:top-6 lg:right-6")}
</header>

{#if menuOpen}
		<div
			data-panel
			class="fixed inset-x-3 bottom-[calc(--spacing(3)+env(safe-area-inset-bottom))] z-50 lg:hidden"
			in:fly={{
				x: prefersReducedMotion.current ? 0 : "-25vw",
				opacity: 0,
				duration: prefersReducedMotion.current ? 120 : 200,
				easing: cubicOut
			}}
			out:fly={{
				x: prefersReducedMotion.current ? 0 : "-50vw",
				opacity: 0,
				duration: prefersReducedMotion.current ? 120 : 300,
				easing: cubicIn
			}}
		>
			<div
				class="origin-bottom-left"
				in:scale={{
					start: prefersReducedMotion.current ? 1 : 0.8,
					opacity: 1,
					duration: prefersReducedMotion.current ? 0 : 200,
					easing: cubicOut
				}}
				out:scale={{
					start: prefersReducedMotion.current ? 1 : 0.8,
					opacity: 1,
					duration: prefersReducedMotion.current ? 0 : 300,
					easing: cubicIn
				}}
			>
				<aside
					class="min-h-[45dvh] max-h-[85dvh] overflow-y-auto rounded-[1.75rem] bg-white px-4 pt-5 pb-6 shadow-2xl ring-1 ring-slate-950/10 dark:bg-slate-900 dark:shadow-none dark:ring-white/10"
				>
					{@render menuPanelContent(true)}
				</aside>
			</div>
		</div>
{/if}

<aside
	data-panel
	class={`fixed inset-y-0 left-0 z-50 hidden h-dvh w-64 overflow-y-auto bg-white px-[14px] pt-4 pb-4 shadow-[0_8px_30px_rgba(15,23,42,0.08)] ring-1 ring-slate-950/10 transition-transform duration-300 lg:block dark:bg-slate-900 dark:shadow-none dark:ring-white/10 ${menuOpen ? "translate-x-0" : "-translate-x-full"}`}
	onmouseleave={closeMenuOnDesktop}
>
	{@render menuPanelContent(false)}
</aside>

{#if settingsOpen}
		<div
			data-panel
			class="fixed inset-x-3 bottom-[calc(--spacing(3)+env(safe-area-inset-bottom))] z-50 lg:hidden"
			in:fly={{
				x: prefersReducedMotion.current ? 0 : "25vw",
				opacity: 0,
				duration: prefersReducedMotion.current ? 120 : 200,
				easing: cubicOut
			}}
			out:fly={{
				x: prefersReducedMotion.current ? 0 : "50vw",
				opacity: 0,
				duration: prefersReducedMotion.current ? 120 : 300,
				easing: cubicIn
			}}
		>
			<div
				class="origin-bottom-right"
				in:scale={{
					start: prefersReducedMotion.current ? 1 : 0.8,
					opacity: 1,
					duration: prefersReducedMotion.current ? 0 : 200,
					easing: cubicOut
				}}
				out:scale={{
					start: prefersReducedMotion.current ? 1 : 0.8,
					opacity: 1,
					duration: prefersReducedMotion.current ? 0 : 300,
					easing: cubicIn
				}}
			>
				<aside
					class="min-h-[45dvh] max-h-[85dvh] overflow-y-auto rounded-[1.75rem] bg-white px-4 pt-5 pb-6 shadow-2xl ring-1 ring-slate-950/10 dark:bg-slate-900 dark:shadow-none dark:ring-white/10"
				>
					{@render settingsPanelContent("mobile-expiration", "mobile-theme")}
				</aside>
			</div>
		</div>
{/if}

<aside
	data-panel
	class={`fixed inset-y-0 right-0 z-50 hidden h-dvh w-72 overflow-y-auto bg-white px-4 pt-4 pb-4 shadow-[0_8px_30px_rgba(15,23,42,0.08)] ring-1 ring-slate-950/10 transition-transform duration-300 lg:block dark:bg-slate-900 dark:shadow-none dark:ring-white/10 ${settingsOpen ? "translate-x-0" : "translate-x-full"}`}
	onmouseleave={closeSettingsOnDesktop}
>
	{@render settingsPanelContent("desktop-expiration", "desktop-theme")}
</aside>

<div class="mx-auto max-w-[960px] px-4 pb-[var(--mobile-browser-bottom-space)] lg:pb-8">
	<div class="mt-8 grid grid-cols-1 gap-6 md:grid-cols-3 lg:mt-0">
		{#each columns as column (column.id)}
			<div
				class="md:flex md:min-h-[80vh] md:flex-col"
				ondrop={(event) => onDrop(event, column.id)}
				ondragover={(e) => e.preventDefault()}
				role="none"
			>
				<div class="flex">
					<p class="grow rounded-t-lg font-semibold text-slate-950 dark:text-slate-50">{column.name}</p>
					{#if column.id === "todo" && tasks.length > 0 && tasks.length < boardTaskMaxCount}
						<button
							type="button"
							class="grid size-6 place-items-center rounded-lg text-slate-950 transition-[background-color,scale] duration-200 active:scale-[0.96] hover:bg-slate-50 focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600 dark:text-slate-100 dark:hover:bg-slate-800"
							aria-label="Add task"
							onclick={() => onCreateTaskPressed()}
						>
							<Plus class="size-5" aria-hidden="true" />
						</button>
					{/if}
				</div>
				<div>
					{#if column.id === "todo"}
						<div
							contenteditable="plaintext-only"
							class="selected my-3 box-border min-h-4 cursor-default rounded-2xl bg-white p-4 whitespace-pre-line text-pretty text-slate-700 outline-none dark:bg-slate-900 dark:text-slate-100"
							class:hidden={!addTask && tasks.length !== 0}
							onfocusin={() => { addTask = true; activeEditableElement = addTaskInput; }}
							onfocusout={() => { addTask = false; activeEditableElement = undefined; }}
							onkeydown={(event) => onKeyDownCreateTask(event)}
							oninput={(event) => onEditableInput(event)}
							onpaste={(event) => onPaste(event)}
							bind:this={addTaskInput}
							role="none"
						></div>
					{/if}
					{#each column.tasks as task (task.id)}
						{#if task.editable}
							<div
								contenteditable="plaintext-only"
								onkeydown={(event) => onKeyDownUpdateTask(event, task)}
								onfocusin={() => { activeEditableElement = task.instance; }}
								onblur={() => { task.editable = false; activeEditableElement = undefined; }}
								oninput={(event) => onEditableInput(event)}
								onpaste={(event) => onPaste(event)}
								class="selected my-3 box-border min-h-4 skew-x-0 cursor-text rounded-2xl bg-white p-4 whitespace-pre-line text-pretty text-slate-700 outline-none dark:bg-slate-900 dark:text-slate-100"
								bind:this={task.instance}
								bind:innerText={task.title}
								role="none"
							></div>
						{:else}
							<div
								class="my-3 box-border min-h-[58px] skew-x-0 cursor-default rounded-2xl border-2 border-transparent bg-white p-[15px] whitespace-pre-line text-pretty text-slate-700 shadow-[0_1px_2px_rgba(15,23,42,0.04)] ring-1 ring-slate-950/10 transition-shadow duration-200 hover:shadow-[0_4px_14px_rgba(15,23,42,0.08)] dark:bg-slate-900 dark:text-slate-100 dark:shadow-none dark:ring-white/10"
								draggable="true"
								onclick={(event) => onTaskClicked(event, task)}
								ondragstart={(event) => onDrag(event, task)}
								bind:this={task.instance}
								role="none"
							>
								{@html toHtml(task.title)}
							</div>
						{/if}
					{/each}
				</div>
			</div>
		{/each}
	</div>
</div>

<EmojiPicker bind:this={emojiPicker} element={activeEditableElement} onselect={onEmojiSelect} />

<style lang="postcss">
	.selected {
		border: 2px solid rgb(37 99 235);
		padding: 15px;
	}

	:global(article > *) {
		margin-bottom: 1rem;
		overflow-wrap: break-word;
	}

	:global(article > :last-child) {
		margin-bottom: 0;
	}

	:global(article code) {
		background-color: rgb(241 245 249);
		padding-left: 0.25rem;
		padding-right: 0.25rem;
		padding-top: 0.125rem;
		padding-bottom: 0.125rem;
		border-radius: 0.375rem;
		font-size: 0.875rem;
		line-height: 1.25rem;
	}

	:global(article ul) {
		padding-left: 1rem;
		list-style-type: disc;
	}

	:global(article ol) {
		padding-left: 1rem;
		list-style-type: decimal;
	}

	:global(article a) {
		color: rgb(37 99 235);
		text-decoration: underline;
	}

	@media (prefers-color-scheme: dark) {
		.selected {
			border-color: rgb(59 130 246);
		}

		:global(article code) {
			background-color: rgb(30 41 59);
			box-shadow: inset 0 0 0 1px rgb(255 255 255 / 0.06);
			color: rgb(226 232 240);
		}

		:global(article a) {
			color: rgb(96 165 250);
		}
	}
</style>
