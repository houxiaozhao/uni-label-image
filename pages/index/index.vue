<template>
	<ImageLabel :url="url" @updateRect="updateRect" />
</template>

<script>
	import ImageLabel from "./ImageLabel.vue";
	export default {
		components: {
			ImageLabel
		},
		data() {
			return {
				url: 'https://construction-management-system.obs.cn-north-4.myhuaweicloud.com/cms/2023/10/10/2Vj7Z7tJVr.JPG?x-image-process=style/small'
			}
		},
		onLoad() {

		},
		methods: {
			updateRect(e) {
				console.log(e);
			},
		}
	}
</script>

<style>

</style>