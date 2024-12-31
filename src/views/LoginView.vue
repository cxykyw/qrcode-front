<template>
  <div class="login-container">
    <div class="qrcode-box">
      <h2>扫码登录</h2>
      <div class="qrcode-wrapper">
        <qrcode-vue v-if="qrcodeContent" :value="qrcodeContent" :size="200" level="H" />
        <div v-if="isExpired || hasError" class="qrcode-mask">
          <button class="refresh-btn" @click="generateQRCode">
            <span class="refresh-icon">↻</span>
            刷新二维码
          </button>
        </div>
      </div>
      <p class="countdown" v-if="!isExpired && countdown > 0">
        {{ Math.floor(countdown / 60) }}分{{ countdown % 60 }}秒后过期
      </p>
      <p class="status-text">{{ statusText }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import QrcodeVue from 'qrcode.vue'
import axios from 'axios'
import { useRouter } from 'vue-router'

const router = useRouter()
const qrcodeContent = ref('')
const statusText = ref('请使用手机APP扫描二维码')
const isExpired = ref(false)
const hasError = ref(false)
const countdown = ref(300) // 5分钟倒计时
let pollingTimer = null
let countdownTimer = null
let qrcodeId = ''

// 开始倒计时
const startCountdown = () => {
  // 清除之前的倒计时
  if (countdownTimer) {
    clearInterval(countdownTimer)
  }
  
  countdown.value = 300 // 重置为5分钟
  isExpired.value = false
  
  countdownTimer = setInterval(() => {
    countdown.value--
    if (countdown.value <= 0) {
      clearInterval(countdownTimer)
      clearInterval(pollingTimer)
      isExpired.value = true
      statusText.value = '二维码已过期,请点击刷新'
    }
  }, 1000)
}

// 生成二维码
const generateQRCode = async () => {
  try {
    hasError.value = false
    statusText.value = '正在生成二维码...'
    const res = await axios.get('/api/qrcode/generate')
    qrcodeId = res.data.qrCodeId
    qrcodeContent.value = qrcodeId
    statusText.value = '请使用手机APP扫描二维码'
    
    // 开始轮询状态和倒计时
    startPolling()
    startCountdown()
  } catch (error) {
    console.error('生成二维码失败:', error)
    statusText.value = '生成二维码失败,请点击刷新'
    hasError.value = true
  }
}

// 轮询检查状态
const startPolling = () => {
  // 清除之前的轮询
  if (pollingTimer) {
    clearInterval(pollingTimer)
  }
  
  pollingTimer = setInterval(async () => {
    try {
      const res = await axios.get(`/api/qrcode/check/${qrcodeId}`)
      const status = res.data.status
      
      switch(status) {
        case 'WAITING':
          statusText.value = '请使用手机APP扫描二维码'
          break
        case 'SCANNED':
          statusText.value = '已扫描,请在手机上确认'
          break
        case 'CONFIRMED':
          statusText.value = '登录成功,正在跳转...'
          clearInterval(pollingTimer)
          clearInterval(countdownTimer)
          if (res.data.token) {
            localStorage.setItem('token', res.data.token)
          }
          router.push('/success')
          break
        case 'EXPIRED':
          statusText.value = '二维码已过期,请点击刷新'
          isExpired.value = true
          clearInterval(pollingTimer)
          clearInterval(countdownTimer)
          break
      }
    } catch (error) {
      console.error('查询状态失败:', error)
      statusText.value = '查询状态失败'
    }
  }, 2000)
}

onMounted(() => {
  generateQRCode()
})

onUnmounted(() => {
  if (pollingTimer) {
    clearInterval(pollingTimer)
  }
  if (countdownTimer) {
    clearInterval(countdownTimer)
  }
})
</script>

<style scoped>
.login-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background-color: #f5f5f5;
}

.qrcode-box {
  background: white;
  padding: 30px;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  text-align: center;
}

.qrcode-wrapper {
  position: relative;
  display: inline-block;
  margin: 20px 0;
}

.qrcode-mask {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255, 255, 255, 0.9);
  display: flex;
  justify-content: center;
  align-items: center;
}

.refresh-btn {
  background: #409eff;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  display: flex;
  align-items: center;
  gap: 5px;
  transition: background-color 0.3s;
}

.refresh-btn:hover {
  background: #66b1ff;
}

.refresh-icon {
  font-size: 16px;
}

.status-text {
  margin-top: 20px;
  color: #666;
}

.countdown {
  margin-top: 10px;
  color: #909399;
  font-size: 14px;
}
</style> 