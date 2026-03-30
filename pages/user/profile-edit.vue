<template>
	<view class="container">
		<view class="avatar-section" @click="changeAvatar">
			<image class="avatar" :src="formData.avatarUrl || '/static/default-avatar.png'" mode="aspectFill"></image>
			<view class="avatar-hint">
				<text>点击更换头像</text>
				<text class="arrow">&gt;</text>
			</view>
		</view>

		<view class="form-section">
			<view class="form-item">
				<text class="label">昵称</text>
				<input class="input" type="text" v-model="formData.nickname" placeholder="请输入昵称(最多20个字)" maxlength="20" />
			</view>
			
			<view class="form-item bio-item">
				<text class="label">个性签名</text>
				<textarea class="textarea" v-model="formData.bio" placeholder="介绍一下你自己和你的宠物吧..." maxlength="100"></textarea>
				<text class="word-count">{{ formData.bio ? formData.bio.length : 0 }}/100</text>
			</view>
		</view>

		<view class="btn-section">
			<button class="save-btn" @click="saveProfile">保存修改</button>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				formData: {
					nickname: '',
					avatarUrl: '',
					bio: ''
				},
				baseUrl: 'http://localhost:8080' // 请替换为你的实际后端基础地址
			}
		},
		onLoad() {
			this.fetchUserProfile();
		},
		methods: {
			// 获取当前用户信息
			fetchUserProfile() {
				uni.request({
					url: `${this.baseUrl}/users/user/profile`,
					method: 'GET',
					header: { 'token': uni.getStorageSync('token') },
					success: (res) => {
						if (res.data.code === 1) { // 假设 Result.success 返回 code 为 1
							const user = res.data.data;
							this.formData.nickname = user.nickname || '';
							this.formData.avatarUrl = user.avatarUrl || '';
							this.formData.bio = user.bio || '';
						}
					}
				});
			},
			// 更换头像
			changeAvatar() {
				uni.chooseImage({
					count: 1,
					success: (chooseImageRes) => {
						const tempFilePaths = chooseImageRes.tempFilePaths;
						uni.showLoading({ title: '上传中...' });
						
						// 注意：这里需要调用你后端的图片上传接口（例如 /upload）
						// 如果你还没有写上传接口，这里需要后续对接
						uni.uploadFile({
							url: `${this.baseUrl}/upload/image`, // 替换为你的上传接口
							filePath: tempFilePaths[0],
							name: 'file', // 后端接收文件的参数名
							header: { 'token': uni.getStorageSync('token') },
							success: (uploadFileRes) => {
								const res = JSON.parse(uploadFileRes.data);
								if (res.code === 1) {
									// 假设上传成功后返回的是图片的网络地址
									this.formData.avatarUrl = res.data; 
									uni.showToast({ title: '上传成功', icon: 'success' });
								} else {
									uni.showToast({ title: '上传失败', icon: 'none' });
								}
							},
							complete: () => {
								uni.hideLoading();
							}
						});
					}
				});
			},
			// 保存资料
			saveProfile() {
				if (!this.formData.nickname.trim()) {
					return uni.showToast({ title: '昵称不能为空', icon: 'none' });
				}
				
				uni.showLoading({ title: '保存中...' });
				uni.request({
					url: `${this.baseUrl}/users/user/update`,
					method: 'PUT',
					header: { 
					    'token': uni.getStorageSync('token'), 
					    'Content-Type': 'application/json'
					},
					data: this.formData,
					success: (res) => {
						if (res.data.code === 1) {
							uni.showToast({ title: '保存成功', icon: 'success' });
							setTimeout(() => {
								uni.navigateBack(); // 保存成功后返回上一页
							}, 1000);
						} else {
							uni.showToast({ title: res.data.msg || '保存失败', icon: 'none' });
						}
					},
					complete: () => {
						uni.hideLoading();
					}
				});
			}
		}
	}
</script>

<style scoped lang="scss">
	.container { background-color: #F7F9FC; min-height: 100vh; padding-top: 20rpx; }
	
	.avatar-section { background: #FFFFFF; display: flex; justify-content: space-between; align-items: center; padding: 40rpx; margin-bottom: 20rpx; }
	.avatar { width: 120rpx; height: 120rpx; border-radius: 50%; background-color: #eee; }
	.avatar-hint { display: flex; align-items: center; color: #999; font-size: 28rpx; }
	.arrow { margin-left: 10rpx; font-family: consolas; }
	
	.form-section { background: #FFFFFF; padding: 0 40rpx; }
	.form-item { display: flex; align-items: center; padding: 30rpx 0; border-bottom: 2rpx solid #F1F3F7; }
	.bio-item { align-items: flex-start; flex-direction: column; border-bottom: none; position: relative; padding-bottom: 60rpx;}
	.label { width: 160rpx; font-size: 30rpx; color: #333; font-weight: 500; }
	.input { flex: 1; font-size: 30rpx; color: #333; text-align: right; }
	
	.textarea { width: 100%; height: 160rpx; margin-top: 20rpx; padding: 20rpx; background: #F7F9FC; border-radius: 16rpx; font-size: 28rpx; box-sizing: border-box; color: #333;}
	.word-count { position: absolute; right: 10rpx; bottom: 10rpx; font-size: 24rpx; color: #999; }
	
	.btn-section { margin-top: 80rpx; padding: 0 40rpx; }
	.save-btn { background: #4FA2FE; color: #FFFFFF; border-radius: 50rpx; font-size: 32rpx; font-weight: bold; box-shadow: 0 8rpx 20rpx rgba(79, 162, 254, 0.3); }
	.save-btn::after { border: none; }
</style>