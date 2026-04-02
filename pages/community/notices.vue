<template>
	<view class="notice-page">
		<view v-if="noticeList.length === 0" class="empty">暂无消息通知</view>
		<view class="notice-item" v-for="(item, index) in noticeList" :key="index" @click="goToDetail(item.postId)">
			<image class="avatar" :src="item.senderAvatar || '/static/default-avatar.png'"></image>
			<view class="right">
				<view class="title">
					<text class="nickname">{{ item.senderNickname }}</text>
					<text class="action">{{ formatAction(item.type) }}</text>
				</view>
				<view class="post-preview">“{{ item.postContentTeaser }}”</view>
				<view class="comment-content" v-if="item.type === 'COMMENT'">回复内容：{{ item.content }}</view>
				<view class="time">{{ item.createTime }}</view>
			</view>
		</view>
	</view>
</template>

<script>
export default {
	data() { return { noticeList: [] }; },
	onLoad() { this.fetchNotices(); },
	methods: {
		fetchNotices() {
			uni.request({
				url: 'http://localhost:8080/interaction-notifications/my',
				method: 'GET',
				header: { 'token': uni.getStorageSync('token') },
				success: (res) => {
					if (res.data.code === 200) { this.noticeList = res.data.data; }
				}
			});
		},
		formatAction(type) {
			const map = { 'LIKE': '点赞了你的动态', 'FAVORITE': '收藏了你的动态', 'COMMENT': '评论了你的动态' };
			return map[type] || '互动了你的动态';
		},
		goToDetail(id) { uni.navigateTo({ url: `/pages/community/post-detail?id=${id}` }); }
	}
}
</script>