<template>
  <v-container>
    <v-row justify="center">
      <v-col cols="4" class="text-center">
        <h1 class="my-5">My ERC 721 Token</h1>
        <v-card
          elevation="2"
        >
          <v-card-text>
            <v-col>
              <h1>{{currentSupply}}/{{maxSupply}}</h1>
              <a v-if="isCorrectChain" target="_blank" :href='(chainId == "0x4") ? `https://rinkeby.etherscan.io/address/${walletAddress}` : `https://polygonscan.io/address/${walletAddress}`'>{{walletAddress}}</a>
              <p>Chaind Id: {{currentChainId}}</p>
              <h3 v-if='currentChainId == "0x4"'>You are currently in Rinkeby Test network.</h3>
              <h3 v-if='currentChainId == "0x89"'>You are currently in Polygon mainnet.</h3>
              <h3 v-if='!isCorrectChain'>Please connect to the correct network! 😠</h3>
              <h3 v-if="isCorrectChain">cost: {{mintingCostText}}</h3>
              <v-btn v-if="isCorrectChain" icon @click="decrementMintAmount()"><v-icon>mdi-minus-circle</v-icon></v-btn>
              <b v-if="isCorrectChain">{{mintAmount}}</b>
              <v-btn v-if="isCorrectChain" icon @click="incrementMintAmount()"><v-icon>mdi-plus-circle</v-icon></v-btn>
              <br>
              <v-btn
                color="primary"
                :loading="transactionLoading"
                v-if="isCorrectChain && (currentSupply != maxSupply) && (mintingCost != null) && (currentSupply != 0) && (maxSupply != 0) && (maxMintPerTx != 0)"
                class="my-3"
                @click="mint()"
              >
                MINT
              </v-btn>
            </v-col>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>

    <v-col cols="auto">
      <v-dialog
        max-width="600"
        v-model="transactionDialog"
      >
        <template v-slot:default="dialog">
          <v-card>
            <v-card-text>
              <div class="text-h3 pa-12">Transaction sent!</div>
              <div class="text-h4 pa-12">view your transaction here: {{}}</div>
            </v-card-text>
            <v-card-actions class="justify-end">
              <v-btn
                text
                @click="dialog.value = false"
              >Close</v-btn>
            </v-card-actions>
          </v-card>
        </template>
      </v-dialog>
    </v-col>

  </v-container>
</template>

<script>
import Moralis from 'moralis';
import { myContractAddress, ABI } from '../constants/contract';
import { getEllipsisTxt, tokenValue } from '../helpers/formatters.js';
const { createAlchemyWeb3 } = require("@alch/alchemy-web3");

const serverUrl = process.env.VUE_APP_MORALIS_SERVER_URL;
const appId =  process.env.VUE_APP_MORALIS_APPLICATION_ID;

const alchemyKey =  process.env.VUE_APP_ALCHEMY_KEY;

Moralis.start({ serverUrl, appId });

export default {
  data: () => ({
    currentUser: null,
    mintingCost: null,
    newMintingCost: null,
    currentSupply: 0,
    maxSupply: 0,
    web3Js: null,
    contract: null,
    chainId: process.env.VUE_APP_CHAIN_ID,
    currentChainId: null,
    walletAddress: "",
    options: {
      contractAddress: myContractAddress,
      abi: ABI
    },
    mintAmount: 1,
    maxMintPerTx: 0,
    mintTransaction: null,

    transactionLoading: false,
    transactionDialog: false,

    txHash: "",
  }),

  methods: {
    
    async init(){
      this.currentUser = Moralis.User.current();

      if(!this.currentUser){
        this.currentUser = await Moralis.authenticate();
      }

      await Moralis.enableWeb3();
      this.web3Js = new createAlchemyWeb3(alchemyKey);
      this.contract = new this.web3Js.eth.Contract(ABI, myContractAddress);

      this.walletAddress = window.ethereum.selectedAddress;
      this.currentChainId = await Moralis.chainId;

      // check if in correct network
      if(this.isCorrectChain){
        this.getContractDetails();
      }

      // event listeners
      // Subscribe to onChainChanged events
      Moralis.onChainChanged((chain) => {
        this.currentChainId = chain;
        this.refreshContractDetails();
      });

      window.ethereum.on('accountsChanged', (accounts) => {
        // Time to reload your interface with accounts[0]!
        // this.updateWalletText(accounts[0]);
        this.walletAddress = accounts[0];
      });

      // Subscribe to onWeb3Deactivated events
      Moralis.onWeb3Deactivated((result) => {
        this.logOut(result);
      });
      //
      //event listeners

    },

    async getContractDetails(){
      this.maxSupply = await Moralis.executeFunction({functionName: 'maxSupply', ...this.options});
      this.currentSupply = await Moralis.executeFunction({functionName: 'totalSupply', ...this.options});
      this.mintingCost = await Moralis.executeFunction({functionName: 'cost', ...this.options});
      this.newMintingCost = this.mintingCost;
      let resultMaxMint = await Moralis.executeFunction({functionName: 'maxMintAmountPerTx', ...this.options});
      this.maxMintPerTx = resultMaxMint.toNumber();
    },

    updateWalletText(walletText){
      this.walletAddress = getEllipsisTxt(walletText);
    },

    // do chain change event
    refreshContractDetails(){

      // reset mint amount
      this.mintAmount = 1;

      // check if in correct network
      if(this.isCorrectChain){
        this.getContractDetails();
      }
      else{
        this.maxSupply = 0;
        this.currentSupply = 0;
        this.mintingCost = 0;
        this.newMintingCost = this.mintingCost;
        this.maxMintPerTx = 0;
      }
      
    },

    async logOut(result){
      await Moralis.User.logOut();
      console.log(result);
    },

    async mint(){
      
      // do loading
      this.transactionLoading = true;

      // fire metamask tx
      this.contract.methods.mint(this.mintAmount).send({from: window.ethereum.selectedAddress, value: this.newMintingCost})
      .on('transactionHash', (hash) => {
        if(hash){
          this.transactionDialog = true;
          this.txHash = hash;
        }
        // remove loading
        this.transactionLoading = false;
      })
      .on('receipt', (receipt) => {
        console.log(receipt);
      })
      .on('error', (error, receipt) => {
        console.log(error);
        console.log(receipt);
        // remove loading
        this.transactionLoading = false;
      });

    },

    incrementMintAmount(){
      this.mintAmount = (this.mintAmount == this.maxMintPerTx) ? this.mintAmount : this.mintAmount+1;
      this.newMintingCost = this.mintAmount * this.mintingCost;
    },

    decrementMintAmount(){
      this.mintAmount = (this.mintAmount == 1) ? this.mintAmount : this.mintAmount-1;
      this.newMintingCost = this.mintAmount * this.mintingCost;
    }

  },

  mounted(){

  },

  created(){
    this.init();
  },

  computed: {

    isCorrectChain(){
      return this.chainId == this.currentChainId;
    },

    mintingCostText(){
      return (this.newMintingCost > 0) ? tokenValue(this.newMintingCost, 18) : 0;
    }

  },

  asyncComputed: {

  }

}
</script>
