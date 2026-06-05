<template>
  <div class="api-container">
    <el-scrollbar class="scroll">
      <div class="scroll-body">
        
        <!-- Header Section -->
        <div class="api-header">
          <div class="header-icon">
            <Icon icon="material-symbols:api-rounded" width="36" height="36" />
          </div>
          <div class="header-text">
            <h2>{{ isZh ? 'API 接口管理' : 'API Management' }}</h2>
            <p>{{ isZh ? '在这里，您可以查看 API 接入说明，生成与复制用于外部程序集成的 API Token，并进行可视化配置和测试。' : 'Here you can view API integration guides, generate/copy API tokens for external services, and configure visual requests.' }}</p>
          </div>
        </div>

        <div class="card-grid">
          
          <!-- Authentication & Token Card -->
          <div class="settings-card auth-card">
            <div class="card-title">
              <Icon icon="fluent:key-16-regular" width="20" height="20" />
              <span>{{ isZh ? '接口身份认证 (Authentication)' : 'Authentication' }}</span>
            </div>
            <div class="card-content">
              <div class="alert-info">
                <Icon icon="ep:info-filled" width="18" height="18" class="info-icon" />
                <p>
                  {{ isZh ? '所有 API 请求均需要包含认证请求头。认证 Token 的有效期与您的登录状态同步。请注意保管，切勿泄漏。' : 'All API requests must include the authentication header. The API Token\'s validity is synced with your login session. Keep it secure.' }}
                </p>
              </div>
              
              <div class="token-display">
                <div class="token-label">Authorization Token</div>
                <div class="token-input-group">
                  <el-input 
                    type="text" 
                    v-model="token" 
                    readonly 
                    :show-password="hideToken"
                    class="token-input"
                  >
                    <template #suffix>
                      <el-button link @click="hideToken = !hideToken">
                        <Icon :icon="hideToken ? 'ep:view' : 'ep:hide'" width="16" height="16" />
                      </el-button>
                    </template>
                  </el-input>
                  <el-button type="primary" @click="copyText(token, 'token')">
                    <Icon :icon="copiedType === 'token' ? 'ep:select' : 'solar:copy-linear'" width="16" height="16" style="margin-right: 5px;" />
                    {{ copiedType === 'token' ? (isZh ? '已复制' : 'Copied') : (isZh ? '复制 Token' : 'Copy Token') }}
                  </el-button>
                </div>
              </div>
            </div>
          </div>

          <!-- Interactive cURL Constructor Card -->
          <div class="settings-card constructor-card">
            <div class="card-title">
              <Icon icon="solar:code-file-linear" width="20" height="20" />
              <span>{{ isZh ? 'cURL 配置构造器 (cURL Constructor)' : 'cURL Constructor' }}</span>
            </div>
            
            <div class="card-content constructor-grid">
              
              <!-- Left side: Configuration Fields -->
              <div class="config-panel">
                <el-form label-position="top">
                  <el-form-item :label="isZh ? '选择 API 接口' : 'Select API Endpoint'">
                    <el-select v-model="selectedEndpoint" style="width: 100%;">
                      <el-option value="listAccounts" :label="isZh ? '获取邮箱列表 (GET /account/list)' : 'Get Mailboxes (GET /account/list)'" />
                      <el-option value="latestEmails" :label="isZh ? '轮询最新邮件 (GET /email/latest)' : 'Poll Latest Emails (GET /email/latest)'" />
                      <el-option value="sendEmail" :label="isZh ? '发送邮件 (POST /email/send)' : 'Send Email (POST /email/send)'" />
                    </el-select>
                  </el-form-item>

                  <!-- Dynamic inputs for account list (No options needed, it is just a simple GET) -->
                  <div v-if="selectedEndpoint === 'listAccounts'">
                    <p class="config-desc">
                      {{ isZh ? '无额外请求参数，该接口用于获取当前登录用户下所有的邮箱账号及其 ID。' : 'No extra parameters needed. This endpoint retrieves all email accounts and their IDs under your user profile.' }}
                    </p>
                  </div>

                  <!-- Dynamic inputs for latest emails -->
                  <div v-if="selectedEndpoint === 'latestEmails'">
                    <el-form-item :label="isZh ? '选择邮箱账号' : 'Select Mailbox Account'">
                      <el-select v-model="latestParams.accountId" placeholder="Select Account" style="width: 100%;">
                        <el-option :value="0" :label="isZh ? '所有账号 (All Accounts)' : 'All Accounts'" />
                        <el-option 
                          v-for="acc in userAccounts" 
                          :key="acc.accountId" 
                          :value="acc.accountId" 
                          :label="acc.email" 
                        />
                      </el-select>
                    </el-form-item>
                    
                    <el-form-item :label="isZh ? '起始邮件 ID (emailId)' : 'Starting Email ID (emailId)'">
                      <el-input-number v-model="latestParams.emailId" :min="0" style="width: 100%;" />
                      <span class="input-tip">{{ isZh ? '只获取 ID 大于此值的邮件，首次加载可填 0。' : 'Only returns emails with ID greater than this value. Defaults to 0.' }}</span>
                    </el-form-item>

                    <el-form-item :label="isZh ? '接收所有邮箱新邮件 (allReceive)' : 'Receive All Accounts (allReceive)'">
                      <el-switch 
                        v-model="latestParams.allReceive" 
                        :active-value="1" 
                        :inactive-value="0" 
                        :active-text="isZh ? '开启' : 'ON'" 
                        :inactive-text="isZh ? '关闭' : 'OFF'" 
                      />
                    </el-form-item>
                  </div>

                  <!-- Dynamic inputs for sending emails -->
                  <div v-if="selectedEndpoint === 'sendEmail'">
                    
                    <!-- Send Permission Warning -->
                    <div v-if="!canSend" class="permission-warning">
                      <Icon icon="material-symbols:warning-rounded" width="20" height="20" class="warn-icon" />
                      <span>{{ isZh ? '您的账号目前没有邮件发送权限，无法使用发信 API。' : 'Your account currently does not have email sending permissions, and cannot use the send API.' }}</span>
                    </div>

                    <div :class="{ 'disabled-form-mask': !canSend }">
                      <el-form-item :label="isZh ? '发送邮箱账号 (Sender Account)' : 'Sender Account'" required>
                        <el-select v-model="sendParams.accountId" placeholder="Select Sender" style="width: 100%;" :disabled="!canSend">
                          <el-option 
                            v-for="acc in userAccounts" 
                            :key="acc.accountId" 
                            :value="acc.accountId" 
                            :label="acc.email" 
                          />
                        </el-select>
                      </el-form-item>

                      <el-form-item :label="isZh ? '发件人昵称 (Sender Name)' : 'Sender Nickname'">
                        <el-input v-model="sendParams.name" :placeholder="isZh ? '选填，默认使用邮箱前缀' : 'Optional, defaults to email name'" :disabled="!canSend" />
                      </el-form-item>

                      <el-form-item :label="isZh ? '收件人邮箱 (Recipients)' : 'Recipient Emails'" required>
                        <el-input 
                          v-model="sendParams.recipientsInput" 
                          :placeholder="isZh ? '输入收件人邮箱，多个请用英文逗号(,)分隔' : 'Enter recipient emails, separate multiple with comma (,)'" 
                          :disabled="!canSend"
                        />
                      </el-form-item>

                      <el-form-item :label="isZh ? '邮件主题 (Subject)' : 'Subject'" required>
                        <el-input v-model="sendParams.subject" :placeholder="isZh ? '请输入邮件主题' : 'Enter email subject'" :disabled="!canSend" />
                      </el-form-item>

                      <el-form-item :label="isZh ? '纯文本正文 (Plain Text)' : 'Plain Text Body'">
                        <el-input type="textarea" :rows="3" v-model="sendParams.text" :placeholder="isZh ? '纯文本内容' : 'Plain text content'" :disabled="!canSend" />
                      </el-form-item>

                      <el-form-item :label="isZh ? 'HTML正文 (HTML Content)' : 'HTML Body'">
                        <el-input type="textarea" :rows="3" v-model="sendParams.content" :placeholder="isZh ? 'HTML 格式内容' : 'HTML formatted content'" :disabled="!canSend" />
                      </el-form-item>
                    </div>
                  </div>
                </el-form>
              </div>

              <!-- Right side: Real-time cURL Display -->
              <div class="code-panel">
                <div class="code-header">
                  <span class="code-title">Generated cURL Command</span>
                  <el-button type="success" size="small" class="copy-code-btn" @click="copyText(curlCommand, 'curl')">
                    <Icon :icon="copiedType === 'curl' ? 'ep:select' : 'solar:copy-linear'" width="14" height="14" style="margin-right: 4px;" />
                    {{ copiedType === 'curl' ? (isZh ? '已复制' : 'Copied') : (isZh ? '复制命令' : 'Copy cURL') }}
                  </el-button>
                </div>
                <div class="code-body">
                  <pre><code>{{ curlCommand }}</code></pre>
                </div>
              </div>

            </div>
          </div>

          <!-- API Detailed Reference Card -->
          <div class="settings-card reference-card">
            <div class="card-title">
              <Icon icon="fluent:document-search-20-regular" width="20" height="20" />
              <span>{{ isZh ? 'API 详细参考指南 (API Reference)' : 'API Reference Guide' }}</span>
            </div>
            
            <div class="card-content">
              <el-tabs>
                <!-- Get Mailbox List Tab -->
                <el-tab-pane :label="isZh ? '1. 获取账号列表' : '1. Get Mailbox List'">
                  <div class="api-doc-item">
                    <div class="api-route-tag get">GET</div>
                    <code class="api-route-path">/api/account/list</code>
                  </div>
                  <p class="api-doc-desc">
                    {{ isZh ? '获取当前登录用户下所有的邮箱账号及配置状态。' : 'Retrieve all email accounts and configurations under the logged-in user profile.' }}
                  </p>

                  <h5 class="section-sub-title">Request Headers</h5>
                  <table class="doc-table">
                    <thead>
                      <tr><th>Header</th><th>Type</th><th>Required</th><th>Description</th></tr>
                    </thead>
                    <tbody>
                      <tr><td>Authorization</td><td>String</td><td>Yes</td><td>{{ isZh ? '接口验证 Token' : 'API Session Token' }}</td></tr>
                    </tbody>
                  </table>

                  <h5 class="section-sub-title">Response Example (JSON)</h5>
                  <div class="code-block">
                    <pre><code>{
  "code": 200,
  "message": "success",
  "data": [
    {
      "accountId": 1,
      "email": "test@freshmail.com",
      "name": "Test User",
      "status": 0,
      "allReceive": 1,
      "createTime": "2026-06-05 15:30:00"
    }
  ]
}</code></pre>
                  </div>
                </el-tab-pane>

                <!-- Check Email Tab -->
                <el-tab-pane :label="isZh ? '2. 轮询最新邮件' : '2. Poll Latest Emails'">
                  <div class="api-doc-item">
                    <div class="api-route-tag get">GET</div>
                    <code class="api-route-path">/api/email/latest</code>
                  </div>
                  <p class="api-doc-desc">
                    {{ isZh ? '增量查询新收到的邮件，通常用于外部程序进行邮件接收轮询。' : 'Incrementally query newly received emails. Typically used for email polling in external apps.' }}
                  </p>

                  <h5 class="section-sub-title">Request Headers</h5>
                  <table class="doc-table">
                    <thead>
                      <tr><th>Header</th><th>Type</th><th>Required</th><th>Description</th></tr>
                    </thead>
                    <tbody>
                      <tr><td>Authorization</td><td>String</td><td>Yes</td><td>{{ isZh ? '接口验证 Token' : 'API Session Token' }}</td></tr>
                    </tbody>
                  </table>

                  <h5 class="section-sub-title">Query Parameters</h5>
                  <table class="doc-table">
                    <thead>
                      <tr><th>Parameter</th><th>Type</th><th>Required</th><th>Default</th><th>Description</th></tr>
                    </thead>
                    <tbody>
                      <tr><td>accountId</td><td>Number</td><td>No</td><td>0</td><td>{{ isZh ? '指定的邮箱账号 ID，为 0 时查询全部' : 'Mailbox ID, query all if 0' }}</td></tr>
                      <tr><td>emailId</td><td>Number</td><td>No</td><td>0</td><td>{{ isZh ? '获取邮件 ID 大于此值的新邮件' : 'Only fetch emails with ID greater than this value' }}</td></tr>
                      <tr><td>allReceive</td><td>Number</td><td>No</td><td>0</td><td>{{ isZh ? '是否接收该账户所拥有的全部域名的邮件 (1: 开启, 0: 关闭)' : 'Whether to receive all domains (1: ON, 0: OFF)' }}</td></tr>
                    </tbody>
                  </table>

                  <h5 class="section-sub-title">Response Example (JSON)</h5>
                  <div class="code-block">
                    <pre><code>{
  "code": 200,
  "message": "success",
  "data": [
    {
      "emailId": 125,
      "sendEmail": "sender@external.com",
      "name": "External Sender",
      "accountId": 1,
      "userId": 5,
      "subject": "Important Notification",
      "content": "&lt;p&gt;Hello, this is a test email content.&lt;/p&gt;",
      "text": "Hello, this is a test email content.",
      "createTime": "2026-06-05 15:40:00",
      "isDel": 0,
      "unread": 0,
      "recipient": "[{\"address\":\"test@freshmail.com\",\"name\":\"\"}]",
      "toEmail": "test@freshmail.com",
      "toName": "test"
    }
  ]
}</code></pre>
                  </div>
                </el-tab-pane>

                <!-- Send Email Tab -->
                <el-tab-pane :label="isZh ? '3. 发送邮件' : '3. Send Email'">
                  <div class="api-doc-item">
                    <div class="api-route-tag post">POST</div>
                    <code class="api-route-path">/api/email/send</code>
                  </div>
                  <p class="api-doc-desc">
                    {{ isZh ? '通过后台配置的邮件发送通道（如 Resend）向外部或内部邮箱发送邮件。' : 'Send emails to external or internal recipients via configured email delivery channels (e.g. Resend).' }}
                  </p>

                  <h5 class="section-sub-title">Request Headers</h5>
                  <table class="doc-table">
                    <thead>
                      <tr><th>Header</th><th>Type</th><th>Required</th><th>Description</th></tr>
                    </thead>
                    <tbody>
                      <tr><td>Authorization</td><td>String</td><td>Yes</td><td>{{ isZh ? '接口验证 Token' : 'API Session Token' }}</td></tr>
                      <tr><td>Content-Type</td><td>String</td><td>Yes</td><td>application/json</td></tr>
                    </tbody>
                  </table>

                  <h5 class="section-sub-title">Request Body (JSON)</h5>
                  <table class="doc-table">
                    <thead>
                      <tr><th>Field</th><th>Type</th><th>Required</th><th>Description</th></tr>
                    </thead>
                    <tbody>
                      <tr><td>accountId</td><td>Number</td><td>Yes</td><td>{{ isZh ? '发送邮箱所属的账号 ID (必须是该用户拥有的账号)' : 'The sender mailbox ID (must be owned by the user)' }}</td></tr>
                      <tr><td>name</td><td>String</td><td>No</td><td>{{ isZh ? '发件人显示名称' : 'Sender display name' }}</td></tr>
                      <tr><td>receiveEmail</td><td>Array[String]</td><td>Yes</td><td>{{ isZh ? '收件人邮箱地址列表' : 'List of recipient email addresses' }}</td></tr>
                      <tr><td>subject</td><td>String</td><td>Yes</td><td>{{ isZh ? '邮件标题' : 'Email subject' }}</td></tr>
                      <tr><td>text</td><td>String</td><td>No</td><td>{{ isZh ? '邮件纯文本内容' : 'Plain text email body' }}</td></tr>
                      <tr><td>content</td><td>String</td><td>No</td><td>{{ isZh ? '邮件 HTML 富文本内容' : 'HTML email body' }}</td></tr>
                      <tr><td>attachments</td><td>Array</td><td>No</td><td>{{ isZh ? '附件列表' : 'Attachment list (defaults to empty)' }}</td></tr>
                    </tbody>
                  </table>

                  <h5 class="section-sub-title">Response Example (JSON)</h5>
                  <div class="code-block">
                    <pre><code>{
  "code": 200,
  "message": "success",
  "data": [
    {
      "sendEmail": "test@freshmail.com",
      "name": "Test User",
      "subject": "Hello World",
      "content": "<p>HTML content</p>",
      "text": "Plain text content",
      "accountId": 1,
      "status": 1,
      "type": 1,
      "userId": 5,
      "recipient": "[{\"address\":\"recipient@example.com\",\"name\":\"\"}]",
      "emailId": 126,
      "createTime": "2026-06-05 15:45:00"
    }
  ]
}</code></pre>
                  </div>
                </el-tab-pane>
              </el-tabs>
            </div>
          </div>

        </div>

      </div>
    </el-scrollbar>
  </div>
</template>

<script setup>
import { ref, computed, reactive, onMounted, defineOptions } from 'vue'
import { useI18n } from 'vue-i18n'
import { Icon } from '@iconify/vue'
import { hasPerm } from '@/perm/perm.js'
import { accountList } from '@/request/account.js'

defineOptions({
  name: 'api-docs'
})

const { locale } = useI18n()
const isZh = computed(() => locale.value === 'zh')

// Token info
const token = ref(localStorage.getItem('token') || '')
const hideToken = ref(true)
const copiedType = ref('')

// User Accounts for dynamic selection
const userAccounts = ref([])

// Endpoint choice
const selectedEndpoint = ref('listAccounts')

// Configuration parameters
const latestParams = reactive({
  accountId: 0,
  emailId: 0,
  allReceive: 0
})

const sendParams = reactive({
  accountId: '',
  name: '',
  recipientsInput: 'recipient@example.com',
  subject: 'Test Email via API',
  text: 'This is a test email sent using the Fresh Mail API.',
  content: '<p>This is a test email sent using the <b>Fresh Mail API</b>.</p>'
})

// Permissions check
const canSend = computed(() => hasPerm('email:send'))

// Fetch user mailbox accounts on mount
onMounted(() => {
  if (localStorage.getItem('token')) {
    accountList(0, 100).then(list => {
      userAccounts.value = list || []
      if (userAccounts.value.length > 0) {
        sendParams.accountId = userAccounts.value[0].accountId
      }
    }).catch(err => {
      console.error('Failed to load accounts for API helper', err)
    })
  }
})

// Dynamic cURL Command Generator
const curlCommand = computed(() => {
  const origin = window.location.origin
  const baseUrlEnv = import.meta.env.VITE_BASE_URL
  let apiBaseUrl = origin + '/api'
  
  if (baseUrlEnv && baseUrlEnv.startsWith('http')) {
    apiBaseUrl = baseUrlEnv
  } else if (baseUrlEnv) {
    const cleanBase = baseUrlEnv.replace(/\/$/, '')
    apiBaseUrl = `${origin}${cleanBase}`
  }

  // Ensure authorization token is visible in curl
  const tokenHeader = ` -H "Authorization: ${token.value || 'YOUR_API_TOKEN'}"`

  if (selectedEndpoint.value === 'listAccounts') {
    return `curl -X GET "${apiBaseUrl}/account/list" \\\n${tokenHeader}`
  }

  if (selectedEndpoint.value === 'latestEmails') {
    const queryParts = []
    if (latestParams.accountId !== 0) queryParts.push(`accountId=${latestParams.accountId}`)
    if (latestParams.emailId !== 0) queryParts.push(`emailId=${latestParams.emailId}`)
    if (latestParams.allReceive !== 0) queryParts.push(`allReceive=${latestParams.allReceive}`)
    
    const queryStr = queryParts.length > 0 ? `?${queryParts.join('&')}` : ''
    return `curl -X GET "${apiBaseUrl}/email/latest${queryStr}" \\\n${tokenHeader}`
  }

  if (selectedEndpoint.value === 'sendEmail') {
    // Process recipients
    const recipients = sendParams.recipientsInput
      ? sendParams.recipientsInput.split(',').map(e => e.trim()).filter(Boolean)
      : []

    const bodyObj = {
      accountId: sendParams.accountId || 0,
      name: sendParams.name || undefined,
      receiveEmail: recipients,
      subject: sendParams.subject,
      text: sendParams.text,
      content: sendParams.content,
      attachments: []
    }

    const bodyStr = JSON.stringify(bodyObj, null, 2)
    return `curl -X POST "${apiBaseUrl}/email/send" \\\n` +
           `  -H "Content-Type: application/json" \\\n` +
           `${tokenHeader} \\\n` +
           `  -d '${bodyStr.replace(/'/g, "'\\''")}'`
  }

  return ''
})

// Clipboard Helper
function copyText(text, type) {
  if (!text) return
  navigator.clipboard.writeText(text).then(() => {
    copiedType.value = type
    ElMessage({
      message: isZh.value ? '复制成功！' : 'Copied successfully!',
      type: 'success',
      plain: true
    })
    setTimeout(() => {
      copiedType.value = ''
    }, 2000)
  }).catch(() => {
    ElMessage({
      message: isZh.value ? '复制失败，请手动选择复制。' : 'Copy failed, please copy manually.',
      type: 'error',
      plain: true
    })
  })
}
</script>

<style scoped lang="scss">
.api-container {
  height: 100%;
  width: 100%;
  padding: 0;
  
  .scroll {
    height: 100%;
    width: 100%;
  }

  .scroll-body {
    padding: 30px 40px;
    max-width: 1200px;
    margin: 0 auto;
    
    @media (max-width: 767px) {
      padding: 20px;
    }
  }
}

.api-header {
  display: flex;
  align-items: center;
  gap: 20px;
  margin-bottom: 30px;
  
  .header-icon {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 64px;
    height: 64px;
    border-radius: 16px;
    background: linear-gradient(135deg, rgba(71, 136, 107, 0.15), rgba(53, 147, 105, 0.3));
    color: var(--el-color-primary);
    border: 1px solid rgba(71, 136, 107, 0.2);
    box-shadow: 0 4px 12px rgba(71, 136, 107, 0.08);
  }

  .header-text {
    h2 {
      margin: 0 0 6px 0;
      font-size: 24px;
      font-weight: 700;
      color: var(--el-text-color-primary);
    }
    p {
      margin: 0;
      font-size: 14px;
      color: var(--secondary-text-color);
    }
  }
}

.card-grid {
  display: flex;
  flex-direction: column;
  gap: 25px;
}

.settings-card {
  background: var(--extra-light-fill);
  border: 1px solid var(--light-border);
  border-radius: 12px;
  padding: 24px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.02);
  backdrop-filter: blur(8px);
  transition: all 0.3s ease;
  
  &:hover {
    box-shadow: 0 6px 24px rgba(0, 0, 0, 0.04);
    border-color: var(--el-color-primary-light-5);
  }

  .card-title {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 16px;
    font-weight: 600;
    margin-bottom: 20px;
    color: var(--el-text-color-primary);
    border-bottom: 1px solid var(--light-border-color);
    padding-bottom: 12px;
    
    span {
      display: inline-block;
    }
  }
}

.alert-info {
  display: flex;
  gap: 12px;
  background: rgba(71, 136, 107, 0.06);
  border: 1px solid rgba(71, 136, 107, 0.15);
  border-radius: 8px;
  padding: 12px 16px;
  margin-bottom: 20px;
  color: var(--regular-text-color);
  font-size: 13.5px;
  align-items: flex-start;
  line-height: 1.6;
  
  .info-icon {
    color: var(--el-color-primary);
    flex-shrink: 0;
    margin-top: 2px;
  }
  p {
    margin: 0;
  }
}

.token-display {
  display: flex;
  flex-direction: column;
  gap: 8px;
  
  .token-label {
    font-size: 13px;
    font-weight: 600;
    color: var(--secondary-text-color);
  }

  .token-input-group {
    display: flex;
    gap: 12px;
    
    @media (max-width: 576px) {
      flex-direction: column;
    }
    
    .token-input {
      flex-grow: 1;
      font-family: monospace;
    }
  }
}

.constructor-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 24px;
  
  @media (max-width: 1024px) {
    grid-template-columns: 1fr;
  }
}

.config-panel {
  display: flex;
  flex-direction: column;
  
  .config-desc {
    font-size: 13px;
    color: var(--secondary-text-color);
    line-height: 1.6;
    margin-bottom: 15px;
  }

  .input-tip {
    font-size: 12px;
    color: var(--secondary-text-color);
    margin-top: 4px;
  }

  .permission-warning {
    display: flex;
    align-items: center;
    gap: 8px;
    background: rgba(230, 162, 60, 0.1);
    border: 1px solid rgba(230, 162, 60, 0.2);
    border-radius: 6px;
    padding: 10px 14px;
    color: #e6a23c;
    font-size: 13px;
    margin-bottom: 15px;
    line-height: 1.4;
    
    .warn-icon {
      flex-shrink: 0;
    }
  }

  .disabled-form-mask {
    opacity: 0.5;
    pointer-events: none;
    user-select: none;
  }
}

.code-panel {
  display: flex;
  flex-direction: column;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid var(--light-border);
  background: #1e1e1e;
  height: 100%;
  min-height: 320px;
  
  .code-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    background: #2d2d2d;
    padding: 8px 16px;
    border-bottom: 1px solid #3d3d3d;
    
    .code-title {
      font-size: 12px;
      font-weight: 600;
      color: #9cdcfe;
      font-family: monospace;
    }
  }

  .code-body {
    flex-grow: 1;
    padding: 16px;
    overflow: auto;
    font-family: "Fira Code", Consolas, Monaco, "Andale Mono", "Ubuntu Mono", monospace;
    font-size: 13px;
    line-height: 1.5;
    color: #d4d4d4;
    
    pre {
      margin: 0;
      white-space: pre-wrap;
      word-break: break-all;
    }
  }
}

.api-doc-item {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 10px;
  margin-bottom: 10px;
  
  .api-route-tag {
    font-size: 11px;
    font-weight: 700;
    padding: 2px 8px;
    border-radius: 4px;
    text-transform: uppercase;
    font-family: monospace;
    
    &.get {
      background: rgba(64, 158, 255, 0.12);
      color: #409eff;
      border: 1px solid rgba(64, 158, 255, 0.2);
    }
    
    &.post {
      background: rgba(103, 194, 58, 0.12);
      color: #67c23a;
      border: 1px solid rgba(103, 194, 58, 0.2);
    }
  }

  .api-route-path {
    font-family: monospace;
    font-size: 15px;
    font-weight: 600;
    color: var(--el-text-color-primary);
  }
}

.api-doc-desc {
  font-size: 13.5px;
  color: var(--regular-text-color);
  line-height: 1.6;
  margin-bottom: 24px;
}

.section-sub-title {
  font-size: 14px;
  font-weight: 600;
  margin: 20px 0 10px 0;
  color: var(--el-text-color-primary);
  border-left: 3px solid var(--el-color-primary);
  padding-left: 8px;
}

.doc-table {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 20px;
  font-size: 13px;
  
  th, td {
    padding: 10px 12px;
    text-align: left;
    border-bottom: 1px solid var(--light-border-color);
  }
  
  th {
    font-weight: 600;
    background: rgba(0, 0, 0, 0.02);
    color: var(--el-text-color-primary);
  }
  
  td {
    color: var(--regular-text-color);
  }
}

.code-block {
  background: #1e1e1e;
  border-radius: 6px;
  padding: 16px;
  overflow: auto;
  font-family: monospace;
  font-size: 13px;
  color: #d4d4d4;
  max-height: 250px;
  border: 1px solid var(--light-border);
  
  pre {
    margin: 0;
  }
}

/* Dark mode overrides */
.dark {
  .settings-card {
    background: var(--extra-light-fill);
    border-color: var(--light-border);
    box-shadow: none;
    
    &:hover {
      border-color: var(--el-color-primary-light-5);
    }
  }
  
  .alert-info {
    background: rgba(71, 136, 107, 0.04);
    border-color: rgba(71, 136, 107, 0.1);
  }
  
  .code-panel {
    border-color: #333;
  }
  
  .doc-table {
    th {
      background: rgba(255, 255, 255, 0.02);
    }
  }
}
</style>
