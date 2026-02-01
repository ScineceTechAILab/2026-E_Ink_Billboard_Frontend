<template>
  <view class="page-container">
    <view class="login-wrap" v-if="!token">
      <view class="brand-area">
        <text class="brand-en">E-Ink Preview</text>
        <text class="brand-cn">墨水屏预览助手</text>
      </view>
      <button 
        class="login-btn" 
        hover-class="btn-hover"
        @click="handleLogin"
        :disabled="isLogining"
      >
        <text class="icon-wechat">❖</text>
        {{ isLogining ? '正在登录...' : '微信一键登录' }}
      </button>
      <text class="error-text" v-if="errorMsg">{{ errorMsg }}</text>
    </view>

    <view class="main-wrap" v-else>
      <view class="nav-bar">
        <text class="nav-title">我的作品</text>
        <view class="logout-link" @click="handleLogout">退出</view>
      </view>

      <view class="tab-control">
        <view 
          class="tab-item" 
          :class="{ active: currentTab === 'image' }" 
          @click="switchTab('image')"
        >图片</view>
        <view 
          class="tab-item" 
          :class="{ active: currentTab === 'video' }" 
          @click="switchTab('video')"
        >视频</view>
      </view>

      <view class="action-card">
        <view class="upload-area" @click="handleUpload" hover-class="upload-hover">
          <view v-if="isUploading" class="loading-box">
            <view class="spinner"></view>
            <text>上传处理中...</text>
          </view>
          <view v-else class="upload-placeholder">
            <text class="plus-icon">+</text>
            <text class="upload-tip">
              {{ currentTab === 'image' ? '上传图片预览' : '上传视频处理' }}
            </text>
          </view>
        </view>
      </view>

      <view class="preview-modal" v-if="previewItem">
        <view class="modal-content">
          <view class="modal-header">
            <text>预览效果</text>
            <text class="close-btn" @click="closePreview">×</text>
          </view>
          
          <view class="ink-screen">
            <image 
              v-if="previewItem.type === 'image'"
              :src="previewItem.url" 
              class="preview-media" 
              mode="widthFix"
              show-menu-by-longpress
            ></image>
            
            <video 
              v-else-if="previewItem.type === 'video'"
              :src="previewItem.url" 
              class="preview-media" 
              autoplay 
              loop 
              controls
            ></video>
          </view>
          <text class="ink-tip">
            {{ previewItem.type === 'image' ? '已应用墨水屏抖动算法' : '视频已转码适配墨水屏刷新率' }}
          </text>
        </view>
      </view>

      <view class="history-section">
        <text class="section-title">
          {{ currentTab === 'image' ? '图片记录' : '视频记录' }}
        </text>
        
        <view class="grid-list" v-if="currentList.length > 0">
          <view 
            class="grid-item" 
            v-for="(item, index) in currentList" 
            :key="item.id"
            @click="handleViewDetail(item)"
          >
            <view class="media-box">
              <image 
                v-if="currentTab === 'image'"
                :src="item.processedUrl || item.originalUrl" 
                class="item-img" 
                mode="aspectFill"
                lazy-load
              ></image>
              <view v-else class="video-placeholder">
                <text class="video-icon">▶</text>
                <text class="video-name">{{ item.fileName }}</text>
              </view>
            </view>
            
            <view class="status-group">
               <view :class="['status-badge', getAuditClass(item.auditStatus)]">
                 {{ getAuditText(item.auditStatus) }}
               </view>
               <view 
                 v-if="currentTab === 'video' && item.processingStatus !== 'SUCCESS'"
                 :class="['status-badge', getProcessClass(item.processingStatus)]"
               >
                 {{ getProcessText(item.processingStatus) }}
               </view>
            </view>

            <view class="item-footer">
               <text class="time-text">{{ formatDate(item.createTime) }}</text>
               <view class="del-btn" @click.stop="handleDelete(item.id, index)">删除</view>
            </view>
            
            <view class="reject-reason" v-if="item.auditStatus === 'REJECTED' && item.auditReason">
              拒: {{ item.auditReason }}
            </view>
            <view class="reject-reason" v-if="currentTab === 'video' && item.processingStatus === 'FAILED' && item.failReason">
              错: {{ item.failReason }}
            </view>
          </view>
        </view>

        <view class="list-status">
           <text v-if="currentList.length === 0 && !isListLoading" class="empty-text">暂无记录</text>
           <text v-if="isListLoading" class="loading-text">加载中...</text>
           <text v-if="isNoMore && currentList.length > 0" class="no-more-text">- 到底了 -</text>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue';
import { onReachBottom, onPullDownRefresh } from '@dcloudio/uni-app';

// ===================== 基础配置 =====================
const BASE_API = 'http://62.234.92.187:8081';
const TOKEN_KEY = 'ink_preview_token';

// ===================== 状态管理 =====================
const token = ref('');
const isLogining = ref(false);
const isUploading = ref(false);
const errorMsg = ref('');

// Tab管理: 'image' | 'video'
const currentTab = ref('image');

// 预览对象: { type: 'image'|'video', url: '' }
const previewItem = ref(null);

// 列表数据
const imageList = ref([]);
const videoList = ref([]);
const isListLoading = ref(false);

// 分页参数 (分开管理)
const paging = ref({
  image: { current: 1, size: 10, isNoMore: false },
  video: { current: 1, size: 10, isNoMore: false }
});

// 计算属性：获取当前显示的列表
const currentList = computed(() => {
  return currentTab.value === 'image' ? imageList.value : videoList.value;
});

const isNoMore = computed(() => {
  return paging.value[currentTab.value].isNoMore;
});

// ===================== 生命周期 =====================
onMounted(() => {
  initToken();
});

onPullDownRefresh(async () => {
  if (token.value) {
    await refreshCurrentList();
  }
  uni.stopPullDownRefresh();
});

onReachBottom(() => {
  if (!isNoMore.value && !isListLoading.value && token.value) {
    loadList();
  }
});

// ===================== 核心逻辑 =====================

/** 切换 Tab */
const switchTab = (type) => {
  if (currentTab.value === type) return;
  currentTab.value = type;
  // 切换时如果列表为空，自动加载一次
  if (currentList.value.length === 0) {
    loadList(true);
  }
};

/** 初始化 */
const initToken = () => {
  const savedToken = uni.getStorageSync(TOKEN_KEY);
  if (savedToken) {
    token.value = savedToken;
    loadList(true); // 加载默认 Tab 的列表
  }
};

/** 统一上传入口 */
const handleUpload = async () => {
  if (currentTab.value === 'image') {
    await uploadImage();
  } else {
    await uploadVideo();
  }
};

/** 图片上传 */
const uploadImage = async () => {
  try {
    const filePath = await chooseMedia('image');
    isUploading.value = true;
    const res = await uploadFile('/api/image/upload', filePath);
    // 图片是同步返回URL，可以直接预览
    previewItem.value = { type: 'image', url: res.url };
    uni.showToast({ title: '上传成功', icon: 'success' });
    await refreshCurrentList();
  } catch (err) {
    handleError(err);
  } finally {
    isUploading.value = false;
  }
};

/** 视频上传 */
const uploadVideo = async () => {
  try {
    const filePath = await chooseMedia('video');
    isUploading.value = true;
    // 视频返回的是ID (Long)
    const res = await uploadFile('/api/video/upload', filePath);
    
    uni.showToast({ title: '后台处理中', icon: 'success' });
    // 视频是异步处理，不能立马预览，需要刷新列表看状态
    await refreshCurrentList();
  } catch (err) {
    handleError(err);
  } finally {
    isUploading.value = false;
  }
};

/** 加载列表 (通用) */
const loadList = async (isRefresh = false) => {
  const type = currentTab.value;
  const page = paging.value[type];
  
  if (isRefresh) {
    page.current = 1;
    page.isNoMore = false;
  }
  
  isListLoading.value = true;
  const apiPath = type === 'image' ? '/api/image/list' : '/api/video/list';
  
  try {
    // 对接列表接口
    const res = await request(apiPath, 'GET', {
      current: page.current,
      size: page.size
    });
    
    const newRecords = res.records || [];
    
    if (type === 'image') {
      imageList.value = isRefresh ? newRecords : [...imageList.value, ...newRecords];
    } else {
      videoList.value = isRefresh ? newRecords : [...videoList.value, ...newRecords];
    }

    if (newRecords.length < page.size) {
      page.isNoMore = true;
    } else {
      page.current++;
    }
  } catch (err) {
    handleError(err);
  } finally {
    isListLoading.value = false;
  }
};

const refreshCurrentList = () => loadList(true);

/** 删除条目 */
const handleDelete = (id, index) => {
  const type = currentTab.value;
  uni.showModal({
    title: '确认删除',
    content: '删除后无法恢复，确定吗？',
    success: async (res) => {
      if (res.confirm) {
        try {
          // 对接删除接口
          const apiPath = type === 'image' ? `/api/image/${id}` : `/api/video/${id}`;
          await request(apiPath, 'DELETE');
          
          if (type === 'image') imageList.value.splice(index, 1);
          else videoList.value.splice(index, 1);
          
          uni.showToast({ title: '已删除', icon: 'none' });
        } catch (err) {
          uni.showToast({ title: err.message || '删除失败', icon: 'none' });
        }
      }
    }
  });
};

/** 查看详情/预览 */
const handleViewDetail = (item) => {
  const type = currentTab.value;
  
  // 视频特殊处理：检查处理状态
  if (type === 'video') {
    if (item.processingStatus === 'PROCESSING') {
      uni.showToast({ title: '视频转码中，请稍候...', icon: 'none' });
      return;
    }
    if (item.processingStatus === 'FAILED') {
      uni.showToast({ title: '转码失败: ' + (item.failReason || '未知错误'), icon: 'none' });
      return;
    }
  }

  const url = item.processedUrl || item.originalUrl;
  if (!url) {
    uni.showToast({ title: '文件链接无效', icon: 'none' });
    return;
  }

  previewItem.value = { type, url };
};

const closePreview = () => {
  previewItem.value = null;
};

// ===================== 工具函数 =====================

/** 选择媒体文件 */
const chooseMedia = (type) => {
  return new Promise((resolve, reject) => {
    if (type === 'image') {
      uni.chooseImage({
        count: 1,
        sizeType: ['compressed'],
        success: (res) => resolve(res.tempFilePaths[0]),
        fail: reject
      });
    } else {
      uni.chooseVideo({
        sourceType: ['album', 'camera'],
        compressed: true,
        success: (res) => resolve(res.tempFilePath),
        fail: reject
      });
    }
  });
};

/** 请求封装 */
const request = (url, method, data) => {
  return new Promise((resolve, reject) => {
    uni.request({
      url: BASE_API + url,
      method,
      data,
      header: { 
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token.value}`
      },
      success: (res) => {
        if (res.statusCode === 200 && res.data.code === 200) {
          resolve(res.data.data);
        } else if (res.statusCode === 401 || res.data.code === 401) {
          reject(new Error('401'));
        } else {
          reject(new Error(res.data.info || '请求失败'));
        }
      },
      fail: () => reject(new Error('网络异常'))
    });
  });
};

/** 上传封装 */
const uploadFile = (url, filePath) => {
  return new Promise((resolve, reject) => {
    uni.uploadFile({
      url: BASE_API + url,
      filePath,
      name: 'file',
      header: { 'Authorization': `Bearer ${token.value}` },
      success: (res) => {
        // uni.uploadFile 返回的 data 是字符串
        try {
          const data = JSON.parse(res.data);
          if (res.statusCode === 200 && data.code === 200) {
            resolve(data.data);
          } else {
            reject(new Error(data.info || '上传失败'));
          }
        } catch (e) {
          reject(new Error('响应解析失败'));
        }
      },
      fail: () => reject(new Error('网络上传失败'))
    });
  });
};

const handleLogin = async () => {
  isLogining.value = true;
  try {
    const { code } = await uni.login({ provider: 'weixin' });
    const res = await request('/api/auth/login', 'POST', { code }); // 注意：登录接口可能不需要 Bearer Token，这里需根据实际情况调整 request 封装
    // 修正：request 默认带 token，登录时可能还没 token，但 request 内部取的是 ref，初始为空串没问题。
    // 如果登录接口也是标准结构返回 token，则：
    token.value = res.token;
    uni.setStorageSync(TOKEN_KEY, res.token);
    loadList(true);
  } catch (err) {
    errorMsg.value = err.message;
  } finally {
    isLogining.value = false;
  }
};

const handleLogout = () => {
  token.value = '';
  imageList.value = [];
  videoList.value = [];
  uni.removeStorageSync(TOKEN_KEY);
};

const handleError = (err) => {
  if (err.message === '401') handleLogout();
  else uni.showToast({ title: err.message || '操作失败', icon: 'none' });
};

// 状态格式化
const getAuditText = (s) => ({ 'PENDING': '审核中', 'APPROVED': '已过审', 'REJECTED': '被驳回' }[s] || s);
const getAuditClass = (s) => ({ 'PENDING': 'badge-orange', 'APPROVED': 'badge-green', 'REJECTED': 'badge-red' }[s] || 'badge-gray');

const getProcessText = (s) => ({ 'PROCESSING': '转码中', 'SUCCESS': '成功', 'FAILED': '失败' }[s] || s);
const getProcessClass = (s) => ({ 'PROCESSING': 'badge-blue', 'SUCCESS': 'badge-green', 'FAILED': 'badge-red' }[s] || 'badge-gray');

const formatDate = (isoStr) => {
  if (!isoStr) return '';
  const date = new Date(isoStr);
  return `${date.getMonth()+1}/${date.getDate()} ${date.getHours()}:${String(date.getMinutes()).padStart(2,'0')}`;
};
</script>

<style scoped>
/* 引入之前的样式，并增加新样式 */
.page-container { min-height: 100vh; background: #f7f7f7; font-family: -apple-system, sans-serif; }
.login-wrap { height: 100vh; display: flex; flex-direction: column; align-items: center; justify-content: center; background: #fff; }
.brand-area { text-align: center; margin-bottom: 100rpx; }
.brand-en { font-size: 48rpx; font-weight: 900; letter-spacing: 4rpx; display: block; }
.brand-cn { font-size: 28rpx; color: #999; letter-spacing: 10rpx; }
.login-btn { width: 600rpx; background: #333; color: #fff; border-radius: 8rpx; }
.main-wrap { padding: 30rpx; }
.nav-bar { display: flex; justify-content: space-between; align-items: flex-end; margin-bottom: 30rpx; }
.nav-title { font-size: 44rpx; font-weight: 800; }
.logout-link { color: #999; font-size: 28rpx; padding: 10rpx; }

/* Tab 样式 */
.tab-control { display: flex; background: #e0e0e0; border-radius: 12rpx; padding: 6rpx; margin-bottom: 30rpx; }
.tab-item { flex: 1; text-align: center; padding: 16rpx 0; border-radius: 8rpx; font-size: 28rpx; font-weight: 600; color: #666; transition: all 0.2s; }
.tab-item.active { background: #fff; color: #333; shadow: 0 2rpx 8rpx rgba(0,0,0,0.1); }

/* 上传卡片 */
.action-card { background: #fff; padding: 40rpx; border-radius: 16rpx; margin-bottom: 40rpx; }
.upload-area { height: 200rpx; background: #fafafa; border: 2rpx dashed #ccc; border-radius: 12rpx; display: flex; align-items: center; justify-content: center; flex-direction: column; }
.plus-icon { font-size: 60rpx; color: #999; font-weight: 300; }
.upload-tip { font-size: 28rpx; color: #666; margin-top: 10rpx; }
.spinner { width: 40rpx; height: 40rpx; border: 4rpx solid #e1e1e1; border-top-color: #333; border-radius: 50%; animation: spin 0.8s linear infinite; margin-bottom: 10rpx; }
@keyframes spin { to { transform: rotate(360deg); } }

/* 列表样式 */
.section-title { font-weight: 700; margin-bottom: 20rpx; display: block; }
.grid-list { display: grid; grid-template-columns: 1fr 1fr; gap: 20rpx; }
.grid-item { background: #fff; border-radius: 12rpx; overflow: hidden; position: relative; box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.03); }
.media-box { height: 300rpx; background: #eee; display: flex; align-items: center; justify-content: center; }
.item-img { width: 100%; height: 100%; }
.video-placeholder { display: flex; flex-direction: column; align-items: center; color: #999; }
.video-icon { font-size: 60rpx; margin-bottom: 10rpx; }
.video-name { font-size: 20rpx; padding: 0 20rpx; text-align: center; overflow: hidden; white-space: nowrap; text-overflow: ellipsis; max-width: 100%; }

/* 状态标签 */
.status-group { position: absolute; top: 10rpx; right: 10rpx; display: flex; flex-direction: column; align-items: flex-end; gap: 6rpx; }
.status-badge { font-size: 18rpx; padding: 4rpx 10rpx; border-radius: 6rpx; color: #fff; font-weight: 600; }
.badge-green { background: #52c41a; }
.badge-orange { background: #faad14; }
.badge-red { background: #ff4d4f; }
.badge-blue { background: #1890ff; }
.badge-gray { background: #d9d9d9; }

.item-footer { padding: 16rpx; display: flex; justify-content: space-between; align-items: center; }
.time-text { font-size: 22rpx; color: #bbb; }
.del-btn { font-size: 22rpx; color: #ff4d4f; border: 1rpx solid #ffccc7; padding: 2rpx 10rpx; border-radius: 4rpx; }
.reject-reason { padding: 0 16rpx 16rpx; font-size: 18rpx; color: #ff4d4f; }

/* 模态框 */
.preview-modal { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.9); z-index: 999; display: flex; align-items: center; justify-content: center; }
.modal-content { width: 90%; background: #fff; padding: 30rpx; border-radius: 16rpx; }
.modal-header { display: flex; justify-content: space-between; margin-bottom: 20rpx; font-weight: bold; }
.close-btn { font-size: 40rpx; padding: 0 20rpx; }
.ink-screen { background: #f0f0f0; border: 4rpx solid #333; padding: 10rpx; min-height: 400rpx; display: flex; align-items: center; }
.preview-media { width: 100%; display: block; image-rendering: pixelated; filter: contrast(1.1) grayscale(100%); }
.ink-tip { text-align: center; color: #999; font-size: 24rpx; margin-top: 20rpx; display: block; }
.list-status { text-align: center; padding: 30rpx; color: #999; font-size: 24rpx; }
</style>