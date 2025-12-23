<template>
	<div class="common-layout">
		<el-container>
			<el-header>
				<Icon class="icon" icon="prime:discord" />
				<span class="title"> Discord Emoticon & Sticker Downloader </span>
			</el-header>
			<el-main>
				<index />
			</el-main>
		</el-container>
	</div>
</template>

<script setup>
import { Icon } from "@iconify/vue";
import index from "./components/download.vue"
</script>

<style lang="scss" scoped>
@import "./assets/handle.scss";

.el-header {
	height: 64px;
	padding: 0 64px;
	border-bottom: 2px dashed var(--el-border-color);
	@include flex-center($direction: row, $content: start);

	.icon {
		font-size: 2rem;
		color: $themeColor;
	}
	.title {
		margin-left: 24px;
		font-size: 1.1rem;
		font-weight: bold;
	}
}
.el-main {
	height: calc(100vh - 64px);
}

@media (max-width: 900px) {
	.el-header {
		padding: 0 20px;
	}
}
</style>
