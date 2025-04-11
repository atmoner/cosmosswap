<template>
  <h1>🔄 Cosmos Pro Swap</h1>

  <div id="wallet-connect" class="card">
    <button v-if="!walletAddress" class="button-primary" @click="connectWallet()">
      Connect Keplr
    </button>
    <button v-else class="button-primary" @click="sendToken()">Send token</button>
    <div v-if="walletAddress !== ''" id="wallet-info" style="margin-top: 12px">
      <div>
        <span>Connected: </span>
        <span id="wallet-address">({{ getKey.name }}) {{ walletAddress }}</span>
      </div>
    </div>
    <div v-if="walletAddress === ''" id="wallet-info" style="margin-top: 12px">
      <span>Connect to Keplr wallet to start swapping.</span>
    </div>
  </div>

  <div class="grid-container">
    <div class="main-content">
      <div class="card">
        <div class="form-group">
          <label>Select Liquidity Pool</label>
          <!-- <select id="pool-list" class="form-control" onchange="loadPoolDetails()"></select> -->
          <select v-model="selectedPool" id="pool-list" class="form-control">
            <option disabled value="">Please select one</option>
            <option v-for="(item, key) in poolList" :value="item.id" :key="key">
              {{ formatDenoms(item.reserve_coin_denoms) }}
            </option>
          </select>
        </div>

        <div class="swap-container">
          <div class="form-group">
            <label>From</label>
            <div class="token-selector">
              <select v-model="tokenA" id="from-denom" class="form-control"></select>
              <input
                type="number"
                id="from-amount"
                class="form-control"
                placeholder="0.0"
                step="any"
                oninput="calculatePrice()"
              />
            </div>
            <div class="balance-display">
              <span>Balance: <span id="from-balance">0.00</span></span>
              <span class="max-button" @click="setMaxBalance()">MAX</span>
            </div>
          </div>

          <div class="swap-direction-button" onclick="swapDirection()">↓↑</div>

          <div class="form-group">
            <label>To</label>
            <div class="token-selector">
              <select v-model="tokenB" id="to-denom" class="form-control"></select>
              <input type="text" id="to-amount" class="form-control" placeholder="0.0" readonly />
            </div>
            <div class="balance-display">
              <span>Balance: <span id="to-balance">0.00</span></span>
            </div>
          </div>
        </div>

        <div class="form-group">
          <div class="balance-display">
            <span>Price Impact:</span>
            <span id="price-impact">0.00%</span>
          </div>
          <div id="price-impact-warning" class="price-impact-warning"></div>
          <div class="balance-display">
            <span>Fee:</span>
            <span id="swap-fee">0.00</span>
          </div>
        </div>

        <button class="button-primary" @click="executeSwap()" style="width: 100%">
          Execute Swap
        </button>
      </div>
    </div>

    <div class="card">
      <h3>Pool Information</h3>
      <div id="pool-balances" style="margin-top: 12px"></div>
    </div>
  </div>

  <div id="status"></div>
</template>

<script>
import { watch, defineComponent, ref } from 'vue'

import {
  defaultRegistryTypes,
  assertIsDeliverTxSuccess,
  SigningStargateClient, 
  GasPrice,
  calculateFee,
} from '@cosmjs/stargate'
import { Registry } from '@cosmjs/proto-signing'
import { coins } from '@cosmjs/stargate'

import { liquidity, liquidityProtoRegistry } from 'liquidityjs';

export default defineComponent({
  name: 'App',
  data: () => ({
    apiUrl: 'http://localhost:3000/lcd/arkh',
    rpcUrl: 'http://localhost:3000/rpc/arkh',
    chainId: 'arkh',
    addressPrefix: 'arkh',
    setChainSelected: 'arkh',
    setChainName: 'Arkham',
    setChainSymbol: 'ARKH',
    setChainDenom: 'uarkh',
    setChainDecimals: 6,
    setChainCoinGeckoId: 'arkham',
    setChainCoinLookup: {
      chainDenom: 'arkh',
      viewDenom: 'ARKH',
      addressPrefix: 'arkh',
    },
    setChainGasPrice: '0.025arkh',
    setChainFeeMultiplier: 1.2,
    setChainGasPriceStep: {
      low: 0.01,
      average: 0.025,
      high: 0.03,
    },
    decimalMultiplier: 1e6,
    walletAddress: '',
    getKey: null,
    poolList: [],
    selectedPool: null,
    userBalances: {},
    tokenA: '',
    tokenB: '',
    fromDenom: '',
    toDenom: '',
    fromAmount: 0,
    toAmount: 0,
    priceImpact: 0,
    swapFee: 0,
    fromBalance: 0,
    toBalance: 0,
    tokenRegistry: {
      uarkh: { symbol: 'ARKH', decimals: 6 },
      uatom: { symbol: 'ATOM', decimals: 6 },
      uosmo: { symbol: 'OSMO', decimals: 6 },
    },
  }),
  async mounted() {
    this.loadPools()
 
  },
  watch: {
    selectedPool(newVal, oldVal) {
      console.log('oldVal:', oldVal)
      if (newVal) {
        console.log('Selected Pool:', newVal)
        this.loadPoolDetails()
      }
    },
    tokenA(newVal) {
      this.updateSwapParams()
    },
    tokenB(newVal) {
      this.updateSwapParams()
    },
  },
  methods: {
    async connectWallet() {
      // Connect to Keplr wallet
      if (!window.getOfflineSigner) {
        console.error('Keplr is not installed')
        return
      }
      await window.keplr.experimentalSuggestChain({
        chainId: this.chainId,
        chainName: this.setChainName,
        rpc: this.rpcUrl,
        rest: this.apiUrl,
        bip44: {
          coinType: 118,
        },
        bech32Config: {
          bech32PrefixAccAddr: this.addressPrefix,
          bech32PrefixAccPub: this.addressPrefix + 'pub',
          bech32PrefixValAddr: this.addressPrefix + 'valoper',
          bech32PrefixValPub: this.addressPrefix + 'valoperpub',
          bech32PrefixConsAddr: this.addressPrefix + 'valcons',
          bech32PrefixConsPub: this.addressPrefix + 'valconspub',
        },
        currencies: [
          {
            coinDenom: this.setChainCoinLookup.viewDenom,
            coinMinimalDenom: this.setChainCoinLookup.chainDenom,
            coinDecimals: this.setChainDecimals,
            coinGeckoId: this.coingeckoId,
          },
        ],
        feeCurrencies: [
          {
            coinDenom: this.setChainCoinLookup.viewDenom,
            coinMinimalDenom: this.setChainCoinLookup.chainDenom,
            coinDecimals: this.setChainDecimals,
            coinGeckoId: this.coingeckoId,
            gasPriceStep: {
              low: this.setChainGasPriceStep.low,
              average: this.setChainGasPriceStep.average,
              high: this.setChainGasPriceStep.high,
            },
          },
        ],
        stakeCurrency: {
          coinDenom: this.setChainCoinLookup.viewDenom,
          coinMinimalDenom: this.setChainCoinLookup.chainDenom,
          coinDecimals: this.setChainDecimals,
          coinGeckoId: this.coingeckoId,
        },
      })

      await window.keplr.enable(this.chainId)
      const offlineSigner = await window.getOfflineSignerAuto(this.chainId)
      const accounts = await offlineSigner.getAccounts()
      const getKey = await window.keplr.getKey(this.chainId)
      console.log(getKey)
      console.log(accounts)
      this.getKey = getKey
      this.walletAddress = accounts[0].address

      await this.loadUserBalances()
      await this.loadPools()
    },
    async loadUserBalances() {
      try {
        const response = await fetch(
          this.apiUrl + `/cosmos/bank/v1beta1/balances/` + this.walletAddress,
        )
        const { balances } = await response.json()

        this.userBalances = balances.reduce((acc, coin) => {
          acc[coin.denom] = (coin.amount / this.decimalMultiplier).toFixed(6)
          return acc
        }, {})
      } catch (error) {
        console.error('Error loading user balances:', error)
        //showStatus(`Balance Load Error: ${error.message}`, 'error');
      }
    },
    async loadPools() {
      try {
        const response = await fetch(this.apiUrl + `/cosmos/liquidity/v1beta1/pools`)
        const { pools } = await response.json()
        this.poolList = pools
      } catch (error) {
        console.error('Error loading pools:', error)
        // showStatus(`Pool Load Error: ${error.message}`, 'error');
      }
    },
    async loadPoolDetails() {
      console.log(this.selectedPool)
      const poolId = document.getElementById('pool-list').value
      try {
        const poolRes = await fetch(this.apiUrl + `/cosmos/liquidity/v1beta1/pools/${poolId}`)
        const { pool } = await poolRes.json()

        const balancesRes = await fetch(
          this.apiUrl + `/cosmos/bank/v1beta1/balances/${pool.reserve_account_address}`,
        )
        const { balances } = await balancesRes.json()

        this.currentPool = {
          id: poolId,
          denoms: pool.reserve_coin_denoms,
          reserves: balances.reduce((acc, coin) => {
            acc[coin.denom] = (coin.amount / this.decimalMultiplier).toFixed(6)
            return acc
          }, {}),
        }

        this.updateTokenSelection()
        this.updateBalances()
        //calculatePrice();
      } catch (error) {
        console.error('Error loading pools:', error)
        //showStatus(`Pool Error: ${error.message}`, 'error');
      }
    },
    updateTokenSelection() {
      console.log(this.currentPool.denoms)
      const fromDenomSelect = document.getElementById('from-denom')
      const toDenomSelect = document.getElementById('to-denom')

      fromDenomSelect.innerHTML = this.currentPool.denoms
        .map(
          (denom) => `
                <option value="${denom}">${this.formatDenom(denom)}</option>
            `,
        )
        .join('')

      toDenomSelect.innerHTML = this.currentPool.denoms
        .map(
          (denom) => `
                <option value="${denom}">${this.formatDenom(denom)}</option>
            `,
        )
        .join('')
    },
    setMaxBalance() {
      const fromDenom = document.getElementById('from-denom').value
      document.getElementById('from-amount').value = this.userBalances[fromDenom] || 0
      this.calculatePrice()
    },

    calculatePrice() {
      try {
        const fromDenom = document.getElementById('from-denom').value
        const toDenom = document.getElementById('to-denom').value
        const fromAmount = parseFloat(document.getElementById('from-amount').value) || 0

        const fromReserve = parseFloat(this.currentPool.reserves[fromDenom])
        const toReserve = parseFloat(this.currentPool.reserves[toDenom])

        console.log('fromReserve:', this.currentPool.reserves)
        console.log('toReserve:', toReserve)
        console.log('fromAmount:', fromAmount)

        // Constant product formula
        const k = fromReserve * toReserve
        const newFrom = fromReserve + fromAmount
        const newTo = k / newFrom
        const output = toReserve - newTo
        const priceImpact = ((fromAmount / (fromReserve + fromAmount)) * 100).toFixed(2)
        const fee = (fromAmount * 0.003).toFixed(6)

        document.getElementById('to-amount').value = output.toFixed(6)
        document.getElementById('price-impact').textContent = `${priceImpact}%`
        document.getElementById('swap-fee').textContent = `${fee} ${this.formatDenom(fromDenom)}`
      } catch (error) {
        console.error('Error calculating price:', error)
        //showStatus(`Calculation Error: ${error.message}`, 'error');
      }
    },
    updateBalances() {
      const fromDenom = document.getElementById('from-denom').value
      const toDenom = document.getElementById('to-denom').value

      document.getElementById('from-balance').textContent = this.userBalances[fromDenom] || '0.00'
      document.getElementById('to-balance').textContent = this.userBalances[toDenom] || '0.00'

      // Update pool balances display
      document.getElementById('pool-balances').innerHTML = Object.entries(this.currentPool.reserves)
        .map(
          ([denom, amount]) => `
                    <div class="balance-display">
                        <span>${this.formatDenom(denom)}:</span>
                        <span>${amount}</span>
                    </div>
                `,
        )
        .join('')
    },

    // Fixed function added here
    updateSwapParams() {
      this.updateBalances()
      this.calculatePrice()
    },

    async executeSwap() {
      // Execute the swap transaction
    },
    formatDenom(denom) {
      return this.tokenRegistry[denom]?.symbol || denom
    },
    formatDenoms(denoms) {
      return denoms.map((d) => this.formatDenom(d)).join('/')
    },

    async executeSwap() {
      const registry = new Registry([...defaultRegistryTypes, ...liquidityProtoRegistry])
      await window.keplr.enable(this.chainId)
      const offlineSigner = await window.getOfflineSignerAuto(this.chainId)
      const client = await SigningStargateClient.connectWithSigner(this.rpcUrl, offlineSigner, {
        gasPrice: GasPrice.fromString(0.025 + this.setChainCoinLookup.chainDenom),
        registry: registry,
      })  

      console.log('client:', client)
 
      const fromDenom = this.tokenA
      const toDenom = this.tokenB
      const fromAmount = parseFloat(document.getElementById('from-amount').value) || 0
      const toAmount = parseFloat(document.getElementById('to-amount').value) || 0
      const fee = {
        amount: coins(0.025 * 1e6, this.setChainCoinLookup.chainDenom),
        gas: '200000',
      }


          const {
          swap
      } = liquidity.v1beta1.MessageComposer.withTypeUrl;
      let swapMessage = swap({
        swapRequesterAddress: this.walletAddress,
        poolId: this.selectedPool,
        swapTypeId: 1, // Assuming 1 is the swap type ID for a standard swap
        offerCoin: {
          denom: fromDenom,
          amount: "9,998249",
        },
        demandCoinDenom: toDenom,
        offerCoinFee: {
          denom: fromDenom,
          amount: (fromAmount * 0.003 * this.decimalMultiplier).toString(), // 0.3% fee
        },
        orderPrice: "0.01",
      });
      console.log('Swap Message:', swapMessage); 


      try {
        const result = await client.signAndBroadcast(
          this.walletAddress,
          [
          swapMessage
          ],
          "auto",
          '',
        )
        assertIsDeliverTxSuccess(result)
        console.log(result)
      } catch (error) {
        console.error(error)
      }


    },

    async sendToken() {
      const registry = new Registry([...defaultRegistryTypes, ...liquidityProtoRegistry])

      await window.keplr.enable(this.chainId)
      const offlineSigner = await window.getOfflineSignerAuto(this.chainId)

      const client = await SigningStargateClient.connectWithSigner(this.rpcUrl, offlineSigner, {
        gasPrice: GasPrice.fromString(0.025 + this.setChainCoinLookup.chainDenom),
        registry: registry,
      })

      console.log(client)

      const feeEstimation = await client.simulate(
        this.walletAddress,
        [
          {
            typeUrl: '/cosmos.bank.v1beta1.MsgSend',
            value: {
              fromAddress: this.walletAddress,
              toAddress: this.walletAddress,
              amount: coins(10000, this.setChainCoinLookup.chainDenom),
            },
          },
        ],
        'Simple arkh send Tokens',
      )
      const usedFee = calculateFee(
        Math.round(feeEstimation * this.setChainFeeMultiplier),
        GasPrice.fromString(0.025 + this.setChainCoinLookup.chainDenom),
      )
      this.gasFee = { fee: usedFee.amount[0].amount / 1000000, gas: usedFee.gas }
      const feeAmount = coins(usedFee.amount[0].amount, 'arkh')
      let finalFee = {
        amount: feeAmount,
        gas: usedFee.gas,
        //granter: this.store.setFeePayer,
      }
      try {
        const result = await client.signAndBroadcast(
          this.walletAddress,
          [
            {
              typeUrl: '/cosmos.bank.v1beta1.MsgSend',
              value: {
                fromAddress: this.walletAddress,
                toAddress: this.walletAddress,
                amount: coins(10000, this.setChainCoinLookup.chainDenom),
              },
            },
          ],
          finalFee,
          '',
        )
        assertIsDeliverTxSuccess(result)
        console.log(result)
      } catch (error) {
        console.error(error)
      }
    },
  },
})
</script>
