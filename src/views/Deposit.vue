<template>
  <div>
    <h1 class="title">Deposit to Layer2</h1>

    <div class="row">
      <div class="col-md-3">
        <h3>Account Info</h3>
        <hr />

        <p><strong>ETH Address</strong>: {{ ethAddress }}</p>
        <p><strong>Polyjuice Address</strong>: {{ polyjuiceAddress }}</p>
        <p><strong>Nervos Layer2(godwoken) CKB Balance</strong>: {{ balance }}</p>
        <p><strong>Layer2 deposit recipient </strong>: {{ recipient }}</p>
        <p><strong>Nervos Layer2(godwoken) SUDT Balance</strong>: {{ sudtBalance }}</p>
        <p><strong>SUDT contract address</strong>: {{ sudtContractAddress }}</p>
        <p>
          <strong>Deposit to Layer2</strong>:
          <button
            class="btn btn-primary"
            :disabled="disableSubmit"
            @click="bridge"
          >
            Fore Bridge
          </button>
        </p>
      </div>
    </div>

    <div class="row" v-show="errorMessage">
      <div class="col-md-6">
        <div class="alert alert-danger mt-4">
          {{ errorMessage }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import mixin from "../libs/mixinViews";
import Web3 from "web3";
import { PolyjuiceHttpProvider } from "@polyjuice-provider/web3";
import { AddressTranslator } from "nervos-godwoken-integration";
import { CONFIG } from "../assets/config";
const ProxyContract = require("../assets/contract/ERC20");

async function createWeb3() {
  // Modern dapp browsers...
  if (window.ethereum) {
    const godwokenRpcUrl = CONFIG.WEB3_PROVIDER_URL;
    const providerConfig = {
      rollupTypeHash: CONFIG.ROLLUP_TYPE_HASH,
      ethAccountLockCodeHash: CONFIG.ETH_ACCOUNT_LOCK_CODE_HASH,
      web3Url: godwokenRpcUrl,
    };

    const provider = new PolyjuiceHttpProvider(godwokenRpcUrl, providerConfig);
    const web3 = new Web3(provider || Web3.givenProvider);
    try {
      // Request account access if needed
      await window.ethereum.enable();
    } catch (error) {
      // User denied account access...
    }

    return web3;
  }

  console.log(
    "Non-Ethereum browser detected. You should consider trying MetaMask!"
  );
  return null;
}

const web3 = await createWeb3();
const _accounts = [window.ethereum.selectedAddress];
const balance = BigInt(await web3.eth.getBalance(_accounts[0]));
const addressTranslator = new AddressTranslator();
const polyjuiceAddress = addressTranslator.ethAddressToGodwokenShortAddress(
  _accounts?.[0]
);
const depositAddress = await addressTranslator.getLayer2DepositAddress(
  web3,
  _accounts?.[0]
);
console.log(`Layer 2 Deposit Address on Layer 1: \n${depositAddress.addressString}`);

const contract = new web3.eth.Contract(
  ProxyContract.abi,
  CONFIG.SUDT_PROXY_CONTRACT_ADDRESS
);
const sudtBalance = await contract.methods
  .balanceOf(polyjuiceAddress)
  .call({ from: _accounts?.[0] });
console.log(sudtBalance);

export default {
  mixins: [mixin],
  data() {
    return {
      polyjuiceAddress: "",
      ethAddress: "",
      balance: 0,
      sudtBalance: "",
      recipient: "",
      sudtContractAddress: "",
      errorMessage: null,
    };
  },
  computed: {},
  methods: {
    bridge() {
      window.open(
        "https://force-bridge-test.ckbapp.dev/bridge/Ethereum/Nervos?xchain-asset=0x0000000000000000000000000000000000000000"
      );
    },
  },
  async created() {
    this.ethAddress = _accounts?.[0];
    this.balance = (balance / 10n ** 8n).toString();
    this.polyjuiceAddress = polyjuiceAddress;
    this.recipient = depositAddress.addressString;
    this.sudtContractAddress = CONFIG.SUDT_PROXY_CONTRACT_ADDRESS;
    this.sudtBalance = sudtBalance;
  },
};
</script>
