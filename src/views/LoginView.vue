<template>
  <div class="login-container">
    <div class="qrcode-box">
      <h2>扫码登录</h2>
      <qrcode-vue v-if="qrcodeContent" :value="qrcodeContent" :size="200" level="H" />
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
let pollingTimer = null
let qrcodeId = ''

// 生成二维码
const generateQRCode = async () => {
  try {
    const res = await axios.get('/api/qrcode/generate')
    qrcodeId = res.data.qrCodeId
    // 使用 qrCodeId 作为二维码内容
    qrcodeContent.value = qrcodeId
    
    // 开始轮询状态
    startPolling()
  } catch (error) {
    console.error('生成二维码失败:', error)
    statusText.value = '生成二维码失败,请刷新重试'
  }
}

// 轮询检查状态
const startPolling = () => {
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
          // 登录成功,保存token并跳转
          if (res.data.token) {
            localStorage.setItem('token', res.data.token)
          }
          router.push('/success')
          break
        case 'EXPIRED':
          statusText.value = '二维码已过期,请刷新重试'
          clearInterval(pollingTimer)
          break
      }
    } catch (error) {
      console.error('查询状态失败:', error)
      statusText.value = '查询状态失败'
    }
  }, 2000) // 每2秒查询一次
}

onMounted(() => {
  generateQRCode()
})

onUnmounted(() => {
  if(pollingTimer) {
    clearInterval(pollingTimer)
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

.status-text {
  margin-top: 20px;
  color: #666;
}
</style> 