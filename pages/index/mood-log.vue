<template>
	<view class="book-page">
		<view class="wood-desk">
			<view class="scrapbook">
				<view class="book-spine"></view>

				<swiper class="book-swiper" :current="currentIdx" @change="onPageChange" :duration="500">
					<swiper-item v-if="historyList.length === 0">
						<view class="page-content empty-page">
							<view class="stamp-box">
								<text class="stamp-text">MY DIARY</text>
							</view>
							<text class="diary-title">《{{ petName }}的心情手账》</text>
							<text class="empty-tips">这里目前空空如也...\n快点识别表情，开启第一篇日志吧！</text>
						</view>
					</swiper-item>

					<swiper-item v-for="(item, index) in historyList" :key="item.moodId">
						<view class="page-content diary-page">
							<view class="page-header">
								<view class="date-tag">
									<text class="day">{{ getDay(item.recordDate) }}</text>
									<text class="month">{{ getMonth(item.recordDate) }}</text>
								</view>
								<view class="mood-label" :class="item.moodType.toLowerCase()">
									{{ translateMood(item.moodType) }}
								</view>
							</view>
							
							<view class="polaroid">
								<image :src="item.imageUrl" mode="aspectFill" class="photo" @click="preview(item.imageUrl)"></image>
								<view class="tape top-right"></view> </view>

							<scroll-view scroll-y class="handwriting-area">
								<text class="diary-text">{{ item.diaryContent || '这天的小脑袋里不知道在想什么呢...' }}</text>
							</scroll-view>
							
							<view class="page-footer">
								<text class="page-num">Page {{ index + 1 }}</text>
							</view>
						</view>
					</swiper-item>
				</swiper>
			</view>
		</view>

		<view class="bottom-bar">
			<button class="ai-btn" @click="handleUploadMood" :disabled="isLoading">
				<text class="btn-icon">📸</text>
				<text>识别宠物表情</text>
			</button>
		</view>

		<view class="loading-layer" v-if="isLoading">
			<view class="loading-content">
				<view class="paw-spinner">
					<view class="paw-icon">🐾</view>
				</view>
				<text class="loading-title">正在倾听爱宠的心声...</text>
				<view class="skeleton-text">
					<view class="line"></view>
					<view class="line short"></view>
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
			petName: '宝贝',
			historyList: [],
			currentIdx: 0,
			isLoading: false,
			// 丰富的即时反馈文案（按 4 类情绪分类）
			feedbackDict: {
				happy: [
					"目前心情超棒，它说明天会有好事告诉你哦！✨",
					"现在开心地想转圈圈，快去陪它玩会儿吧！🎾",
					"感觉世界都亮了，它最喜欢现在的陪伴啦！",
					"尾巴都要摇成螺旋桨了，明天日记里肯定全是夸主人的话！"
				],
				sad: [
					"目前不是很开心，快去抱抱安慰一下它吧。它的心事明天再偷偷告诉你...☁️",
					"眼神里有一丝忧郁，它在等你一个暖暖的抚摸呢。",
					"看起来有点小委屈，是不是想去草地上奔跑了？",
					"心情有些低落，它的不开心明天会化作心语写进手账里。"
				],
				angry: [
					"现在像个小炸药桶，是不是谁抢它的玩具啦？🔥",
					"表情超凶的！建议先用好吃的“物理降温”一下。",
					"它现在有点小脾气，明天日记里估计要吐槽主人咯~",
					"本宝宝生气了，哄不好的那种（除非有肉干）！"
				],
				other: [
					"情绪稳定，正在思考猫生/狗生的终极意义。🌱",
					"目前状态很佛系，只想安安静静地做个美男/女纸。",
					"心情平静如水，这种安稳的陪伴它很享受呢。",
					"不悲不喜，它说明天再给你整点新花样！"
				]
			}
		}
	},
	onLoad(options) {
		this.petId = options.petId;
		this.fetchPetInfo();
		this.fetchHistory();
	},
	methods: {
		fetchPetInfo() {
			uni.request({
				url: `http://127.0.0.1:8080/pets/${this.petId}`,
				method: 'GET',
				header: { 'token': uni.getStorageSync('token') },
				success: (res) => {
					if (res.data.code === 200) this.petName = res.data.data.name;
				}
			});
		},
		fetchHistory() {
			uni.request({
				url: `http://127.0.0.1:8080/mood-records/history/${this.petId}`,
				method: 'GET',
				header: { 'token': uni.getStorageSync('token') },
				success: (res) => {
					if (res.data.code === 200) {
						this.historyList = res.data.data;
						// 翻到最新一条
						this.currentIdx = this.historyList.length;
					}
				}
			});
		},
		handleUploadMood() {
			uni.chooseImage({
				count: 1,
				sizeType: ['compressed'],
				success: async (res) => {
					const tempPath = res.tempFilePaths[0];
					this.isLoading = true;

					try {
						// 1. 上传图片
						const uploadRes = await uniCloud.uploadFile({
							filePath: tempPath,
							cloudPath: `mood_${Date.now()}.jpg`
						});
						
						// 2. 调用 Java 后端
						uni.request({
							url: 'http://127.0.0.1:8080/mood-records/generate',
							method: 'POST',
							header: { 
								'token': uni.getStorageSync('token'),
								'content-type': 'application/x-www-form-urlencoded'
							},
							data: {
								userId: uni.getStorageSync('userInfo').userId || 1,
								petId: this.petId,
								imageUrl: uploadRes.fileID
							},
							success: (res) => {
								this.isLoading = false;
								if (res.data.code === 200) {
									const mood = res.data.data.moodType.toLowerCase();
									// 随机选取对应情绪的文案
									const msgList = this.feedbackDict[mood] || this.feedbackDict.other;
									const randomMsg = msgList[Math.floor(Math.random() * msgList.length)];
									
									uni.showModal({
										title: '识别成功',
										content: `${this.petName}${randomMsg}`,
										showCancel: false,
										success: () => { this.fetchHistory(); }
									});
								} else {
									uni.showToast({ title: res.data.msg, icon: 'none' });
								}
							},
							fail: () => {
								this.isLoading = false;
								uni.showToast({ title: 'AI由于太累罢工了', icon: 'none' });
							}
						});
					} catch (e) {
						this.isLoading = false;
						uni.showToast({ title: '上传失败', icon: 'none' });
					}
				}
			});
		},
		onPageChange(e) { this.currentIdx = e.detail.current; },
		translateMood(mood) {
			const map = { happy: '✨ 开心', sad: '☁️ 忧郁', angry: '🔥 暴躁', other: '🌱 平静' };
			return map[mood.toLowerCase()] || '❓ 神秘';
		},
		getDay(d) { return d ? d.split('-')[2] : ''; },
		getMonth(d) { 
			const months = ['JAN','FEB','MAR','APR','MAY','JUN','JUL','AUG','SEP','OCT','NOV','DEC'];
			return d ? months[parseInt(d.split('-')[1]) - 1] : '';
		},
		preview(url) { uni.previewImage({ urls: [url] }); }
	}
}
</script>

<style lang="scss" scoped>
.book-page { min-height: 100vh; background-color: #d2b48c; display: flex; flex-direction: column; }
.wood-desk { flex: 1; padding: 30rpx 15rpx; background: radial-gradient(circle, #ead5b5 0%, #c4a484 100%); }

.scrapbook {
	height: 86vh; background-color: #fffdf7; border-radius: 10rpx 40rpx 40rpx 10rpx;
	box-shadow: 15rpx 15rpx 30rpx rgba(0,0,0,0.2), inset 20rpx 0 25rpx rgba(0,0,0,0.05);
	position: relative; overflow: hidden; border-left: 8rpx solid #a08060;
}

.book-spine { position: absolute; left: 40rpx; top: 0; bottom: 0; width: 2rpx; background: rgba(0,0,0,0.1); z-index: 10; border-left: 1rpx dashed #d2b48c; }
.book-swiper { height: 100%; }

.page-content { padding: 40rpx 40rpx 40rpx 70rpx; display: flex; flex-direction: column; height: 100%; }

.empty-page {
	align-items: center; justify-content: center; text-align: center;
	.stamp-box { border: 4rpx double #a08060; padding: 10rpx 20rpx; margin-bottom: 40rpx; color: #a08060; font-weight: bold; }
	.diary-title { font-size: 40rpx; color: #5c4a3d; margin-bottom: 20rpx; }
	.empty-tips { color: #8c7b6c; line-height: 1.8; font-size: 28rpx; }
}

.page-header {
	display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 40rpx;
	.date-tag { display: flex; flex-direction: column; align-items: center; border-bottom: 4rpx solid #5c4a3d;
		.day { font-size: 44rpx; font-weight: 900; line-height: 1; }
		.month { font-size: 20rpx; font-weight: bold; }
	}
	.mood-label { 
		font-size: 24rpx; padding: 6rpx 20rpx; border-radius: 40rpx; font-weight: bold; border: 2rpx solid;
		&.happy { color: #ff6b6b; border-color: #ff6b6b; background: #fff5f5; }
		&.sad { color: #54a0ff; border-color: #54a0ff; background: #f0f7ff; }
		&.angry { color: #ee5253; border-color: #ee5253; background: #fff1f1; }
		&.other { color: #1dd1a1; border-color: #1dd1a1; background: #f0fffb; }
	}
}

.polaroid {
	background: #fff; padding: 15rpx 15rpx 60rpx 15rpx; box-shadow: 0 10rpx 20rpx rgba(0,0,0,0.1);
	transform: rotate(-1.5deg); margin-bottom: 40rpx; position: relative;
	.photo { width: 100%; height: 380rpx; background: #f0f0f0; }
	.tape { position: absolute; width: 100rpx; height: 30rpx; background: rgba(255,255,255,0.4); }
	.top-right { top: -10rpx; right: -20rpx; transform: rotate(30deg); background: rgba(173, 216, 230, 0.4); }
}

.handwriting-area {
	flex: 1; background: repeating-linear-gradient(#fffdf7, #fffdf7 49rpx, #e8dfcc 50rpx);
	padding: 10rpx 0; overflow: hidden;
	.diary-text { font-size: 30rpx; color: #4b3621; line-height: 50rpx; font-family: 'Courier New', Courier, monospace; }
}

.page-footer { text-align: center; padding-top: 10rpx; .page-num { font-size: 20rpx; color: #bcae9e; } }

.bottom-bar { padding: 30rpx; background: #fff;
	.ai-btn { background: #ffb3a7; color: #fff; border-radius: 60rpx; font-weight: bold;
		display: flex; align-items: center; justify-content: center; box-shadow: 0 8rpx 20rpx rgba(255,179,167,0.4);
		.btn-icon { font-size: 40rpx; margin-right: 15rpx; }
	}
}

/* Loading 动画层 */
.loading-layer {
	position: fixed; inset: 0; background: rgba(255,255,255,0.95);
	display: flex; align-items: center; justify-content: center; z-index: 1000;
	.loading-content { display: flex; flex-direction: column; align-items: center; }
	.loading-title { color: #ffb3a7; font-weight: bold; margin-top: 40rpx; font-size: 32rpx; }
}

.paw-spinner {
	width: 120rpx; height: 120rpx; border: 8rpx solid #fff1f1; border-top: 8rpx solid #ffb3a7;
	border-radius: 50%; animation: spin 1s infinite linear; display: flex; align-items: center; justify-content: center;
	.paw-icon { animation: reverse-spin 1s infinite linear; font-size: 40rpx; }
}

.skeleton-text {
	margin-top: 30rpx; width: 300rpx;
	.line { height: 12rpx; background: #eee; border-radius: 6rpx; margin-bottom: 15rpx; }
	.line.short { width: 180rpx; }
}

@keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
@keyframes reverse-spin { from { transform: rotate(0deg); } to { transform: rotate(-360deg); } }
</style>