<style scoped lang="less">
@import '../less/common.less';
</style>

<template>
  <div class="transaction-page explorer-page page">
    <div class="transaction-body explorer-body">
      <div class="container" v-if="!loading && transaction">
        <div class="explorer-card">
          <header>
            <h1>{{ isStaking ? 'Staking Transaction' : 'Transaction' }}</h1>
          </header>
          <div class="explorer-card-body">
            <table class="explorer-table">
              <tr v-if="isStaking">
                <td class="td-title">Type</td>
                <td>{{ transaction.type }}</td>
              </tr>
              <tr>
                <td class="td-title">ID</td>
                <td>{{ transaction.hash }}</td>
              </tr>
              <tr>
                <td class="td-title">Status</td>
                <td>{{ status }}</td>
              </tr>
              <tr v-if="!isStaking">
                <td class="td-title">Value</td>
                <td>{{ transaction.value | amount }}</td>
              </tr>

              <tr v-if="isStaking">
                <td class="td-title">Value</td>
                <td>
                  <vue-json-pretty :data="transaction.msg" />
                </td>
              </tr>
              <!-- <tr>
                <td class="td-title">Size (bytes)</td>
                <td>{{ transaction.bytes }}</td>
              </tr>-->
              <!-- <tr>
                <td class="td-title">Received Time</td>
                <td>{{ transaction.receivedTime }}</td>
              </tr>-->
              <tr>
                <td class="td-title">Timestamp</td>
                <td>
                  {{ (Number(transaction.timestamp) * 1000) | timestamp }}
                </td>
              </tr>
              <tr>
                <td class="td-title">
                  {{
                    transaction.shardID === transaction.toShardID
                      ? 'Shard ID'
                      : 'From Shard'
                  }}
                </td>
                <td>{{ transaction.shardID }}</td>
              </tr>
              <tr>
                <td class="td-title">Sender shard block</td>
                <td>
                  <router-link :to="'/block/' + transaction.blockHash">{{
                    Number(transaction.blockNumber)
                  }}</router-link>
                </td>
              </tr>
              <tr>
                <td class="td-title">From Address</td>
                <td>
                  <router-link
                    :to="'/address/' + transaction.from"
                    v-if="transaction.from"
                    >{{ transaction.from }}</router-link
                  >
                </td>
              </tr>
              <tr v-if="transaction.shardID !== transaction.toShardID">
                <td class="td-title">To Shard</td>
                <td>{{ transaction.toShardID }}</td>
              </tr>
              <tr v-if="receipt">
                <td class="td-title">Receiving shard block</td>
                <td>
                  <router-link :to="'/block/' + receipt.blockHash">{{
                    Number(receipt.blockNumber)
                  }}</router-link>
                </td>
              </tr>
              <tr v-if="!isStaking">
                <td class="td-title">To Address</td>
                <td>
                  <router-link
                    :to="'/address/' + transaction.to"
                    v-if="transaction.to"
                    >{{ transaction.to }}</router-link
                  >
                </td>
              </tr>

              <!--              <tr v-if="isStaking">-->
              <!--                <td class="td-title">Validator Address</td>-->
              <!--                <td>-->
              <!--                  <router-link-->
              <!--                    :to="'/address/' + transaction.validator"-->
              <!--                    v-if="transaction.validator"-->
              <!--                  >{{ transaction.validator }}</router-link>-->
              <!--                </td>-->
              <!--              </tr>-->
              <!--              <tr v-if="isStaking">-->
              <!--                <td class="td-title">Delegator Address</td>-->
              <!--                <td>-->
              <!--                  <router-link-->
              <!--                    :to="'/address/' + transaction.delegator"-->
              <!--                    v-if="transaction.delegator"-->
              <!--                  >{{ transaction.delegator }}</router-link>-->
              <!--                </td>-->
              <!--              </tr>-->

              <tr>
                <td class="td-title">Network Fee</td>
                <td>{{ normalizedGas() }} ONE</td>
              </tr>
              <tr v-if="sequence">
                <td class="td-title">Sequence</td>
                <td>{{ sequence }}</td>
              </tr>
            </table>

            <expand-panel>
              <table class="explorer-table">
                <tr></tr>
                <tr>
                  <td class="td-title">Data (Hex)</td>
                  <td>{{ transaction.input || '-' }}</td>
                </tr>
                <tr>
                  <td class="td-title">Data (UTF-8)</td>
                  <td>{{ hexToUTF8(transaction.input) || '-' }}</td>
                </tr>
              </table>
            </expand-panel>
          </div>
        </div>
      </div>
      <div class="container" v-else>
        <loading-message />
      </div>
    </div>
  </div>
</template>

<script>
import service from '../explorer/service';
import store from '../explorer/store';
import LoadingMessage from './LoadingMessage';
import VueJsonPretty from 'vue-json-pretty';
import ExpandPanel from '@/ui/ExpandPanel';

export default {
  name: 'TransactionPage',
  props: {
    isStaking: {
      type: Boolean
    }
  },
  data() {
    return {
      loading: true,
      transaction: null,
      receipt: null,
      sequence: null,
      globalData: store.data,
    };
  },
  components: {
    LoadingMessage,
    VueJsonPretty,
    ExpandPanel
  },
  watch: {
    $route() {
      this.getTransaction();
    }
  },
  mounted() {
    this.getTransaction();
  },
  computed: {
    isCrossShard() {
      return (
        this.transaction &&
        this.transaction.shardID === this.transaction.toShardID
      );
    },
    status() {
      const txId = this.$route.params.transactionId;

      if (this.globalData.txPools.includes(txId)) {
        return 'Pending';
      }

      if (this.globalData.txFailures.includes(txId)) {
        return 'Failed';
      }

      return 'Success';
    }
  },
  methods: {
    getSequence() {
      const data = this.transaction.input;
      const re = /.+?7c7c((30|31|32|33|34|35|36|37|38|39|4c|52|55|44)+) 7c7c0*$/;
      const match = data.match(re);
      if (match && match[1] && match[1].length % 2 == 0) {
        this.sequence = this.hexToAscii(match[1]);
      }
    },
    getTransaction() {
      this.loading = true;

      const getTx = this.isStaking
        ? service.getStakingTransaction
        : service.getTransaction;

      getTx(this.$route.params.transactionId)
        .then(transaction => {
          this.transaction = transaction;
          if (this.transaction.shardID !== this.transaction.toShardID) {
            service
              .getCxReceipt(this.$route.params.transactionId)
              .then(receipt => {
                this.receipt = receipt;
                console.log('receipt', receipt);
              });
          }
          this.getSequence();
        })
        .finally(() => (this.loading = false));
    },
    hexToUTF8(h) {
      try {
        let s = this.hexToAscii(h);
        return decodeURIComponent(escape(s));
      } catch (e) {
        return null;
        // return "[Unknown Binary Content]";
      }
    },
    hexToAscii(h) {
      var s = '';
      for (var i = 0; i < h.length; i += 2) {
        s += String.fromCharCode(parseInt(h.substr(i, 2), 16));
      }
      return s;
    },
    normalizedGas() {
      return isNaN(this.transaction.gas)
        ? 0
        : (Number(this.transaction.gas) * Number(this.transaction.gasPrice)) /
            10 ** 14 /
            10000;
    }
  }
};
</script>
