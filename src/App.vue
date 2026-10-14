<script setup lang="ts">
import { ref, computed } from 'vue'
import liff from '@line/liff'

// 取貨明細資料 (預設 item 顯示一項，時間預設為 YYYY-MM-DDTHH:mm 格式相容 datetime-local)
const pickupNotification = ref({
  items: [{ name: '', quantity: 1 }],
  pickupTime: '',
  paidAmount: null,
})

// 格式化取件時間給 LINE Flex Message 顯示 (例如: 2026/10/12 18:30)
const formattedPickupTime = computed(() => {
  const timeStr = pickupNotification.value.pickupTime
  if (!timeStr) return ''
  return timeStr.replace('T', ' ').replace(/-/g, '/')
})

// 新增明細項目
const addItem = () => {
  pickupNotification.value.items.push({ name: '', quantity: 1 })
}

// 移除明細項目 (至少保留一項)
const removeItem = (index: number) => {
  if (pickupNotification.value.items.length > 1) {
    pickupNotification.value.items.splice(index, 1)
  }
}

// 驗證是否所有欄位皆已填寫
const isFormValid = computed(() => {
  const { items, pickupTime, paidAmount } = pickupNotification.value

  // 檢查取件時間
  if (!pickupTime || !pickupTime.trim()) return false

  // 檢查付款金額
  if (
    paidAmount === null ||
    paidAmount === undefined ||
    isNaN(Number(paidAmount)) ||
    Number(paidAmount) < 0
  ) {
    return false
  }

  // 檢查品項清單
  if (!items || items.length === 0) return false

  for (const item of items) {
    if (!item.name || !item.name.trim()) return false
    if (item.quantity === null || item.quantity === undefined || Number(item.quantity) <= 0)
      return false
  }

  return true
})

// 顯示目標選擇器（用於選擇群組或好友的畫面），並將開發者所創建的消息發送到選定的目標
const shareTargetPicker = () => {
  if (!isFormValid.value) return

  const { items, paidAmount } = pickupNotification.value
  const displayTime = formattedPickupTime.value

  liff
    .shareTargetPicker([
      {
        type: 'flex',
        altText: '【買家已取貨通知】買家已完成取件！',
        contents: {
          type: 'bubble',
          hero: {
            type: 'image',
            url: 'https://images.unsplash.com/photo-1549465220-1a8b9238cd48?auto=format&fit=crop&w=800&q=80',
            size: 'full',
            aspectRatio: '20:13',
            aspectMode: 'cover',
          },
          body: {
            type: 'box',
            layout: 'vertical',
            contents: [
              {
                type: 'text',
                text: '買家已取貨通知 📦',
                weight: 'bold',
                size: 'xl',
                color: '#1DB446',
              },
              {
                type: 'text',
                text: '買家已順利完成取件，清單資訊如下：',
                size: 'xs',
                color: '#8c8c8c',
                margin: 'sm',
              },
              {
                type: 'separator',
                margin: 'lg',
              },
              // 商品清單區塊 (動態生成品名 + 數量)
              {
                type: 'box',
                layout: 'vertical',
                margin: 'lg',
                spacing: 'sm',
                contents: [
                  {
                    type: 'text',
                    text: '商品明細',
                    size: 'xs',
                    color: '#aaaaaa',
                    weight: 'bold',
                  },
                  ...items.map(item => ({
                    type: 'box' as const,
                    layout: 'horizontal' as const,
                    contents: [
                      {
                        type: 'text' as const,
                        text: item.name,
                        size: 'sm' as const,
                        color: '#333333',
                        flex: 4,
                        wrap: true,
                      },
                      {
                        type: 'text' as const,
                        text: `x ${item.quantity}`,
                        size: 'sm' as const,
                        color: '#666666',
                        align: 'end' as const,
                        flex: 1,
                      },
                    ],
                  })),
                ],
              },
              {
                type: 'separator',
                margin: 'lg',
              },
              // 取件時間與付款狀態
              {
                type: 'box',
                layout: 'vertical',
                margin: 'lg',
                spacing: 'sm',
                contents: [
                  {
                    type: 'box',
                    layout: 'baseline',
                    contents: [
                      {
                        type: 'text',
                        text: '取件時間',
                        color: '#aaaaaa',
                        size: 'sm',
                        flex: 2,
                      },
                      {
                        type: 'text',
                        text: displayTime,
                        wrap: true,
                        color: '#333333',
                        size: 'sm',
                        flex: 4,
                      },
                    ],
                  },
                  {
                    type: 'box',
                    layout: 'baseline',
                    contents: [
                      {
                        type: 'text',
                        text: '付款狀態',
                        color: '#aaaaaa',
                        size: 'sm',
                        flex: 2,
                      },
                      {
                        type: 'text',
                        text: `已付款（NT$${Number(paidAmount).toLocaleString('zh-TW')}）`,
                        wrap: true,
                        color: '#1DB446',
                        size: 'sm',
                        flex: 4,
                        weight: 'bold',
                      },
                    ],
                  },
                ],
              },
            ],
          },
          footer: {
            type: 'box',
            layout: 'vertical',
            spacing: 'sm',
            contents: [
              {
                type: 'button',
                style: 'link',
                height: 'sm',
                action: {
                  type: 'uri',
                  label: '查看庫存',
                  uri: 'https://www.icloud.com/notes/05asjVt-XnFXvrOvG-8TLjJ_w',
                },
              },
            ],
            flex: 0,
          },
        },
      },
    ])
    .then(function (res) {
      if (res) {
        console.log(`[${res.status}] Message sent!`)
      } else {
        console.log('TargetPicker was closed!')
      }
    })
    .catch(function (error) {
      console.log('something wrong happen', error)
    })
}

const initLiff = async () => {
  try {
    await liff.init({ liffId: '1563584152-HhSGjZI9' })
    if (!liff.isLoggedIn()) liff.login()
  } catch (e) {
    console.error('liff.init error', e)
  }
}
initLiff()
</script>

<template>
  <div class="page-container">
    <div class="card">
      <header class="header">
        <h1 class="title">📦 取貨明細設定</h1>
        <p class="subtitle">請填妥以下取貨資訊，填寫完畢即可送出 LINE 訊息。</p>
      </header>

      <form @submit.prevent="shareTargetPicker" class="form">
        <!-- 取件時間 (時間選擇器) -->
        <div class="form-group">
          <label class="label" for="pickupTime"> 取件時間 <span class="required">*</span> </label>
          <input
            id="pickupTime"
            v-model="pickupNotification.pickupTime"
            type="datetime-local"
            class="input"
            required
          />
        </div>

        <!-- 付款金額 -->
        <div class="form-group">
          <label class="label" for="paidAmount">
            付款金額 (NT$) <span class="required">*</span>
          </label>
          <input
            id="paidAmount"
            v-model.number="pickupNotification.paidAmount"
            type="number"
            min="0"
            placeholder="請輸入付款金額"
            class="input"
            required
          />
        </div>

        <!-- 商品明細清單 -->
        <div class="form-group">
          <div class="items-header">
            <label class="label"> 商品明細 <span class="required">*</span> </label>
            <button type="button" class="btn btn-outline" @click="addItem">+ 新增品項</button>
          </div>

          <div class="items-list">
            <div v-for="(item, index) in pickupNotification.items" :key="index" class="item-row">
              <div class="item-field item-name">
                <input
                  v-model="item.name"
                  type="text"
                  placeholder="品項名稱"
                  class="input"
                  required
                />
              </div>
              <div class="item-field item-qty">
                <select v-model.number="item.quantity" class="input select-input" required>
                  <option v-for="n in 10" :key="n" :value="n">
                    {{ n }}
                  </option>
                </select>
              </div>
              <button
                v-if="pickupNotification.items.length > 1"
                type="button"
                class="btn btn-danger"
                aria-label="刪除品項"
                @click="removeItem(index)"
              >
                ✕
              </button>
            </div>
          </div>
        </div>

        <!-- 提示訊息 -->
        <p v-if="!isFormValid" class="warning-text">
          ⚠️ 尚有未填寫或格式不正確的欄位，請完整填寫後再送出。
        </p>

        <!-- 送出按鈕 -->
        <button type="submit" class="btn btn-primary" :disabled="!isFormValid">送出訊息</button>
      </form>
    </div>
  </div>
</template>

<style scoped>
:global(body) {
  margin: 0;
}

.page-container {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: flex-start;
  padding: 24px 16px;
  background-color: #0f172a;
  color: #f1f5f9;
  box-sizing: border-box;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
}

.card {
  width: 100%;
  max-width: 580px;
  background: #1e293b;
  border: 1px solid #334155;
  border-radius: 16px;
  box-shadow:
    0 10px 25px -5px rgba(0, 0, 0, 0.5),
    0 8px 10px -6px rgba(0, 0, 0, 0.4);
  padding: 32px 24px;
  box-sizing: border-box;
}

.header {
  margin-bottom: 24px;
}

.title {
  margin: 0;
  font-size: 1.5rem;
  font-weight: 700;
  color: #f8fafc;
}

.subtitle {
  margin: 6px 0 0;
  font-size: 0.875rem;
  color: #94a3b8;
}

.form {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.label {
  font-size: 0.9rem;
  font-weight: 600;
  color: #cbd5e1;
}

.required {
  color: #f87171;
}

.input {
  width: 100%;
  max-width: 100%;
  min-width: 0;
  padding: 10px 14px;
  font-size: 0.95rem;
  color: #f8fafc;
  background-color: #0f172a;
  border: 1px solid #475569;
  border-radius: 8px;
  box-sizing: border-box;
  outline: none;
  transition:
    border-color 0.2s,
    box-shadow 0.2s;
  color-scheme: dark;
  -webkit-appearance: none;
  appearance: none;
}

.select-input {
  cursor: pointer;
  padding-right: 14px;
}

/* 針對 datetime-local 在 WebKit / iOS LIFF 瀏覽器中的寬度修正 */
input[type='datetime-local'] {
  display: block;
  -webkit-appearance: none;
  appearance: none;
  min-height: 44px;
}

.input:focus {
  border-color: #10b981;
  box-shadow: 0 0 0 3px rgba(16, 185, 129, 0.2);
}

.input::placeholder {
  color: #64748b;
}

.items-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.items-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.item-row {
  display: flex;
  gap: 8px;
  align-items: center;
  width: 100%;
}

.item-name {
  flex: 1;
  min-width: 0;
}

.item-qty {
  width: 72px;
  flex-shrink: 0;
}

.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  border: none;
  outline: none;
}

.btn-outline {
  padding: 6px 12px;
  font-size: 0.825rem;
  background-color: transparent;
  color: #34d399;
  border: 1px solid #059669;
}

.btn-outline:hover {
  background-color: rgba(16, 185, 129, 0.15);
  border-color: #34d399;
}

.btn-danger {
  padding: 10px 12px;
  font-size: 0.95rem;
  background-color: rgba(239, 68, 68, 0.2);
  color: #f87171;
  border: 1px solid rgba(239, 68, 68, 0.3);
  flex-shrink: 0;
}

.btn-danger:hover {
  background-color: rgba(239, 68, 68, 0.35);
}

.btn-primary {
  width: 100%;
  padding: 12px;
  font-size: 1rem;
  background-color: #059669;
  color: #ffffff;
  margin-top: 8px;
}

.btn-primary:hover:not(:disabled) {
  background-color: #10b981;
}

.btn-primary:disabled {
  background-color: #334155;
  color: #64748b;
  cursor: not-allowed;
  opacity: 0.7;
}

.warning-text {
  margin: 0;
  font-size: 0.85rem;
  color: #fbbf24;
}

/* 響應式調整 */
@media (max-width: 480px) {
  .card {
    padding: 20px 14px;
  }

  .item-qty {
    width: 68px;
  }

  .btn-danger {
    padding: 10px 10px;
  }
}
</style>
