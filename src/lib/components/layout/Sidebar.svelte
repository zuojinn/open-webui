<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { v4 as uuidv4 } from 'uuid';

	import { goto } from '$app/navigation';
	import {
		user,
		chats,
		settings,
		showSettings,
		chatId,
		tags,
		showSidebar,
		mobile,
		showArchivedChats,
		pinnedChats,
		scrollPaginationEnabled,
		currentChatPage,
		temporaryChatEnabled,
		channels,
		socket,
		config,
		isApp
	} from '$lib/stores';
	import { onMount, getContext, tick, onDestroy } from 'svelte';

	const i18n = getContext('i18n');

	import {
		deleteChatById,
		getChatList,
		getAllTags,
		getChatListBySearchText,
		createNewChat,
		getPinnedChatList,
		toggleChatPinnedStatusById,
		getChatPinnedStatusById,
		getChatById,
		updateChatFolderIdById,
		importChat
	} from '$lib/apis/chats';
	import { createNewFolder, getFolders, updateFolderParentIdById } from '$lib/apis/folders';
	import { WEBUI_BASE_URL } from '$lib/constants';

	import ArchivedChatsModal from './Sidebar/ArchivedChatsModal.svelte';
	import UserMenu from './Sidebar/UserMenu.svelte';
	import ChatItem from './Sidebar/ChatItem.svelte';
	import Spinner from '../common/Spinner.svelte';
	import Loader from '../common/Loader.svelte';
	import AddFilesPlaceholder from '../AddFilesPlaceholder.svelte';
	import SearchInput from './Sidebar/SearchInput.svelte';
	import Folder from '../common/Folder.svelte';
	import Plus from '../icons/Plus.svelte';
	import Tooltip from '../common/Tooltip.svelte';
	import Folders from './Sidebar/Folders.svelte';
	import { getChannels, createNewChannel } from '$lib/apis/channels';
	import ChannelModal from './Sidebar/ChannelModal.svelte';
	import ChannelItem from './Sidebar/ChannelItem.svelte';
	import PencilSquare from '../icons/PencilSquare.svelte';
	import Home from '../icons/Home.svelte';

	const BREAKPOINT = 768;

	let navElement;
	let search = '';

	let shiftKey = false;

	let selectedChatId = null;
	let showDropdown = false;
	let showPinnedChat = true;

	let showCreateChannel = false;

	// Pagination variables
	let chatListLoading = false;
	let allChatsLoaded = false;

	let folders = {};

	const initFolders = async () => {
		const folderList = await getFolders(localStorage.token).catch((error) => {
			toast.error(`${error}`);
			return [];
		});

		folders = {};

		// First pass: Initialize all folder entries
		for (const folder of folderList) {
			// Ensure folder is added to folders with its data
			folders[folder.id] = { ...(folders[folder.id] || {}), ...folder };
		}

		// Second pass: Tie child folders to their parents
		for (const folder of folderList) {
			if (folder.parent_id) {
				// Ensure the parent folder is initialized if it doesn't exist
				if (!folders[folder.parent_id]) {
					folders[folder.parent_id] = {}; // Create a placeholder if not already present
				}

				// Initialize childrenIds array if it doesn't exist and add the current folder id
				folders[folder.parent_id].childrenIds = folders[folder.parent_id].childrenIds
					? [...folders[folder.parent_id].childrenIds, folder.id]
					: [folder.id];

				// Sort the children by updated_at field
				folders[folder.parent_id].childrenIds.sort((a, b) => {
					return folders[b].updated_at - folders[a].updated_at;
				});
			}
		}
	};

	const createFolder = async (name = 'Untitled') => {
		if (name === '') {
			toast.error($i18n.t('Folder name cannot be empty.'));
			return;
		}

		const rootFolders = Object.values(folders).filter((folder) => folder.parent_id === null);
		if (rootFolders.find((folder) => folder.name.toLowerCase() === name.toLowerCase())) {
			// If a folder with the same name already exists, append a number to the name
			let i = 1;
			while (
				rootFolders.find((folder) => folder.name.toLowerCase() === `${name} ${i}`.toLowerCase())
			) {
				i++;
			}

			name = `${name} ${i}`;
		}

		// Add a dummy folder to the list to show the user that the folder is being created
		const tempId = uuidv4();
		folders = {
			...folders,
			tempId: {
				id: tempId,
				name: name,
				created_at: Date.now(),
				updated_at: Date.now()
			}
		};

		const res = await createNewFolder(localStorage.token, name).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		if (res) {
			await initFolders();
		}
	};

	const initChannels = async () => {
		await channels.set(await getChannels(localStorage.token));
	};

	const initChatList = async () => {
		// Reset pagination variables
		tags.set(await getAllTags(localStorage.token));
		pinnedChats.set(await getPinnedChatList(localStorage.token));
		initFolders();

		currentChatPage.set(1);
		allChatsLoaded = false;

		if (search) {
			await chats.set(await getChatListBySearchText(localStorage.token, search, $currentChatPage));
		} else {
			await chats.set(await getChatList(localStorage.token, $currentChatPage));
		}

		// Enable pagination
		scrollPaginationEnabled.set(true);
	};

	const loadMoreChats = async () => {
		chatListLoading = true;

		currentChatPage.set($currentChatPage + 1);

		let newChatList = [];

		if (search) {
			newChatList = await getChatListBySearchText(localStorage.token, search, $currentChatPage);
		} else {
			newChatList = await getChatList(localStorage.token, $currentChatPage);
		}

		// once the bottom of the list has been reached (no results) there is no need to continue querying
		allChatsLoaded = newChatList.length === 0;
		await chats.set([...($chats ? $chats : []), ...newChatList]);

		chatListLoading = false;
	};

	let searchDebounceTimeout;

	const searchDebounceHandler = async () => {
		console.log('search', search);
		chats.set(null);

		if (searchDebounceTimeout) {
			clearTimeout(searchDebounceTimeout);
		}

		if (search === '') {
			await initChatList();
			return;
		} else {
			searchDebounceTimeout = setTimeout(async () => {
				allChatsLoaded = false;
				currentChatPage.set(1);
				await chats.set(await getChatListBySearchText(localStorage.token, search));

				if ($chats.length === 0) {
					tags.set(await getAllTags(localStorage.token));
				}
			}, 1000);
		}
	};

	const importChatHandler = async (items, pinned = false, folderId = null) => {
		console.log('importChatHandler', items, pinned, folderId);
		for (const item of items) {
			console.log(item);
			if (item.chat) {
				await importChat(localStorage.token, item.chat, item?.meta ?? {}, pinned, folderId);
			}
		}

		initChatList();
	};

	const inputFilesHandler = async (files) => {
		console.log(files);

		for (const file of files) {
			const reader = new FileReader();
			reader.onload = async (e) => {
				const content = e.target.result;

				try {
					const chatItems = JSON.parse(content);
					importChatHandler(chatItems);
				} catch {
					toast.error($i18n.t(`Invalid file format.`));
				}
			};

			reader.readAsText(file);
		}
	};

	const tagEventHandler = async (type, tagName, chatId) => {
		console.log(type, tagName, chatId);
		if (type === 'delete') {
			initChatList();
		} else if (type === 'add') {
			initChatList();
		}
	};

	let draggedOver = false;

	const onDragOver = (e) => {
		e.preventDefault();

		// Check if a file is being draggedOver.
		if (e.dataTransfer?.types?.includes('Files')) {
			draggedOver = true;
		} else {
			draggedOver = false;
		}
	};

	const onDragLeave = () => {
		draggedOver = false;
	};

	const onDrop = async (e) => {
		e.preventDefault();
		console.log(e); // Log the drop event

		// Perform file drop check and handle it accordingly
		if (e.dataTransfer?.files) {
			const inputFiles = Array.from(e.dataTransfer?.files);

			if (inputFiles && inputFiles.length > 0) {
				console.log(inputFiles); // Log the dropped files
				inputFilesHandler(inputFiles); // Handle the dropped files
			}
		}

		draggedOver = false; // Reset draggedOver status after drop
	};

	let touchstart;
	let touchend;

	function checkDirection() {
		const screenWidth = window.innerWidth;
		const swipeDistance = Math.abs(touchend.screenX - touchstart.screenX);
		if (touchstart.clientX < 40 && swipeDistance >= screenWidth / 8) {
			if (touchend.screenX < touchstart.screenX) {
				showSidebar.set(false);
			}
			if (touchend.screenX > touchstart.screenX) {
				showSidebar.set(true);
			}
		}
	}

	const onTouchStart = (e) => {
		touchstart = e.changedTouches[0];
		console.log(touchstart.clientX);
	};

	const onTouchEnd = (e) => {
		touchend = e.changedTouches[0];
		checkDirection();
	};

	const onKeyDown = (e) => {
		if (e.key === 'Shift') {
			shiftKey = true;
		}
	};

	const onKeyUp = (e) => {
		if (e.key === 'Shift') {
			shiftKey = false;
		}
	};

	const onFocus = () => {};

	const onBlur = () => {
		shiftKey = false;
		selectedChatId = null;
	};

	onMount(async () => {
		showPinnedChat = localStorage?.showPinnedChat ? localStorage.showPinnedChat === 'true' : true;

		mobile.subscribe((value) => {
			if ($showSidebar && value) {
				showSidebar.set(false);
			}

			if ($showSidebar && !value) {
				const navElement = document.getElementsByTagName('nav')[0];
				if (navElement) {
					navElement.style['-webkit-app-region'] = 'drag';
				}
			}

			if (!$showSidebar && !value) {
				showSidebar.set(true);
			}
		});

		showSidebar.set(!$mobile ? localStorage.sidebar === 'true' : false);
		showSidebar.subscribe((value) => {
			localStorage.sidebar = value;

			// nav element is not available on the first render
			const navElement = document.getElementsByTagName('nav')[0];

			if (navElement) {
				if ($mobile) {
					if (!value) {
						navElement.style['-webkit-app-region'] = 'drag';
					} else {
						navElement.style['-webkit-app-region'] = 'no-drag';
					}
				} else {
					navElement.style['-webkit-app-region'] = 'drag';
				}
			}
		});

		await initChannels();
		await initChatList();

		window.addEventListener('keydown', onKeyDown);
		window.addEventListener('keyup', onKeyUp);

		window.addEventListener('touchstart', onTouchStart);
		window.addEventListener('touchend', onTouchEnd);

		window.addEventListener('focus', onFocus);
		window.addEventListener('blur-sm', onBlur);

		const dropZone = document.getElementById('sidebar');

		dropZone?.addEventListener('dragover', onDragOver);
		dropZone?.addEventListener('drop', onDrop);
		dropZone?.addEventListener('dragleave', onDragLeave);
	});

	onDestroy(() => {
		window.removeEventListener('keydown', onKeyDown);
		window.removeEventListener('keyup', onKeyUp);

		window.removeEventListener('touchstart', onTouchStart);
		window.removeEventListener('touchend', onTouchEnd);

		window.removeEventListener('focus', onFocus);
		window.removeEventListener('blur-sm', onBlur);

		const dropZone = document.getElementById('sidebar');

		dropZone?.removeEventListener('dragover', onDragOver);
		dropZone?.removeEventListener('drop', onDrop);
		dropZone?.removeEventListener('dragleave', onDragLeave);
	});
       // color: "linear-gradient(to bottom, #A8D8FF, #007BFF)",
	  // color: "linear-gradient(to bottom, #C69C6D, #8B5A2B)",
	  let sections = [
		{
		  title: "通用工具",
		  color:"#ececec",
		  items: ["AI阅读", "AI写作", "AI画图"],
		  open: true
		},
		{
		  title: "办公场景",
		  color:"#ececec",
		  items: ["AI会议纪要", "HR简历助手"],
		  open: true
		}
	  ];

  function toggleSection(index) {
    sections[index].open = !sections[index].open;
  }
</script>

<ArchivedChatsModal
	bind:show={$showArchivedChats}
	on:change={async () => {
		await initChatList();
	}}
/>

<ChannelModal
	bind:show={showCreateChannel}
	onSubmit={async ({ name, access_control }) => {
		const res = await createNewChannel(localStorage.token, {
			name: name,
			access_control: access_control
		}).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		if (res) {
			$socket.emit('join-channels', { auth: { token: $user.token } });
			await initChannels();
			showCreateChannel = false;
		}
	}}
/>

<!-- svelte-ignore a11y-no-static-element-interactions -->

{#if $showSidebar}
	<div
		class=" {$isApp
			? ' ml-[4.5rem] md:ml-0'
			: ''} fixed md:hidden z-40 top-0 right-0 left-0 bottom-0 bg-black/60 w-full min-h-screen h-screen flex justify-center overflow-hidden overscroll-contain"
		on:mousedown={() => {
			showSidebar.set(!$showSidebar);
		}}
	/>
{/if}

<div
	bind:this={navElement}
	id="sidebar"
	class="h-screen max-h-[100dvh] min-h-screen select-none {$showSidebar
		? 'md:relative w-[260px] max-w-[260px]'
		: '-translate-x-[260px] w-[0px]'} {$isApp
		? `ml-[4.5rem] md:ml-0 `
		: 'transition-width duration-200 ease-in-out'}  shrink-0 bg-gray-50 text-gray-900 dark:bg-gray-950 dark:text-gray-200 fixed z-50 top-0 left-0 overflow-x-hidden
        "
	data-state={$showSidebar}
>
	<div
		class="py-2 my-auto flex flex-col justify-between h-screen max-h-[100dvh] w-[260px] overflow-x-hidden z-50 {$showSidebar
			? ''
			: 'invisible'}"
	>
		<div class="px-1.5 flex justify-between space-x-1 text-gray-600 dark:text-gray-400">
			<button
				class=" cursor-pointer p-[7px] flex rounded-xl hover:bg-gray-100 dark:hover:bg-gray-900 transition"
				on:click={() => {
					showSidebar.set(!$showSidebar);
				}}
			>
				<div class=" m-auto self-center">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						fill="none"
						viewBox="0 0 24 24"
						stroke-width="2"
						stroke="currentColor"
						class="size-5"
					>
						<path
							stroke-linecap="round"
							stroke-linejoin="round"
							d="M3.75 6.75h16.5M3.75 12h16.5m-16.5 5.25H12"
						/>
					</svg>
				</div>
			</button>

			<a
				id="sidebar-new-chat-button"
				class="flex justify-between items-center flex-1 rounded-lg px-2 py-1 h-full text-right hover:bg-gray-100 dark:hover:bg-gray-900 transition no-drag-region"
				href="/"
				draggable="false"
				on:click={async () => {
					selectedChatId = null;
					await goto('/');
					const newChatButton = document.getElementById('new-chat-button');
					setTimeout(() => {
						newChatButton?.click();
						if ($mobile) {
							showSidebar.set(false);
						}
					}, 0);
				}}
			>
				<div class="flex items-center">
					<div class="self-center mx-1.5">
						<img
							crossorigin="anonymous"
							src="/static/favicon.png"
							class=" size-5 -translate-x-1.5 rounded-full"
							alt="logo"
						/>
					</div>
					<div class=" self-center font-medium text-gray-850 dark:text-white font-primary">
						<!--{$i18n.t('New Chat')}-->
						AI对话
					</div>
				</div>

				<div>
					<PencilSquare className=" size-5" strokeWidth="2" />
				</div>
			</a>
		</div>

		<!-- {#if $user?.role === 'admin'}
			<div class="px-1.5 flex justify-center text-gray-800 dark:text-gray-200">
				<a
					class="grow flex items-center space-x-3 rounded-lg px-2 py-[7px] hover:bg-gray-100 dark:hover:bg-gray-900 transition"
					href="/home"
					on:click={() => {
						selectedChatId = null;
						chatId.set('');

						if ($mobile) {
							showSidebar.set(false);
						}
					}}
					draggable="false"
				>
					<div class="self-center">
						<Home strokeWidth="2" className="size-[1.1rem]" />
					</div>

					<div class="flex self-center translate-y-[0.5px]">
						<div class=" self-center font-medium font-primary">{$i18n.t('Home')}</div>
					</div>
				</a>
			</div>
		{/if} -->

		{#if $user?.role === 'admin' || $user?.permissions?.workspace?.models || $user?.permissions?.workspace?.knowledge || $user?.permissions?.workspace?.prompts || $user?.permissions?.workspace?.tools}
			<div class="px-1.5 flex justify-center text-gray-800 dark:text-gray-200">
				<a
					class="grow flex items-center space-x-3 rounded-lg px-2 py-[7px] hover:bg-gray-100 dark:hover:bg-gray-900 transition"
					href="/workspace"
					on:click={() => {
						selectedChatId = null;
						chatId.set('');

						if ($mobile) {
							showSidebar.set(false);
						}
					}}
					draggable="false"
				>
					<div class="self-center">
						<svg
							xmlns="http://www.w3.org/2000/svg"
							fill="none"
							viewBox="0 0 24 24"
							stroke-width="2"
							stroke="currentColor"
							class="size-[1.1rem]"
						>
							<path
								stroke-linecap="round"
								stroke-linejoin="round"
								d="M13.5 16.875h3.375m0 0h3.375m-3.375 0V13.5m0 3.375v3.375M6 10.5h2.25a2.25 2.25 0 0 0 2.25-2.25V6a2.25 2.25 0 0 0-2.25-2.25H6A2.25 2.25 0 0 0 3.75 6v2.25A2.25 2.25 0 0 0 6 10.5Zm0 9.75h2.25A2.25 2.25 0 0 0 10.5 18v-2.25a2.25 2.25 0 0 0-2.25-2.25H6a2.25 2.25 0 0 0-2.25 2.25V18A2.25 2.25 0 0 0 6 20.25Zm9.75-9.75H18a2.25 2.25 0 0 0 2.25-2.25V6A2.25 2.25 0 0 0 18 3.75h-2.25A2.25 2.25 0 0 0 13.5 6v2.25a2.25 2.25 0 0 0 2.25 2.25Z"
							/>
						</svg>
					</div>

					<div class="flex self-center translate-y-[0.5px]">
						<div class=" self-center font-medium font-primary">{$i18n.t('Workspace')}</div>
					</div>
				</a>
			</div>
		{/if}

		<div class="px-1.5 flex justify-center text-gray-800 dark:text-gray-200">
				<a
					class="grow flex items-center space-x-3 rounded-lg px-2 py-[7px] transition"
					href=""
					on:click={() => {
						selectedChatId = null;
						chatId.set('');

						if ($mobile) {
							showSidebar.set(false);
						}
					}}
					draggable="false"
				>
					<div class="self-center">
						<svg t="1742457529719"  stroke="currentColor"  stroke-width="2" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" class="size-[1.1rem]"><path d="M554.666667 221.866667h-174.250667v580.266666H716.8a85.333333 85.333333 0 0 0 85.333333-85.333333V307.2c0-38.229333-25.122133-70.5536-59.733333-81.442133V409.6a25.6 25.6 0 0 1-43.690667 18.090667L648.533333 377.514667l-50.176 50.176a25.6 25.6 0 0 1-43.690666-18.090667V221.866667z m-225.450667 0H307.2A85.333333 85.333333 0 0 0 221.866667 307.2v409.6A85.333333 85.333333 0 0 0 307.2 802.133333h22.016v-580.266666zM307.2 170.666667h409.6a136.533333 136.533333 0 0 1 136.533333 136.533333v409.6a136.533333 136.533333 0 0 1-136.533333 136.533333H307.2a136.533333 136.533333 0 0 1-136.533333-136.533333V307.2a136.533333 136.533333 0 0 1 136.533333-136.533333z m298.666667 59.733333v117.418667l24.576-24.576a25.6 25.6 0 0 1 36.181333 0l24.576 24.576V230.4h-85.333333z"></path></svg>
					</div>

					<div class="flex self-center translate-y-[0.5px]">
						<div class=" self-center font-medium font-primary">
							<!--{$i18n.t('Workspace')}-->
							知识库
						</div>
					</div>
				</a>
		</div>

		<div class="px-1.5 flex justify-center text-gray-800 dark:text-gray-200">
				<a
					class="grow flex items-center space-x-3 rounded-lg px-2 py-[7px] hover:bg-gray-100 dark:hover:bg-gray-900 transition"
					href="/"
					draggable="false"
					on:click={async () => {
						selectedChatId = null;
						await goto('/');
						const newChatButton = document.getElementById('new-chat-button');
						setTimeout(() => {
							newChatButton?.click();
							if ($mobile) {
								showSidebar.set(false);
							}
						}, 0);
					}}
				>
					<div class="self-center">
						<svg t="1742457529719"  stroke="currentColor"  stroke-width="2" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" class="size-[1.1rem]"><path d="M540.553846 83.298462c58.88 0.393846 114.609231 13.981538 166.596923 41.156923 76.996923 40.172308 134.104615 99.84 167.778462 179.790769 51.396923 122.092308 39.975385 239.655385-32.098462 350.72-5.513846 8.467692-5.513846 12.996923 2.166154 20.283077 58.486154 54.350769 116.381538 108.898462 174.276923 163.643077 16.738462 15.753846 21.267692 34.461538 13.784616 52.381538-12.8 30.326154-49.427692 38.4-74.24 15.753846-20.676923-18.707692-40.763077-38.006154-61.046154-57.107692-39.187692-36.824615-78.572308-73.452308-117.563077-110.670769-5.513846-5.12-9.058462-5.316923-14.769231-0.787693-48.64 39.187692-103.975385 63.409231-165.415385 73.255385-97.673846 15.753846-188.258462-3.150769-269.587692-59.864615-90.190769-62.818462-143.556923-149.661538-155.766154-259.15077-13.587692-121.698462 25.206154-225.673846 113.821539-310.350769 60.455385-57.895385 133.513846-89.206154 216.615384-97.673846 11.815385-0.787692 23.630769-0.787692 35.446154-1.378461z m302.867692 364.504615c0.787692-161.870769-132.332308-296.96-293.415384-302.08-175.852308-5.513846-310.744615 133.513846-315.864616 294.793846-5.513846 172.307692 132.529231 314.683077 304.64 315.076923 169.353846 0.590769 308.184615-139.815385 304.64-307.790769z" p-id="3788"></path><path d="M649.846154 574.818462c-0.196923 13.587692-7.68 21.858462-20.676923 24.812307-10.043077 2.166154-21.858462-4.332308-26.584616-15.950769-4.923077-12.209231-9.058462-24.615385-12.996923-37.218462-2.363077-7.876923-6.695385-10.830769-14.966154-10.830769-22.646154 0.393846-45.292308 0.393846-67.938461 0-8.861538-0.196923-13.193846 3.150769-15.556923 11.421539-3.544615 11.618462-7.68 23.04-12.012308 34.264615-5.710769 14.769231-17.723077 20.676923-30.916923 15.950769-12.012308-4.332308-17.92-18.313846-12.8-32.886154 13.587692-38.990769 27.569231-77.981538 41.550769-116.775384 13.784615-38.990769 27.569231-77.981538 41.747693-116.775385 2.166154-5.907692 5.12-12.603077 9.64923-16.738461 11.618462-10.24 29.144615-5.12 35.249231 10.436923 7.483077 18.904615 13.587692 38.203077 20.283077 57.501538 21.070769 61.046154 42.338462 122.092308 63.409231 183.138462 1.181538 3.938462 1.969231 7.876923 2.56 9.649231z m-107.126154-86.843077h15.753846c8.073846 0 10.633846-4.332308 8.270769-11.421539-5.907692-17.723077-11.815385-35.249231-18.313846-52.775384-0.787692-2.363077-3.741538-4.726154-6.104615-5.12-1.575385-0.196923-4.726154 2.56-5.316923 4.52923-6.498462 18.116923-12.8 36.430769-18.707693 54.744616-1.969231 6.301538 0.984615 10.24 7.876924 10.24 5.316923-0.196923 10.830769-0.196923 16.541538-0.196923zM313.304615 448c0.590769-45.292308 15.950769-84.48 47.064616-117.563077 7.286154-7.876923 15.950769-11.618462 26.584615-8.664615 8.664615 2.363077 13.587692 8.861538 15.753846 17.32923 2.56 10.633846-2.953846 18.313846-9.64923 25.403077-14.375385 15.753846-24.418462 34.067692-29.144616 54.941539-9.255385 40.566154 0.590769 76.406154 26.584616 108.504615 5.316923 6.695385 10.633846 14.572308 12.406153 22.646154 1.969231 9.649231-4.135385 17.723077-13.390769 21.858462-9.255385 4.135385-18.904615 2.756923-25.403077-4.726154-11.027692-12.8-22.449231-25.993846-30.72-40.566154-13.784615-24.024615-20.283077-50.806154-20.086154-79.163077z" p-id="3789"></path><path d="M687.458462 480.098462c0-28.553846-0.393846-57.107692 0.196923-85.661539 0.196923-19.692308 18.116923-30.916923 34.855384-22.252308 9.058462 4.726154 13.193846 12.996923 13.390769 22.646154 0.393846 57.304615 0.590769 114.806154 0.196924 172.110769 0 14.178462-12.406154 25.6-25.206154 25.206154-13.193846-0.393846-23.04-10.633846-23.236923-25.6-0.590769-28.750769-0.393846-57.698462-0.196923-86.44923-0.196923 0 0 0 0 0zM711.286154 340.676923c-12.8-0.196923-23.827692-10.633846-23.630769-22.843077 0-12.012308 11.224615-22.646154 23.827692-22.449231 12.209231 0 24.221538 11.421538 23.827692 22.843077-0.196923 12.012308-11.618462 22.646154-24.024615 22.449231z"></path></svg>
					</div>

					<div class="flex self-center translate-y-[0.5px]">
						<div class=" self-center font-medium font-primary">
							<!--{$i18n.t('Workspace')}-->
							AI搜索
						</div>
					</div>
				</a>
		</div>

		<div class="px-1.5 flex justify-center text-gray-800 dark:text-gray-200" style="display:none;">
				<a
					class="grow flex items-center space-x-3 rounded-lg px-2 py-[7px] hover:bg-gray-100 dark:hover:bg-gray-900 transition"
					href="/generalTools"
					on:click={() => {
						selectedChatId = null;
						chatId.set('');

						if ($mobile) {
							showSidebar.set(false);
						}
					}}
					draggable="false"
				>
					<div class="self-center">
						<svg t="1742457529719"  stroke="currentColor"  stroke-width="2" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" class="size-[1.1rem]"><path d="M227.328 139.5712a149.3504 149.3504 0 0 1 190.8736 190.8736l462.6944 462.7456-90.5216 90.5216L327.68 421.0176a149.3504 149.3504 0 0 1-190.8224-190.9248l95.4368 95.4368a64 64 0 0 0 90.5216-90.4704L227.3792 139.5712z m442.368 80.384l135.7824-75.4176L865.8432 204.8l-75.4688 135.7824-75.4176 15.104-90.5216 90.4704-60.3136-60.3136 90.5216-90.5216 15.0528-75.4176z m-301.6704 331.8784l90.5216 90.5216L247.296 853.504a64 64 0 0 1-94.6688-85.9136l4.1472-4.608 211.2-211.1488z"></path></svg>
					</div>

					<div class="flex self-center translate-y-[0.5px]">
						<div class=" self-center font-medium font-primary">
							<!--{$i18n.t('Workspace')}-->
							通用工具
						</div>
					</div>
				</a>
		</div>

		<div class="px-1.5 flex justify-center text-gray-800 dark:text-gray-200" style="display:none;">
				<a
					class="grow flex items-center space-x-3 rounded-lg px-2 py-[7px] hover:bg-gray-100 dark:hover:bg-gray-900 transition"
					href="/officeScene"
					on:click={() => {
						selectedChatId = null;
						chatId.set('');

						if ($mobile) {
							showSidebar.set(false);
						}
					}}
					draggable="false"
				>
					<div class="self-center">
						<svg t="1742457529719"  stroke="currentColor"  stroke-width="2" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" class="size-[1.1rem]"><path d="M85.333333 128a42.666667 42.666667 0 0 1 42.666667-42.666667h469.333333a42.666667 42.666667 0 0 1 42.666667 42.666667v298.666667a42.666667 42.666667 0 0 1-42.666667 42.666666H128a42.666667 42.666667 0 0 1-42.666667-42.666666V128z m85.333334 42.666667v213.333333h384V170.666667H170.666667zM42.666667 576a42.666667 42.666667 0 0 1 42.666666-42.666667h853.333334a42.666667 42.666667 0 0 1 42.666666 42.666667v341.333333a42.666667 42.666667 0 0 1-42.666666 42.666667H682.666667a42.666667 42.666667 0 0 1-42.666667-42.666667v-133.333333H298.666667V917.333333a42.666667 42.666667 0 0 1-42.666667 42.666667H85.333333a42.666667 42.666667 0 0 1-42.666666-42.666667V576z m85.333333 42.666667v256h85.333333v-133.333334a42.666667 42.666667 0 0 1 42.666667-42.666666h426.666667a42.666667 42.666667 0 0 1 42.666666 42.666666V874.666667h170.666667V618.666667H128z" fill="#000000" p-id="6132"></path><path d="M362.666667 416a42.666667 42.666667 0 0 1 42.666666 42.666667v85.333333a42.666667 42.666667 0 1 1-85.333333 0v-85.333333a42.666667 42.666667 0 0 1 42.666667-42.666667zM682.666667 448a42.666667 42.666667 0 0 1 42.666666 42.666667v85.333333a42.666667 42.666667 0 1 1-85.333333 0v-85.333333a42.666667 42.666667 0 0 1 42.666667-42.666667zM810.666667 384a42.666667 42.666667 0 0 1 42.666666 42.666667v149.333333a42.666667 42.666667 0 1 1-85.333333 0v-149.333333a42.666667 42.666667 0 0 1 42.666667-42.666667zM938.666667 298.666667a42.666667 42.666667 0 0 1 42.666666 42.666666v234.666667a42.666667 42.666667 0 1 1-85.333333 0V341.333333a42.666667 42.666667 0 0 1 42.666667-42.666666z"></path></svg>
					</div>

					<div class="flex self-center translate-y-[0.5px]">
						<div class=" self-center font-medium font-primary">
							<!--{$i18n.t('Workspace')}-->
							办公场景
						</div>
					</div>
				</a>
		</div>

		<div class="menu">
		  {#each sections as section, i}
			<div class="section" style="background: {section.color};">
			  <div class="header" on:click={() => toggleSection(i)}>
				  {#if i=== 0}
					  <div class="self-center">
							<svg t="1742457529719"  stroke="currentColor"  stroke-width="2" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" class="size-[1.1rem]"><path d="M227.328 139.5712a149.3504 149.3504 0 0 1 190.8736 190.8736l462.6944 462.7456-90.5216 90.5216L327.68 421.0176a149.3504 149.3504 0 0 1-190.8224-190.9248l95.4368 95.4368a64 64 0 0 0 90.5216-90.4704L227.3792 139.5712z m442.368 80.384l135.7824-75.4176L865.8432 204.8l-75.4688 135.7824-75.4176 15.104-90.5216 90.4704-60.3136-60.3136 90.5216-90.5216 15.0528-75.4176z m-301.6704 331.8784l90.5216 90.5216L247.296 853.504a64 64 0 0 1-94.6688-85.9136l4.1472-4.608 211.2-211.1488z"></path></svg>
					  </div>
				  {/if}
				  {#if i=== 1}
					  <div class="self-center">
							<svg t="1742457529719"  stroke="currentColor"  stroke-width="2" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" class="size-[1.1rem]"><path d="M85.333333 128a42.666667 42.666667 0 0 1 42.666667-42.666667h469.333333a42.666667 42.666667 0 0 1 42.666667 42.666667v298.666667a42.666667 42.666667 0 0 1-42.666667 42.666666H128a42.666667 42.666667 0 0 1-42.666667-42.666666V128z m85.333334 42.666667v213.333333h384V170.666667H170.666667zM42.666667 576a42.666667 42.666667 0 0 1 42.666666-42.666667h853.333334a42.666667 42.666667 0 0 1 42.666666 42.666667v341.333333a42.666667 42.666667 0 0 1-42.666666 42.666667H682.666667a42.666667 42.666667 0 0 1-42.666667-42.666667v-133.333333H298.666667V917.333333a42.666667 42.666667 0 0 1-42.666667 42.666667H85.333333a42.666667 42.666667 0 0 1-42.666666-42.666667V576z m85.333333 42.666667v256h85.333333v-133.333334a42.666667 42.666667 0 0 1 42.666667-42.666666h426.666667a42.666667 42.666667 0 0 1 42.666666 42.666666V874.666667h170.666667V618.666667H128z" fill="#000000" p-id="6132"></path><path d="M362.666667 416a42.666667 42.666667 0 0 1 42.666666 42.666667v85.333333a42.666667 42.666667 0 1 1-85.333333 0v-85.333333a42.666667 42.666667 0 0 1 42.666667-42.666667zM682.666667 448a42.666667 42.666667 0 0 1 42.666666 42.666667v85.333333a42.666667 42.666667 0 1 1-85.333333 0v-85.333333a42.666667 42.666667 0 0 1 42.666667-42.666667zM810.666667 384a42.666667 42.666667 0 0 1 42.666666 42.666667v149.333333a42.666667 42.666667 0 1 1-85.333333 0v-149.333333a42.666667 42.666667 0 0 1 42.666667-42.666667zM938.666667 298.666667a42.666667 42.666667 0 0 1 42.666666 42.666666v234.666667a42.666667 42.666667 0 1 1-85.333333 0V341.333333a42.666667 42.666667 0 0 1 42.666667-42.666666z"></path></svg>
					  </div>
				  {/if}
				  {section.title}
			  </div>
			  {#if section.open}
				<div class="items">
				  {#each section.items as item}
					<div class="item">{item}</div>
				  {/each}
				</div>
			  {/if}
			</div>
		  {/each}
		</div>



		<div class="relative {$temporaryChatEnabled ? 'opacity-20' : ''}">
			{#if $temporaryChatEnabled}
				<div class="absolute z-40 w-full h-full flex justify-center"></div>
			{/if}

			<SearchInput
				bind:value={search}
				on:input={searchDebounceHandler}
				placeholder={$i18n.t('Search')}
			/>
		</div>

		<div
			class="relative flex flex-col flex-1 overflow-y-auto overflow-x-hidden {$temporaryChatEnabled
				? 'opacity-20'
				: ''}"
		>
			{#if $config?.features?.enable_channels && ($user.role === 'admin' || $channels.length > 0) && !search}
				<Folder
					className="px-2 mt-0.5"
					name={$i18n.t('Channels')}
					dragAndDrop={false}
					onAdd={async () => {
						if ($user.role === 'admin') {
							await tick();

							setTimeout(() => {
								showCreateChannel = true;
							}, 0);
						}
					}}
					onAddLabel={$i18n.t('Create Channel')}
				>
					{#each $channels as channel}
						<ChannelItem
							{channel}
							onUpdate={async () => {
								await initChannels();
							}}
						/>
					{/each}
				</Folder>
			{/if}

			<Folder
				collapsible={!search}
				className="px-2 mt-0.5"
				name={$i18n.t('Chats')}
				onAdd={() => {
					createFolder();
				}}
				onAddLabel={$i18n.t('New Folder')}
				on:import={(e) => {
					importChatHandler(e.detail);
				}}
				on:drop={async (e) => {
					const { type, id, item } = e.detail;

					if (type === 'chat') {
						let chat = await getChatById(localStorage.token, id).catch((error) => {
							return null;
						});
						if (!chat && item) {
							chat = await importChat(localStorage.token, item.chat, item?.meta ?? {});
						}

						if (chat) {
							console.log(chat);
							if (chat.folder_id) {
								const res = await updateChatFolderIdById(localStorage.token, chat.id, null).catch(
									(error) => {
										toast.error(`${error}`);
										return null;
									}
								);
							}

							if (chat.pinned) {
								const res = await toggleChatPinnedStatusById(localStorage.token, chat.id);
							}

							initChatList();
						}
					} else if (type === 'folder') {
						if (folders[id].parent_id === null) {
							return;
						}

						const res = await updateFolderParentIdById(localStorage.token, id, null).catch(
							(error) => {
								toast.error(`${error}`);
								return null;
							}
						);

						if (res) {
							await initFolders();
						}
					}
				}}
			>
				{#if $temporaryChatEnabled}
					<div class="absolute z-40 w-full h-full flex justify-center"></div>
				{/if}

				{#if !search && $pinnedChats.length > 0}
					<div class="flex flex-col space-y-1 rounded-xl">
						<Folder
							className=""
							bind:open={showPinnedChat}
							on:change={(e) => {
								localStorage.setItem('showPinnedChat', e.detail);
								console.log(e.detail);
							}}
							on:import={(e) => {
								importChatHandler(e.detail, true);
							}}
							on:drop={async (e) => {
								const { type, id, item } = e.detail;

								if (type === 'chat') {
									let chat = await getChatById(localStorage.token, id).catch((error) => {
										return null;
									});
									if (!chat && item) {
										chat = await importChat(localStorage.token, item.chat, item?.meta ?? {});
									}

									if (chat) {
										console.log(chat);
										if (chat.folder_id) {
											const res = await updateChatFolderIdById(
												localStorage.token,
												chat.id,
												null
											).catch((error) => {
												toast.error(`${error}`);
												return null;
											});
										}

										if (!chat.pinned) {
											const res = await toggleChatPinnedStatusById(localStorage.token, chat.id);
										}

										initChatList();
									}
								}
							}}
							name={$i18n.t('Pinned')}
						>
							<div
								class="ml-3 pl-1 mt-[1px] flex flex-col overflow-y-auto scrollbar-hidden border-s border-gray-100 dark:border-gray-900"
							>
								{#each $pinnedChats as chat, idx}
									<ChatItem
										className=""
										id={chat.id}
										title={chat.title}
										{shiftKey}
										selected={selectedChatId === chat.id}
										on:select={() => {
											selectedChatId = chat.id;
										}}
										on:unselect={() => {
											selectedChatId = null;
										}}
										on:change={async () => {
											initChatList();
										}}
										on:tag={(e) => {
											const { type, name } = e.detail;
											tagEventHandler(type, name, chat.id);
										}}
									/>
								{/each}
							</div>
						</Folder>
					</div>
				{/if}

				{#if !search && folders}
					<Folders
						{folders}
						on:import={(e) => {
							const { folderId, items } = e.detail;
							importChatHandler(items, false, folderId);
						}}
						on:update={async (e) => {
							initChatList();
						}}
						on:change={async () => {
							initChatList();
						}}
					/>
				{/if}

				<div class=" flex-1 flex flex-col overflow-y-auto scrollbar-hidden">
					<div class="pt-1.5">
						{#if $chats}
							{#each $chats as chat, idx}
								{#if idx === 0 || (idx > 0 && chat.time_range !== $chats[idx - 1].time_range)}
									<div
										class="w-full pl-2.5 text-xs text-gray-500 dark:text-gray-500 font-medium {idx ===
										0
											? ''
											: 'pt-5'} pb-1.5"
									>
										{$i18n.t(chat.time_range)}
										<!-- localisation keys for time_range to be recognized from the i18next parser (so they don't get automatically removed):
							{$i18n.t('Today')}
							{$i18n.t('Yesterday')}
							{$i18n.t('Previous 7 days')}
							{$i18n.t('Previous 30 days')}
							{$i18n.t('January')}
							{$i18n.t('February')}
							{$i18n.t('March')}
							{$i18n.t('April')}
							{$i18n.t('May')}
							{$i18n.t('June')}
							{$i18n.t('July')}
							{$i18n.t('August')}
							{$i18n.t('September')}
							{$i18n.t('October')}
							{$i18n.t('November')}
							{$i18n.t('December')}
							-->
									</div>
								{/if}

								<ChatItem
									className=""
									id={chat.id}
									title={chat.title}
									{shiftKey}
									selected={selectedChatId === chat.id}
									on:select={() => {
										selectedChatId = chat.id;
									}}
									on:unselect={() => {
										selectedChatId = null;
									}}
									on:change={async () => {
										initChatList();
									}}
									on:tag={(e) => {
										const { type, name } = e.detail;
										tagEventHandler(type, name, chat.id);
									}}
								/>
							{/each}

							{#if $scrollPaginationEnabled && !allChatsLoaded}
								<Loader
									on:visible={(e) => {
										if (!chatListLoading) {
											loadMoreChats();
										}
									}}
								>
									<div
										class="w-full flex justify-center py-1 text-xs animate-pulse items-center gap-2"
									>
										<Spinner className=" size-4" />
										<div class=" ">Loading...</div>
									</div>
								</Loader>
							{/if}
						{:else}
							<div class="w-full flex justify-center py-1 text-xs animate-pulse items-center gap-2">
								<Spinner className=" size-4" />
								<div class=" ">Loading...</div>
							</div>
						{/if}
					</div>
				</div>
			</Folder>
		</div>
		<div class="px-2">
			<div class="px-1.5 flex justify-center text-gray-800 dark:text-gray-200">
				<a
					class="grow flex items-center space-x-3 rounded-lg px-2 py-[7px] transition"
					href="/applicationCenter"
					on:click={() => {
						selectedChatId = null;
						chatId.set('');

						if ($mobile) {
							showSidebar.set(false);
						}
					}}
					draggable="false"
				>
					<div class="self-center">
						<svg
							xmlns="http://www.w3.org/2000/svg"
							fill="none"
							viewBox="0 0 24 24"
							stroke-width="2"
							stroke="currentColor"
							class="size-[1.1rem]"
						>
							<path
								stroke-linecap="round"
								stroke-linejoin="round"
								d="M13.5 16.875h3.375m0 0h3.375m-3.375 0V13.5m0 3.375v3.375M6 10.5h2.25a2.25 2.25 0 0 0 2.25-2.25V6a2.25 2.25 0 0 0-2.25-2.25H6A2.25 2.25 0 0 0 3.75 6v2.25A2.25 2.25 0 0 0 6 10.5Zm0 9.75h2.25A2.25 2.25 0 0 0 10.5 18v-2.25a2.25 2.25 0 0 0-2.25-2.25H6a2.25 2.25 0 0 0-2.25 2.25V18A2.25 2.25 0 0 0 6 20.25Zm9.75-9.75H18a2.25 2.25 0 0 0 2.25-2.25V6A2.25 2.25 0 0 0 18 3.75h-2.25A2.25 2.25 0 0 0 13.5 6v2.25a2.25 2.25 0 0 0 2.25 2.25Z"
							/>
						</svg>
					</div>

					<div class="flex self-center translate-y-[0.5px]">
						<div class=" self-center font-medium font-primary">
							<!--{$i18n.t('Workspace')}-->
							我的应用
						</div>
					</div>
				</a>
			</div>
		</div>
		<div class="px-2">
			<div class="px-1.5 flex justify-center text-gray-800 dark:text-gray-200">
				<a
					class="grow flex items-center space-x-3 rounded-lg px-2 py-[7px] transition"
					href="/applicationStore"
					on:click={() => {
						selectedChatId = null;
						chatId.set('');

						if ($mobile) {
							showSidebar.set(false);
						}
					}}
					draggable="false"
				>
					<div class="self-center">
						<svg t="1742536539513" class="size-[1.1rem]" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg"><path d="M128.704 832l0-137.856c-0.128-1.216-0.64-2.24-0.64-3.52L128 545.216C128 526.72 142.4 512 160 512c17.664-0.128 30.72 22.144 30.72 40.384l0 145.472c0 2.24 0.448-3.008 0-0.896l0 71.616L768 768.576 768 619.008c0.064 0.128-0.064 0.064 0 0.128L768 545.216C768 526.72 782.656 512 800.256 512 817.92 511.872 832 526.848 832 545.152l0 145.472c0 0.448 0.064 0.832 0 1.28L832 832 128.704 832zM264.448 335.872C264.448 397.76 312.448 448 371.648 448 430.72 448 478.72 397.76 478.72 335.872 478.72 397.76 526.72 448 585.856 448c59.2 0 107.2-50.24 107.2-112.128C693.056 397.76 729.664 448 788.8 448 848 448 896 397.76 896 335.872L788.8 64 171.136 64 64 335.872C64 397.76 98.176 448 157.312 448S264.448 397.76 264.448 335.872zM896 928c0-17.664-14.336-32-32-32l-768 0C78.336 896 64 910.336 64 928l0 0C64 945.664 78.336 960 96 960l768 0C881.664 960 896 945.664 896 928L896 928z" fill="#020202" p-id="2768"></path></svg>
					</div>

					<div class="flex self-center translate-y-[0.5px]">
						<div class=" self-center font-medium font-primary">
							<!--{$i18n.t('Workspace')}-->
							应用商店
						</div>
					</div>
				</a>
			</div>
		</div>
		<div class="px-2">
			<div class="flex flex-col font-primary">
				{#if $user !== undefined}
					<UserMenu
						role={$user.role}
						on:show={(e) => {
							if (e.detail === 'archived-chat') {
								showArchivedChats.set(true);
							}
						}}
					>
						<button
							class=" flex items-center rounded-xl py-2.5 px-2.5 w-full hover:bg-gray-100 dark:hover:bg-gray-900 transition"
							on:click={() => {
								showDropdown = !showDropdown;
							}}
						>
							<div class=" self-center mr-3">
								<img
									src={$user.profile_image_url}
									class=" max-w-[30px] object-cover rounded-full"
									alt="User profile"
								/>
							</div>
							<div class=" self-center font-medium">{$user.name}</div>
						</button>
					</UserMenu>
				{/if}
			</div>
		</div>
	</div>
</div>

<style>
	.scrollbar-hidden:active::-webkit-scrollbar-thumb,
	.scrollbar-hidden:focus::-webkit-scrollbar-thumb,
	.scrollbar-hidden:hover::-webkit-scrollbar-thumb {
		visibility: visible;
	}
	.scrollbar-hidden::-webkit-scrollbar-thumb {
		visibility: hidden;
	}

	.menu {
        width: 240px;
		margin-left: 7px;
		font-family: Arial, sans-serif;
		margin-top: 4px;
	  }
  .section {
    border-radius: 8px;
    margin-bottom: 10px;
    overflow: hidden;
  }
  .header {
	display: flex;
    padding: 10px;
    font-weight: bold;
    cursor: pointer;
	color:#000;
  }
  .header .self-center{
	  padding-right:10px;
  }
  .items {
    padding: 10px 10px 10px 36px;
    background: white;
    display: flex;
    flex-direction: column;
  }
  .item {
    padding: 7px 0;
    color: black;
  }
</style>
