<script>
import { onMount } from 'svelte';
import { get } from "svelte/store";
import orderBy from 'lodash/orderBy';
import groupBy from 'lodash/groupBy';

import { _ } from "svelte-i18n";
import debounce from "lodash/debounce";
import BigNumber from "bignumber.js";
import { ethers } from "ethers";
import images from "../config/images.json";
import poolsConfig from "../config/pools.json";
import ovenABI from '../config/ovenABI.json';
import displayNotification from "../notifications.js";
import {
    balanceKey,
    balances,
    subject,
    contract,
    eth
} from "../stores/eth.js";

import {
  maxAmount,
  getTokenImage,
  fetchEthBalance,
  fetchCalcToPie,
  formatFiat,
  subscribeToBalance,
  toFixed
} from "../components/helpers.js";

import Gauge from '../components/charts/gauge.svelte';

export let ovenAddress;
export let pieAddress;

let initialized = false;
let amount = 0;
let instance;
let ethKey;

$: balanceKeyOven = balanceKey(ethers.constants.AddressZero, ovenAddress);
$: selectedTab = 1;
$: ovenData = {
  ethBalance: 0,
  pieBalance: 0,
  logs: []
}

$: pie = poolsConfig[pieAddress];

$: if($eth.address) {
  fetchEthBalance($eth.address);
  ethKey = balanceKey(ethers.constants.AddressZero, $eth.address);
}

$: ethBalance = BigNumber($balances[ethKey]).toFixed(4);

onMount(async () => {
  const { provider, signer } = get(eth);
  instance = new ethers.Contract(ovenAddress, ovenABI, signer);
  
  await fetch();

  let bakeLogs = await instance.queryFilter(instance.filters.Bake(), 3604155, "latest");
  ovenData.logs = orderBy(bakeLogs.map( log => {
    return {
      amount: (log.args.amount / 1e18).toFixed(2),
      price: log.args.price / 1e18,
      user: log.args.user,
      tx: log.transactionHash,
      blockNumber: log.blockNumber
    }
  }), ['blockNumber'], ['desc']);

  console.log('bakeLogs', bakeLogs);
});

const fetch = async () => {
    ovenData.ethBalance = await instance.ethBalanceOf($eth.address) / 1e18;
    ovenData.pieBalance = await instance.outputBalanceOf($eth.address) / 1e18;
    ovenData.cap = await instance.cap() / 1e18;
    initialized = true;  
};

const withdrawPie = async () => {
    if (!$eth.address || !$eth.signer) {
      displayNotification({ message: $_("piedao.please.connect.wallet"), type: "hint" });
      connectWeb3();
      return;
    }
    const { emitter } = displayNotification(await instance.withdrawOutput($eth.address) );

    emitter.on("txConfirmed", ({ hash }) => {
      const { dismiss } = displayNotification({
        message: "Confirming...",
        type: "pending",
      });

      const subscription = subject("blockNumber").subscribe({
        next: async () => {
          displayNotification({
            autoDismiss: 15000,
            message: `${oven.baking.symbol} successfully withdrew from the Oven`,
            type: "success",
          });
          dismiss();
          await fetch();
          subscription.unsubscribe();
        },
      });

      return {
        autoDismiss: 1,
        message: "Mined",
        type: "success",
      };
    });
};

const withdrawEth = async () => {
    const requestedAmount = BigNumber(amount);
    const max = BigNumber(ovenData.ethBalance).multipliedBy(10 ** 18).toFixed(0);

    if (!$eth.address || !$eth.signer) {
      displayNotification({ message: $_("piedao.please.connect.wallet"), type: "hint" });
      connectWeb3();
      return;
    }

    if (BigNumber(requestedAmount).isGreaterThan(BigNumber(max)) ) {
      const message = `Not enough ETH in the Oven`;
      displayNotification({ message, type: "error", autoDismiss: 30000 });
      return;
    }

    const amountWei = requestedAmount.multipliedBy(10 ** 18).toFixed(0);

    const { emitter } = displayNotification(await instance.withdrawETH(amountWei, $eth.address) );

    emitter.on("txConfirmed", ({ hash }) => {
      const { dismiss } = displayNotification({
        message: "Confirming...",
        type: "pending",
      });

      const subscription = subject("blockNumber").subscribe({
        next: async () => {
          displayNotification({
            autoDismiss: 15000,
            message: `${requestedAmount.toFixed()} ETH successfully withdrew from the Oven`,
            type: "success",
          });
          await fetch();
          dismiss();
          subscription.unsubscribe();
        },
      });

      return {
        autoDismiss: 1,
        message: "Mined",
        type: "success",
      };
    });
};

const deposit = async () => {
    const requestedAmount = BigNumber(amount);
    const max = BigNumber(ethBalance).multipliedBy(10 ** 18).toFixed(0);

    if (!$eth.address || !$eth.signer) {
      displayNotification({ message: $_("piedao.please.connect.wallet"), type: "hint" });
      connectWeb3();
      return;
    }

    if (BigNumber(requestedAmount).isGreaterThan(BigNumber(max)) ) {
      const maxFormatted = amountFormatter({ amount: max, displayDecimals: 8 });
      const message = `Not enough ETH`;
      displayNotification({ message, type: "error", autoDismiss: 30000 });
      return;
    }

    const amountWei = requestedAmount.multipliedBy(10 ** 18).toFixed(0);
    let overrides = {
      value: amountWei
    }

    const { emitter } = displayNotification(await instance.deposit(overrides) );

    emitter.on("txConfirmed", ({ hash }) => {
      const { dismiss } = displayNotification({
        message: "Confirming...",
        type: "pending",
      });

      const subscription = subject("blockNumber").subscribe({
        next: async () => {
          displayNotification({
            autoDismiss: 15000,
            message: `${requestedAmount.toFixed()} ETH successfully deposited in the Oven`,
            type: "success",
          });
          await fetch();
          dismiss();
          subscription.unsubscribe();
        },
      });

      return {
        autoDismiss: 1,
        message: "Mined",
        type: "success",
      };
    });
  };

</script>


<div class="liquidity-container flex-col justify-items-center bg-grey-243 rounded-4px p-4 w-100pc md:p-6 ">
  <div class="flex justify-center font-thin mb-2">

    <div class="flex w-full text-black text-center text-xs md:text-xs lg:text-base justify-around mt-2 md:mt-0">
      <div class="p-0 mr-8 ">
        <div class="">
          Your ETH in the Oven
        </div>
        <div class="font-bold">{toFixed(ovenData.ethBalance, 6)} ETH</div>
      </div>
      <div class="p-0 mr-8">
        <div class="">
          Pie Ready to Withdraw
        </div>
        <div class="font-bold">{toFixed(ovenData.pieBalance, 2)} {pie.symbol}</div>
      </div>
    </div>

  </div>

  <div class="w-100pc flex justify-center justify-items-center content-center text-center">
    <button on:click={ () => selectedTab = 0} class:oven-button-active={selectedTab === 0} class="oven-button m-0 mt-4 mb-4 w-50pc rounded-8px min-w-100px lg:w-20pc lg:min-w-100px">
        Status
    </button>
    <button on:click={ () => selectedTab = 1} class:oven-button-active={selectedTab === 1} class="oven-button m-0 mt-4 mb-4 w-50pc rounded-8px min-w-100px lg:w-20pc lg:min-w-100px">
        Deposit
    </button>
    <button on:click={ () => selectedTab = 2} class:oven-button-active={selectedTab === 2} class="oven-button m-0 mt-4 mb-4 w-50pc rounded-8px min-w-100px lg:w-20pc lg:min-w-100px">
        Withdraw
    </button>
  </div>

  {#if selectedTab === 0}
    <div class="flex justify-center py-4">
      <Gauge value={BigNumber($balances[balanceKeyOven]).toFixed(4)} max={10} />
    </div>
    <table class="breakdown-table table-auto w-full mx-1">
        <thead>
          <tr>
            <th class="font-thin border-b-2 px-4 py-2 text-left">Baked {pie.symbol}</th>
            <th class="font-thin border-b-2 px-4 py-2">Block</th>
            <th class="font-thin border-b-2 px-4 py-2">Tx</th>
          </tr>
        </thead>
        <tbody>
          {#each ovenData.logs as log}
            <tr class="row-highlight">
              <td class="pointer border border-gray-800 px-2 py-2 text-left">
                {log.amount} {pie.symbol}
              </td>
              <td class="pointer border px-4 ml-8 py-2 font-thin text-center">
                {log.blockNumber}
              </td>
              <td class="border px-4 ml-8 py-2 font-thin text-center">
                  <a href={`https://etherscan.io/tx/${log.tx}`} target="_blank">Etherscan</a>
              </td>
              
            </tr>
          {/each}
        </tbody>
      </table>
  {/if}
  
  {#if selectedTab === 1}
      <div class="input bg-white border border-solid rounded-8px border-grey-204 mx-0 md:mx-4">
          <div class="top h-32px text-sm font-thin px-4 py-4 md:py-2">
            <div class="left float-left">You Deposit</div>
            <div class="right text-white font-bold text-xs py-1px text-center align-right float-right rounded">
              <button on:click={() => amount = ethBalance} class="percentage-btn inline-block rounded-20px h-20px bg-black w-50px cursor-pointer">100%</button>
            </div>
          </div>
          <div class="bottom  px-4 py-4 md:px-4 pb-4">
            <input bind:value={amount} type="number" class="font-thin text-base w-60pc md:w-75pc md:text-xl">
            <div class="asset-btn float-right h-32px bg-grey-243 rounded-32px px-2px flex
          align-middle justify-center items-center pointer mt-0 md:mt-14px">
          <img class="token-icon w-20px h-20px md:h-26px md:w-26px my-4px mx-2px" src="https://raw.githubusercontent.com/trustwallet/assets/master/blockchains/ethereum/info/logo.png" alt="ETH">
          <span class="py-2px px-4px">ETH</span></div> 
        </div>
      </div>

    <div class="flex justify-center">
      <button on:click={deposit} class="btn m-0 mt-4 rounded-8px px-56px py-15px" >Bake</button>
    </div>
    {/if}



    {#if selectedTab === 2}
      <div class="flex justify-between flex-col items-center  px-4 py-4 rounded-8px lg:flex-row">            
        <div class="input bg-white border border-solid rounded-8px border-grey-204 mx-0 w-100pc md:mr-4">
            <div class="top h-32px text-sm font-thin px-4 py-4 md:py-2">
              <div class="left float-left">Amount</div>
              <div class="right text-white font-bold text-xs py-1px text-center align-right float-right rounded">
                <button on:click={() => amount = ovenData.ethBalance} class="percentage-btn inline-block rounded-20px h-20px bg-black w-50px cursor-pointer">MAX</button>
              </div>
            </div>
            <div class="bottom  px-4 py-1 md:px-4">
              <input bind:value={amount} type="number" class="font-thin text-base w-90pc lg:w-70pc md:w-70pc md:text-xl">
              <div class="asset-btn float-right h-32px bg-grey-243 rounded-32px px-2px flex
            align-middle justify-center items-center pointer mt-0 md:mt-14px">
            <img class="token-icon w-20px mb-2 h-20px md:h-26px md:w-26px my-4px mx-2px " src="https://raw.githubusercontent.com/trustwallet/assets/master/blockchains/ethereum/info/logo.png" alt="ETH">
            <span class="py-2px px-4px">ETH</span></div> 
          </div>
        </div>
        <div class="flex justify-center">
          <button disabled={ovenData.ethBalance === 0} on:click={withdrawEth} class="btn m-0 mt-4 rounded-8px px-56px py-24px lg:mt-0" >Withdraw ETH</button>
        </div>
      </div>

      <div class="flex justify-between flex-col items-center  px-4 py-4 rounded-8px lg:flex-row">            
        <div class="input flex items-center h-97px bg-white border border-solid rounded-8px border-grey-204 mx-0 md:mr-4 px-4 py-1 w-100pc md:px-4 lg:w-76pc">
          <img class="token-icon w-40px h-40px  my-4px mx-2px" src={getTokenImage(pieAddress)} alt={`PieDAO ` + pie.symbol}>
          <span class="w-90pc lg:w-70pc md:w-70pc md:text-xl py-2px px-4px">{toFixed(ovenData.pieBalance, 2)} {pie.symbol}</span>
        </div>
        
        <div class="flex justify-center">
          <button disabled={ovenData.pieBalance === 0} on:click={withdrawPie} class="btn m-0 mt-4 rounded-8px px-56px py-24px lg:mt-0" >Withdraw Pie</button>
        </div>
      </div>
    {/if}


</div>