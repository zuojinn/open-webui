<script lang="ts">
	import { goto } from '$app/navigation';
	import { user } from '$lib/stores';
	import { onMount } from 'svelte';

	onMount(() => {
		if ($user?.role !== 'admin') {
			if ($user?.permissions?.workspace?.models) {
				goto('/officeScene/models');
			} else if ($user?.permissions?.workspace?.knowledge) {
				goto('/officeScene/knowledge');
			} else if ($user?.permissions?.workspace?.prompts) {
				goto('/officeScene/prompts');
			} else if ($user?.permissions?.workspace?.tools) {
				goto('/officeScene/tools');
			} else {
				goto('/');
			}
		} else {
			goto('/officeScene/writeRead');
		}
	});
</script>
