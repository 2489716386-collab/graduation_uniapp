<template>
	<view class="container">
		<view class="info-card" v-if="petInfo.petid">
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
			</view>
		</view>

		<view class="history-list">
			<view class="section-title">历史养护计划</view>
			
			<view class="no-data" v-if="historyWeeks.length === 0">
				<text>暂无历史计划记录</text>
			</view>

			<view class="week-card" v-for="(week, wIndex) in historyWeeks" :key="week.id">
				
				<view class="week-header" @click="toggleWeekDetail(wIndex, week.id)">
					<view class="week-title-box">
						<text class="week-date">{{ week.startDate }} ~ {{ week.endDate }}</text>
						<text class="snapshot-weight" v-if="week.snapshotWeight">当时体重: {{ week.snapshotWeight }}kg</text>
					</view>
					<text class="arrow" :class="{ 'arrow-down': week.expanded }">❯</text>
				</view>

				<view class="focus-box" v-if="week.expanded && week.weeklyFocus">
					<text class="focus-title">重点：</text>
					<text class="focus-text">{{ week.weeklyFocus }}</text>
				</view>

				<view class="daily-tasks-box" v-if="week.expanded">
					<view class="loading-text" v-if="week.loading">加载详细任务中...</view>
					
					<view class="day-block" v-for="(dayTasks, date) in week.groupedTasks" :key="date">
						<view class="day-title">{{ formatDate(date) }} {{ getDayLabel(date) }}</view>
						<view class="task-line" v-for="(task, tIndex) in dayTasks" :key="tIndex">
							<text class="task-cate">[{{ task.taskCategory }}]</text>
							<text class="task-val">{{ task.taskContent }}</text>
						</view>
					</view>
					
					<view class="no-tasks" v-if="!week.loading && Object.keys(week.groupedTasks || {}).length === 0">
						未查到该周的具体任务
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
			petId: null,
			petInfo: {},
			historyWeeks: [] // 历史周计划数组
		};
	},
	onLoad(options) {
		// 【防呆设计】兼容大小写
		const id = options.petid || options.petId;
		if (id && id !== 'undefined' && id !== 'null') {
			this.petId = id;
			this.fetchPetInfo();
			this.fetchHistoryWeeks();
		} else {
			uni.showToast({ title: '参数异常', icon: 'none' });
		}
	},
	methods: {
		// 1. 获取宠物基础信息
		async fetchPetInfo() {
			const token = uni.getStorageSync('token');
			const res = await uni.request({
				url: `http://localhost:8080/pets/${this.petId}`,
				method: 'GET',
				header: { 'token': token }
			});
			if (res.data.code === 200) {
				this.petInfo = res.data.data;
			}
		},

		// 2. 获取历史周列表
		async fetchHistoryWeeks() {
			const token = uni.getStorageSync('token');
			try {
				const res = await uni.request({
					url: `http://localhost:8080/care-plans-week/history-list/${this.petId}`,
					method: 'GET',
					header: { 'token': token }
				});
				if (res.data.code === 200) {
					// 给每一周附加 expanded (展开状态)、loading 和 groupedTasks 属性，用于前端控制显示
					this.historyWeeks = res.data.data.map(week => ({
						...week,
						expanded: false,
						loading: false,
						groupedTasks: {}
					}));
				}
			} catch (e) {
				console.error("加载历史列表失败", e);
			}
		},

		// 3. 点击某一周：展开/收起，并懒加载该周的每日任务
		async toggleWeekDetail(index, weekPlanId) {
			const week = this.historyWeeks[index];
			
			// 如果是关闭动作，直接关闭
			if (week.expanded) {
				week.expanded = false;
				return;
			}

			// 展开动作
			week.expanded = true;

			// 如果已经加载过任务数据了，就不重复请求接口
			if (Object.keys(week.groupedTasks).length > 0) return;

			week.loading = true;
			const token = uni.getStorageSync('token');
			
			try {
				// 调用与 detail.vue 中一样的接口获取周详情
				const res = await uni.request({
					url: `http://localhost:8080/care-plans-day/week-details/${weekPlanId}`,
					method: 'GET',
					header: { 'token': token }
				});
				
				if (res.data.code === 200) {
					const tasks = res.data.data;
					const groups = {};
					// 将任务按日期分组
					tasks.forEach(task => {
						if (!groups[task.planDate]) groups[task.planDate] = [];
						groups[task.planDate].push(task);
					});
					week.groupedTasks = groups;
				}
			} catch (e) {
				uni.showToast({ title: '加载任务失败', icon: 'none' });
			} finally {
				week.loading = false;
			}
		},

		// 日期格式化工具方法
		getDayLabel(dateStr) {
			const days = ['周日', '周一', '周二', '周三', '周四', '周五', '周六'];
			return days[new Date(dateStr).getDay()];
		},
		formatDate(dateStr) {
			return dateStr.substring(5); // 将 2023-10-01 截取为 10-01
		}
	}
}
</script>

<style lang="scss">
.container { padding: 15px; background: #f8f8f8; min-height: 100vh; }

/* 宠物信息卡片 (与 detail.vue 一致) */
.info-card { 
	background: #fff; padding: 15px; border-radius: 12px; margin-bottom: 15px; display: flex; align-items: center;
	box-shadow: 0 2px 10px rgba(0,0,0,0.02);
	.main-avatar { width: 70px; height: 70px; border-radius: 10px; margin-right: 15px; background: #eee; }
	.info-content { 
		flex: 1; 
		.name-row { display: flex; align-items: center; margin-bottom: 8px;
			.pet-name { font-size: 18px; font-weight: bold; color: #333; margin-right: 8px; }
			.breed-tag { font-size: 11px; color: #007AFF; background: #eef6ff; padding: 2px 6px; border-radius: 4px; }
		}
		.detail-grid { display: flex;
			.grid-item { margin-right: 20px; .label { font-size: 11px; color: #999; display: block; } .value { font-size: 14px; font-weight: bold; color: #444; } }
		}
	}
}

/* 历史记录部分 */
.history-list {
	.section-title { font-size: 15px; font-weight: bold; color: #333; margin-bottom: 12px; padding-left: 5px; }
	.no-data { text-align: center; color: #ccc; padding: 30px 0; font-size: 13px; }
	
	.week-card {
		background: #fff; border-radius: 12px; margin-bottom: 15px; overflow: hidden;
		box-shadow: 0 2px 8px rgba(0,0,0,0.03);
		
		/* 头部 (点击区域) */
		.week-header {
			padding: 15px; display: flex; justify-content: space-between; align-items: center;
			background: #fafafa;
			.week-title-box {
				display: flex; flex-direction: column;
				.week-date { font-size: 14px; font-weight: bold; color: #333; }
				.snapshot-weight { font-size: 11px; color: #888; margin-top: 4px; }
			}
			.arrow { color: #ccc; font-size: 14px; transition: transform 0.3s; }
			.arrow-down { transform: rotate(90deg); color: #007AFF; }
		}
		
		/* 重点内容 */
		.focus-box {
			padding: 12px 15px; border-bottom: 1px dashed #eee; background: #fff;
			.focus-title { font-size: 13px; font-weight: bold; color: #007AFF; }
			.focus-text { font-size: 13px; color: #555; }
		}
		
		/* 每日任务详情 */
		.daily-tasks-box {
			padding: 15px; background: #fff;
			.loading-text { font-size: 12px; color: #999; text-align: center; padding: 10px; }
			.no-tasks { font-size: 12px; color: #ccc; text-align: center; padding: 10px; }
			
			.day-block {
				margin-bottom: 15px;
				&:last-child { margin-bottom: 0; }
				.day-title { font-size: 13px; font-weight: bold; color: #444; margin-bottom: 8px; border-left: 3px solid #007AFF; padding-left: 6px; }
				.task-line { 
					font-size: 12px; margin-bottom: 6px; color: #666; display: flex;
					.task-cate { color: #007AFF; margin-right: 5px; }
				}
			}
		}
	}
}
</style>