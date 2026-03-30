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
				<input class="input" v-model="breedNamePlaceholder" disabled placeholder="哈士奇" @click="chooseBreed"/>
			</view>
			
			<view class="form-item">
				<text class="label">性别</text>
				<view class="gender-radio-group">
					<view class="gender-option boy" :class="{active: formData.gender === 1}" @click="formData.gender = 1">♂ 公</view>
					<view class="gender-option girl" :class="{active: formData.gender === 2}" @click="formData.gender = 2">♀ 母</view>
				</view>
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
				<switch :checked="formData.isNeutered" @change="neuteredChange" color="#4FA2FE"/>
			</view>
		</view>

		<view class="btn-group">
			<button class="save-profile-btn" @click="savePet">保存档案</button>
			<button v-if="isEdit" class="delete-pet-btn" @click="deletePet">删除宠物</button>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				isEdit: false,
				breedNamePlaceholder: '哈士奇', // 预留
				formData: {
					petId: null,
					name: '',
					gender: 1, // 默认公
					birthDate: '', 
					weight: '',
					isNeutered: false,
					breedId: 1, 
					avatar: '' // 必须初始化 avatar 字段
				}
			}
		},
		onLoad(options) {
			if (options.petId) {
				this.isEdit = true;
				this.formData.petId = options.petId;
				this.loadPetDetail(options.petId);
			} else {
				// 新增时初始化默认日期（今天）
				this.formData.birthDate = this.getTodayDate();
			}
		},
		methods: {
			loadPetDetail(petId) {
				uni.request({
					url: `http://localhost:8080/pets/user/detail/${petId}`,
					method: 'GET',
					header: { 'Authorization': uni.getStorageSync('token') },
					success: (res) => {
						if (res.data.code === 200) this.formData = res.data.data;
					}
				});
			},
			
			// === 1. 选择并上传照片的方法 (全新布局触发) ===
			chooseAvatar() {
				uni.chooseImage({
					count: 1, 
					sizeType: ['compressed'], 
					success: (res) => {
						const tempFilePath = res.tempFilePaths[0];
						uni.showLoading({ title: '上传中...' });
						
						// 调用上一个回合写好的后端文件上传接口
						uni.uploadFile({
							url: 'http://localhost:8080/upload/image', // 后端专门上传文件的 Controller
							filePath: tempFilePath,
							name: 'file', // 后端接收参数名
							header: { 'Authorization': uni.getStorageSync('token') },
							success: (uploadRes) => {
								uni.hideLoading();
								let resData = JSON.parse(uploadRes.data);
								if (resData.code === 200) {
									// 将后端返回的图片 URL 保存进表单
									this.formData.avatar = resData.data;
									uni.showToast({ title: '上传成功', icon: 'success' });
								} else {
									uni.showToast({ title: '上传失败', icon: 'none' });
								}
							},
							fail: (err) => {
								uni.hideLoading();
								uni.showToast({ title: '网络错误', icon: 'none' });
							}
						});
					}
				});
			},
			
			// 2. 表单处理方法
			dateChange(e) { this.formData.birthDate = e.detail.value; },
			neuteredChange(e) { this.formData.isNeutered = e.detail.value; },
			chooseBreed() { uni.showToast({ title: '品种选择开发中', icon: 'none' }); },
			
			// 3. 提交与删除 (之前已跑通的接口逻辑)
			savePet() {
				if (!this.formData.name) return uni.showToast({ title: '请输入宠物名字', icon: 'none' });
				
				const url = this.isEdit ? 'http://localhost:8080/pets/user/update' : 'http://localhost:8080/pets/user/add';
				const method = this.isEdit ? 'PUT' : 'POST';

				uni.request({
					url: url,
					method: method,
					data: this.formData,
					header: { 'Authorization': uni.getStorageSync('token') },
					success: (res) => {
						if (res.data.code === 200) {
							uni.showToast({ title: '保存成功', icon: 'success' });
							setTimeout(() => { uni.navigateBack(); }, 1000);
						} else {
							uni.showToast({ title: res.data.msg || '保存失败', icon: 'none' });
						}
					}
				});
			},
			deletePet() {
				uni.showModal({ title: '确认删除', content: '确定要删除这个档案吗？', success: (res) => {
						if (res.confirm) {
							uni.request({
								url: `http://localhost:8080/pets/user/delete/${this.formData.petId}`,
								method: 'DELETE',
								header: { 'Authorization': uni.getStorageSync('token') },
								success: (res) => { if (res.data.code === 200) { uni.showToast({ title: '已删除', icon: 'success' }); setTimeout(() => { uni.navigateBack(); }, 1000); } }
							});
						}
					}
				});
			},
			// 辅助方法：获取今天日期字符串
			getTodayDate() { const date = new Date(); return `${date.getFullYear()}-${(date.getMonth() + 1).toString().padStart(2, '0')}-${date.getDate().toString().padStart(2, '0')}`; }
		}
	}
</script>

<style scoped lang="scss">
	/* === 全新图片上传页布局 === */
	.container { background-color: #F7F9FC; min-height: 100vh; padding-bottom: 60rpx; }
	
	/* 1. 顶部大照片上传区布局 */
	.pet-photo-upload-container {
		background-color: #F1F3F7; /* 灰色背景卡片 */
		margin: 40rpx;
		height: 400rpx;
		border-radius: 30rpx;
		border: 4rpx dashed #D0D6E0; /* 虚线框 */
		display: flex;
		justify-content: center;
		align-items: center;
		overflow: hidden; /* 预览图超出后隐藏路由 */
		position: relative;
	}
	.pet-photo-preview { width: 100%; height: 100%; border-radius: 26rpx; }
	
	.upload-guide-centered { display: flex; flex-direction: column; align-items: center; justify-content: center;}
	.plus-icon-circle { width: 100rpx; height: 100rpx; background-color: white; border-radius: 50%; color: #9AA0AF; font-size: 60rpx; display: flex; justify-content: center; align-items: center; margin-bottom: 24rpx; box-shadow: 0 4rpx 10rpx rgba(0,0,0,0.05);}
	.upload-tip { font-size: 26rpx; color: #9AA0AF; font-weight: 500;}

	/* 2. 表单区样式 */
	.form-group-card { margin: 40rpx; background: white; border-radius: 30rpx; padding: 10rpx 40rpx; box-shadow: 0 6rpx 20rpx rgba(0,0,0,0.03); }
	.form-item { display: flex; justify-content: space-between; align-items: center; padding: 36rpx 0; border-bottom: 2rpx solid #F7F9FC; }
	.form-item:last-child { border-bottom: none; }
	.label { font-size: 30rpx; color: #1A1C20; width: 180rpx; font-weight: 500;}
	.input, .picker-text { flex: 1; text-align: right; font-size: 30rpx; color: #1A1C20; }
	.picker-text { color: #4FA2FE; font-weight: bold;} /* 选中日期后变为蓝色 */

	/* 性别单选按钮样式 */
	.gender-radio-group { display: flex; align-items: center;}
	.gender-option { font-size: 26rpx; padding: 10rpx 24rpx; border-radius: 30rpx; border: 2rpx solid #F1F3F7; margin-left: 20rpx; color: #787E8C;}
	.gender-option.active.boy { background-color: #F1F7FE; color: #4FA2FE; border-color: #E2F0FD; font-weight: bold;}
	.gender-option.active.girl { background-color: #FEF1F5; color: #E85757; border-color: #FEE2EA; font-weight: bold;}
	
	/* 3. 操作按钮 */
	.btn-group { margin: 60rpx 40rpx; }
	.save-profile-btn { background-color: #4FA2FE; color: white; border-radius: 50rpx; font-size: 32rpx; font-weight: bold; margin-bottom: 30rpx; border: none;}
	.save-profile-btn::after { border: none; }
	.delete-pet-btn { background-color: white; color: #E85757; border-radius: 50rpx; font-size: 30rpx; border: 2rpx solid #FEE2EA; font-weight: bold;}
	.delete-pet-btn::after { border: none; }
</style>