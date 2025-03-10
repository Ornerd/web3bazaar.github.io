<template>
  <section class="create-new-trade darker">
    <v-container>
      
        <div class="trade_header">
          <h1 class="gradient-text">Create a trade</h1>
        </div>
      <v-row  v-if="isWalletConnected">
        <v-col >
          <trade-list-wrapper class="trade_header" :new-trade="true" :disabled="loadingBtn" />
        </v-col>
        <v-col cols="12" class="trade_button_area">
          <ui-action-btn
            v-if="isSelectedContractApproved"
            class="mb-8"
            :loading="loadingBtn"
            :btn-text="ADD_TRADE"
            @click="newTrade"
          >
          </ui-action-btn>

          <ui-action-btn
            v-else
            class="mb-8"
            :loading="loadingBtn"
            :btn-text="APPROVE"
            @click="approveSelectedContract"
          >
          </ui-action-btn>
        </v-col>
      </v-row>

      <history-section class="mt-14"> </history-section>
    </v-container>
  </section>
</template>

<script>
import { ethers } from 'ethers'
import { mapState, mapActions } from 'vuex'

const createNewTrade = {
  log: require('debug')('w3b:view:createNewTrade'),
  error: require('debug')('w3b:view:error:createNewTrade'),
}

const ADD_TRADE = 'Create Trade'
const APPROVE = 'Approve Contract'

const TRADE_TYPE = {
  NON: 0,
  ERC20: 1,
  ERC1155: 2,
  ERC721: 3,
  NATIVE: 4,
}

export default {
  data() {
    return {
      ADD_TRADE,
      APPROVE,
      contractsToApprove: [],
      loadingBtn: false,
    }
  },
  async fetch() {
    // await this.$store.dispatch('trader/GET_PROJECT_DATA')
  },
  computed: {
    ...mapState('connector', ['isWalletConnected', 'account']),
    ...mapState('trader', [
      'executorAddress',
      'creatorSelectedAssets',
      'executorSelectedAssets',
    ]),
    isSelectedContractApproved() {
      return this.contractsToApprove.length === 0
    },
  },
  watch: {
    async creatorSelectedAssets(val) {
      createNewTrade.log('creatorSelectedAssets watch', val)
      this.loadingBtn = true
      this.contractsToApprove = []
      for (const project in val) {
        const {
          token_address: creatorAssetContract,
          contract_type: creatorAssetType,
        } = val[project][0] || {}
        createNewTrade.log(
          'creatorAssetContract || !creatorAssetType watch',
          creatorAssetContract || creatorAssetType
        )

        if (!creatorAssetContract || !creatorAssetType) continue
        const isApproved = await this.checkIfContractIsApprovedForWallet(
          creatorAssetContract,
          creatorAssetType,
          this.account
        )
        createNewTrade.log('isApproved watch', isApproved)

        if (!isApproved) {
          this.contractsToApprove.push({
            creatorAssetContract,
            creatorAssetType,
          })
        }
      }
      this.loadingBtn = false
      createNewTrade.log('contractsToApprove watch', this.contractsToApprove)
    },
  },
  mounted() {
    this.$store.commit(`trader/resetSelectedAssets`)
  },
  methods: {
    ...mapActions('sound', ['playSFXAudio']),

    async checkIfContractIsApprovedForWallet(
      contractAddress,
      contractType,
      walletAddress
    ) {
      let result
      try {
        result = await this.$store.dispatch('bazaar-connector/isApproved', {
          contractAddress,
          contractType,
          walletAddress,
        })
        createNewTrade.log('isApproved', result)
      } catch (error) {
        createNewTrade.error('error', error)
      }

      return result
    },

    async setApproval(
      contractAddress,
      contractType,
      walletAddress,
      approveValue
    ) {
      let result
      try {
        this.$store.commit('modals/setPopupState', {
          type: 'loading',
          isShow: true,
          data: {
            state: 'approveContract',
          },
        })

        result = await this.$store.dispatch('bazaar-connector/setApproval', {
          contractAddress,
          contractType,
          walletAddress,
          approveValue,
        })
        if (result === 'REJECTED') {
          return result
        }
      } catch (error) {
        if (error?.message === 'REJECTED') {
          return error
        }
        createNewTrade.error('error', error)
        throw new Error(error)
      }

      return result
    },

    async removeApprove() {
      if (!this.creatorSelectedAssets) return
      const {
        token_address: creatorAssetContract,
        contract_type: creatorAssetType,
      } = this.creatorSelectedAssets.find((asset) => asset.selected > 0)

      const res = await this.setApproval(
        creatorAssetContract,
        creatorAssetType,
        this.account,
        false
      )
      return res
    },

    async approveSelectedContract() {
      createNewTrade.log('contractsToApprove', this.contractsToApprove)
      try {
        if (this.contractsToApprove.length === 0) return

        const promises = []

        for (let i = 0; i < this.contractsToApprove.length; i++) {
          const { creatorAssetContract, creatorAssetType } =
            this.contractsToApprove[i] || {}

          if (!creatorAssetContract || !creatorAssetType) return

          this.loadingBtn = true
          this.$store.commit('modals/setPopupState', {
            type: 'loading',
            isShow: true,
          })

          const isApproved = await this.checkIfContractIsApprovedForWallet(
            creatorAssetContract,
            creatorAssetType,
            this.account
          )
          createNewTrade.log('isApproved', isApproved)

          if (!isApproved) {
            promises.push(
              this.setApproval(
                creatorAssetContract,
                creatorAssetType,
                this.account
              )
            )
          }
        }

        const promiseResults = await Promise.all(promises)
        promiseResults.forEach((result) => {
          if (result === 'REJECTED' || result?.message === 'REJECTED') {
            throw new Error('REJECTED')
          }
        })
        this.$store.commit('modals/closeModal')
        this.$store.commit('modals/setPopupState', {
          type: 'success',
          isShow: true,
          data: {
            message: `You have successfully approved Bazaar contracts to move your assets.
                You can now Create the Trade and write it on the blockchain.`,
            skipRedirectToMainSquare: true,
          },
        })
        this.contractsToApprove = []
        this.loadingBtn = false
      } catch (error) {
        createNewTrade.error('approveSelectedContract error', error)
        this.$store.commit('modals/closeModal')
        if (error?.message === 'REJECTED') {
          this.$store.commit('modals/setPopupState', {
            type: 'error',
            isShow: true,
            data: {
              message: `It seems like the transaction didn't go through.
                Please try canceling the trade again`,
            },
          })
        }
        this.loadingBtn = false
      }
    },
    async newTrade() {
      try {
        createNewTrade.log('creatorSelectedAssets', this.creatorSelectedAssets)
        createNewTrade.log(
          'executorSelectedAssets',
          this.executorSelectedAssets
        )

        if (!this.creatorSelectedAssets) return
        if (Object.keys(this.executorSelectedAssets).length === 0) return

        const creatorObject = {
          creatorAssetContract: [],
          creatorAssetId: [],
          creatorAmount: [],
          creatorAssetType: [],
        }

        // const {
        //   token_address: creatorAssetContract,
        //   token_id: creatorAssetId,
        //   chosenAmount: creatorAmount,
        //   contract_type: creatorAssetType,
        // } = this.creatorSelectedAssets.filter((asset) => asset.selected > 0)
        for (const project in this.creatorSelectedAssets) {
          this.creatorSelectedAssets[project].forEach((asset) => {
            if (asset.selected) {
              const amount =
                asset.contract_type === 'ERC20'
                  ? ethers.utils.parseEther(asset.chosenAmount.toString())
                  : asset.chosenAmount

              creatorObject.creatorAssetContract.push(asset.token_address)
              creatorObject.creatorAssetId.push(asset.token_id)
              creatorObject.creatorAmount.push(amount)
              creatorObject.creatorAssetType.push(
                TRADE_TYPE[asset.contract_type]
              )
            }
          })
        }
        createNewTrade.log('creatorObject', creatorObject)

        // const {
        //   token_address: executorAssetContract,
        //   token_id: executorAssetId,
        //   chosenAmount: executorAmount,
        //   contract_type: executorAssetType,
        // } = this.executorSelectedAssets.find((asset) => asset.selected > 0)

        const executorWalletAdd = this.executorAddress

        const executorObject = {
          executorAssetContract: [],
          executorAssetId: [],
          executorAmount: [],
          executorAssetType: [],
          executorWalletAdd,
        }
        for (const project in this.executorSelectedAssets) {
          this.executorSelectedAssets[project].forEach((asset) => {
            if (asset.selected) {
              const amount =
                asset.contract_type === 'ERC20'
                  ? ethers.utils.parseEther(asset.chosenAmount.toString())
                  : asset.chosenAmount

              executorObject.executorAssetContract.push(asset.token_address)
              executorObject.executorAssetId.push(asset.token_id)
              executorObject.executorAmount.push(amount)
              executorObject.executorAssetType.push(
                TRADE_TYPE[asset.contract_type]
              )
            }
          })
        }
        createNewTrade.log('executorObject', executorObject)

        if (
          creatorObject.creatorAmount.length === 0 ||
          executorObject.executorAmount.length === 0
        )
          return

        this.loadingBtn = true

        this.$store.commit('modals/setPopupState', {
          type: 'loading',
          isShow: true,
        })

        const startTrade = await this.$store.dispatch(
          'bazaar-connector/startTrade',
          {
            ...creatorObject,
            ...executorObject,
          }
        )

        this.$store.commit('modals/setPopupState', {
          type: 'loading',
          isShow: true,
          data: {
            state: 'mining',
          },
        })
        await this.$nextTick()

        await startTrade.wait()
        // .catch((e) => {
        //   createNewTrade.error(e)

        //   if (e.code === 3) {
        //     createNewTrade.error(e.message)
        //   }
        // })

        // TODO: remove token from the users lists (update list?? )
        createNewTrade.log('startTrade ', startTrade)
        if (startTrade) {
          createNewTrade.log('Trade successfully created')

          // this.$store.commit('modals/closeModal')

          this.$store.commit('modals/setPopupState', {
            type: 'success',
            isShow: true,
            data: {
              txHash: startTrade.hash,
              message:
                'Your trade is now open in the Bazaar. Check its status in the Main Square.',
            },
          })
          this.playSFXAudio({ audioToPlay: 'successState' })
        }
      } catch (error) {
        createNewTrade.log('error', error)
        this.$store.commit('modals/closeModal')
        this.$store.commit('modals/setPopupState', {
          type: 'error',
          isShow: true,
          data: {},
        })
      }
      this.loadingBtn = false
    },
  },
}
</script>

<style>
  .create-new-trade {
    padding-top:8%;
    padding-bottom:8%;
    padding-left:5%;
    padding-right:5%;
  }

  .trade_header {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-bottom:3%;
  }

  .trade_button_area {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-top:1%;
  }

  @media screen and (max-width:960px) and (min-height:1070px) {
      .create-new-trade {
      height:60vh;
      padding-top:15%;
      padding-bottom:8%;
      padding-left:5%;
      padding-right:5%;
      }

  }

   @media screen and (max-width:799px) and (min-width:500.5px) {
      .create-new-trade {
        padding-top:15%;
        padding-bottom:8%;
        padding-left:5%;
        padding-right:5%;
      }
      .trade_button_area {
        display: flex;
        justify-content: center;
        align-items: center;
        margin-top:-10%;
      }
    }

  @media screen and (max-width:500px) {
      .trade_header {
        display: flex;
        justify-content: center;
        align-items: center;
        margin-top:20%;
        margin-bottom:-20%;
      }

      .trade_button_area {
        display: flex;
        justify-content: center;
        align-items: center;
        margin-top: 30%;
      }
  }

  </style>
