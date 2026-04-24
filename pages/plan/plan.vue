<template>
	<view class="container">
		
		<view class="empty-state" v-if="!isLoggedIn">
			<image class="empty-img" src="/static/images/empty-notice.png" mode="aspectFit"></image>
			<text class="empty-text">您尚未登录，无法查看宠物关爱计划</text>
			<button class="login-btn" @click="goToLogin">立即登录</button>
		</view>

		<block v-else>
			<view class="header">请选择要查看计划的宠物</view>

			<view class="pet-list" v-if="petList.length > 0">
				<view class="pet-group" v-for="(pet, index) in petList" :key="pet.petid || index">
					<view class="pet-card" @click="goToDetail(pet)">
						<image class="avatar" :src="pet.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
						<view class="info">
							<view class="name-row">
								<text class="name">{{ pet.name }}</text>
							</view>
							<text class="desc">
								{{ pet.breedName }} · {{ pet.age }}岁
							</text>
						</view>
						<text class="arrow">❯</text>
					</view>

					<view class="todo-section">
						<view class="todo-header">
							<text class="todo-title">今日待办事项</text>
						</view>

						<view class="task-list" v-if="pet.tasks && pet.tasks.length > 0">
							<view class="task-item" v-for="(task, tIndex) in pet.tasks" :key="task.id">
								<view class="task-left">
									<view class="task-dot" :class="{ completed: task.isCompleted === 1 }"></view>
									<view class="task-text-content">
										<text class="task-name" :class="{ 'text-grey': task.isCompleted === 1 }">
											[{{ task.taskCategory }}] {{ task.taskContent }}
										</text>
										<text class="complete-time" v-if="task.isCompleted === 1 && task.completeTime">
											打卡于 {{ formatTime(task.completeTime) }}
										</text>
									</view>
								</view>
								<button 
									class="check-btn" 
									:disabled="task.isCompleted === 1" 
									@click="handleCheckIn(task)"
								>
									{{ task.isCompleted === 1 ? '已完成' : '打卡' }}
								</button>
							</view>
						</view>
						
						<view class="no-task" v-else>
							<text>今日暂无养护任务</text>
						</view>
					</view>
				</view>
			</view>

			<view class="empty-state" v-else>
				<text class="empty-text">您还没有添加任何宠物，快去添加吧</text>
			</view>
		</block>
		
	</view>
</template>

<script>
export default {
	data() {
		return {
			petList: [],
			isLoggedIn: false // 新增：用于标记用户是否已登录
		};
	},
	onShow() {
		this.checkLoginStatus();
	},
	methods: {
		// 检查登录状态
		checkLoginStatus() {
			const token = uni.getStorageSync('token');
			if (token) {
				this.isLoggedIn = true;
				this.fetchPetsAndTasks(); // 有 token 才去请求数据
			} else {
				this.isLoggedIn = false;
				this.petList = []; // 清空可能残留的数据
			}
		},

		// 跳转到登录页
		goToLogin() {
			uni.navigateTo({
				url: '/pages/login/login'
			});
		},

		// 1. 获取宠物及其今日任务
		async fetchPetsAndTasks() {
			const token = uni.getStorageSync('token');
			try {
				const res = await uni.request({
					url: 'http://localhost:8080/pets/user/list',
					method: 'GET',
					header: { 'token': token }
				});

				if (res.data.code === 200) {
					const pets = res.data.data;
					
					// 并行请求每只宠物的任务
					const taskPromises = pets.map(async (pet) => {
						const taskRes = await uni.request({
							url: `http://localhost:8080/care-plans-day/today-plan/${pet.petid}`,
							method: 'GET',
							header: { 'token': token }
						});
						return {
							...pet,
							tasks: taskRes.data.code === 200 ? taskRes.data.data : []
						};
					});

					this.petList = await Promise.all(taskPromises);
				}
			} catch (e) {
				console.error("数据加载失败", e);
			}
		},

		// 2. 打卡处理
		async handleCheckIn(task) {
			if (task.isCompleted === 1) return;

			const token = uni.getStorageSync('token');
			try {
				const res = await uni.request({
					url: `http://localhost:8080/care-plans-day/check-in/${task.id}`,
					method: 'POST',
					header: { 'token': token }
				});

				if (res.data.code === 200) {
					uni.showToast({ title: '打卡成功', icon: 'success' });
					this.fetchPetsAndTasks(); // 刷新数据
				}
			} catch (e) {
				uni.showToast({ title: '网络异常', icon: 'none' });
			}
		},

		goToDetail(pet) {
			const id = pet.petid || pet.petId;
			uni.navigateTo({ url: `/pages/plan/detail?petid=${id}` });
		},

		formatTime(dateTimeStr) {
			if (!dateTimeStr) return '';
			// 处理 LocalDateTime 格式 2026-04-07T03:30:00 -> 03:30
			const parts = dateTimeStr.split('T');
			return parts.length > 1 ? parts[1].substring(0, 5) : dateTimeStr;
		}
	}
}
</script>

<style lang="scss" scoped>
.container { padding: 15px; background: #f8f8f8; min-height: 100vh; }

/* ======== 新增的空状态样式 ======== */
.empty-state {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	padding-top: 200rpx;
}
.empty-img {
	width: 300rpx;
	height: 300rpx;
	margin-bottom: 30rpx;
}
.empty-text {
	font-size: 28rpx;
	color: #999;
	margin-bottom: 50rpx;
}
.login-btn {
	width: 280rpx;
	height: 80rpx;
	line-height: 80rpx;
	background-color: #42b983;
	color: #fff;
	border-radius: 40rpx;
	font-size: 30rpx;
	text-align: center;
	box-shadow: 0 4rpx 10rpx rgba(66, 185, 131, 0.3);
}
.login-btn::after {
	border: none;
}
/* ================================ */

.header { font-size: 14px; color: #999; margin-bottom: 15px; padding-left: 5px; }

.pet-group { 
	background: #fff; border-radius: 12px; margin-bottom: 20px; overflow: hidden; 
	box-shadow: 0 4px 12px rgba(0,0,0,0.03);
}

.pet-card {
	display: flex; align-items: center; padding: 15px; border-bottom: 1px solid #f5f5f5;
	.avatar { width: 50px; height: 50px; border-radius: 25px; margin-right: 12px; background: #fafafa; }
	.info { 
		flex: 1; 
		.name-row { display: flex; justify-content: space-between; align-items: center;
			.name { font-size: 16px; font-weight: bold; color: #333; }
		}
		.desc { font-size: 12px; color: #888; margin-top: 4px; display: block; }
	}
	.arrow { color: #ccc; font-size: 14px; margin-left: 10px; }
}

.todo-section { 
	padding: 12px 15px;
	.todo-header { margin-bottom: 10px; .todo-title { font-size: 13px; font-weight: bold; color: #666; } }
	.no-task { font-size: 12px; color: #ccc; text-align: center; padding: 10px; }
}

.task-item {
	display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px;
	.task-left { 
		display: flex; align-items: flex-start; flex: 1;
		.task-dot { width: 6px; height: 6px; border-radius: 3px; background: #007AFF; margin-top: 6px; margin-right: 8px; 
			&.completed { background: #eee; }
		}
		.task-text-content { display: flex; flex-direction: column;
			.task-name { font-size: 13px; color: #444; line-height: 1.4; }
			.complete-time { font-size: 10px; color: #07c160; margin-top: 2px; }
			.text-grey { color: #bbb; text-decoration: line-through; }
		}
	}
	.check-btn { 
		background: #007AFF; color: #fff; font-size: 11px; height: 26px; line-height: 26px; 
		border-radius: 13px; padding: 0 12px; margin: 0; margin-left: 10px;
		&::after { border: none; }
		&[disabled] { background: #f0f0f0; color: #ccc; }
	}
}
</style>