<template>
	<view class="container">
		<view class="menu-list">
			<view class="menu-item">
				<text class="menu-text">当前版本</text>
				<text class="version">v1.0.0</text>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {}
		}
	}
</script>

<style scoped>
	.container { 
		background-color: #F8F8F8; 
		min-height: 100vh; 
		padding-top: 20rpx;
	}
	
	.menu-list { 
		margin: 20rpx; 
		background: white; 
		border-radius: 20rpx; 
		padding: 0 30rpx; 
	}
	
	.menu-item { 
		display: flex; 
		justify-content: space-between; 
		align-items: center; 
		padding: 36rpx 0; 
	}
	
	.menu-text { 
		font-size: 30rpx; 
		color: #333; 
	}
	
	.version { 
		font-size: 30rpx; 
		color: #999; 
	}
</style>