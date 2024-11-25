<template>
  <el-card v-loading="pending">
    <h2>前台解析中心 | {{ getAppName() }}</h2>

    <el-alert
      show-icon
      type="warning"
      :closable="false"
      :title="config.custom_copyright"
      v-if="config.show_copyright"
    />

    <el-alert
      :closable="false"
      class="alert"
      :title="`当前解析状态：正常可用！冲鸭！！！`"
      type="success"
      v-if="config.have_account"
    />
    <el-alert :closable="false" class="alert" :title="`当前解析状态：已失效，等修复！`" type="error" v-else />

    <el-alert
      class="alert"
      title="当前网站开启了DEBUG模式,非调试请关闭!!!!"
      type="error"
      v-if="config.debug"
      :closable="false"
    />

    <el-alert
      class="alert"
      title="当前网站未开启SSL,可能出现无法请求Aria2服务器的问题"
      type="error"
      v-if="!config.is_https"
      :closable="false"
    />

    <el-alert v-if="limitMessage === ''" class="alert" type="success" :closable="false">
      <span v-if="isToken">
        <span>当前卡密: {{ limitForm.group_name }} </span>
        <span>剩余可解析文件数: {{ limitForm.count }}</span>
        <span>剩余可解析大小: {{ formatBytes(limitForm.size) }}</span>
        <span>
          到期时间:
          {{
            limitForm.expired_at === '未使用'
              ? limitForm.expired_at
              : new Date(limitForm.expired_at ?? 0).toLocaleString()
          }}
        </span>
      </span>
      <span v-else>
        <span>当前用户组: {{ limitForm.group_name }}</span>
        <span>剩余可解析文件数: {{ limitForm.count }}</span>
        <span>剩余可解析大小: {{ formatBytes(limitForm.size) }}</span>
      </span>
    </el-alert>
    <el-alert v-else class="alert" type="error" :closable="false">
      {{ limitMessage ?? '未知错误' }}
    </el-alert>

    <el-form
      ref="getFileListFormRef"
      :model="getFileListForm"
      :rules="getFileListFormRule"
      label-width="auto"
      class="form"
      :label-position="position"
    >
      <el-form-item label="链接" prop="url">
        <el-input
          v-model.trim="getFileListForm.url"
          @change="clearLinks()"
          @blur="checkLink()"
        ></el-input>
      </el-form-item>
      <el-form-item label="密码" prop="pwd">
        <el-input v-model.trim="getFileListForm.pwd" @change="clearLinks()"></el-input>
      </el-form-item>
      <el-form-item label="解析密码" prop="password" v-if="config.need_password">
        <el-input v-model.trim="getFileListForm.password"></el-input>
      </el-form-item>
      <el-form-item label="卡密(不用留空即可)" prop="token" v-if="config.token_mode">
        <el-input v-model.trim="getFileListForm.token" @blur="tokenBlur"></el-input>
      </el-form-item>
      <el-form-item label="当前路径" prop="dir">
        <el-input v-model="getFileListForm.dir" disabled></el-input>
      </el-form-item>
      <template v-if="vcode.hit_captcha">
        <el-form-item label="验证码编号" prop="vcode_str">
          <el-input v-model="vcode.vcode_str" disabled></el-input>
        </el-form-item>
        <el-form-item label="验证码图片" prop="vcode_img">
          <img
            :src="`${vcode.vcode_img}&t=${timestamp}`"
            alt="验证码图片"
            @click="changeTimestamp"
          />
        </el-form-item>
        <el-form-item label="验证码字符" prop="vcode_input">
          <el-input v-model="vcode.vcode_input"></el-input>
        </el-form-item>
      </template>
      <el-form-item
        label="解析账号id,多个使用,分割"
        prop="account_ids"
        v-if="getLoginRole() === 'admin'"
      >
        <el-input v-model="getFileListForm.account_ids"></el-input>
      </el-form-item>
      <el-form-item label=" " class="buttons">
        <el-button type="primary" @click="fileListStore.getFileList()">获取/刷新列表</el-button>
        <el-button
          type="primary"
          v-if="shouldShowBatchButton"
          @click="fileListStore.getDownloadLinks()"
        >
          批量解析
        </el-button>
        <el-button
          type="primary"
          v-if="shouldShowAdButton && getLoginRole() !== 'admin'"
          @click="showAdPrompt"
        >
          看广告免费解析(输入可卡密可免广告)
        </el-button>
        <el-button type="primary" v-if="config.button_link !== ''" @click="go(config.button_link)">
          前往购买卡密
        </el-button>
        <template v-if="config.show_login_button">
          <el-button type="primary" @click="goLogin()" v-if="getLoginState() === '0'">登陆</el-button>
          <template v-if="getLoginState() === '1'">
            <el-button type="primary" @click="goAdmin()" v-if="getLoginRole() === 'admin'">进入后台</el-button>
            <el-button type="danger" @click="mainStore.logout()"> 注销 </el-button>
          </template>
        </template>
      </el-form-item>
    </el-form>
    
    <el-card v-loading="pending">
      <div class="instructions">
        <h3>使用说明</h3>
        <p>简易使用说明：</p> 
        <p>输入网盘链接→点击【获取/刷新列表】→选中需要下载的文件→看广告或输入卡密→点击【批量解析】→下载 motrix并打开和设置下载配置→点击【发送 Aria2】→等待下载完成</p> 
        <p>如果还是不会用，点击右边链接查看
          <a href="https://blog.91ios.fun/1954.html" target="_blank">详细使用说明</a>。
        </p>
      </div>
    </el-card>
  </el-card>
</template>

<style scoped>
.instructions {
  padding: 20px;
  font-size: 16px;
  line-height: 1.6;
}

.instructions h3 {
  color: #2c3e50;
  margin-bottom: 10px;
}

.instructions ol {
  padding-left: 20px;
}

.instructions li {
  margin: 10px 0;
}

.instructions a {
  color: #409eff;
  text-decoration: none;
}

.instructions a:hover {
  text-decoration: underline;
}
</style>

<script lang="ts" setup>
import { useFileListStore } from '@/stores/fileListStore.js'
import { useMainStore } from '@/stores/mainStore.js'
import { copy } from '@/utils/copy.js'
import { getAppName, getLoginRole, getLoginState } from '@/utils/env.js'
import { formatBytes } from '@/utils/format.js'
import { isMobile } from '@/utils/isMobile.js'
import type { RuleItem } from 'async-validator'
import { ElMessage, type FormInstance, type FormRules } from 'element-plus'
import { storeToRefs } from 'pinia'
import { nextTick, onMounted, ref, computed } from 'vue'
import { useRouter } from 'vue-router'
import Swal from 'sweetalert2'
import CryptoJS from 'crypto-js'

const position = ref('right')
if (isMobile()) position.value = 'top'

const fileListStore = useFileListStore()
const {
  pending,
  getFileListForm,
  getFileListFormRef,
  selectedRows,
  limitForm,
  limitMessage,
  vcode,
  downloadLinks,
  dialogVisible
} = storeToRefs(fileListStore)
const mainStore = useMainStore()
const { config } = storeToRefs(mainStore)

const adsWatched = ref(false) // Signal to track if ads are watched

// Boolean that determines if the batch parse button should be shown
const shouldShowBatchButton = computed(() => {
  return (getLoginRole() === 'admin' || adsWatched.value || (isToken.value && limitMessage.value === ''))
})

// Boolean to determine if the ad button should be shown
const shouldShowAdButton = computed(() => {
  return !adsWatched.value && getLoginRole() !== 'admin' && (limitForm.value.group_name === '游客分组' || (!isToken.value && limitMessage.value === ''))
})

const uid = 1 // 示例UID，请替换为实际值
const secret = "sdg45ffhxxfgsfgd" // 用户签名密钥，请替换为实际值
const maxAdsPerDay = 5 // 每日最多观看广告次数

const showAdPrompt = () => {
  const adid = generateAdID()
  const key = generateKey(uid, secret, adid)

  fetch('https://ad.91ios.fun/api/create.php', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams({ uid: uid.toString(), adid, key })
  })
    .then(response => response.json())
    .then(data => {
      if (data.code === 200) {
        const ticket = data.link
        const query = encodeURIComponent(data.query)
        const wxLink = `${ticket}&query=${query}`

        const userAgent = navigator.userAgent.toLowerCase()
        const isMobile = /iphone|ipod|android|blackberry|iemobile|opera mini/i.test(userAgent)
        let htmlContent

        if (isMobile) {
          htmlContent = `<p>1.微信小程序看完广告后可免费解析。</p><p>2.看1次广告可以解析1次，每日上限5次。</p><p>3.解析不够可以直接购买天卡，次数、大小不限量。</p><a href="${wxLink}" target="_blank" style="color: blue;">点击观看广告解锁</a>`
        } else {
          const redirectTo = encodeURIComponent(`https://ad.91ios.fun/api/h5.html?wxLink=${wxLink}`)
          const qrCodeURL = `https://api.qrserver.com/v1/create-qr-code/?data=${redirectTo}&size=150x150`
          htmlContent = `<p>微信扫码观看广告后可免费解析</p><div style="display: flex; justify-content: center; align-items: center;"><img src="${qrCodeURL}" width="150" height="150"></div><p>1.激励视频广告需播放完毕才能加速。</p><p>2.看1次广告可以解析1次，每日上限5次。</p><p> 3.解析次数不够用可以直接购买天卡，次数、大小不限量。</p>`
        }

        Swal.fire({
          html: htmlContent,
          showConfirmButton: false,
          showCancelButton: false,
          allowOutsideClick: false,
          allowEscapeKey: false
        })

        const adCheckInterval = setInterval(() => {
          fetch('https://ad.91ios.fun/api/check.php', {
            method: 'POST',
            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
            body: new URLSearchParams({ uid: uid.toString(), adid, key })
          })
            .then(response => response.json())
            .then(checkData => {
              if (checkData.code === 200) {
                clearInterval(adCheckInterval)
                Swal.fire({
                  title: '广告观看完成，感谢您的支持',
                  icon: 'success',
                  showConfirmButton: false,
                  timer: 2000
                }).then(() => {
                  adsWatched.value = true // 设置广告已观看
                  fileListStore.getDownloadLinks() // 执行批量解析
                })
              } else if (checkData.code === 500) {
                clearInterval(adCheckInterval)
                Swal.fire({
                  title: '广告不存在或加载失败，请重试',
                  icon: 'error',
                  showConfirmButton: true,
                  timer: 2000
                })
              }
            })
            .catch(error => {
              console.error('Error checking ad status:', error)
            })
        }, 5000)
      } else {
        Swal.fire({
          icon: 'error',
          title: '加载广告失败，请稍后再试',
          showConfirmButton: true,
          timer: 1500
        })
      }
    })
    .catch(error => {
      console.error('Failed to load ad:', error)
      Swal.fire({
        icon: 'error',
        title: '加载广告失败，请稍后再试',
        showConfirmButton: true,
        timer: 1500
      })
    })
}

const generateAdID = () => {
  const base = 'custom'
  const timestamp = new Date().getTime()
  const randomString = Math.random().toString(36).substring(2, 15)
  return base + timestamp + randomString
}

const generateKey = (uid: number, secret: string, token: string) => {
  return CryptoJS.MD5(uid + secret + token).toString()
}

const urlValidator: RuleItem['validator'] = (rule, value, callback) => {
  if (value === '') return callback(new Error('请先输入需要解析的链接'))
  return getUrlId(value) ? callback() : callback(new Error('请输入合法的链接'))
}

const checkLink = () => {
  getFileListForm.value.dir = '/'
  getFileListForm.value.surl = ''

  const data = getUrlId(getFileListForm.value.url)
  if (!data) return
  if (data.id) {
    if (data.surl) {
      getFileListForm.value.url = `https://pan.baidu.com/share/init?surl=${data.id}`
      getFileListForm.value.surl = `1${data.id}`
    } else {
      getFileListForm.value.url = `https://pan.baidu.com/s/${data.id}`
      getFileListForm.value.surl = data.id
    }
  }

  if (data.pwd) {
    getFileListForm.value.pwd = data.pwd
    ElMessage.success('已自动填写密码')
  }
}

const getUrlId = (url: string) => {
  const fullMatch = url.match(/s\/([a-zA-Z0-9_-]+)/)
  const fullMatch2 = url.match(/surl=([a-zA-Z0-9_-]+)/)
  const pwdMatch = url.match(/\?pwd=([a-zA-Z0-9_-]+)/)
  const pwdMatch2 = url.match(/&pwd=([a-zA-Z0-9_-]+)/)
  const pwdMatch3 = url.match(/提取码[:：]\s?([a-zA-Z0-9_-]+)/)

  let id: string
  if (fullMatch2) {
    id = fullMatch2[1]
  } else if (fullMatch) {
    id = fullMatch[1]
  } else {
    return false
  }

  const pwd = pwdMatch ? pwdMatch[1] : pwdMatch2 ? pwdMatch2[1] : pwdMatch3 ? pwdMatch3[1] : null
  return fullMatch2 ? { surl: true, id, pwd } : { id, pwd }
}

const getFileListFormRule: FormRules = {
  url: [{ required: true, validator: urlValidator, trigger: 'blur' }]
}

const copyLink = async (formEl: FormInstance | null) => {
  if (!formEl || !(await formEl.validate())) return

  const search = new URLSearchParams()
  search.set('url', getFileListForm.value.url)
  search.set('surl', getFileListForm.value.surl)
  search.set('pwd', getFileListForm.value.pwd)
  search.set('dir', getFileListForm.value.dir)

  copy(`${location.host}/?${search.toString()}`, '复制成功')
}

onMounted(() => {
  nextTick(() => {
    const token = localStorage.getItem('token')
    if (token && token !== '') {
      ElMessage.success('已自动填充 token')
      getFileListForm.value.token = token
    }

    const searchParams = new URLSearchParams(location.search)
    if (searchParams.size < 4) return

    const url = searchParams.get('url')
    const pwd = searchParams.get('pwd')
    const dir = searchParams.get('dir')
    const surl = searchParams.get('surl')

    if (!url || !pwd || !dir || !surl) return

    getFileListForm.value = { url, pwd, dir, surl }

    ElMessage.success('已读取到参数,正在加载')

    setTimeout(fileListStore.getFileList, 500)
  })

  fileListStore.getLimit()
  changeTimestamp()
})

const router = useRouter()
const goLogin = () => router.push('/login')
const goAdmin = () => router.push('/admin')
const go = (link: string) => window.open(link)

const timestamp = ref(0)

const changeTimestamp = () => (timestamp.value = Date.now())
const clearLinks = () => (downloadLinks.value = [])

const isToken = ref(false)
const tokenBlur = () => {
  fileListStore.getLimit().then(() => {
    if (getFileListForm.value.token !== '') {
      if (limitMessage.value === '') {
        isToken.value = true
        localStorage.setItem('token', getFileListForm.value.token ?? '')
      } else {
        isToken.value = false
        localStorage.removeItem('token')
        ElMessage.error('无效的卡密')
      }
    } else {
      isToken.value = false
      localStorage.removeItem('token')
    }
  })
}
</script>

<style lang="scss" scoped>
img:hover {
  cursor: pointer;
}

a {
  text-decoration: none;
  color: inherit;
}

.el-button + .el-button {
  margin-left: 0;
}

.alert span span + span {
  margin-left: 10px;
}
</style>

<style lang="scss">
.buttons .el-form-item__content {
  gap: 10px;
}
</style>

<style>
  .swal2-title {
    font-size: 16px !important; 
  }
</style>
