<script lang="ts">
	import { WEBUI_BASE_URL } from '$lib/constants';

	export let className = 'size-8';
	// export let src = `${WEBUI_BASE_URL}/static/favicon.png`;
	export let src1 = `/static/favicon.png`;
</script>
<img
	crossorigin="anonymous"
	src={src1 === ''
		? `/static/favicon.png`
		: src1.startsWith(WEBUI_BASE_URL) ||
			  src1.startsWith('https://www.gravatar.com/avatar/') ||
			  src1.startsWith('data:') ||
			  src1.startsWith('/')
			? src1
			: `/user.png`}
	class=" {className} object-cover rounded-full -translate-y-[1px]"
	alt="profile"
	draggable="false"
/>
