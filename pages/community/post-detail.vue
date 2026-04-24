<template>
	<view class="detail-container">
		<view v-if="isLoading" class="loading-state">
			<text>加载中...</text>
		</view>

		<view v-else-if="!post" class="error-state">
			<text>糟糕，动态不见了~</text>
		</view>

		<view v-else class="post-content">
		    <view class="author-header">
		        <image class="avatar" :src="post.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
		        <view class="info">
		            <text class="nickname">{{ post.nickname || '用户_' + (post.userId || '') }}</text>
		            <text class="time">{{ post.createTime }}</text>
		        </view>
		        <view class="post-more-btn" @click="handlePostOptions">
		            <text class="more-dot">•••</text>
		        </view>
		    </view>
		
		    <view class="post-body">
		        <text class="content-text">{{ post.content }}</text>
		        
		        <view class="image-grid" v-if="post.imageList && post.imageList.length > 0">
		            <image 
		                class="grid-img" 
		                v-for="(img, idx) in post.imageList" 
		                :key="idx" 
		                :src="img" 
		                mode="aspectFill" 
		                @click.stop="previewImage(img, post.imageList)"
		            ></image>
		        </view>
		    </view>

			<view class="tags-wrap" v-if="tagsArray.length > 0">
				<text class="tag" v-for="(tag, index) in tagsArray" :key="index">#{{ tag }}</text>
			</view>

			<view class="post-stats">
				<text class="stat-item">点赞 {{ post.likeCount || 0 }}</text>
				<text class="stat-item">评论 {{ post.commentCount || 0 }}</text>
			</view>
		</view>

		<view class="divider"></view>

		<view class="comment-section">
			<view class="comment-title">全部评论 ({{ post ? post.commentCount || 0 : 0 }})</view>
			
			<view v-if="commentList.length === 0" class="empty-comment">
				<text>还没有人评论，快来抢沙发吧~</text>
			</view>

			<view class="comment-list" v-else>
				<view class="comment-item" v-for="(root, index) in commentList" :key="root.commentId">
					<image class="c-avatar" :src="root.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
					
					<view class="c-content-wrap">
						<view class="c-main-box" @click="prepareReply(root.commentId, root.nickname)">
							<view class="c-user">
							    <text class="c-nickname">{{ root.nickname }}</text>
							    <view class="action-right">
							        <text class="report-text" @click.stop="handleCommentOptions(root.commentId)">举报</text>
							        
							        <view class="c-like-btn" @click.stop="likeComment(root)">
							            <text :class="root.isLiked ? 'color-active' : ''">♥</text> 
							            <text class="like-count" :class="root.isLiked ? 'color-active' : ''">{{ root.likeCount || '赞' }}</text>
							        </view>
							    </view>
							</view>
							<text class="c-text">{{ root.content }}</text>
							<text class="c-time">{{ root.createTime }}</text>
						</view>

						<view class="replies-box" v-if="root.replies && root.replies.length > 0">
							<view class="reply-item" 
								  v-for="(sub, subIndex) in root.replies" 
								  :key="sub.commentId"
								  @click="prepareReply(sub.commentId, sub.nickname)"> 
								<image class="r-avatar" :src="sub.avatar || '/static/default-avatar.png'" mode="aspectFill"></image>
								<view class="r-content">
									<view class="r-user">
									    <text class="r-nickname">{{ sub.nickname }}</text>
									    <view class="action-right">
									        <text class="report-text" @click.stop="handleCommentOptions(sub.commentId)">举报</text>
									        
									        <view class="c-like-btn" @click.stop="likeComment(sub)">
									            <text :class="sub.isLiked ? 'color-active' : ''">♥</text> 
									            <text class="like-count" :class="sub.isLiked ? 'color-active' : ''">{{ sub.likeCount || '赞' }}</text>
									        </view>
									    </view>
									</view>
									<view class="r-text-wrap">
										<text v-if="sub.replyToNickname && sub.replyToUserId !== root.userId" class="reply-target">
											回复 @{{ sub.replyToNickname }}: 
										</text>
										<text class="r-text">{{ sub.content }}</text>
									</view>
									<text class="r-time">{{ sub.createTime }}</text>
								</view>
							</view>
						</view>
					</view>
				</view>
			</view>
		</view>

		<view class="bottom-bar" v-if="post">
			<view class="comment-input-wrap">
				<text class="cancel-reply" v-if="replyParentId !== 0" @click="cancelReply">✖</text>
				<input class="comment-input" type="text" v-model="newComment" :placeholder="inputPlaceholder" :focus="isInputFocus" @confirm="submitComment" />
			</view>
			<view class="action-icons" v-if="!newComment">
				<view class="icon-item" @click="toggleLike">
					<text :class="post.isLiked ? 'color-active' : 'color-normal'">{{ post.isLiked ? '已赞' : '点赞' }} {{ post.likeCount || 0 }}</text>
				</view>
				<view class="icon-item" @click="toggleFavorite">
					<text :class="post.isFavorited ? 'color-active' : 'color-normal'">{{ post.isFavorited ? '已收藏' : '收藏' }}</text>
				</view>
			</view>
			<view class="send-btn" v-else @click="submitComment">发送</view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			postId: null,
			post: null,
			isLoading: true,
			tagsArray: [],
			commentList: [], 
			newComment: '',   
			replyParentId: 0, 
			inputPlaceholder: '说点什么...',
			isInputFocus: false
		};
	},
	onLoad(options) {
		if (options.id) {
			this.postId = options.id;
			this.fetchPostDetail();
			this.fetchTreeComments();
		} else {
			this.isLoading = false;
		}
	},
	methods: {
		fetchPostDetail() {
			uni.request({
				url: `http://localhost:8080/community-posts/${this.postId}`,
				method: 'GET',
				header: { 'token': uni.getStorageSync('token') },
				success: (res) => {
					this.isLoading = false;
					if (res.data.code === 200 || res.data.code === 1) {
						let postData = res.data.data;
						
						// 解析图片
						let images = [];
						if (postData.mediaUrls) {
							try {
								images = JSON.parse(postData.mediaUrls);
							} catch (e) {
								if (typeof postData.mediaUrls === 'string') {
									images = postData.mediaUrls.split(',').filter(url => url.trim() !== '');
								}
							}
						}
						postData.imageList = images;
						
						this.post = postData;
						if (this.post.tags) this.tagsArray = this.post.tags.split(',');
					}
				}
			});
		},
		fetchTreeComments() {
			uni.request({
				url: `http://localhost:8080/comments/tree/${this.postId}`,
				method: 'GET',
				success: (res) => {
					if (res.data.code === 200) {
						this.commentList = res.data.data;
					}
				}
			});
		},
		previewImage(current, urls) {
			uni.previewImage({
				current: current,
				urls: urls
			});
		},
		prepareReply(parentId, nickname) {
			const token = uni.getStorageSync('token');
			if (!token) return uni.showToast({ title: '请先登录', icon: 'none' });

			this.replyParentId = parentId;
			this.inputPlaceholder = `回复 @${nickname}:`;
			this.isInputFocus = true; 
		},
		cancelReply() {
			this.replyParentId = 0;
			this.inputPlaceholder = '说点什么...';
			this.isInputFocus = false;
		},
		submitComment() {
			if (!this.newComment.trim()) return;
			const token = uni.getStorageSync('token');
			if (!token) return uni.showToast({ title: '请先登录', icon: 'none' });

			uni.request({
				url: 'http://localhost:8080/comments/user/add',
				method: 'POST',
				header: { 'token': token },
				data: {
					postId: this.postId,
					content: this.newComment,
					parentId: this.replyParentId 
				},
				success: (res) => {
					if (res.data.code === 200) {
						uni.showToast({ title: '发布成功' });
						this.newComment = '';
						this.cancelReply(); 
						this.fetchTreeComments(); 
						if(this.post) this.post.commentCount++;
					}
				}
			});
		},
		likeComment(comment) {
			const token = uni.getStorageSync('token');
			if (!token) return uni.showToast({ title: '请先登录', icon: 'none' });
			
			if(comment.isLiked) {
				comment.likeCount = Math.max(0, (comment.likeCount || 0) - 1);
				comment.isLiked = false;
			} else {
				comment.likeCount = (comment.likeCount || 0) + 1;
				comment.isLiked = true;
				uni.showToast({ title: '点赞成功', icon: 'none' });
			}
		},
		toggleLike() {
			const token = uni.getStorageSync('token');
			if (!token) return uni.showToast({ title: '请先登录', icon: 'none' });
			uni.request({
				url: `http://localhost:8080/likes/toggle/${this.postId}`,
				method: 'POST',
				header: { 'token': token },
				success: (res) => {
					if (res.data.code === 200) {
						this.post.isLiked = res.data.data;
						this.post.likeCount += this.post.isLiked ? 1 : -1;
					}
				}
			});
		},
		handlePostOptions() {
		    uni.showActionSheet({
		        itemList: ['举报该动态'],
		        itemColor: '#ff4d4f', 
		        success: (res) => {
		            if (res.tapIndex === 0) {
		                this.submitReport(this.postId, 'POST'); 
		            }
		        }
		    });
		},
		handleCommentOptions(commentId) {
		    uni.showActionSheet({
		        itemList: ['举报该评论'],
		        itemColor: '#ff4d4f',
		        success: (res) => {
		            if (res.tapIndex === 0) {
		                this.submitReport(commentId, 'COMMENT'); 
		            }
		        }
		    });
		},
		submitReport(targetId, targetType) {
		    const token = uni.getStorageSync('token');
		    if (!token) {
		        return uni.showToast({ title: '请先登录后操作', icon: 'none' });
		    }
		
		    uni.showModal({
		        title: '提示',
		        content: '确定要举报该内容吗？多次被举报的内容将进入人工审核。',
		        success: (modalRes) => {
		            if (modalRes.confirm) {
		                uni.request({
		                    url: 'http://localhost:8080/reports/user/add', 
		                    method: 'POST',
		                    header: { 'token': token },
		                    data: {
		                        targetId: targetId,
		                        targetType: targetType,
		                        reason: '涉嫌违规或引起不适' 
		                    },
		                    success: (res) => {
		                        if (res.data.code === 200 || res.data.code === 1) { 
		                            uni.showToast({ title: '举报已提交，感谢您的反馈', icon: 'none' });
		                        } else {
		                            uni.showToast({ title: res.data.msg || '举报失败', icon: 'none' });
		                        }
		                    }
		                });
		            }
		        }
		    });
		},
		toggleFavorite() {
			const token = uni.getStorageSync('token');
			if (!token) return uni.showToast({ title: '请先登录', icon: 'none' });
			uni.request({
				url: `http://localhost:8080/favorites/toggle/${this.postId}`,
				method: 'POST',
				header: { 'token': token },
				success: (res) => {
					if (res.data.code === 200) {
						this.post.isFavorited = res.data.data;
						uni.showToast({ title: this.post.isFavorited ? '已收藏' : '已取消收藏', icon: 'none' });
					}
				}
			});
		}
	}
}
</script>

<style scoped>
.detail-container { min-height: 100vh; background-color: #FFFFFF; padding-bottom: 120rpx; }
.loading-state, .error-state { text-align: center; padding-top: 200rpx; color: #999; }
.post-content { padding: 30rpx; }
.author-header { display: flex; align-items: center; margin-bottom: 30rpx; position: relative; }
.avatar { width: 80rpx; height: 80rpx; border-radius: 50%; margin-right: 20rpx; background-color: #f0f0f0; flex-shrink: 0; }
.info { display: flex; flex-direction: column; }
.nickname { font-size: 32rpx; font-weight: bold; color: #333; }
.time { font-size: 24rpx; color: #999; margin-top: 6rpx; }
.tags-wrap { display: flex; flex-wrap: wrap; gap: 16rpx; margin-bottom: 30rpx; }
.tag { font-size: 24rpx; color: #42b983; background-color: rgba(66, 185, 131, 0.1); padding: 6rpx 24rpx; border-radius: 30rpx; }

.post-stats { display: flex; gap: 40rpx; margin-bottom: 30rpx; font-size: 26rpx; color: #999; }

.divider { height: 16rpx; background-color: #F8F8F8; margin: 0 -30rpx 30rpx -30rpx; }

.comment-title { font-size: 30rpx; font-weight: bold; color: #333; margin-bottom: 30rpx; }
.empty-comment { text-align: center; padding: 60rpx 0; color: #999; font-size: 28rpx; }

.comment-item { display: flex; margin-bottom: 40rpx; }
.c-avatar { width: 70rpx; height: 70rpx; border-radius: 50%; margin-right: 20rpx; flex-shrink: 0; background-color: #eee; }
.c-content-wrap { flex: 1; }
.c-main-box { margin-bottom: 16rpx; }
.c-user { display: flex; justify-content: space-between; align-items: center; margin-bottom: 8rpx; }
.c-nickname { font-size: 28rpx; color: #666; font-weight: bold; }
.c-text { font-size: 28rpx; color: #333; line-height: 1.5; word-break: break-all; }
.c-time { font-size: 22rpx; color: #CCC; margin-top: 8rpx; display: block; }

.c-like-btn { font-size: 24rpx; color: #999; display: flex; align-items: center; padding: 10rpx; }
.like-count { margin-left: 6rpx; }

.replies-box { background-color: #F9F9F9; border-radius: 12rpx; padding: 20rpx; }
.reply-item { display: flex; margin-bottom: 20rpx; }
.reply-item:last-child { margin-bottom: 0; }
.r-avatar { width: 48rpx; height: 48rpx; border-radius: 50%; margin-right: 16rpx; flex-shrink: 0; background-color: #e0e0e0; }
.r-content { flex: 1; }
.r-user { display: flex; justify-content: space-between; align-items: center; margin-bottom: 6rpx; }
.r-nickname { font-size: 26rpx; color: #666; font-weight: bold; }
.r-text-wrap { font-size: 26rpx; line-height: 1.5; word-break: break-all; }
.reply-target { color: #42b983; margin-right: 8rpx; }
.r-text { color: #333; }
.r-time { font-size: 20rpx; color: #CCC; margin-top: 6rpx; display: block; }

.bottom-bar { position: fixed; bottom: 0; left: 0; width: 100%; height: 110rpx; background-color: #FFFFFF; border-top: 2rpx solid #EEEEEE; display: flex; align-items: center; padding: 0 30rpx; box-sizing: border-box; z-index: 99; padding-bottom: env(safe-area-inset-bottom); }
.comment-input-wrap { flex: 1; background-color: #F5F5F5; height: 70rpx; border-radius: 35rpx; display: flex; align-items: center; padding: 0 30rpx; transition: all 0.3s; }
.cancel-reply { color: #999; font-size: 24rpx; margin-right: 16rpx; padding-right: 16rpx; border-right: 2rpx solid #E0E0E0; }
.comment-input { flex: 1; font-size: 28rpx; }
.action-icons { display: flex; align-items: center; margin-left: 30rpx; gap: 30rpx; }
.icon-item { font-size: 26rpx; }
.send-btn { margin-left: 20rpx; background-color: #42b983; color: white; font-size: 26rpx; padding: 12rpx 30rpx; border-radius: 30rpx; font-weight: bold; }
.color-normal { color: #666; }
.color-active { color: #42b983; font-weight: bold; }

.post-more-btn { position: absolute; right: 0; top: 50%; transform: translateY(-50%); padding: 10rpx 20rpx; }
.more-dot { color: #999; font-size: 36rpx; letter-spacing: 2rpx; font-weight: bold; }

.action-right { display: flex; align-items: center; gap: 20rpx; }
.report-text { font-size: 22rpx; color: #BBB; padding: 4rpx 10rpx; }
.report-text:active { color: #ff4d4f; }
.post-body { margin-top: 20rpx; }
.content-text { font-size: 30rpx; line-height: 1.6; color: #333; display: block; margin-bottom: 20rpx; }
.image-grid { display: flex; flex-wrap: wrap; gap: 12rpx; }
.grid-img { width: 220rpx; height: 220rpx; border-radius: 12rpx; background-color: #f8f8f8; }
</style>