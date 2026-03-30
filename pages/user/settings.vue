<template>
	<view class="container">
		<view class="menu-list">
			<view class="menu-item" @click="changePassword">
				<text class="menu-text">修改密码</text>
				<text class="arrow">></text>
			</view>
			<view class="menu-item">
				<text class="menu-text">关于我们</text>
				<text class="version">v1.0.0</text>
			</view>
		</view>

		<button class="logout-btn" @click="logout">退出登录</button>
	</view>
</template>

<script>
	export default {
		methods: {
			changePassword() {
				uni.showToast({ title: '跳往修改密码页', icon: 'none' });
			},
			logout() {
				uni.showModal({
					title: '提示',
					content: '确定要退出登录吗？',
					success: function (res) {
						if (res.confirm) {
							// 1. 清除本地缓存的 token
							uni.removeStorageSync('token');
							// 2. 跳转到登录页 (假设你的登录页在 pages/login/login)
							// uni.reLaunch({ url: '/pages/login/login' });
							uni.showToast({ title: '已退出', icon: 'success' });
						}
					}
				});
			}
		}
	}
</script>

<style scoped>
	.container { background-color: #F8F8F8; min-height: 100vh; padding-top: 20rpx;}
	.menu-list { margin: 20rpx; background: white; border-radius: 20rpx; padding: 0 30rpx; }
	.menu-item { display: flex; justify-content: space-between; align-items: center; padding: 30rpx 0; border-bottom: 2rpx solid #f5f5f5; }
	.menu-item:last-child { border-bottom: none; }
	.menu-text { font-size: 30rpx; color: #333; }
	.arrow { color: #ccc; font-size: 24rpx; }
	.version { font-size: 26rpx; color: #999; }
	
	.logout-btn { margin: 60rpx 40rpx; background-color: white; color: #ff4d4f; border-radius: 50rpx; font-size: 32rpx; border: 2rpx solid #ff4d4f; }
	.logout-btn::after { border: none; }
</style>