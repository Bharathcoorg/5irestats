<template>
  <div class="wallet-selector">
    <h2 class="text-center d-block">
      {{ $t('components.wallet_selector.title') }}
    </h2>
    <hr />
    <p class="text-center mb-4">
      {{ $t('components.wallet_selector.description') }}
    </p>
    <div v-if="loading">
      <Loading color="gray" />
    </div>
    <div v-else>
      <b-table
        striped
        :fields="fields"
        :items="extensionAccounts"
        class="account-table"
      >
        <template #cell(address)="data">
          <Identicon :address="data.item.address" :size="24" />
          <span v-if="data.item.name">
            {{ data.item.name }}<br />{{ shortAddress(data.item.address) }}
          </span>
          <span v-else>
            {{ shortAddress(data.item.address) }}
          </span>
        </template>
        <template #cell(selected)="data">
          <p class="text-right mb-0">
            <b-button
              variant="info"
              size="sm"
              @click="selectAddress(data.item.address)"
              >{{ $t('components.wallet_selector.connect') }}</b-button
            >
          </p>
        </template>
      </b-table>
    </div>
  </div>
</template>

<script>
import { web3Accounts, web3Enable } from '@polkadot/extension-dapp'
import { ApiPromise, WsProvider } from '@polkadot/api'
import { encodeAddress } from '@polkadot/keyring'
import Identicon from '@/components/Identicon.vue'
import Loading from '@/components/Loading.vue'
import commonMixin from '@/mixins/commonMixin.js'
import { config } from '@/frontend.config.js'

export default {
  components: {
    Identicon,
    Loading,
  },
  mixins: [commonMixin],
  data() {
    return {
      config,
      detectedExtension: false,
      extensionAccounts: [],
      extensionAddresses: [],
      api: null,
      enableWeb3: false,
      error: null,
      noAccountsFound: true,
      loading: true,
      fields: [
        {
          key: 'address',
          label: 'Account',
        },
        {
          key: 'role',
          label: 'Role',
        },
        {
          key: 'available',
          label: 'Available balance',
        },
        {
          key: 'selected',
          label: '',
        },
      ],
    }
  },
  async created() {
    this.enableWeb3 = await web3Enable(
      `${config.title} for ${this.capitalize(config.name)}`
    )
      .then(() => {
        web3Accounts()
          .then((accounts) => {
            const wsProvider = new WsProvider(config.nodeWs)
            ApiPromise.create({ provider: wsProvider }).then(async (api) => {
              this.api = api
              if (accounts.length > 0) {
                this.detectedExtension = true
                for (const account of accounts) {
                  const address = encodeAddress(
                    account.address,
                    config.ss58Format
                  )
                  const balances = await this.getAccountBalances(address)
                  const role = await this.getAddressRole(address)
                  if (
                    balances.availableBalance.gte(0) &&
                    role.includes('controller')
                  ) {
                    this.extensionAccounts.push({
                      address,
                      name: account.meta.name,
                      role,
                      available: this.formatAmount(balances.availableBalance),
                      selected: false,
                    })
                  }
                }
                if (
                  this.extensionAccounts.length > 0 &&
                  this.extensionAddresses.length > 0
                ) {
                  this.noAccountsFound = false
                } else {
                  this.noAccountsFound = true
                }
                this.loading = false
              }
            })
          })
          .catch((error) => {
            // eslint-disable-next-line
            console.log('Error: ', error)
          })
      })
      .catch((error) => {
        // eslint-disable-next-line
        console.log('Error: ', error)
      })
  },
  methods: {
    async getAccountBalances(address) {
      const balances = await this.api.derive.balances.all(address)
      return balances
    },
    async importChainValidatorAddresses(address) {
      const chainStaking = await this.api.query.staking.nominators(address)
      const staking = JSON.parse(JSON.stringify(chainStaking))
      if (staking?.targets.length > 0) {
        await this.$store.dispatch(
          'ranking/importChainValidatorAddresses',
          staking.targets
        )
      }
    },
    async getAddressRole(address) {
      const bonded = await this.api.query.staking.bonded(address)
      if (bonded.toString() && bonded.toString() === address) {
        return `stash/controller`
      } else if (bonded.toString() && bonded.toString() !== address) {
        return `stash`
      } else {
        const stakingLedger = await this.api.query.staking.ledger(address)
        if (stakingLedger.toString()) {
          return `controller`
        } else {
          return `none`
        }
      }
    },
    async selectAddress(address) {
      await this.importChainValidatorAddresses(address)
      await this.$store.dispatch('ranking/updateSelectedAddress', address)
      this.$emit('close')
      return true
    },
  },
}
</script>
