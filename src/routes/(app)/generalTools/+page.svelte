<script lang="ts">
	import { goto } from '$app/navigation';
	import { user } from '$lib/stores';
	import { onMount } from 'svelte';

	onMount(() => {
		if ($user?.role !== 'admin') {
			if ($user?.permissions?.workspace?.models) {
				goto('/generalTools/models');
			} else if ($user?.permissions?.workspace?.knowledge) {
				goto('/generalTools/knowledge');
			} else if ($user?.permissions?.workspace?.prompts) {
				goto('/generalTools/prompts');
			} else if ($user?.permissions?.workspace?.tools) {
				goto('/generalTools/tools');
			} else {
				goto('/');
			}
		} else {
			goto('/generalTools/writeRead');
		}
	});
</script>
