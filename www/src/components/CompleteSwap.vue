<template>
  <div class="full row-spaced">
    <v-dialog/>
    <sign-transaction-dialog modalName="commit-on-stellar" network="stellar">
      <span slot="title">
        COMMIT ON STELLAR
      </span>
      <div slot="description">
        <p>
          MirrorX needs your signature to commit your ETH tokens on Stellar.
        </p>
      </div>
    </sign-transaction-dialog>
    <sign-transaction-dialog modalName="commit-on-ethereum" network="ethereum">
      <span slot="title">
        COMMIT ON ETHEREUM
      </span>
      <div slot="description">
        <p>
          MirrorX wants to commit your Ether on Ethereum.
        </p>
      </div>
    </sign-transaction-dialog>
    <sign-transaction-dialog modalName="claim-on-ethereum" network="ethereum">
      <span slot="title">
        CLAIM ETHEREUM
      </span>
      <div slot="description">
        <p>
          MirrorX wants to claim your Ether on Ethereum.
        </p>
      </div>
    </sign-transaction-dialog>
    <sign-transaction-dialog modalName="claim-on-stellar" network="stellar">
      <span slot="title">
        CLAIM ON STELLAR
      </span>
      <div slot="description">
        <p>
          MirrorX needs your signature to claim your ETH tokens on Stellar.
        </p>
      </div>
    </sign-transaction-dialog>
    <div class="half">
      <div class="big-info">
        <div class="big-info__header">
          CONVERTING
        </div>
        <span v-if="reqInfo" class="big-info__swap-size">
          <span v-if="isWithdrawer">
            ?? XLM TO
          </span>
          <span>
            {{ swapSize || '..' }} {{ currency }}
          </span>
          <span v-if="isDepositor">
            TO ?? XLM
          </span>
        </span>
        <div class="big-info__label">
          {{ this.isWithdrawer ? 'TO' : 'FROM' }}:
        </div>
        <div class="big-info__account">
          <span v-if="reqInfo">
            {{ reqInfo.cryptoAddress }}
          </span>
          <span v-else>
            ..
          </span>
        </div>
        <div class="big-info__label">
          {{ this.isWithdrawer ? 'FROM' : 'TO' }}:
        </div>
        <div class="big-info__account">
          <span v-if="reqInfo">
            {{myStellarAccountTrunc}}
          </span>
          <span v-else>
            ..
          </span>
        </div>
      </div>
    </div>
    <div class="half col-spaced">
      <h3>Progress</h3>
      <swap-progress-log :status="status" :side="side"/>
    </div>
  </div>
</template>

<script>
  import SwapSpecs from '../../../lib/swapSpecs.mjs'
  import {Stellar} from '../../../lib/stellar.mjs'
  import {createHash} from 'crypto'

  import Status from '../util/swapStatus'

  export default {
    name: 'complete-swap',
    props: {
      currency: String,
      side: String,
      swapReqId: String,
      holdingSecret: String,
      preimage: String,
    },
    data() {
      let hashlock = null
      if (this.preimage) {
        const h = createHash('sha256')
        h.update(Buffer.from(this.preimage, 'hex'))
        hashlock = h.digest()
        console.log(`Hashlock=${hashlock.toString('hex')}, preimage=${this.preimage} `)
      }
      return {
        status: null,
        reqInfo: null,
        matchedInfo: null,
        preimageBuf: this.preimage || null,
        hashlock,
      }
    },
    mounted() {
      this.status = Status.RequestingSwapInfo
    },
    beforeRouteLeave(to, from, next) {
      if (this.checkMatchTimeout !== null) {
        clearInterval(this.checkMatchTimeout)
      }
      next()
    },
    methods: {
      async requestSwapInfo() {
        const {currency, swapReqId} = this
        const {data: {status, matchedInfo, reqInfo}} =
          await this.$client.get(`/swap/${currency}/match/${swapReqId}`)
        Object.assign(this, {reqInfo, matchedInfo})
        if (status !== 'matched') {
          this.status = Status.WaitingForMatch
          this.checkMatchTimeout = setTimeout(this.requestSwapInfo.bind(this), 5 * 1000)
          return
        }
        const {holdingAccount} = this.withdrawer
        Object.assign(this, {holdingAccount})
        this.status = Status.CommitOnStellar
      },
      displayError(e) {
        this.$modal.show('dialog', {
          title: 'Error',
          text: e.message || JSON.stringify(e),
        })
      },
      async findHoldingTx({wait}) {
        const {
          swapSize,
          holdingAccount,
          withdrawer: {stellarAccount: withdrawerAccount},
          depositor: {stellarAccount: depositorAccount},
        } = this
        try {
          return await this.swapSpec.findStellarHoldingTx({
            swapSize,
            withdrawerAccount,
            depositorAccount,
            holdingAccount,
            wait,
          })
        } catch (e) {
          this.displayError(e)
          throw e
        }
      },
      async findStellarCommitment() {
        let holdingTxInfo = await this.findHoldingTx({wait: false})
        if (!holdingTxInfo) {
          if (this.isWithdrawer) {
            await this.signStellarCommitment()
          }
          holdingTxInfo = await this.findHoldingTx({wait: true})
        }
        if (!this.hashlock) {
          const {hashlock} = holdingTxInfo
          Object.assign(this, {hashlock})
          console.log(`Found hashlock: ${hashlock.toString('hex')}`)
        }
        this.status = Status.CommitOnEthereum
      },
      async signStellarCommitment() {
        const {
          hashlock,
          swapSize,
          holdingAccount,
          depositor: {stellarAccount: depositorAccount},
          withdrawer: {stellarAccount: withdrawerAccount},
        } = this
        const {holdingTx} = await this.swapSpec.genStellarHoldingTx({
          hashlock,
          swapSize,
          holdingAccount,
          depositorAccount,
          withdrawerAccount,
        })
        holdingTx.sign(this.holdingKeys)
        const envelope = holdingTx.toEnvelope()
        const envelopeXdr = envelope.toXDR('base64')
        this.$modal.show('commit-on-stellar', {envelopeXdr})
      },
      async findEthereumCommitment() {
        this.$modal.hide('commit-on-stellar')
        let eventLog = await this.findPrepareSwapCall({wait: false})
        if (!eventLog) {
          if (this.isDepositor) {
            this.signEthereumCommit()
          }
          eventLog = await this.findPrepareSwapCall({wait: true})
        }
        this.status = Status.ClaimOnEthereum
      },
      signEthereumCommit() {
        const {
          hashlock,
          swapSize,
          withdrawer: {cryptoAddress: withdrawerEthAddress},
          depositor: {cryptoAddress: depositorEthAddress},
        } = this
        const {funcName, params} = this.swapSpec.prepareSwapParams({
          swapSize, hashlock, withdrawerEthAddress, depositorEthAddress,
        })
        this.$modal.show('commit-on-ethereum', {funcName, params})
      },
      async findPrepareSwapCall({wait}) {
        this.$modal.hide('commit-on-ethereum')
        const {
          hashlock,
          swapSize,
          withdrawer: {cryptoAddress: withdrawerEthAddress},
        } = this
        try {
          return await this.swapSpec.findPrepareSwap({
            hashlock, swapSize, withdrawerEthAddress, wait,
          })
        } catch (e) {
          this.displayError(e)
          throw e
        }
      },
      async findEthereumClaim() {
        let eventLog = await this.findFulfillSwapCall({wait: false})
        if (!eventLog) {
          if (this.isWithdrawer) {
            this.signEthereumClaim()
          }
          eventLog = await this.findFulfillSwapCall({wait: true})
        }
        if (!this.preimageBuf) {
          let {preimage} = eventLog
          preimage = Buffer.from(preimage.slice(2), 'hex')
          this.preimageBuf = preimage
          console.log(`Found preimage: ${preimage.toString('hex')}`)
        }
        this.status = Status.ClaimOnStellar
      },
      async findFulfillSwapCall({wait}) {
        const {hashlock} = this
        try {
          return await this.swapSpec.findFulfillSwap({
            hashlock, wait,
          })
        } catch (e) {
          this.displayError(e)
          throw e
        }
      },
      async signEthereumClaim() {
        const {
          preimage,
          withdrawer: {cryptoAddress: withdrawerEthAddress},
        } = this
        const {funcName, params} = this.swapSpec.fulfillSwapParams({preimage, withdrawerEthAddress})
        this.$modal.show('claim-on-ethereum', {funcName, params})
      },
      async findStellarClaim() {
        this.$modal.hide('claim-on-ethereum')
        let claimTx = await this.findStellarClaimTx({wait: false})
        if (!claimTx) {
          if (this.isDepositor) {
            await this.signStellarClaim()
          }
          claimTx = await this.findStellarClaimTx({wait: true})
        }
        this.status = Status.Done
        this.$modal.hide('claim-on-stellar')
      },
      async findStellarClaimTx({wait}) {
        const {
          holdingAccount,
          preimage,
          depositor: {stellarAccount: depositorAccount},
        } = this
        try {
          return await this.swapSpec.findStellarClaimTx({depositorAccount, holdingAccount, preimage, wait})
        } catch (e) {
          this.displayError(e)
          throw e
        }
      },
      async signStellarClaim() {
        const {
          preimageBuf: preimage,
          depositor: {stellarAccount: depositorAccount},
          holdingAccount,
        } = this
        const {claimTx} = await this.swapSpec.genStellarClaimTx({preimage, depositorAccount, holdingAccount})
        const envelopeXdr = claimTx.toEnvelope().toXDR('base64')
        this.$modal.show('claim-on-stellar', {envelopeXdr})
      },
    },
    computed: {
      swapSpec() {
        return SwapSpecs[this.currency]
      },
      swapSize() {
        return (this.reqInfo || {}).swapSize
      },
      myStellarAccountTrunc() {
        const {reqInfo: {stellarAccount: account}} = this
        return `${account.slice(0, 12)}...${account.slice(account.length - 12)}`
      },
      depositor() {
        if (this.isWithdrawer) {
          return this.matchedInfo || {}
        } else {
          return this.reqInfo || {}
        }
      },
      withdrawer() {
        if (this.isWithdrawer) {
          return this.reqInfo || {}
        } else {
          return this.matchedInfo || {}
        }
      },
      isWithdrawer() {
        return this.side === 'withdraw'
      },
      isDepositor() {
        return this.side === 'deposit'
      },
      holdingKeys() {
        if (!this.holdingSecret) {
          throw new Error('No holding secret')
        }
        return Stellar.Keypair.fromSecret(this.holdingSecret)
      },
    },
    watch: {
      status(newStatus, oldStatus) {
        if (newStatus === oldStatus) {
          return
        }
        console.log(`${this.side} status: ${oldStatus} -> ${newStatus}`)
        if (newStatus === Status.RequestingSwapInfo) {
          this.requestSwapInfo()
        } else if (newStatus === Status.CommitOnStellar) {
          this.findStellarCommitment()
        } else if (newStatus === Status.CommitOnEthereum) {
          this.findEthereumCommitment()
        } else if (newStatus === Status.ClaimOnEthereum) {
          this.findEthereumClaim()
        } else if (newStatus === Status.ClaimOnStellar) {
          this.findStellarClaim()
        } else if (newStatus === Status.Done) {
          console.log('Done!')
        }
      },
    },
  }
</script>
