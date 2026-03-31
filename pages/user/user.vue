<template>
	<view class="container">
		<view class="user-header">
			<view class="header-action-btn" @click="editProfile">
				<text class="arrow">❯</text>
			</view>

			<view class="user-profile-centered">
				<image class="avatar" :src="userInfo.avatarUrl || '/static/default-avatar.png'" mode="aspectFill">
				</image>
				<text class="nickname">{{ userInfo.nickname || '未设置昵称' }}</text>
				<text class="bio">{{ userInfo.bio || '这个人很懒，什么都没写~' }}</text>
			</view>
		</view>

		<view class="stats-board-wrapper">
			<view class="stats-board">
				<view class="stat-item" @click="goToMyPosts">
					<text class="stat-num">{{ stats.postCount || 0 }}</text>
					<text class="stat-label">动态</text>
				</view>
				<view class="stat-divider"></view>
				<view class="stat-divider"></view>
				<view class="stat-item" @click="goToDynamicNotices">
					<text class="stat-num">{{ stats.likeCount || 0 }}</text>
					<text class="stat-label">获赞</text>
				</view>
			</view>
		</view>

		<view class="card-section">
			<view class="section-header">
				<text class="title">🐾 我的宠物</text>
				<view class="add-pet-btn" @click="goToPetEdit()">添加新宠物</view>
			</view>

			<view class="pet-list" v-if="petList.length > 0">
				<view class="pet-list-item" v-for="pet in petList" :key="pet.petId" @click="goToPetEdit(pet.petId)">
					<image class="pet-icon-square" :src="pet.avatar || '/static/default-avatar.png'" mode="aspectFill">
					</image>

					<view class="pet-item-info">
						<text class="pet-name">{{ pet.name }}</text>
						<text class="pet-breed">{{ pet.breedName || '普通品种' }}</text>

						<view class="pet-details">
							<text>{{ calculateAge(pet.birthDate) }}</text>
							<text class="dot">·</text>
							<text>{{ pet.birthDate || '未知生日' }}</text>
							<text class="dot">·</text>
							<text>{{ pet.weight ? pet.weight + 'kg' : '暂无体重' }}</text>
						</view>
					</view>

					<view class="view-btn">编辑 ❯</view>
				</view>
			</view>

			<view class="empty-state" v-else>
				<text>还没有添加宠物哦，快去添加吧！</text>
			</view>
		</view>

		<view class="card-section menu-section">
			<view class="menu-item" @click="goToSystemNotices">
				<text class="menu-text">系统通知</text>
				<text class="arrow">❯</text>
			</view>
			<view class="menu-item" @click="goToFeedback">
				<text class="menu-text">问题反馈</text>
				<text class="arrow">❯</text>
			</view>
			<view class="menu-item" @click="goToSettings">
				<text class="menu-text">关于我们</text>
				<text class="arrow">❯</text>
			</view>
			<view class="menu-item logout-item" @click="logout">
				<text class="menu-text logout-text">退出登录</text>
				<text class="arrow red">❯</text>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				userInfo: {},
				petList: [],
				allBreeds: [], // 👈 新增：用来存放所有的品种字典
				stats: {
					postCount: 0,
					likeCount: 0
				}
			}
		},
		// 👈 修改：改成 async 异步，确保先拿到字典，再渲染宠物列表
		async onShow() {
			this.getUserProfile();
			await this.loadBreeds(); // 第一步：拉取品种字典
			this.getPetList(); // 第二步：拉取宠物列表并匹配名字
			this.getUserStats();
		},
		methods: {
			getUserProfile() {
				// 1. 先尝试从本地缓存获取 token
				const token = uni.getStorageSync('token');

				// 2. 如果没有 token，说明还没登录，直接终止函数，不去打扰后端
				if (!token) {
					console.log('用户未登录，不获取个人信息');
					return;
				}

				// 3. 只有存在 token 时，才向后端发起请求
				uni.request({
					url: 'http://localhost:8080/users/user/profile',
					method: 'GET',
					header: {
						'token': token // 使用刚刚获取到的 token
					},
					success: (res) => {
						// 注意检查你的后端 Result 成功状态码到底是 1 还是 200
						if (res.data.code === 1 || res.data.code === 200) {
							this.userInfo = res.data.data;
						}
					}
				});
			},
			// ================= 新增：获取品种字典 =================
			loadBreeds() {
				return new Promise((resolve) => {
					uni.request({
						url: 'http://localhost:8080/pet-breeds/list',
						method: 'GET',
						header: {
							'token': uni.getStorageSync('token')
						},
						success: (res) => {
							if (res.data.code === 200) {
								this.allBreeds = res.data.data;
							}
							resolve(); // 请求完毕放行
						},
						fail: () => resolve()
					});
				});
			},

			// ================= 修改：获取宠物列表并匹配 =================
			getPetList() {
				const token = uni.getStorageSync('token');
				if (!token) return;

				uni.request({
					url: 'http://localhost:8080/pets/user/list',
					method: 'GET',
					header: {
						'token': token
					},
					success: (res) => {
						if (res.data.code === 200) {
							const rawPets = res.data.data;

							// 【核心魔法】遍历后端返回的宠物列表，拿 breedId 去字典里找名字
							this.petList = rawPets.map(pet => {
								const breedObj = this.allBreeds.find(b => b.breedId === pet.breedId);
								return {
									...pet,
									// 如果找到了就用真正的名字，如果没找到就兜底显示'未知品种'
									breedName: breedObj ? breedObj.breedName : '未知品种'
								};
							});
						}
					}
				});
			},
			getUserStats() {
				this.stats = {
					postCount: 12,
					likeCount: 128
				};
			},
			calculateAge(birthDate) {
				if (!birthDate) return '年龄未知';
				const birth = new Date(birthDate);
				const now = new Date();
				let age = now.getFullYear() - birth.getFullYear();
				const m = now.getMonth() - birth.getMonth();
				if (m < 0 || (m === 0 && now.getDate() < birth.getDate())) age--;

				if (age < 0) return '刚出生';
				return age > 0 ? age + '岁' : '不满1岁';
			},
			editProfile() {
				uni.navigateTo({
					url: '/pages/user/profile-edit'
				});
			},
			goToPetEdit(petId) {
				let url = '/pages/user/pet-edit';
				if (petId) url += `?petId=${petId}`;
				uni.navigateTo({
					url
				});
			},
			goToMyPosts() {
				uni.showToast({
					title: '跳转到我的动态',
					icon: 'none'
				});
			},
			goToDynamicNotices() {
				uni.showToast({
					title: '跳转到动态通知页',
					icon: 'none'
				});
			},
			goToSystemNotices() {
				uni.showToast({
					title: '跳转到系统通知',
					icon: 'none'
				});
			},
			goToFeedback() {
				uni.navigateTo({
					url: '/pages/user/feedback'
				});
			},
			goToSettings() {
				uni.navigateTo({
					url: '/pages/user/settings'
				});
			},
			logout() {
				uni.showModal({
					title: '提示',
					content: '确定退出吗？',
					success: (res) => {
						if (res.confirm) {
							uni.removeStorageSync('token');
							uni.showToast({
								title: '已退出',
								icon: 'success'
							});
						}
					}
				});
			}
		}
	}
</script>

<style scoped lang="scss">
	/* 样式与上一版完全一致，无需修改 */
	.container {
		background-color: #F7F9FC;
		min-height: 100vh;
		padding-bottom: 60rpx;
	}

	.user-header {
		background: linear-gradient(180deg, #E2F0FD 0%, #FFFFFF 100%);
		padding: 60rpx 40rpx 120rpx;
		position: relative;
		border-bottom-left-radius: 60rpx;
		border-bottom-right-radius: 60rpx;
	}

	.header-action-btn {
		position: absolute;
		top: 40rpx;
		right: 40rpx;
		width: 60rpx;
		height: 60rpx;
		background: rgba(0, 0, 0, 0.03);
		border-radius: 50%;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	.arrow {
		color: #999;
		font-size: 28rpx;
		font-weight: bold;
	}

	.user-profile-centered {
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.avatar {
		width: 160rpx;
		height: 160rpx;
		border-radius: 50%;
		border: 6rpx solid white;
		background-color: #eee;
		box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.1);
		margin-bottom: 20rpx;
	}

	.nickname {
		font-size: 40rpx;
		font-weight: bold;
		color: #1A1C20;
		margin-bottom: 10rpx;
	}

	.bio {
		font-size: 24rpx;
		color: #787E8C;
	}

	.stats-board-wrapper {
		margin-top: -60rpx;
		padding: 0 40rpx;
		position: relative;
		z-index: 10;
	}

	.stats-board {
		background: white;
		border-radius: 30rpx;
		padding: 40rpx 0;
		display: flex;
		align-items: center;
		box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.05);
	}

	.stat-item {
		flex: 1;
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.stat-num {
		font-size: 44rpx;
		font-weight: bold;
		color: #1A1C20;
		margin-bottom: 8rpx;
	}

	.stat-label {
		font-size: 24rpx;
		color: #787E8C;
		font-weight: 500;
	}

	.stat-divider {
		width: 2rpx;
		height: 70rpx;
		background-color: #F1F3F7;
	}

	.card-section {
		margin: 40rpx;
		background: white;
		border-radius: 30rpx;
		padding: 40rpx;
		box-shadow: 0 6rpx 20rpx rgba(0, 0, 0, 0.03);
	}

	.section-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 30rpx;
	}

	.title {
		font-size: 34rpx;
		font-weight: bold;
		color: #1A1C20;
	}

	.add-pet-btn {
		font-size: 26rpx;
		color: #4FA2FE;
		font-weight: bold;
	}

	.pet-list-item {
		display: flex;
		align-items: center;
		padding: 30rpx 0;
		border-bottom: 2rpx solid #F7F9FC;
	}

	.pet-list-item:last-child {
		border-bottom: none;
	}

	.pet-icon-square {
		width: 110rpx;
		height: 110rpx;
		border-radius: 20rpx;
		margin-right: 24rpx;
		background-color: #f5f5f5;
		border: 2rpx solid #F1F3F7;
	}

	.pet-item-info {
		flex: 1;
	}

	.pet-name {
		font-size: 32rpx;
		font-weight: bold;
		color: #1A1C20;
		margin-bottom: 8rpx;
		display: block;
	}

	.pet-breed {
		font-size: 24rpx;
		color: #787E8C;
		background-color: #F1F7FE;
		color: #4FA2FE;
		padding: 4rpx 16rpx;
		border-radius: 12rpx;
		align-self: flex-start;
		display: inline-block;
		margin-bottom: 10rpx;
	}

	.pet-details {
		font-size: 24rpx;
		color: #9AA0AF;
		display: block;
	}

	.view-btn {
		font-size: 24rpx;
		color: #9AA0AF;
		margin-left: 20rpx;
	}

	.empty-state {
		text-align: center;
		color: #9AA0AF;
		padding: 40rpx 0;
		font-size: 28rpx;
	}

	.menu-section {
		padding: 10rpx 40rpx;
	}

	.menu-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 36rpx 0;
		border-bottom: 2rpx solid #F7F9FC;
	}

	.menu-item:last-child {
		border-bottom: none;
	}

	.menu-text {
		font-size: 30rpx;
		color: #1A1C20;
		font-weight: 500;
	}

	.logout-text {
		color: #E85757;
	}

	.arrow.red {
		color: #E85757;
	}

	.pet-details {
		display: flex;
		align-items: center;
		font-size: 24rpx;
		color: #9AA0AF;
		margin-top: 8rpx;

		.dot {
			margin: 0 10rpx;
			font-weight: bold;
		}
	}

	.pet-breed {
		font-size: 22rpx;
		color: #4FA2FE;
		background-color: #F1F7FE;
		padding: 2rpx 12rpx;
		border-radius: 8rpx;
		display: inline-block;
	}
</style>