<template>
	<view class="container">
		<view class="pet-photo-upload-container" @click="chooseAvatar">
			<image v-if="formData.avatar" class="pet-photo-preview" :src="formData.avatar" mode="aspectFill"></image>

			<view v-else class="upload-guide-centered">
				<view class="plus-icon-circle">+</view>
				<text class="upload-tip">添加宠物超萌美照</text>
			</view>
		</view>

		<view class="form-group-card">
			<view class="form-item">
				<text class="label">宠物名字</text>
				<input class="input" v-model="formData.name" placeholder="请输入名字 (小白)" />
			</view>

			<view class="form-item">
				<text class="label">分类/品种</text>
				<picker mode="selector" :range="breedList" range-key="breedName" @change="breedChange">
					<view class="picker-text">{{ breedNamePlaceholder || '请选择品种' }}</view>
				</picker>
			</view <view class="form-item">
			<text class="label">性别</text>
			<view class="gender-radio-group">
				<view class="gender-option boy" :class="{active: formData.gender === 1}" @click="formData.gender = 1">♂
					公</view>
				<view class="gender-option girl" :class="{active: formData.gender === 2}" @click="formData.gender = 2">♀
					母</view>
			</view>
			<view class="form-item">
				<text class="label">出生日期</text>
				<picker mode="date" :value="formData.birthDate" @change="dateChange">
					<view class="picker-text">{{ formData.birthDate || '请选择日期' }}</view>
				</picker>
			</view>

			<view class="form-item">
				<text class="label">体重 (kg)</text>
				<input class="input" type="digit" v-model="formData.weight" placeholder="5.0" />
			</view>

			<view class="form-item">
				<text class="label">是否绝育</text>
				<switch :checked="formData.isNeutered" @change="neuteredChange" color="#4FA2FE" />
			</view>
		</view>
	</view>
	<view class="btn-group">
		<button class="save-profile-btn" @click="savePet">保存档案</button>
		<button v-if="isEdit" class="delete-pet-btn" @click="deletePet">删除宠物</button>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				isEdit: false,
				breedList: [], // 👈 新增：存放从后端请求回来的品种列表数组
				breedNamePlaceholder: '', // 👈 修改：用来在页面上显示当前选中的品种中文名
				formData: {
					petId: null,
					name: '',
					gender: 1,
					birthDate: '',
					weight: '',
					isNeutered: false,
					breedId: null, // 👈 修改：默认置空，等用户选择
					avatar: ''
				}
			}
		},
		onLoad(options) {
			this.loadBreeds(); // 👈 进页面第一件事就是去后台拉取品种列表

			if (options.petId) {
				this.isEdit = true;
				this.formData.petId = options.petId;
				this.loadPetDetail(options.petId);
			} else {
				this.formData.birthDate = this.getTodayDate();
			}
		},
		methods: {
			// 👈 2. 新增：向后端请求品种列表
			loadBreeds() {
				uni.request({
					url: 'http://localhost:8080/pet-breeds/list',
					method: 'GET',
					header: {
						'token': uni.getStorageSync('token')
					},
					success: (res) => {
						if (res.data.code === 200) {
							this.breedList = res.data.data; // 赋值给本地数组
						}
					}
				});
			},

			// 👈 3. 新增：用户在底部滚动选择完品种后的处理逻辑
			breedChange(e) {
				// e.detail.value 是用户选中的项在数组中的索引 (index)
				const index = e.detail.value;
				const selectedBreed = this.breedList[index];

				// 把后台需要的 breedId 存进表单
				this.formData.breedId = selectedBreed.breedId;
				// 把页面上要显示的中文名更新一下
				this.breedNamePlaceholder = selectedBreed.breedName;
			},

			// === 2. 选择并上传照片 (关键修复：升级为 uniCloud 阿里云上传) ===
			async chooseAvatar() {
				uni.chooseImage({
					count: 1,
					sizeType: ['compressed'],
					success: async (res) => {
						const tempFilePath = res.tempFilePaths[0];
						uni.showLoading({
							title: '传至云端中...'
						});

						try {
							// 提取文件后缀并生成随机云端文件名
							const extension = tempFilePath.substring(tempFilePath.lastIndexOf('.'));
							const cloudFileName = 'pet_' + Date.now() + extension;

							// 直接调用 uniCloud 上传图片！
							const uploadRes = await uniCloud.uploadFile({
								filePath: tempFilePath,
								cloudPath: cloudFileName
							});

							console.log('宠物照片云端链接：', uploadRes.fileID);
							this.formData.avatar = uploadRes.fileID;

							uni.showToast({
								title: '上传成功',
								icon: 'success'
							});

						} catch (error) {
							console.error('uniCloud 上传报错：', error);
							uni.showToast({
								title: '上传失败，请检查网络',
								icon: 'none'
							});
						} finally {
							uni.hideLoading();
						}
					}
				});
			},

			// 3. 表单处理方法
			dateChange(e) {
				this.formData.birthDate = e.detail.value;
			},
			neuteredChange(e) {
				this.formData.isNeutered = e.detail.value;
			},
			chooseBreed() {
				uni.showToast({
					title: '品种选择开发中',
					icon: 'none'
				});
			},

			// 4. 提交宠物档案 (修改了 header 中的 token)
			savePet() {
				if (!this.formData.name) return uni.showToast({
					title: '请输入宠物名字',
					icon: 'none'
				});

				const url = this.isEdit ? 'http://localhost:8080/pets/user/update' : 'http://localhost:8080/pets/user/add';
				const method = this.isEdit ? 'PUT' : 'POST';

				uni.request({
					url: url,
					method: method,
					data: this.formData,
					header: {
						'token': uni.getStorageSync('token')
					}, // 👈 关键修复
					success: (res) => {
						if (res.data.code === 200) {
							uni.showToast({
								title: '保存成功',
								icon: 'success'
							});
							setTimeout(() => {
								uni.navigateBack();
							}, 1000);
						} else {
							uni.showToast({
								title: res.data.msg || '保存失败',
								icon: 'none'
							});
						}
					}
				});
			},

			// 5. 删除宠物 (修改了 header 中的 token)
			deletePet() {
				uni.showModal({
					title: '确认删除',
					content: '确定要删除这个档案吗？',
					success: (res) => {
						if (res.confirm) {
							uni.request({
								url: `http://localhost:8080/pets/user/delete/${this.formData.petId}`,
								method: 'DELETE',
								header: {
									'token': uni.getStorageSync('token')
								}, // 👈 关键修复
								success: (res) => {
									if (res.data.code === 200) {
										uni.showToast({
											title: '已删除',
											icon: 'success'
										});
										setTimeout(() => {
											uni.navigateBack();
										}, 1000);
									}
								}
							});
						}
					}
				});
			},

			// 辅助方法：获取今天日期字符串
			getTodayDate() {
				const date = new Date();
				return `${date.getFullYear()}-${(date.getMonth() + 1).toString().padStart(2, '0')}-${date.getDate().toString().padStart(2, '0')}`;
			}
		}
	}
</script>

<style scoped lang="scss">
	/* === 全新图片上传页布局 === */
	.container {
		background-color: #F7F9FC;
		min-height: 100vh;
		padding-bottom: 60rpx;
	}

	/* 1. 顶部大照片上传区布局 */
	.pet-photo-upload-container {
		background-color: #F1F3F7;
		/* 灰色背景卡片 */
		margin: 40rpx;
		height: 400rpx;
		border-radius: 30rpx;
		border: 4rpx dashed #D0D6E0;
		/* 虚线框 */
		display: flex;
		justify-content: center;
		align-items: center;
		overflow: hidden;
		/* 预览图超出后隐藏路由 */
		position: relative;
	}

	.pet-photo-preview {
		width: 100%;
		height: 100%;
		border-radius: 26rpx;
	}

	.upload-guide-centered {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}

	.plus-icon-circle {
		width: 100rpx;
		height: 100rpx;
		background-color: white;
		border-radius: 50%;
		color: #9AA0AF;
		font-size: 60rpx;
		display: flex;
		justify-content: center;
		align-items: center;
		margin-bottom: 24rpx;
		box-shadow: 0 4rpx 10rpx rgba(0, 0, 0, 0.05);
	}

	.upload-tip {
		font-size: 26rpx;
		color: #9AA0AF;
		font-weight: 500;
	}

	/* 2. 表单区样式 */
	.form-group-card {
		margin: 40rpx;
		background: white;
		border-radius: 30rpx;
		padding: 10rpx 40rpx;
		box-shadow: 0 6rpx 20rpx rgba(0, 0, 0, 0.03);
	}

	.form-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 36rpx 0;
		border-bottom: 2rpx solid #F7F9FC;
	}

	.form-item:last-child {
		border-bottom: none;
	}

	.label {
		font-size: 30rpx;
		color: #1A1C20;
		width: 180rpx;
		font-weight: 500;
	}

	.input,
	.picker-text {
		flex: 1;
		text-align: right;
		font-size: 30rpx;
		color: #1A1C20;
	}

	.picker-text {
		color: #4FA2FE;
		font-weight: bold;
	}

	/* 选中日期后变为蓝色 */

	/* 性别单选按钮样式 */
	.gender-radio-group {
		display: flex;
		align-items: center;
	}

	.gender-option {
		font-size: 26rpx;
		padding: 10rpx 24rpx;
		border-radius: 30rpx;
		border: 2rpx solid #F1F3F7;
		margin-left: 20rpx;
		color: #787E8C;
	}

	.gender-option.active.boy {
		background-color: #F1F7FE;
		color: #4FA2FE;
		border-color: #E2F0FD;
		font-weight: bold;
	}

	.gender-option.active.girl {
		background-color: #FEF1F5;
		color: #E85757;
		border-color: #FEE2EA;
		font-weight: bold;
	}

	/* 3. 操作按钮 */
	.btn-group {
		margin: 60rpx 40rpx;
	}

	.save-profile-btn {
		background-color: #4FA2FE;
		color: white;
		border-radius: 50rpx;
		font-size: 32rpx;
		font-weight: bold;
		margin-bottom: 30rpx;
		border: none;
	}

	.save-profile-btn::after {
		border: none;
	}

	.delete-pet-btn {
		background-color: white;
		color: #E85757;
		border-radius: 50rpx;
		font-size: 30rpx;
		border: 2rpx solid #FEE2EA;
		font-weight: bold;
	}

	.delete-pet-btn::after {
		border: none;
	}
</style>