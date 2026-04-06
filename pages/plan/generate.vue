<template>
	<view class="container" :class="{'bg-white': step === 2 || step === 3}">
		
		<view v-if="step === 1" class="step-form fade-in">
			<view class="section card">
				<view class="section-title">宠物基础资料</view>
				<view class="info-row">
					<text class="label">宠物品种</text>
					<text class="value readonly">{{ petInfo.breedName }}</text>
				</view>
				<view class="info-row">
					<text class="label">当前年龄</text>
					<text class="value readonly">{{ petInfo.age }} 岁</text>
				</view>
				<view class="info-row">
					<text class="label">体重 (kg)</text>
					<input class="value editable" type="digit" v-model="formData.weight" placeholder="输入当前体重" />
				</view>
				<view class="tip">* 更改体重将自动同步到宠物档案</view>
			</view>

			<view class="section card">
				<view class="section-title">健康现状描述</view>
				<textarea 
					class="health-input" 
					v-model="formData.healthStatus" 
					placeholder="请描述宠物的近况（如：胃口好、不喜欢动、正在掉毛等），AI将据此调整计划..."
					maxlength="200"
				></textarea>
				
				<view class="upload-area">
					<view class="upload-title">上传体检报告 (可选)</view>
					<view class="upload-box" @click="chooseImage">
						<image v-if="formData.reportImg" :src="formData.reportImg" mode="aspectFill"></image>
						<view v-else class="upload-placeholder">
							<text class="plus">+</text>
							<text>点此上传</text>
						</view>
					</view>
				</view>
			</view>

			<view class="bottom-bar">
				<button class="gen-btn" @click="startGenerate">提交并生成专属计划</button>
			</view>
		</view>


		<view v-if="step === 2" class="step-loading fade-in">
			<view class="loading-content">
				<view class="paw-icon">🐾</view>
				<text class="loading-title">AI 正在为您定制专属计划</text>
				<text class="loading-subtext">{{ currentLoadingText }}</text>
				
				<view class="progress-bar">
					<view class="progress-inner"></view>
				</view>
			</view>
		</view>


		<view v-if="step === 3" class="step-preview fade-in">
			<view class="preview-header">
				<text class="icon">✨</text>
				<text class="title">本周专属养护方案已生成</text>
			</view>

			<scroll-view scroll-y="true" class="preview-body">
				<view class="ai-card">
					<rich-text :nodes="formattedAiResult"></rich-text>
				</view>
				<view class="disclaimer">
					* 免责声明：此计划由 AI 结合您的宠物资料及兽医学知识库生成，仅供日常养护参考。如宠物出现明显不适，请及时就医。
				</view>
			</scroll-view>

			<view class="bottom-action-bar">
				<button class="btn-outline" @click="reGenerate">重新生成</button>
				<button class="btn-primary" @click="handleConfirmImport">确认并导入计划</button>
			</view>
		</view>

	</view>
</template>

<script>
export default {
	data() {
		return {
			step: 1, // 页面状态机：1-填表, 2-Loading, 3-预览
			petId: null,
			petInfo: {},
			formData: {
				weight: '',
				healthStatus: '',
				reportImg: ''
			},
			aiRawResult: '', 
			
			// Loading 状态相关的动画数据
			loadingTexts: [
				'正在分析宠物身体数据...', 
				'正在检索兽医学专业知识库...', 
				'正在匹配饮食与运动建议...', 
				'正在生成每日打卡任务...', 
				'即将完成，请稍候...'
			],
			currentLoadingText: '正在连接 AI 大脑...',
			loadingTimer: null
		};
	},
	onLoad(options) {
		const id = options.petid || options.petId;
		if (id && id !== 'undefined' && id !== 'null') {
			this.petId = id;
			this.fetchPetDetail();
		}
	},
	computed: {
		formattedAiResult() {
			if (!this.aiRawResult) return '';
			try {
				const data = JSON.parse(this.aiRawResult);
				// 提取并美化显示 AI 的 JSON 结构
				let html = `<div style="line-height: 1.8; color: #333;">`;
				html += `<div style="font-size: 15px; font-weight: bold; color: #007AFF; margin-bottom: 10px;">🎯 本周养护重点</div>`;
				html += `<div style="background: #f0f7ff; padding: 12px; border-radius: 8px; margin-bottom: 20px; font-size: 14px;">${data.weekly_focus}</div>`;
				
				html += `<div style="font-size: 15px; font-weight: bold; color: #007AFF; margin-bottom: 10px;">📅 每日打卡任务概览</div>`;
				html += `<ul style="padding-left: 20px; font-size: 14px; color: #555;">`;
				data.daily_tasks.forEach(t => {
					html += `<li style="margin-bottom: 8px;"><b>[${t.category}]</b> ${t.content}</li>`;
				});
				html += `</ul></div>`;
				return html;
			} catch (e) {
				return `<div style="color: #666;">${this.aiRawResult}</div>`;
			}
		}
	},
	methods: {
		async fetchPetDetail() {
			const res = await uni.request({
				url: `http://localhost:8080/pets/${this.petId}`,
				method: 'GET',
				header: { 'token': uni.getStorageSync('token') }
			});
			if (res.data.code === 200) {
				this.petInfo = res.data.data;
				this.formData.weight = this.petInfo.weight;
				this.formData.healthStatus = this.petInfo.healthStatus || '';
			}
		},

		chooseImage() {
			uni.chooseImage({
				count: 1,
				success: (res) => { this.formData.reportImg = res.tempFilePaths[0]; }
			});
		},

		// 触发生成，进入 Loading 态
		startGenerate() {
			if (!this.formData.weight) {
				return uni.showToast({ title: '请填写宠物体重', icon: 'none' });
			}
			
			// 1. 切换到 Loading 页面
			this.step = 2;
			
			// 2. 开启文案轮播动画
			let textIndex = 0;
			this.currentLoadingText = this.loadingTexts[0];
			this.loadingTimer = setInterval(() => {
				textIndex++;
				if (textIndex < this.loadingTexts.length) {
					this.currentLoadingText = this.loadingTexts[textIndex];
				}
			}, 1500); // 每 1.5 秒换一句话

			// 3. 发起真实的后端 AI 请求
			this.callAiApi();
		},

		async callAiApi() {
			try {
				const res = await uni.request({
					url: 'http://localhost:8080/care-plans-week/generate-preview',
					method: 'POST',
					header: { 'token': uni.getStorageSync('token') },
					data: {
						petId: this.petId,
						weight: this.formData.weight,
						healthStatus: this.formData.healthStatus
					}
				});

				if (res.data.code === 200) {
					this.aiRawResult = res.data.data;
					// 请求成功，切换到预览页
					setTimeout(() => { this.step = 3; }, 500); // 稍微延迟一点，让动画更顺滑
				} else {
					throw new Error(res.data.msg);
				}
			} catch (e) {
				uni.showToast({ title: 'AI生成失败，请重试', icon: 'none' });
				this.step = 1; // 失败则退回表单页
			} finally {
				clearInterval(this.loadingTimer); // 清除定时器
			}
		},

		// 不满意，重新生成
		reGenerate() {
			// 直接返回 Loading 态，并重新调用接口
			this.startGenerate(); 
			// 如果你想让用户退回表单重新修改描述，可以改为 this.step = 1;
		},

		// 确认导入
		async handleConfirmImport() {
			uni.showLoading({ title: '正在保存任务...' });
			try {
				const res = await uni.request({
					url: 'http://localhost:8080/care-plans-week/confirm-import',
					method: 'POST',
					header: { 'token': uni.getStorageSync('token') },
					data: {
						snapshot: {
							petId: this.petId,
							weight: this.formData.weight,
							age: this.petInfo.age,
							healthStatus: this.formData.healthStatus
						},
						aiResultJson: this.aiRawResult
					}
				});

				if (res.data.code === 200) {
					uni.showToast({ title: '计划导入成功', icon: 'success' });
					setTimeout(() => {
						// 成功后跳转回历史详情页
						uni.redirectTo({ url: `/pages/plan/detail?petid=${this.petId}` });
					}, 1500);
				}
			} catch (e) {
				uni.showToast({ title: '导入失败', icon: 'none' });
			} finally {
				uni.hideLoading();
			}
		}
	}
}
</script>

<style lang="scss">
.container { min-height: 100vh; background: #f5f7fa; transition: background 0.3s; }
.bg-white { background: #ffffff !important; }

/* 渐入动画 */
.fade-in { animation: fadeIn 0.4s ease-in-out; }
@keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }

/* ================== Step 1: 表单样式 ================== */
.step-form { padding: 15px; padding-bottom: 100px; }
.card { background: #fff; border-radius: 16px; padding: 20px; margin-bottom: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.02); }
.section-title { font-size: 16px; font-weight: bold; margin-bottom: 15px; color: #2c3e50; border-left: 4px solid #007AFF; padding-left: 10px; }

.info-row { display: flex; justify-content: space-between; align-items: center; padding: 12px 0; border-bottom: 1px solid #f0f0f0; 
	.label { font-size: 14px; color: #7f8c8d; }
	.value { font-size: 14px; color: #2c3e50; font-weight: 500; text-align: right; }
	.readonly { color: #bdc3c7; }
	.editable { color: #007AFF; font-weight: bold; background: #f9f9f9; padding: 4px 10px; border-radius: 4px; width: 80px; }
}
.tip { font-size: 11px; color: #95a5a6; margin-top: 10px; }
.health-input { width: 100%; height: 100px; background: #f8f9fa; border-radius: 8px; padding: 12px; font-size: 13px; box-sizing: border-box; }

.upload-area { margin-top: 20px;
	.upload-title { font-size: 14px; color: #7f8c8d; margin-bottom: 10px; }
	.upload-box { 
		width: 100px; height: 100px; background: #f8f9fa; border: 2px dashed #dcdfe6; border-radius: 8px;
		display: flex; flex-direction: column; align-items: center; justify-content: center;
		image { width: 100%; height: 100%; border-radius: 8px; }
		.plus { font-size: 30px; color: #909399; }
		text { font-size: 12px; color: #909399; }
	}
}

.bottom-bar { position: fixed; bottom: 0; left: 0; right: 0; padding: 20px; background: #fff; box-shadow: 0 -4px 10px rgba(0,0,0,0.05); z-index: 10;
	.gen-btn { background: #007AFF; color: #fff; border-radius: 25px; height: 48px; line-height: 48px; font-weight: bold; font-size: 16px; }
}

/* ================== Step 2: 沉浸式 Loading ================== */
.step-loading { 
	height: 100vh; display: flex; align-items: center; justify-content: center; background: #fff;
	.loading-content { text-align: center; width: 80%; }
	.paw-icon { font-size: 60px; margin-bottom: 20px; animation: pulse 1.5s infinite; display: inline-block; }
	.loading-title { font-size: 18px; font-weight: bold; color: #333; display: block; margin-bottom: 10px; }
	.loading-subtext { font-size: 13px; color: #007AFF; display: block; margin-bottom: 30px; min-height: 20px; }
	
	/* 进度条装X效果 */
	.progress-bar { 
		height: 4px; background: #f0f0f0; border-radius: 2px; overflow: hidden; position: relative;
		.progress-inner { 
			position: absolute; left: 0; top: 0; height: 100%; width: 40%; background: #007AFF; border-radius: 2px;
			animation: scan 2s linear infinite;
		}
	}
}
@keyframes pulse { 0% { transform: scale(1); opacity: 1; } 50% { transform: scale(1.2); opacity: 0.7; } 100% { transform: scale(1); opacity: 1; } }
@keyframes scan { 0% { left: -40%; } 100% { left: 100%; } }

/* ================== Step 3: 预览确认页 ================== */
.step-preview {
	height: 100vh; display: flex; flex-direction: column; background: #fff;
	.preview-header { 
		padding: 40px 20px 20px; text-align: center;
		.icon { font-size: 30px; display: block; margin-bottom: 10px; }
		.title { font-size: 20px; font-weight: bold; color: #333; }
	}
	.preview-body { 
		flex: 1; padding: 0 20px; overflow-y: auto; padding-bottom: 100px;
		.ai-card { background: #fff; border: 1px solid #eee; border-radius: 16px; padding: 20px; box-shadow: 0 10px 30px rgba(0,0,0,0.05); }
		.disclaimer { font-size: 11px; color: #aaa; margin-top: 15px; line-height: 1.5; text-align: justify; }
	}
	.bottom-action-bar { 
		position: fixed; bottom: 0; left: 0; right: 0; padding: 20px; background: #fff; 
		display: flex; justify-content: space-between; gap: 15px; border-top: 1px solid #f9f9f9;
		button { flex: 1; height: 46px; line-height: 46px; border-radius: 23px; font-size: 15px; font-weight: bold; margin: 0; }
		.btn-outline { background: #fff; color: #666; border: 1px solid #ddd; }
		.btn-outline::after { border: none; }
		.btn-primary { background: #007AFF; color: #fff; border: none; }
	}
}
</style>