<template>
	<view class="content">
		<text class="title">🐾 欢迎来到宠物健康管家</text>
		<text class="subtitle">这里将展示宠物心情监测和今日AI养护概览</text>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				title: '首页'
			}
		},
		onLoad() {
		},
		methods: {
		}
	}
</script>

<style>
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		padding-top: 100rpx;
	}
	.title {
		font-size: 40rpx;
		font-weight: bold;
		color: #42b983; /* 使用应用的主色调绿色 */
		margin-bottom: 20rpx;
	}
	.subtitle {
		font-size: 28rpx;
		color: #666;
	}
</style>