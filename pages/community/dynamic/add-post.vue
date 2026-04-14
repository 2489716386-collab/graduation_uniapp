<template>
	<view class="add-post-container">
		<view class="editor-area">
			<textarea class="post-textarea" v-model="content" placeholder="分享宠物的新鲜事..." maxlength="500"
				:auto-height="true" :show-confirm-bar="false"></textarea>
		</view>

		<view class="media-grid">
			<view class="media-item" v-for="(img, index) in mediaList" :key="index">
				<image class="media-img" :src="img" mode="aspectFill"></image>
				<view class="delete-btn" @click="removeMedia(index)">
					<text class="delete-icon">×</text>
				</view>
			</view>

			<view class="add-media-btn" v-if="mediaList.length < 9" @click="chooseMedia">
				<text class="plus">+</text>
			</view>
		</view>

		<view class="bottom-action">
			<button class="publish-btn" :class="{'btn-active': isReady}" :disabled="!isReady"
				@click="submitPost">发表</button>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				content: '',
				mediaList: [],
				isNavigatingBack: false // 防止弹窗死循环的锁
			};
		},
		// 监听内容变化，如果是小程序，实时开启/关闭原生拦截
		watch: {
			content: 'updateWechatIntercept',
			mediaList: {
				handler: 'updateWechatIntercept',
				deep: true
			}
		},
		computed: {
			// 只有填写了内容或选了图片才允许点击发送
			isReady() {
				return this.content.trim().length > 0 || this.mediaList.length > 0;
			}
		},
		// uni-app 专属生命周期：拦截返回事件 (物理返回键、左滑返回、原生导航栏返回)
		// App 和 H5 环境依然保留 onBackPress
		onBackPress(options) {
			// 如果是点击了发表成功后的返回，或者是已经确认过要退出，则不拦截
			if (this.isNavigatingBack) return false;

			if (this.content.trim().length > 0 || this.mediaList.length > 0) {
				this.showExitModal();
				return true; // 拦截
			}
			return false;
		},
		methods: {
			// 核心修复：微信小程序原生拦截
			updateWechatIntercept() {
				// #ifdef MP-WEIXIN
				const isDirty = this.content.trim().length > 0 || this.mediaList.length > 0;
				if (isDirty) {
					uni.enableAlertBeforeUnload({
						message: '退出后编辑的文案不会保存，确认退出吗？'
					});
				} else {
					uni.disableAlertBeforeUnload();
				}
				// #endif
			},

			// 如果你自己写了返回按钮，请在 @click 中调用这个方法，而不是直接写 uni.navigateBack
			handleCustomBack() {
				if (this.content.trim().length > 0 || this.mediaList.length > 0) {
					this.showExitModal();
				} else {
					uni.navigateBack();
				}
			},
			showExitModal() {
			      uni.showModal({
			        title: '提示',
			        content: '退出后编辑的文案不会保存，确认退出吗？',
			        confirmColor: '#E85757',
			        success: (res) => {
			          if (res.confirm) {
			            this.isNavigatingBack = true; 
			            // #ifdef MP-WEIXIN
			            uni.disableAlertBeforeUnload(); // 退出前先关掉小程序的拦截，否则会弹两次窗
			            // #endif
			            uni.navigateBack();
			          }
			        }
			      });
			    },
			chooseMedia() {
				uni.chooseImage({
					count: 9 - this.mediaList.length,
					sizeType: ['compressed'], // 使用压缩图
					sourceType: ['album', 'camera'],
					success: (res) => {
						this.mediaList = this.mediaList.concat(res.tempFilePaths);
					}
				});
			},

			removeMedia(index) {
				this.mediaList.splice(index, 1);
			},

			async submitPost() {
				if (!this.isReady) return;

				uni.showLoading({
					title: '发送中...',
					mask: true
				});

				try {
					// 1. 上传图片到 uniCloud 云存储
					let uploadedUrls = [];
					for (let path of this.mediaList) {
						const ext = path.split('.').pop() || 'jpg';
						const cloudPath = `post_${Date.now()}_${Math.floor(Math.random() * 1000)}.${ext}`;
						const uploadRes = await uniCloud.uploadFile({
							filePath: path,
							cloudPath: cloudPath
						});
						uploadedUrls.push(uploadRes.fileID);
					}

					// 2. 将数据发送给你的 Spring Boot 后端
					uni.request({
						url: 'http://localhost:8080/community-posts/user/add',
						method: 'POST',
						header: {
							'token': uni.getStorageSync('token')
						},
						data: {
							content: this.content,
							 // 2. 将字段名改为 mediaUrls 以匹配后端 CommunityPosts 实体类中的属性名
							// 3. 后端要求 JSON 序列化，所以使用 JSON.stringify
							mediaUrls: uploadedUrls.length > 0 ? JSON.stringify(uploadedUrls) : null
						},
						success: (res) => {
							uni.hideLoading();
							if (res.data.code === 200) {
								uni.showToast({
									title: '已发好动态',
									icon: 'success'
								});

								// 3. 发布成功，清空内容避免触发拦截弹窗
								this.content = '';
								this.mediaList = [];

								// 4. 埋下标记，通知社区页跳转到“个人”Tab
								uni.setStorageSync('switchCommunityTab', 'personal');

								setTimeout(() => {
									// 既然是从社区页进来的，直接退回上一页即可触发社区页的 onShow 刷新
									uni.navigateBack({
										delta: 1,
										fail: () => {
											// 作为兜底方案，如果确实没有上一页（比如特殊场景进入），再走 switchTab
											uni.switchTab({
												url: '/pages/community/community'
											});
										}
									});
								}, 1000);
							} else {
								uni.showToast({
									title: res.data.msg || '发送失败',
									icon: 'none'
								});
							}
						},
						fail: () => {
							uni.hideLoading();
							uni.showToast({
								title: '网络开小差了',
								icon: 'none'
							});
						}
					});
				} catch (e) {
					uni.hideLoading();
					console.error("上传图片失败:", e);
					uni.showToast({
						title: '图片上传失败',
						icon: 'none'
					});
				}
			}
		}
	}
</script>

<style scoped>
	.add-post-container {
		min-height: 100vh;
		background-color: #FFFFFF;
		padding: 30rpx;
		padding-bottom: env(safe-area-inset-bottom);
	}

	.editor-area {
		width: 100%;
		margin-bottom: 30rpx;
	}

	.post-textarea {
		width: 100%;
		min-height: 240rpx;
		font-size: 32rpx;
		color: #333;
		line-height: 1.5;
	}

	/* 仿朋友圈九宫格布局 */
	.media-grid {
		display: flex;
		flex-wrap: wrap;
		gap: 20rpx;
		margin-bottom: 60rpx;
	}

	.media-item {
		width: 216rpx;
		height: 216rpx;
		position: relative;
		border-radius: 12rpx;
		overflow: hidden;
		background-color: #F8F8F8;
	}

	.media-img {
		width: 100%;
		height: 100%;
		display: block;
	}

	.delete-btn {
		position: absolute;
		top: 0;
		right: 0;
		width: 40rpx;
		height: 40rpx;
		background-color: rgba(0, 0, 0, 0.5);
		border-bottom-left-radius: 16rpx;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	.delete-icon {
		color: #FFF;
		font-size: 30rpx;
		line-height: 1;
	}

	.add-media-btn {
		width: 216rpx;
		height: 216rpx;
		background-color: #F5F5F5;
		border-radius: 12rpx;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	.plus {
		font-size: 80rpx;
		color: #CCC;
		font-weight: 300;
		line-height: 1;
	}

	.bottom-action {
		margin-top: 100rpx;
	}

	.publish-btn {
		background-color: #D3EADC;
		color: #FFFFFF;
		border-radius: 50rpx;
		font-size: 32rpx;
		font-weight: bold;
		height: 90rpx;
		line-height: 90rpx;
	}

	.publish-btn::after {
		border: none;
	}

	.btn-active {
		background-color: #42b983;
		box-shadow: 0 10rpx 30rpx rgba(66, 185, 131, 0.3);
	}
</style>