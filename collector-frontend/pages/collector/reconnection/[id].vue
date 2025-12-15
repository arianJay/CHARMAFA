<template>
  <div class="min-h-screen bg-gray-100">
    <header class="bg-white shadow-md no-print">
      <div class="container mx-auto px-4 py-4">
        <button
          @click="router.push(`collector/disconnected/${route.params.id}`)"
          class="text-blue-600 hover:text-blue-800 mb-2"
        >
          ← Back
        </button>
        <h1 class="text-2xl font-bold text-gray-800">Reconnection Service</h1>
      </div>
    </header>

    <main class="container mx-auto px-4 py-8">
      <div v-if="loading" class="text-center py-12">
        <p class="text-gray-600">Loading reconnection information...</p>
      </div>      <!-- Reconnection Form -->
      <div v-else-if="!reconnectionProcessed" class="max-w-2xl mx-auto">
        <div class="bg-white rounded-lg shadow p-6 space-y-6">
          <div>
            <h2 class="text-xl font-bold text-gray-800 mb-4">Reconnection Details</h2>
            
            <div class="space-y-4 mb-6">
              <div class="bg-gray-50 p-4 rounded-lg">
                <p class="text-sm text-gray-600">Account Name</p>
                <p class="font-semibold text-gray-900">
                  {{ reconnectionData?.member.fname }} {{ reconnectionData?.member.lname }}
                </p>
              </div>

              <div class="bg-gray-50 p-4 rounded-lg">
                <p class="text-sm text-gray-600">Account Number</p>
                <p class="font-semibold text-gray-900">{{ reconnectionData?.member.account_no }}</p>
              </div>              <div class="bg-blue-50 border-l-4 border-blue-500 p-4">
                <p class="text-sm text-gray-700">Reconnection Fee</p>
                <p class="text-2xl font-bold text-blue-600">₱{{ formatMoney(reconnectionFee) }}</p>
              </div>

              <div class="bg-yellow-50 border-l-4 border-yellow-500 p-4">
                <p class="text-sm text-gray-700">Previous Balance</p>
                <p class="text-xl font-bold text-red-600">₱{{ formatMoney(prevBalance) }}</p>
              </div>              <div v-if="prevBalance > 0" class="bg-red-50 border-l-4 border-red-500 p-4">
                <p class="text-sm text-red-800 font-semibold">❌ Balance Payment Required First</p>
                <p class="text-sm text-red-700">You must pay the remaining balance before reconnection is allowed.</p>
                <p class="text-sm text-red-700 mt-2">Please pay the balance first, then return for reconnection.</p>
              </div>              <div v-else class="bg-green-50 border-l-4 border-green-500 p-4">
                <p class="text-sm text-green-800 font-semibold">✅ Ready for Reconnection</p>
                <p class="text-sm text-green-700">No outstanding balance. You can proceed with reconnection fee payment.</p>
              </div>

              <!-- Shared Meter Information -->
              <div v-if="sharedMeterInfo && sharedMeterInfo.length > 1" class="bg-blue-50 border-l-4 border-blue-500 p-4">
                <p class="font-medium text-blue-800">🔗 Shared Meter: {{ reconnectionData?.member?.meter_no }}</p>
                <p class="text-sm text-blue-600 mb-2">Reconnection will affect all {{ sharedMeterInfo.length }} members sharing this meter:</p>
                <div class="text-sm space-y-1">
                  <div v-for="member in sharedMeterInfo" :key="member.account_no" 
                       :class="member.account_no === reconnectionData?.member?.account_no ? 'font-semibold text-blue-800' : 'text-blue-600'">
                    • {{ member.account_no }} - {{ member.full_name }} 
                    <span v-if="member.connection_status === 'Connected'" class="text-green-600">(✓ Connected)</span>
                    <span v-else class="text-red-600">(⚠ Disconnected)</span>
                  </div>
                </div>
              </div>

              <div class="bg-blue-50 border-l-4 border-blue-500 p-4">
                <p class="text-sm text-gray-700">Reconnection Fee</p>
                <p class="text-2xl font-bold text-blue-600">₱{{ formatMoney(reconnectionFee) }}</p>
              </div>
            </div>            <div v-if="prevBalance <= 0">
              <label class="block text-sm font-medium text-gray-700 mb-2">
                Cash Amount
              </label>
              <input
                v-model.number="cash"
                type="number"
                step="0.01"
                min="0"
                required
                class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                placeholder="Enter cash amount"
              />
            </div>

            <div v-if="cash > 0 && prevBalance <= 0" class="bg-gray-50 p-4 rounded-lg">
              <div class="flex justify-between">
                <span class="text-gray-700">Change:</span>
                <span :class="change >= 0 ? 'text-green-600' : 'text-red-600'" class="font-bold text-xl">
                  ₱{{ formatMoney(Math.abs(change)) }}
                </span>
              </div>
              <p v-if="change < 0" class="text-red-600 text-sm mt-2">
                Insufficient amount. Need ₱{{ formatMoney(Math.abs(change)) }} more.
              </p>
            </div>

            <div v-if="error" class="bg-red-50 border border-red-200 text-red-700 px-4 py-3 rounded-lg">
              {{ error }}
            </div>
          </div>          <div class="flex gap-4">
            <button
              v-if="prevBalance > 0"
              @click="goToBalancePayment"
              class="flex-1 bg-green-600 hover:bg-green-700 text-white font-semibold py-3 px-6 rounded-lg transition-colors"
            >
              Pay Balance Only (₱{{ formatMoney(prevBalance) }})
            </button>
            <button
              v-else
              @click="processReconnection"
              :disabled="processing || cash < reconnectionFee"
              class="flex-1 bg-green-600 hover:bg-green-700 text-white font-semibold py-3 px-6 rounded-lg transition-colors disabled:bg-gray-400"
            >
              <span v-if="processing">Processing...</span>
              <span v-else>Pay Reconnection Fee</span>
            </button>
            <button
              @click="router.push(`/collector/disconnected/${route.params.id}`)"
              class="flex-1 bg-gray-300 hover:bg-gray-400 text-gray-800 font-semibold py-3 px-6 rounded-lg transition-colors"
            >
              Cancel
            </button>
          </div>
        </div>
      </div>      <!-- Receipt -->
      <div v-else-if="reconnectionProcessed" class="max-w-3xl mx-auto">
        <div class="bg-white rounded-lg shadow p-6">
          <h2 class="text-xl font-bold text-green-600 mb-4 text-center">
            ✓ Reconnection Processed Successfully
          </h2>
          
          <div class="space-y-4 mb-6">
            <div class="flex justify-between">
              <span class="text-gray-700">Receipt Number:</span>
              <span class="font-semibold">{{ resultData?.or_number }}</span>
            </div>
            <div class="flex justify-between">
              <span class="text-gray-700">Reconnection Fee:</span>
              <span class="font-semibold">₱{{ formatMoney(resultData?.reconnection_fee) }}</span>
            </div>
            <div v-if="resultData?.prev_balance_paid > 0" class="flex justify-between">
              <span class="text-gray-700">Previous Balance Paid:</span>
              <span class="font-semibold">₱{{ formatMoney(resultData?.prev_balance_paid) }}</span>
            </div>
            <div class="flex justify-between">
              <span class="text-gray-700">Total Amount:</span>
              <span class="font-semibold">₱{{ formatMoney(resultData?.total_amount) }}</span>
            </div>
            <div class="flex justify-between">
              <span class="text-gray-700">Cash Received:</span>
              <span class="font-semibold">₱{{ formatMoney(resultData?.cash) }}</span>
            </div>
            <div class="flex justify-between">
              <span class="text-gray-700">Change:</span>
              <span class="font-semibold text-green-600">₱{{ formatMoney(resultData?.change) }}</span>
            </div>
          </div>          <!-- Bluetooth Printer Component -->
          <div class="mb-6">
            <BluetoothThermalPrinter 
              ref="bluetoothPrinter"
              @printed="onPrinted"
            />
          </div>

          <!-- Receipt Printing Section -->
          <div class="bg-gray-50 p-4 rounded-lg mb-6">
            <h4 class="font-semibold text-gray-800 mb-3">Receipt Printing</h4>
            
            <div class="grid grid-cols-1 md:grid-cols-3 gap-3 mb-3">
              <button 
                @click="printMemberCopy" 
                :disabled="printingMember || !bluetoothPrinter?.isConnected"
                class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-lg disabled:bg-gray-400"
              >
                {{ printingMember ? 'Printing...' : 'Print Consumer\'s Copy' }}
              </button>
              
              <button 
                @click="printOfficeCopy" 
                :disabled="printingOffice || !bluetoothPrinter?.isConnected"
                :class="memberCopyPrinted ? 'bg-green-600 hover:bg-green-700' : 'bg-orange-600 hover:bg-orange-700'"
                class="text-white px-4 py-2 rounded-lg disabled:bg-gray-400"
              >
                {{ printingOffice ? 'Printing...' : 'Print Office Copy' }}
              </button>
              
              <button 
                @click="printBothCopies" 
                v-if="bothCopiesPrinted"
                :disabled="printingMember || printingOffice"
                class="bg-purple-600 hover:bg-purple-700 text-white px-4 py-2 rounded-lg disabled:bg-gray-400"
              >
                🔄 Reprint Both
              </button>
            </div>
            
            <div class="text-sm text-gray-600 space-y-1">
              <div v-if="memberCopyPrinted" class="text-green-600">✓ Consumer's copy printed</div>
              <div v-if="officeCopyPrinted" class="text-green-600">✓ Office copy printed</div>
              <div v-if="!bluetoothPrinter?.isConnected" class="text-red-600">⚠ Please connect to Bluetooth printer first</div>
            </div>
          </div>

          <div class="flex gap-4">
            <button
              @click="router.push('/collector/dashboard')"
              class="flex-1 bg-gray-300 hover:bg-gray-400 text-gray-800 font-semibold py-3 px-6 rounded-lg transition-colors"
            >
              Back to Dashboard
            </button>
          </div>          <!-- Receipt Preview -->
          <div class="mt-6 bg-gray-50 rounded-lg p-4 font-mono text-xs">
            <div class="text-center mb-2">
              <div class="font-bold">CHARMAFA</div>
              <div>Water Association</div>
              <div v-if="memberCopyPrinted || officeCopyPrinted" class="text-sm font-bold text-blue-600">
                RECEIPT PREVIEW
              </div>
              <div class="border-b border-dashed border-gray-400 my-2"></div>
            </div><div class="space-y-1">
              <div>Account: {{ thermalReceiptData?.member?.account_no }}</div>
              <div>Name: {{ getMemberName(thermalReceiptData?.member) }}</div>
              <div>OR: {{ thermalReceiptData?.or_number }}</div>
              <div v-if="thermalReceiptData?.reference">Ref: {{ thermalReceiptData?.reference }}</div>
              <div v-if="thermalReceiptData?.billing_date">Billing Date: {{ formatDate(thermalReceiptData?.billing_date) }}</div>
              <div v-if="thermalReceiptData?.meter_reading">Meter: {{ thermalReceiptData?.meter_reading }}</div>
              <div v-if="thermalReceiptData?.cum_usage">Usage: {{ thermalReceiptData?.cum_usage }} CUM</div>
              <div class="border-b border-dashed border-gray-400 my-2"></div>
              <div>Reconnection Fee: ₱{{ formatMoney(thermalReceiptData?.reconnection_fee) }}</div>
              <div v-if="thermalReceiptData?.prev_balance_paid > 0">
                Previous Balance: ₱{{ formatMoney(thermalReceiptData?.prev_balance_paid) }}
              </div>
              <div class="font-bold">Total: ₱{{ formatMoney(thermalReceiptData?.total_amount) }}</div>
              <div>Cash: ₱{{ formatMoney(thermalReceiptData?.cash) }}</div>
              <div class="font-bold">Change: ₱{{ formatMoney(thermalReceiptData?.change) }}</div>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup>
import { useNotifications } from '~/composables/useNotifications'

definePageMeta({
  middleware: 'auth'
})

const route = useRoute()
const router = useRouter()
const api = useApi()
const authStore = useAuthStore()
const bluetoothPrinter = ref(null)

const loading = ref(false)
const processing = ref(false)
const error = ref('')
const cash = ref(0)
const reconnectionData = ref(null)
const reconnectionFee = ref(100)
const prevBalance = ref(0)
const reconnectionProcessed = ref(false)
const resultData = ref(null)
const thermalReceiptData = ref(null)
const sharedMeterInfo = ref([])

// Receipt printing states
const memberCopyPrinted = ref(false)
const officeCopyPrinted = ref(false)
const printingMember = ref(false)
const printingOffice = ref(false)

const change = computed(() => cash.value - reconnectionFee.value)
const canProceedWithReconnection = computed(() => prevBalance.value <= 0)

const bothCopiesPrinted = computed(() => 
  memberCopyPrinted.value && officeCopyPrinted.value
)

const formatMoney = (amount) => {
  return Number(amount || 0).toFixed(2)
}

const getMemberName = (member) => {
  if (!member) return ''
  return `${member.fname || ''} ${member.mname || ''} ${member.lname || ''}`.trim()
}

const loadReconnectionInfo = async () => {
  loading.value = true
  error.value = ''
  
  try {
    const data = await api(`/reconnection/${route.params.id}`)
    reconnectionData.value = data
    reconnectionFee.value = data.reconnection_fee
    prevBalance.value = data.prev_balance || 0
    
    // Load shared meter information
    if (data.member && data.member.meter_no) {
      try {
        const sharedMeterResponse = await api(`/members/shared-meter/${data.member.meter_no}`)
        sharedMeterInfo.value = sharedMeterResponse.shared_members || []
      } catch (err) {
        console.warn('Could not load shared meter info:', err)
        sharedMeterInfo.value = []
      }
    }
  } catch (err) {
    error.value = 'Failed to load reconnection information.'
    console.error(err)
  } finally {
    loading.value = false
  }
}

const processReconnection = () => {
  // Check if member has remaining balance
  if (prevBalance.value > 0) {
    error.value = `Cannot proceed with reconnection. Please pay the remaining balance of ₱${formatMoney(prevBalance.value)} first.`
    return
  }
  
  // Check if cash is sufficient for reconnection fee only
  if (cash.value < reconnectionFee.value) {
    error.value = `Insufficient cash amount. Reconnection fee required: ₱${formatMoney(reconnectionFee.value)}`
    return
  }

  // For reconnection with no balance, only pay reconnection fee
  confirmPayBalance(false)
}

const confirmPayBalance = async (payBalance) => {
  processing.value = true
  error.value = ''
  const { showSuccess } = useNotifications()
  
  try {
    const result = await api(`/reconnection/${route.params.id}`, {
      method: 'POST',
      body: {
        cash: cash.value,
        reconnection_fee: reconnectionFee.value,
        pay_balance: payBalance // Only pay balance if explicitly requested (false for reconnection fee only)
      }
    })
    
    resultData.value = result
    reconnectionProcessed.value = true
      // Prepare thermal receipt data
    thermalReceiptData.value = {
      member: {
        account_no: result.member.account_no,
        fname: result.member.fname,
        mname: result.member.mname,
        lname: result.member.lname,
        meter_no: result.member.meter_no
      },
      or_number: result.or_number,
      billing_date: result.payment?.billing_date,
      reference: result.payment?.reference,
      meter_reading: result.payment?.meter_reading,
      cum_usage: result.payment?.cum_usage,
      reconnection_fee: result.reconnection_fee,
      prev_balance_paid: payBalance ? result.prev_balance_paid : 0,
      remaining_balance: result.remaining_balance,
      total_amount: result.total_amount,
      cash: result.cash,
      change: result.change,
      processed_by: `${authStore.user?.fname} ${authStore.user?.lname}`
    }
    
    showSuccess('Reconnection processed successfully')
    
    // Set success flag for dashboard refresh
    sessionStorage.setItem('reconnectionSuccess', 'true')
  } catch (err) {
    error.value = err.response?.data?.message || 'Reconnection processing failed. Please try again.'
    console.error(err)
  } finally {
    processing.value = false
  }
}

const goToBalancePayment = () => {
  router.push(`/collector/payment/${route.params.id}?balanceOnly=true&disconnected=true`)
}

const printMemberCopy = async () => {
  if (!bluetoothPrinter.value?.isConnected) {
    error.value = 'Please connect to Bluetooth printer first'
    return
  }

  printingMember.value = true
  try {
    const receiptData = {
      ...thermalReceiptData.value,
      copyType: 'CONSUMER\'S COPY',
      copyNote: 'Please keep this receipt for your records'
    }
    
    const success = await bluetoothPrinter.value.printReceipt(receiptData)
    if (success) {
      memberCopyPrinted.value = true
      const { showSuccess } = useNotifications()
      showSuccess('Consumer\'s copy printed successfully!')
    } else {
      error.value = 'Failed to print consumer\'s copy'
    }
  } catch (err) {
    console.error('Print error:', err)
    error.value = 'Failed to print consumer\'s copy'
  } finally {
    printingMember.value = false
  }
}

const printOfficeCopy = async () => {
  if (!bluetoothPrinter.value?.isConnected) {
    error.value = 'Please connect to Bluetooth printer first'
    return
  }

  printingOffice.value = true
  try {
    const receiptData = {
      ...thermalReceiptData.value,
      copyType: 'OFFICE COPY',
      copyNote: 'For office records and filing'
    }
    
    const success = await bluetoothPrinter.value.printReceipt(receiptData)
    if (success) {
      officeCopyPrinted.value = true
      const { showSuccess } = useNotifications()
      showSuccess('Office copy printed successfully!')
    } else {
      error.value = 'Failed to print office copy'
    }
  } catch (err) {
    console.error('Print error:', err)
    error.value = 'Failed to print office copy'
  } finally {
    printingOffice.value = false
  }
}

const printBothCopies = async () => {
  await printMemberCopy()
  
  // Wait 2 seconds for member to cut the first receipt
  setTimeout(async () => {
    await printOfficeCopy()
  }, 2000)
}

const printBluetoothReceipt = async () => {
  // Legacy function - redirect to consumer's copy
  await printMemberCopy()
}

const onPrinted = () => {
  const { showSuccess } = useNotifications()
  showSuccess('Receipt printed successfully!')
}

onMounted(() => {
  loadReconnectionInfo()
})
</script>