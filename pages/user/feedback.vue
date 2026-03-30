<template>
	<view class="container">
		<view class="feedback-card">
			<image class="icon" src="/static/logo.png" mode="aspectFit"></image>
			<text class="title">遇到问题了吗？</text>
			<text class="desc">为了更好地解决您的问题，请将详细情况或报错截图发送至管理员邮箱，我们会尽快回复处理。</text>
			
			<view class="email-box">
				<text class="email-text">v0v0_lukka@gmail.com</text>
				<button class="copy-btn" @click="copyEmail">一键复制邮箱</button>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		methods: {
			copyEmail() {
				uni.setClipboardData({
					data: 'v0v0_lukka@gmail.com',
					success: () => {
						uni.showToast({ title: '邮箱已复制', icon: 'success' });
					}
				});
			}
		}
	}
</script>

<style scoped>
	.container { padding: 40rpx; background-color: #F8F8F8; min-height: 100vh; display: flex; justify-content: center; align-items: flex-start; }
	.feedback-card { background: white; width: 100%; border-radius: 20rpx; padding: 60rpx 40rpx; display: flex; flex-direction: column; align-items: center; box-shadow: 0 4rpx 12rpx rgba(0,0,0,0.05); margin-top: 40rpx;}
	.icon { width: 120rpx; height: 120rpx; margin-bottom: 30rpx; opacity: 0.8;}
	.title { font-size: 36rpx; font-weight: bold; color: #333; margin-bottom: 20rpx; }
	.desc { font-size: 28rpx; color: #666; text-align: center; line-height: 1.6; margin-bottom: 60rpx; }
	.email-box { width: 100%; background: #f9f9f9; padding: 40rpx 20rpx; border-radius: 12rpx; display: flex; flex-direction: column; align-items: center;}
	.email-text { font-size: 32rpx; font-weight: bold; color: #42b983; margin-bottom: 30rpx; }
	.copy-btn { background-color: #42b983; color: white; border-radius: 50rpx; font-size: 28rpx; width: 80%; }
	.copy-btn::after { border: none; }
</style>