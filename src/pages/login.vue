<template>
  <div class="home">
    <v-container>
      <v-row class="mt-16">
        <v-col cols="12">
          <div
            class="text-h4 font-bold font-weight-bold black--text ma-16 text-center"
          >
            Take Your Web3 Storage Needs to the Next Level
          </div>
          <div
            class="text-Subtitle-2 font-weight-bold grey--text text--lighten-1 ma-16 text-center"
          >
            Store your NFTs and metadata in a fast and convenient way with a
            web3 aggregator
          </div>
          <div class="wallet-box ma-16 black--text text-center py-16">
            <div class="text-h6 font-weight-bold mb-8">Login</div>
            <div
              class="text-Subtitle-2 font-weight-bold grey--text text--lighten-1 mb-16"
            >
              Connect your wallet to access the Bucket
            </div>
            <v-btn
              class="d-block start-btn ma-auto px-10 text-Subtitle-2 font-weight-bold white--text mb-4"
              x-large
              @click="connectOverlay = true"
              >Connect Wallet</v-btn
            >
          </div>
        </v-col>
      </v-row>
    </v-container>
    <v-dialog v-model="connectOverlay" max-width="500">
      <div class="connect-box pa-8">
        <div class="text-h5 font-weight-black purple-color mb-5">
          Connect Wallet
        </div>
        <div class="text-caption grey--text text--darken-2 mb-7">
          After connecting to your wallet, you'll be able to make changes in
          custom settings. Pleses select the Ethereum Mainnet network.
        </div>
        <div class="d-flex justify-space-between align-center">
          <div class="d-flex align-center">
            <v-img
              max-height="36"
              max-width="36"
              :src="require('@/assets/imgs/metamask.png')"
            ></v-img>
            <span class="text-subtitle-1 font-weight-black purple-color ml-3"
              >MetaMask</span
            >
          </div>
          <v-btn
            class="start-btn text-subtitle-1 font-weight-black px-10 white--text"
            @click="connect"
            >Connect</v-btn
          >
        </div>
      </div>
    </v-dialog>
    <v-dialog v-model="gitOverlay" max-width="500">
      <div class="connect-box pa-14">
        <div class="text-caption grey--text text--darken-2 mb-7">
          The Git account is not registered on the bucket, <br />Please use your
          wallet to log in
        </div>
        <v-btn
          class="start-btn text-subtitle-1 font-weight-black px-10"
          @click="gitOverlay = false"
          >OK</v-btn
        >
      </div>
    </v-dialog>
    <v-dialog v-model="lockOverlay" max-width="500">
      <div class="connect-box pa-14">
        <div class="text-caption grey--text text--darken-2 mb-7">
          Metamask is locked, please open the extension before continuing.
        </div>
        <v-btn
          class="start-btn text-subtitle-1 font-weight-black px-10"
          @click="lockOverlay = false"
          >RETRY</v-btn
        >
      </div>
    </v-dialog>
  </div>
</template>
<script>
import contracts from "@/contracts";

const authApi = process.env.VUE_APP_AUTH_URL;
export default {
  name: "Home",
  components: {},
  data() {
    return {
      connectOverlay: false,
      gitOverlay: false,
      lockOverlay: false,
      accounts: "",
    };
  },
  mounted() {
    const code = this.$route.query.code;
    if (code) {
      this.getAuth(code);
    }
  },
  methods: {
    async getAuth(code) {
      try {
        const { data } = await this.$axios.post(`${authApi}/auth/${code}`);
        if (data.code === 430) {
          this.gitOverlay = true;
        }
        if (data.code === 200 && data.data.stoken) {
          this.ssoLogin(data.data.stoken);
        }
      } catch (error) {
        console.log(error);
      }
    },
    async connect() {
      if (!window.ethereum) {
        window.open("https://metamask.io/download.html", "_blank");
        return;
      }

      const isUnlocked = await window.ethereum._metamask.isUnlocked();
      if (!isUnlocked) {
        console.log("Metamask has been locked, please unlock it.");
        this.connectOverlay = false;
        this.lockOverlay = true;
        return;
      }

      let accounts = await window.ethereum.request({
        method: "eth_accounts",
      });
      if (accounts.length === 0) {
        accounts = await window.ethereum.request({
          method: "eth_requestAccounts",
        });
      }

      this.$axios.get(`${authApi}/web3code/${accounts[0]}`).then((res) => {
        console.log(res);
        this.accounts = accounts[0];
        this.sign(res.data.data.nonce);
      });
      //   window.location.reload();
    },
    async sign(nonce) {
      try {
        const accounts = this.accounts;
        this.msg = nonce;
        const signature = await contracts.signer.signMessage(this.msg);
        const data = {
          signature,
          appName: "BUCKET",
        };
        this.$axios
          .post(`${authApi}/web3login/${accounts}`, data)
          .then((res) => {
            if (res.data.data.stoken) {
              this.ssoLogin(res.data.data.stoken);
            }
          });
      } catch (e) {
        console.log(e);
      } finally {
        this.loading = false;
      }
    },
    async ssoLogin(stoken) {
      try {
        const { data } = await this.$http.post(`/st/${stoken}`, null, {
          params: {
            _auth: 1,
          },
        });
        this.$onLoginData(data);
      } catch (error) {
        this.onErr(error);
      }
    },
    onErr(error) {
      console.log(error);
      this.$alert(error.message).then(() => {
        location.href = this.$loginUrl;
      });
    },
  },
};
</script>
<style lang="scss" scoped>
.home {
  .wallet-box {
    background: linear-gradient(150deg, #e1f2ff, #fff6f6);
    border-radius: 10px;
    .start-btn {
      background: linear-gradient(90deg, #fdb6fe, #acc0fd, #31adfe);
      border-radius: 44px;
    }
  }
}
.connect-box {
  width: 500px !important;
  min-height: 200px;
  background-color: #fff;
  border-radius: 15px;
  text-align: center;
  margin: 0 auto;
  .start-btn {
    background: linear-gradient(90deg, #fdb6fe, #acc0fd, #31adfe);
    border-radius: 44px;
  }
}
::v-deep body .v-dialog {
  background: transparent !important;
  flex-grow: 0 !important;
}
</style>
