<template>
	<view class="container">
		<view class="header">请选择要查看计划的宠物</view>

		<view class="pet-list">
			<view class="pet-group" v-for="(pet, index) in petList" :key="pet.petid || index">
				<view class="pet-card" @click="goToDetail(pet)">				        <image class="avatar" :src="pet.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
				        <view class="info">
				            <view class="name-row">
				                <text class="name">{{ pet.name }}</text>
				                <text class="progress-tag">{{ pet.progress || 0 }}% 已完成</text>
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
						<text class="todo-count" v-if="pet.tasks">{{ getCompletedCount(pet.tasks) }}/{{ pet.tasks.length }}</text>
					</view>
					
					<view class="task-list" v-if="pet.tasks && pet.tasks.length > 0">
						<view class="task-item" v-for="(task, tIndex) in pet.tasks" :key="task.id">
							<view class="task-left">
								<text class="task-dot" :class="{ 'dot-done': task.isCompleted === 1 }"></text>
								<text class="task-text" :class="{ 'text-done': task.isCompleted === 1 }">
									[{{ task.taskCategory }}] {{ task.taskContent }}
								</text>
							</view>
							<button 
								class="check-btn" 
								:class="{ 'btn-done': task.isCompleted === 1 }"
								@click.stop="handleCheckIn(pIndex, tIndex)"
							>
								{{ task.isCompleted === 1 ? '已打卡' : '打卡' }}
							</button>
						</view>
					</view>
					<view class="no-task" v-else>
						<text>今日暂无养护计划，请点击上方更新</text>
					</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			petList: [] // 存储宠物及其关联任务
		};
	},
	onShow() {
	        // plan.vue 是列表主页，不需要接收特定宠物的 ID，只需要直接请求全部列表即可
	        this.fetchPetsAndTasks();
	    },
	methods: {
		// 1. 串行加载：先拿宠物，再拿任务
		async fetchPetsAndTasks() {
		    const token = uni.getStorageSync('token');
		    try {
		        const res = await uni.request({
		            url: 'http://localhost:8080/pets/user/list',
		            method: 'GET',
		            header: { 'token': token }
		        });
		
		        if (res.data.code === 200 && res.data.data) {
		            const pets = res.data.data;
		            
		            const petWithTasks = await Promise.all(pets.map(async (pet) => {
		                // 1. 兼容 petid 或 petId
		                const targetId = pet.petid || pet.petId; 
		                
		                try {
		                    const taskRes = await uni.request({
		                        url: `http://localhost:8080/care-plans-day/today-plan/${targetId}`,
		                        method: 'GET',
		                        header: { 'token': token }
		                    });
		                    
		                    if (taskRes.data.code === 200 && taskRes.data.data) {
		                        // 【核心修复】使用解构赋值确保 pet 的原有属性（breed, age, avatar）不丢失
		                        return {
		                            ...pet, // 保留原有的 breed, age, avatar, name 等
		                            tasks: taskRes.data.data.tasks || [],
		                            progress: taskRes.data.data.progress || 0
		                        };
		                    }
		                } catch (err) {
		                    console.error("加载今日计划失败:", err);
		                }
		                // 如果请求失败，也要返回原始 pet 数据，否则页面会显示 undefined
		                return { ...pet, tasks: [], progress: 0 };
		            }));
		
		            this.petList = petWithTasks;
		        }
		    } catch (e) {
		        uni.showToast({ title: '加载失败', icon: 'none' });
		    }
		},

		// 2. 打卡逻辑：调用后端 check-in 接口
		async handleCheckIn(pIndex, tIndex) {
			const task = this.petList[pIndex].tasks[tIndex];
			// 如果已经打卡，视业务逻辑决定是否允许取消（本处支持 toggle）
			try {
				const res = await uni.request({
					url: `http://localhost:8080/care-plans-day/check-in/${task.id}`,
					method: 'POST',
					header: { 'Authorization': uni.getStorageSync('token') }
				});

				if (res.data.code === 200) {
					// 更新本地 UI 状态
					task.isCompleted = (task.isCompleted === 1 ? 0 : 1);
					// 后端会返回最新的总进度百分比
					this.petList[pIndex].progress = res.data.data;
					
					if (task.isCompleted === 1) {
						uni.showToast({ title: '打卡成功', icon: 'success' });
					}
				}
			} catch (e) {
				uni.showToast({ title: '打卡接口异常', icon: 'none' });
			}
		},

		goToDetail(pet) {
		    // 暴力获取法：不管后端传出来叫什么名字，我全抓一遍！
		    const id = pet.petid || pet.petId || pet.id;
		    
		    if (!id || id === 'undefined' || id === 'null') {
		        uni.showToast({ title: '宠物数据加载中...', icon: 'none' });
		        return;
		    }
		    // 跳转全部用小写 petid
		    uni.navigateTo({ url: `/pages/plan/detail?petid=${id}` });
		},

		getCompletedCount(tasks) {
			return tasks.filter(t => t.isCompleted === 1).length;
		}
	}
}
</script>

<style lang="scss">
.container { padding: 20px; background-color: #f8f8f8; min-height: 100vh; }
.header { font-size: 13px; color: #999; margin-bottom: 12px; }

.pet-group { 
	background: #fff; border-radius: 12px; margin-bottom: 20px; overflow: hidden; 
	box-shadow: 0 4px 12px rgba(0,0,0,0.03); 
}

.pet-card {
	display: flex; align-items: center; padding: 15px; border-bottom: 1px solid #f9f9f9;
	.avatar { width: 50px; height: 50px; border-radius: 25px; margin-right: 12px; background: #eee; }
	.info { 
		flex: 1; 
		.name-row { display: flex; align-items: center; justify-content: space-between; }
		.name { font-size: 16px; font-weight: bold; color: #333; }
		.progress-tag { font-size: 11px; color: #007AFF; background: #eef6ff; padding: 2px 6px; border-radius: 4px; }
		.desc { font-size: 12px; color: #888; margin-top: 4px; display: block; }
	}
	.arrow { color: #ccc; font-size: 14px; margin-left: 10px; }
}

.todo-section {
	padding: 12px 15px;
	.todo-header { display: flex; justify-content: space-between; margin-bottom: 12px;
		.todo-title { font-size: 13px; font-weight: bold; color: #666; }
		.todo-count { font-size: 12px; color: #999; }
	}
	.task-item {
		display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px;
		.task-left { display: flex; align-items: center; flex: 1;
			.task-dot { width: 6px; height: 6px; border-radius: 3px; background: #007AFF; margin-right: 10px; }
			.dot-done { background: #ddd; }
			.task-text { font-size: 13px; color: #333; }
			.text-done { color: #bbb; text-decoration: line-through; }
		}
		.check-btn { 
			font-size: 11px; padding: 0 10px; height: 24px; line-height: 24px; border-radius: 12px; 
			background: #007AFF; color: #fff; margin: 0;
			&::after { border: none; }
		}
		.btn-done { background: #f0f0f0; color: #999; }
	}
	.no-task { text-align: center; font-size: 12px; color: #ccc; padding: 10px 0; }
}
</style>