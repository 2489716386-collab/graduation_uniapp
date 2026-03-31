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
				<input class="input" v-model="formData.name" placeholder="请输入名字" />
			</view>
			
			<view class="form-item">
				<text class="label">宠物种类</text>
				<picker mode="selector" :range="speciesList" @change="speciesChange">
					<view class="picker-text">{{ currentSpecies || '请选择种类' }}</view>
				</picker>
			</view>

			<view class="form-item">
				<text class="label">具体品种</text>
				<picker mode="selector" :range="filteredBreeds" range-key="breedName" @change="breedChange" :disabled="!currentSpecies">
					<view class="picker-text" :class="{'disabled-text': !currentSpecies}">{{ breedNamePlaceholder || '请先选择种类' }}</view>
				</picker>
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
				<picker mode="date" :value="formData.birthDate" start="2000-01-01" :end="getTodayDate()" @change="dateChange">
					<view class="picker-text">{{ formData.birthDate || '请选择出生日期' }}</view>
				</picker>
			</view>

			<view class="form-item">
				<text class="label">体重 (kg)</text>
				<input class="input" type="digit" v-model="formData.weight" placeholder="请输入体重" />
			</view>

			<view class="form-item">
				<text class="label">是否绝育</text>
				<switch :checked="formData.isNeutered" @change="neuteredChange" color="#4FA2FE" style="transform:scale(0.8)"/>
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
				allBreeds: [],       // 后端原始全量数据
				speciesList: [],     // 种类数组 ['猫', '狗']
				currentSpecies: '',  // 当前选中的种类名
				filteredBreeds: [],  // 过滤后的品种数组
				breedNamePlaceholder: '', 
				formData: {
					petId: null, name: '', gender: 1, birthDate: '', 
					weight: '', isNeutered: false, breedId: null, avatar: '' 
				}
			}
		},
		async onLoad(options) {
			await this.loadBreeds();
			
			if (options.petId) {
				this.isEdit = true;
				this.formData.petId = options.petId;
				this.loadPetDetail(options.petId);
			} else {
				this.formData.birthDate = this.getTodayDate();
			}
		},
		methods: {
			// 获取全量品种并提取种类
			loadBreeds() {
				return new Promise((resolve) => {
					uni.request({
						url: 'http://localhost:8080/pet-breeds/list',
						method: 'GET',
						header: { 'token': uni.getStorageSync('token') },
						success: (res) => {
							if (res.data.code === 200) {
								this.allBreeds = res.data.data;
								this.speciesList = [...new Set(this.allBreeds.map(item => item.speciesType))];
							}
							resolve();
						}
					});
				});
			},

			// 种类切换逻辑
			speciesChange(e) {
				const index = e.detail.value;
				this.currentSpecies = this.speciesList[index];
				this.filteredBreeds = this.allBreeds.filter(item => item.speciesType === this.currentSpecies);
				this.formData.breedId = null;
				this.breedNamePlaceholder = '请选择品种';
			},

			// 品种切换逻辑
			breedChange(e) {
				const index = e.detail.value;
				const selected = this.filteredBreeds[index];
				this.formData.breedId = selected.breedId;
				this.breedNamePlaceholder = selected.breedName;
			},

			// 详情回显逻辑
			loadPetDetail(petId) {
				uni.request({
					url: `http://localhost:8080/pets/user/detail/${petId}`,
					method: 'GET',
					header: { 'token': uni.getStorageSync('token') },
					success: (res) => {
						if (res.data.code === 200) {
							this.formData = res.data.data;
							const breedItem = this.allBreeds.find(b => b.breedId === this.formData.breedId);
							if (breedItem) {
								this.currentSpecies = breedItem.speciesType;
								this.breedNamePlaceholder = breedItem.breedName;
								this.filteredBreeds = this.allBreeds.filter(item => item.speciesType === this.currentSpecies);
							}
						}
					}
				});
			},
			
			// 云端上传照片
			async chooseAvatar() {
				uni.chooseImage({
					count: 1, 
					sizeType: ['compressed'], 
					success: async (res) => {
						const tempFilePath = res.tempFilePaths[0];
						uni.showLoading({ title: '传至云端中...' });
						try {
							const extension = tempFilePath.substring(tempFilePath.lastIndexOf('.'));
							const cloudFileName = 'pet_' + Date.now() + extension;
							const uploadRes = await uniCloud.uploadFile({
								filePath: tempFilePath,
								cloudPath: cloudFileName
							});
							this.formData.avatar = uploadRes.fileID; 
							uni.showToast({ title: '上传成功', icon: 'success' });
						} catch (error) {
							console.error('uniCloud 上传报错：', error);
							uni.showToast({ title: '上传失败', icon: 'none' });
						} finally {
							uni.hideLoading();
						}
					}
				});
			},

			// 表单组件事件
			dateChange(e) { this.formData.birthDate = e.detail.value; },
			neuteredChange(e) { this.formData.isNeutered = e.detail.value; },
			
			// 保存档案
			savePet() {
				if (!this.formData.name) return uni.showToast({ title: '请输入宠物名字', icon: 'none' });
				
				const url = this.isEdit ? 'http://localhost:8080/pets/user/update' : 'http://localhost:8080/pets/user/add';
				const method = this.isEdit ? 'PUT' : 'POST';

				uni.request({
					url: url,
					method: method,
					data: this.formData,
					header: { 'token': uni.getStorageSync('token') },
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

			// 删除档案
			deletePet() {
				uni.showModal({ 
					title: '确认删除', 
					content: '确定要删除这个档案吗？', 
					success: (res) => {
						if (res.confirm) {
							uni.request({
								url: `http://localhost:8080/pets/user/delete/${this.formData.petId}`,
								method: 'DELETE',
								header: { 'token': uni.getStorageSync('token') },
								success: (res) => { 
									if (res.data.code === 200) { 
										uni.showToast({ title: '已删除', icon: 'success' }); 
										setTimeout(() => { uni.navigateBack(); }, 1000); 
									} 
								}
							});
						}
					}
				});
			},
			
			getTodayDate() { const date = new Date(); return `${date.getFullYear()}-${(date.getMonth() + 1).toString().padStart(2, '0')}-${date.getDate().toString().padStart(2, '0')}`; }
		}
	}
</script>

<style scoped lang="scss">
	.container { background-color: #F7F9FC; min-height: 100vh; padding: 30rpx 40rpx 100rpx;}
	
	.pet-photo-upload-container { display: flex; justify-content: center; margin-bottom: 40rpx; }
	.pet-photo-preview { width: 200rpx; height: 200rpx; border-radius: 40rpx; box-shadow: 0 10rpx 30rpx rgba(0,0,0,0.08); border: 6rpx solid white; }
	.upload-guide-centered { width: 200rpx; height: 200rpx; border-radius: 40rpx; background: white; border: 4rpx dashed #DCE3EE; display: flex; flex-direction: column; justify-content: center; align-items: center; }
	.plus-icon-circle { font-size: 60rpx; color: #4FA2FE; font-weight: 300; margin-bottom: 10rpx; line-height: 1;}
	.upload-tip { font-size: 24rpx; color: #9AA0AF; }

	.form-group-card { background: white; border-radius: 30rpx; padding: 0 40rpx; box-shadow: 0 6rpx 20rpx rgba(0,0,0,0.03); }
	.form-item { display: flex; justify-content: space-between; align-items: center; padding: 36rpx 0; border-bottom: 2rpx solid #F1F3F7; }
	.form-item:last-child { border-bottom: none; }
	.label { font-size: 30rpx; color: #1A1C20; font-weight: bold; width: 160rpx; }
	.input { flex: 1; text-align: right; font-size: 30rpx; color: #1A1C20; }
	.picker-text { flex: 1; text-align: right; font-size: 30rpx; color: #1A1C20; min-width: 200rpx;}
	.disabled-text { color: #9AA0AF; }
	
	.gender-radio-group { display: flex; gap: 20rpx; }
	.gender-option { padding: 10rpx 30rpx; border-radius: 50rpx; font-size: 28rpx; color: #787E8C; background: #F7F9FC; border: 2rpx solid transparent; transition: all 0.2s;}
	.gender-option.active.boy { background: #F1F7FE; color: #4FA2FE; border-color: #4FA2FE; font-weight: bold;}
	.gender-option.active.girl { background: #FEF1F4; color: #FF6B8B; border-color: #FF6B8B; font-weight: bold;}

	.btn-group { margin-top: 60rpx; }
	.save-profile-btn { background: #4FA2FE; color: white; border-radius: 50rpx; font-size: 32rpx; font-weight: bold; height: 90rpx; line-height: 90rpx; box-shadow: 0 10rpx 30rpx rgba(79,162,254,0.3); margin-bottom: 30rpx;}
	.save-profile-btn::after { border: none; }
	.delete-pet-btn { background: #FFF4F4; color: #E85757; border-radius: 50rpx; font-size: 32rpx; font-weight: bold; height: 90rpx; line-height: 90rpx; border: 2rpx solid #FFE0E0;}
	.delete-pet-btn::after { border: none; }
</style>