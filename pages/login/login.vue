<template>
	<view class="container">
		<view class="header">
			<image class="logo" src="/static/logo.png" mode="aspectFit"></image>
			<text class="title">宠物健康管理</text>
			<text class="subtitle">科学养宠，陪伴你的每一天</text>
		</view>

		<view class="login-section">
			<button class="wx-login-btn" @click="handleWxLogin">
				<text class="btn-text">微信一键登录</text>
			</button>

			<button class="cancel-btn" @click="goBack">暂不登录，先看看</button>
		</view>

		<view class="agreement-box">
			<label class="checkbox-label" @click="isAgreed = !isAgreed">
				<radio value="1" :checked="isAgreed" color="#42b983" style="transform:scale(0.7)" />
				<text class="agreement-text">我已阅读并同意</text>
			</label>
			<text class="link">《用户协议》</text>和<text class="link">《隐私政策》</text>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				isAgreed: false // 默认不勾选，防合规风险
			}
		},
		methods: {
			handleWxLogin() {
				if (!this.isAgreed) {
					return uni.showToast({
						title: '请先勾选同意隐私政策',
						icon: 'none'
					});
				}

				uni.showLoading({
					title: '登录中...'
				});

				uni.login({
					provider: 'weixin',
					success: (loginRes) => {
						if (loginRes.code) {
							console.log('1. 获取微信临时code成功:', loginRes.code);
							this.sendCodeToBackend(loginRes.code);
						} else {
							uni.hideLoading();
							console.error('获取code失败:', loginRes);
							uni.showToast({
								title: '微信登录失败',
								icon: 'none'
							});
						}
					},
					fail: (err) => {
						uni.hideLoading();
						// 如果在浏览器里点，就会打印这个报错
						console.error('调用微信原生登录失败(请检查是否在微信环境下运行):', err);
						uni.showToast({
							title: '请在微信小程序中打开',
							icon: 'none'
						});
					}
				});
			},

			sendCodeToBackend(code) {
				// 注意：这里去掉了 nickname 和 avatarUrl，因为最新版微信规则不再允许一键获取头像昵称，需要用户后续自己在个人中心填写
				uni.request({
					url: 'http://localhost:8080/auth/wx-login', // 对应你 LoginController 里的接口
					method: 'POST',
					data: {
						code: code
					},
					success: (res) => {
						uni.hideLoading();
						if (res.data.code === 200) {
							// 登录成功，保存后端给的 Token
							uni.setStorageSync('token', res.data.data.token);
							uni.showToast({
								title: '登录成功',
								icon: 'success'
							});

							// 跳转回首页
							setTimeout(() => {
								uni.switchTab({
									url: '/pages/index/index'
								});
							}, 1000);
						} else {
							uni.showToast({
								title: res.data.msg || '登录失败',
								icon: 'none'
							});
						}
					},
					fail: () => {
						uni.hideLoading();
						uni.showToast({
							title: '网络连接失败',
							icon: 'none'
						});
					}
				});
			},

			goBack() {
				uni.switchTab({
					url: '/pages/index/index'
				});
			}
		}
	}
</script>

<style scoped>
	.container {
		background-color: #FFFFFF;
		min-height: 100vh;
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 0 60rpx;
	}

	.header {
		margin-top: 200rpx;
		display: flex;
		flex-direction: column;
		align-items: center;
		width: 100%;
	}

	.logo {
		width: 200rpx;
		height: 200rpx;
		margin-bottom: 40rpx;
		border-radius: 40rpx;
		box-shadow: 0 10rpx 30rpx rgba(66, 185, 131, 0.2);
	}

	.title {
		font-size: 48rpx;
		font-weight: bold;
		color: #333;
		margin-bottom: 16rpx;
	}

	.subtitle {
		font-size: 28rpx;
		color: #999;
	}

	.login-section {
		width: 100%;
		margin-top: 160rpx;
	}

	.wx-login-btn {
		background-color: #07C160;
		color: white;
		border-radius: 50rpx;
		height: 96rpx;
		display: flex;
		justify-content: center;
		align-items: center;
		font-size: 34rpx;
		font-weight: bold;
		box-shadow: 0 10rpx 30rpx rgba(7, 193, 96, 0.3);
	}

	.wx-login-btn::after {
		border: none;
	}

	.cancel-btn {
		margin-top: 30rpx;
		background-color: transparent;
		color: #999;
		font-size: 30rpx;
		border: none;
	}

	.cancel-btn::after {
		border: none;
	}

	.agreement-box {
		position: absolute;
		bottom: 100rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 24rpx;
		width: 100%;
		left: 0;
	}

	.checkbox-label {
		display: flex;
		align-items: center;
	}

	.agreement-text {
		color: #999;
	}

	.link {
		color: #42b983;
	}
</style>