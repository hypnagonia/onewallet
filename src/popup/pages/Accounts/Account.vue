<template>
  <div>
    <app-header @refresh="refreshAccount" headerTab="main-tab" />
    <main class="main">
      <div class="relative">
        <connected-status v-if="!isExtendedView" />
        <div class="main-logo">
          <img src="images/harmony-big.png" class="logo-img" alt="Harmony" />
        </div>
        <span v-if="active.isLedger" class="badge big account-badge"
          >Ledger</span
        >
      </div>
      <div class="container">
        <div class="account-container">
          <div
            class="account-box"
            @click="onClickAccount()"
            v-tooltip.top="'Click to copy'"
          >
            <h2 class="name-label">
              {{ compressString(active.name, 10, 5) }}
            </h2>
            <div class="box-address">{{ compressString(address, 20, 5) }}</div>
          </div>
          <AccountMenu />
        </div>
        <div class="box-label">Account Balance</div>

        <div class="box-balance">
          {{ formatBalance(account.balance, 4) }}
          <span class="box-balance-code">ONE</span>
        </div>
        <div class="box-usd-balance">
          <span v-if="!tokenPrice">---</span>
          <span v-else>≈ {{ formatBalance(getUSDBalance, 4) }}</span>
          <span class="box-usd-balance-code">USD</span>
        </div>

        <!-- Shard -->
        <div class="shard-box">
          <div>Shard</div>
          <select v-model="shard" v-tooltip.top="'Select the shards'">
            <option
              v-for="item in account.shardArray"
              :value="item.shardID"
              :key="item.shardID"
            >
              {{ item.shardID }}
            </option>
          </select>
        </div>
        <div class="button-group">
          <button class="outline" @click="$router.push('/deposit')">
            Deposit
          </button>
          <button class="primary" @click="onSendClick">
            Send
          </button>
        </div>
        <div class="divider"></div>
        <div class="footer price-bar" v-if="tokenPrice">
          <marquee-text :duration="20">
            <span class="token-symbol-indicator">Harmony:</span>
            <span class="token-price-indicator"
              >{{ tokenPrice["one"] }} USD</span
            >
            <span class="token-symbol-indicator">Bitcoin:</span>
            <span class="token-price-indicator"
              >{{ tokenPrice["btc"] }} USD</span
            >
            <span class="token-symbol-indicator">Ethereum:</span>
            <span class="token-price-indicator"
              >{{ tokenPrice["eth"] }} USD</span
            >
          </marquee-text>
        </div>
      </div>
      <notifications
        group="notify"
        :width="180"
        :max="2"
        class="notifiaction-container"
      />
    </main>
  </div>
</template>

<script>
import helper from "mixins/helper";
import account from "mixins/account";
import MainTab from "components/MainTab.vue";
import { mapState } from "vuex";
import BigNumber from "bignumber.js";
import ConnectedStatus from "./ConnectedStatus";
import axios from "axios";
import AccountMenu from "./AccountMenu";
export default {
  mixins: [account, helper],

  components: {
    MainTab,
    ConnectedStatus,
    AccountMenu,
  },

  data: () => ({
    shard: 0,
    tokenPrice: null,
  }),

  computed: {
    ...mapState({
      active: (state) => state.wallets.active,
      dontShowSwitchModal: (state) => state.settings.dontShowSwitchModal,
    }),
    getUSDBalance() {
      return new BigNumber(this.account.balance)
        .multipliedBy(this.tokenPrice["one"])
        .toFixed();
    },
  },

  mounted() {
    if (
      typeof this.account.shard !== "undefined" ||
      this.account.shard !== null
    ) {
      this.shard = this.account.shard;
    } else {
      this.$store.commit("account/shard", 0);
      this.shard = 0;
    }

    this.loadShardingInfo();
    this.loadOneBalance();
    this.fetchTokenPrice();
  },

  watch: {
    shard(newValue, oldValue) {
      this.$store.commit("account/shard", newValue);
      this.loadOneBalance();
    },
    active(newVal, oldVal) {
      this.refreshAccount();
      if (!this.isSessionExist(this.currentTab)) return;
      const session = this.getSessionByHost(this.currentTab);
      this.$store.dispatch("provider/updateAllSessions", {
        newAddr: newVal.address,
      });
      if (
        !this.dontShowSwitchModal &&
        !session.accounts.includes(newVal.address)
      ) {
        this.$modal.show("modal-switch-account");
        this.$store.commit("global/showCheckBox", true);
      }
    },
  },

  methods: {
    async fetchTokenPrice() {
      const {
        data: {
          bitcoin: { usd: btcUSD },
          ethereum: { usd: ethUSD },
          harmony: { usd: hmyUSD },
        },
      } = await axios.get(
        "https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum,harmony&vs_currencies=usd"
      );
      this.tokenPrice = { btc: btcUSD, eth: ethUSD, one: hmyUSD };
      setTimeout(this.fetchTokenPrice, 30000);
    },
    onSendClick() {
      if (this.active.isLedger) this.openExpandPopup("/send");
      else this.$router.push("/send");
    },
    onClickAccount() {
      this.$copyText(this.address).then(() => {
        this.$notify({
          group: "notify",
          type: "info",
          text: "Copied to Clipboard",
        });
      });
    },
  },
};
</script>
<style lang="scss" scoped>
.shard-box {
  display: flex;
  flex-direction: row;
  justify-content: center;
  margin-top: 10px;
}
.shard-box select {
  margin-left: 20px;
}
.container {
  text-align: center;
}
.name-label {
  margin: 0.5rem;
}
.account-container {
  position: relative;
}
.account-box {
  border-radius: 10px;
  padding: 0.5rem;
  margin: 0 3rem 0.5rem 3rem;
  word-wrap: break-word;
  transition: box-shadow 0.5s ease;
}
.account-box:hover {
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  cursor: pointer;
}
.account-box:active {
  background: #f0f0f0;
}
.toast-container {
  border-radius: 5px;
  max-width: 200px;
}
.logo-img {
  height: 50px;
  width: 50px;
}
.account-badge {
  position: absolute;
  right: 10px;
  top: 5px;
}
.relative {
  position: relative;
}
.price-bar {
  left: 0;
  right: 0;
  font-size: 14px;
  opacity: 1;
  animation-name: fadeInOpacity;
  animation-iteration-count: 1;
  animation-timing-function: ease-in;
  animation-duration: 2s;
  margin-bottom: -1rem;
}

.token-symbol-indicator {
  color: black;
}

.token-price-indicator {
  font-weight: 600;
  margin-right: 2rem;
}
@keyframes fadeInOpacity {
  0% {
    opacity: 0;
  }
  100% {
    opacity: 1;
  }
}
</style>
