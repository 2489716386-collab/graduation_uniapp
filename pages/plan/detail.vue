<template>
	<view class="container">
		<view class="info-card">
			<image class="main-avatar" :src="petInfo.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
			<view class="info-content">
				<view class="name-row">
					<text class="pet-name">{{ petInfo.name }}</text>
					<text class="breed-tag">{{ petInfo.breedName }}</text>
				</view>
				<view class="detail-grid">
					<view class="grid-item">
						<text class="label">年龄</text>
						<text class="value">{{ petInfo.age || 0 }}岁</text>
					</view>
					<view class="grid-item">
						<text class="label">体重</text>
						<text class="value">{{ petInfo.weight || 0 }}kg</text>
					</view>
				</view>
				<view class="status-box">
					<text class="label">当前状态：</text>
					<text class="status-val">{{ petInfo.healthStatus || '健康状态良好' }}</text>
				</view>
			</view>
		</view>

		<view v-if="loading" class="loading-box">
			<text>加载中...</text>
		</view>

		<block v-else-if="weekPlan && weekPlan.id">
			<view class="action-group">
				<button class="btn btn-history" @click="goToHistory">
					<text class="icon">📜</text> 历史记录
				</button>
				<button class="btn btn-update" @click="goToGenerate">
					<text class="icon">🔄</text> 更新计划
				</button>
			</view>

			<view class="focus-section" v-if="weekPlan.weeklyFocus">
				<view class="section-title">本周重点</view>
				<view class="focus-content">
					<text class="focus-text">{{ weekPlan.weeklyFocus }}</text>
				</view>
			</view>

			<view class="weekly-section">
				<view class="section-title">养护详细清单</view>
				
				<view class="day-block" v-for="(day, date) in groupedTasks" :key="date">
					<view class="day-header">
						<text class="day-name">{{ getDayLabel(date) }}</text>
						<text class="day-date">{{ formatDate(date) }}</text>
					</view>
					<view class="day-content">
						<view class="task-line" v-for="(task, tIndex) in day" :key="tIndex">
							<text class="task-cate">[{{ task.taskCategory }}]</text>
							<text class="task-val">{{ task.taskContent }}</text>
						</view>
					</view>
				</view>
				
				<view class="no-data" v-if="Object.keys(groupedTasks).length === 0">
					<text>暂无生成的详细计划</text>
				</view>
			</view>
		</block>

		<view v-else class="empty-state">
			<image class="empty-img" src="/static/images/empty-notice.png" mode="aspectFit"></image>
			<text class="tip-text">本周养护计划还没生成</text>
			<text class="sub-tip">上周计划已归档至历史记录</text>
			<button class="generate-btn" @click="goToGenerate">立即生成本周计划</button>
			<button class="history-btn-plain" @click="goToHistory">查看历史记录</button>
		</view>

	</view>
</template>

<script>
export default {
	data() {
		return {
			petId: null,
			petInfo: {},
			weekPlan: null, // 初始化为 null 方便判断
			groupedTasks: {},
			loading: true // 新增：控制加载状态
		};
	},
	onLoad(options) {
		const id = options.petid || options.petId;
		if (id && id !== 'undefined' && id !== 'null') {
			this.petId = id;
			this.fetchAllData();
		} else {
			uni.showToast({ title: '参数丢失，请返回', icon: 'none' });
		}
	},
	methods: {
		async fetchAllData() {
			if (!this.petId) return;
			
			this.loading = true; // 开始加载
			const token = uni.getStorageSync('token');
			
			try {
				// 1. 获取宠物详细信息 (独立于计划，总是获取并展示)
				const petRes = await uni.request({
					url: `http://localhost:8080/pets/${this.petId}`,
					method: 'GET',
					header: { 'token': token }
				});
				if (petRes.data.code === 200) this.petInfo = petRes.data.data;

				// 2. 获取当前生效的周计划 (调用我们后端修改过的严格本周接口)
				const weekRes = await uni.request({
					url: `http://localhost:8080/care-plans-week/current/${this.petId}`,
					method: 'GET',
					header: { 'token': token }
				});
				
				// 如果有数据，说明本周已经生成
				if (weekRes.data.code === 200 && weekRes.data.data) {
					this.weekPlan = weekRes.data.data;
					// 3. 获取该周计划下的所有每日任务
					await this.fetchWeekTasks(this.weekPlan.id);
				} else {
					// 明确设置为 null，触发 v-else 空状态
					this.weekPlan = null; 
				}
			} catch (e) {
				uni.showToast({ title: '加载数据失败', icon: 'none' });
			} finally {
				this.loading = false; // 无论成功失败，结束加载状态
			}
		},

		async fetchWeekTasks(weekPlanId) {
			const token = uni.getStorageSync('token');
			const res = await uni.request({
				url: `http://localhost:8080/care-plans-day/week-details/${weekPlanId}`,
				method: 'GET',
				header: { 'token': token }
			});
			if (res.data.code === 200) {
				const tasks = res.data.data;
				const groups = {};
				tasks.forEach(task => {
					if (!groups[task.planDate]) groups[task.planDate] = [];
					groups[task.planDate].push(task);
				});
				this.groupedTasks = groups;
			}
		},

		getDayLabel(dateStr) {
			const days = ['周日', '周一', '周二', '周三', '周四', '周五', '周六'];
			return days[new Date(dateStr).getDay()];
		},
		formatDate(dateStr) {
			return dateStr.substring(5); 
		},
		goToHistory() {
			uni.navigateTo({ url: `/pages/plan/history?petid=${this.petId}` });
		},
		goToGenerate() {
			uni.navigateTo({ url: `/pages/plan/generate?petid=${this.petId}` });		
		}
	}
}
</script>

<style lang="scss">
.container { padding: 15px; background: #f8f8f8; min-height: 100vh; }

/* ======== 保留你原有的完美样式 ======== */
.info-card { 
	background: #fff; padding: 15px; border-radius: 12px; margin-bottom: 15px; display: flex; align-items: center;
	box-shadow: 0 2px 10px rgba(0,0,0,0.02);
	.main-avatar { width: 90px; height: 90px; border-radius: 12px; margin-right: 15px; background: #eee; }
	.info-content { 
		flex: 1; 
		.name-row { display: flex; align-items: center; margin-bottom: 8px;
			.pet-name { font-size: 18px; font-weight: bold; color: #333; margin-right: 8px; }
			.breed-tag { font-size: 11px; color: #007AFF; background: #eef6ff; padding: 2px 6px; border-radius: 4px; }
		}
		.detail-grid { display: flex; margin-bottom: 8px;
			.grid-item { margin-right: 20px; .label { font-size: 11px; color: #999; display: block; } .value { font-size: 14px; font-weight: bold; color: #444; } }
		}
		.status-box { font-size: 12px; .label { color: #999; } .status-val { color: #f39c12; font-weight: bold; } }
	}
}

.action-group { display: flex; justify-content: space-between; margin-bottom: 15px;
	.btn { 
		width: 48%; font-size: 14px; border-radius: 10px; height: 40px; line-height: 40px; margin: 0;
		display: flex; align-items: center; justify-content: center;
		.icon { margin-right: 5px; }
	}
	.btn-history { background: #fff; color: #666; border: 1px solid #eee; }
	.btn-update { background: #007AFF; color: #fff; border: none; }
}

.focus-section {
	background: #fff; padding: 15px; border-radius: 12px; margin-bottom: 15px; border-left: 5px solid #007AFF;
	.section-title { font-size: 15px; font-weight: bold; margin-bottom: 8px; color: #333; }
	.focus-text { font-size: 13px; color: #555; line-height: 1.6; }
}

.weekly-section {
	background: #fff; padding: 15px; border-radius: 12px;
	.section-title { font-size: 15px; font-weight: bold; margin-bottom: 15px; color: #333; padding-bottom: 10px; border-bottom: 1px solid #f5f5f5; }
	
	.day-block {
		margin-bottom: 20px;
		&:last-child { margin-bottom: 0; }
		.day-header { display: flex; align-items: baseline; margin-bottom: 10px;
			.day-name { font-size: 14px; font-weight: bold; color: #007AFF; margin-right: 10px; }
			.day-date { font-size: 12px; color: #ccc; }
		}
		.day-content {
			background: #fafafa; border-radius: 8px; padding: 10px;
			.task-line { font-size: 13px; margin-bottom: 8px; line-height: 1.5; display: flex;
				&:last-child { margin-bottom: 0; }
				.task-cate { color: #007AFF; margin-right: 6px; font-weight: bold; white-space: nowrap; }
				.task-val { color: #555; }
			}
		}
	}
}
.no-data { text-align: center; color: #ccc; padding: 30px 0; font-size: 13px; }

/* ======== 新增：加载中和空状态样式 ======== */
.loading-box {
	text-align: center; padding: 100rpx 0; font-size: 28rpx; color: #999;
}

.empty-state {
	display: flex; flex-direction: column; align-items: center; padding-top: 80rpx;
	background: #fff; border-radius: 12px; padding-bottom: 60rpx;
	
	.empty-img { width: 240rpx; height: 240rpx; margin-bottom: 30rpx; }
	.tip-text { font-size: 32rpx; color: #333; font-weight: bold; }
	.sub-tip { font-size: 24rpx; color: #999; margin-top: 10rpx; margin-bottom: 50rpx; }
	
	.generate-btn {
		width: 80%; height: 88rpx; line-height: 88rpx;
		background-color: #007AFF; /* 配合你原有的蓝色主色调 */
		color: #fff; border-radius: 44rpx;
		font-size: 30rpx; margin-bottom: 30rpx;
		box-shadow: 0 4rpx 10rpx rgba(0, 122, 255, 0.2);
		&::after { border: none; }
	}
	
	.history-btn-plain {
		background: transparent; color: #666; font-size: 28rpx; text-decoration: underline;
		&::after { border: none; }
	}
}
</style>