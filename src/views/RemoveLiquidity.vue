<template>
  <div class="max-w-xl mx-auto">
    <NavTabs class="mb-6" />

    <Form :style="processing && 'pointer-events:none'">
      <NavInvest />

      <FormField
        placeholder="0.0"
        label="Shares to remove"
        :withTezos="false"
        :subLabelName="myShares ? 'Your shares: ' : undefined"
        :subLabelValue="myShares || undefined"
        :isLoading="tokenLoading"
        v-model="sharesToRemove"
        @input="(e) => handleSharesToRemoveChange(e.target.value)"
        @selectToken="handleTokenSelect"
        :selectedToken="selectedToken"
      />

      <FormIcon>
        <img :src="require('@/assets/arrow-down.svg')" />
      </FormIcon>

      <div class="-mx-3 shadow-lg xs:-mx-4">
        <div class="relative field rounded-3px">
          <div class="flex flex-col justify-start flex-1 py-6">
            <div class="w-full mb-1 font-light label xs:mb-2 sm:text-lg">
              Output
              <span class="text-sm">(estimated)</span>
            </div>

            <div v-if="inTokens" class="flex flex-col fieldval">
              <div class="mb-1">
                <span class="mr-1 opacity-75">+</span>
                <span class="tracking-wide">
                  {{ formatNum(inTokens.tezos, 6) }}
                </span>
                <span class="ml-1 text-sm opacity-90">XTZ</span>
              </div>

              <div class="mb-1">
                <span class="mr-1 opacity-75">+</span>
                <span class="tracking-wide">
                  {{ formatNum(inTokens.token, selectedToken.decimals) }}
                </span>
                <span class="ml-1 text-sm opacity-90">{{
                  selectedToken.name
                }}</span>
              </div>
            </div>

            <div v-else>-</div>
          </div>
        </div>
      </div>

      <FormInfo class="overflow-x-auto whitespace-no-wrap">
        <div class="flex justify-between mb-1">
          <span class="mr-2">Dex contract</span>
          <span class="font-mono text-gray-400">{{ dexAddress || "-" }}</span>
        </div>

        <div class="flex justify-between mb-1">
          <span class="mr-2">Exchange rate</span>
          <span>{{ exchangeRate || "-" }}</span>
        </div>

        <div class="flex justify-between mb-1">
          <span class="mr-2">Pooled Tokens</span>
          <span>{{ poolMeta ? poolMeta.tokenFull : "-" }}</span>
        </div>

        <div class="flex justify-between mb-1">
          <span class="mr-2">Pooled XTZ</span>
          <span>{{ poolMeta ? poolMeta.tezFull : "-" }}</span>
        </div>

        <div class="flex justify-between mb-1">
          <span class="mr-2">Your pool tokens</span>
          <span>{{ poolMeta ? poolMeta.myTokens : "-" }}</span>
        </div>

        <div class="flex justify-between mb-1">
          <span class="mr-2">Your pool share</span>
          <span>{{ poolMeta ? poolMeta.myShare : "-" }}</span>
        </div>

        <div class="flex mb-1">
          <span class="mr-2">Slippage tolerance</span>
          <span class="flex-1"></span>
          <button
            v-for="percentage in slippagePercentages"
            :key="percentage"
            class="px-2 py-px ml-2 text-xs rounded-md shadow-xs focus:outline-none"
            :class="
              activeSlippagePercentage === percentage ? 'bg-alphawhite' : ''
            "
            v-on:click="setActiveSlippagePercentage(percentage)"
          >
            {{ percentage }}
            <span class="opacity-75">%</span>
          </button>

          <div
            class="px-2 py-2 ml-2 text-xs font-light leading-tight rounded-md shadow-xs focus:outline-none"
            :class="
              slippagePercentages.includes(activeSlippagePercentage)
                ? ''
                : 'bg-alphawhite'
            "
          >
            <input
              class="w-12 text-right bg-transparent outline-none"
              v-bind:placeholder="activeSlippagePercentage"
              v-model="customSlippagePercentage"
              @input="e => handleCustomSlippageChange(e.target.value)"
            />
            %
          </div>
        </div>
      </FormInfo>
    </Form>

    <div
      class="mx-auto mt-8 mb-8 text-sm font-normal text-center text-lightgray"
    ></div>
    <div class="flex justify-center text-center">
      <SubmitBtn @click="removeLiquidity" :disabled="!valid">
        <template v-if="!processing">{{ remLiqStatus }}</template>
        <template v-if="processing">
          <Loader size="large" />
        </template>
      </SubmitBtn>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Watch } from "vue-property-decorator";
import NavTabs from "@/components/NavTabs.vue";
import NavInvest from "@/components/NavInvest.vue";
import Loader from "@/components/Loader.vue";
import Form, { FormField, FormIcon, FormInfo } from "@/components/Form";
import SubmitBtn from "@/components/SubmitBtn.vue";

import BigNumber from "bignumber.js";
import store, { getAccount, useWallet } from "@/store";
import {
  QSAsset,
  isAddressValid,
  toValidAmount,
  getBalance,
  getDexStorage,
  getDexShares,
  getContract,
  estimateInTokens,
  estimateInTezos,
  tzToMutez,
  mutezToTz,
  clearMem,
  toNat,
  fromNat,
  sharesFromNat,
  sharesToNat,
  toAssetSlug,
  findTezDex,
} from "@/core";
import { XTZ_TOKEN } from "@/core/defaults";
import { notifyConfirm } from "../toast";

type InTokens = {
  tezos: string;
  token: string;
};

type PoolMeta = {
  tezFull: string;
  tokenFull: string;
  myShare: string;
  myTokens: string;
};

@Component({
  components: {
    NavTabs,
    NavInvest,
    Form,
    FormField,
    FormIcon,
    FormInfo,
    SubmitBtn,
    Loader,
  },
})
export default class RemoveLiquidity extends Vue {
  sharesToRemove = "";
  myShares: string | null = null;
  tokenLoading = false;

  inTokens: InTokens | null = null;
  poolMeta: PoolMeta | null = null;
  dexAddress: string | null = null;

  processing = false;
  remLiqStatus = this.defaultRemLiqStatus;

  slippagePercentages = [0.5, 1, 3];
  activeSlippagePercentage: number | undefined = 1;
  lastValidCustomSlippagePercentage: string = "";
  customSlippagePercentage: string = "";

  get selectedToken(): QSAsset | null {
    const tokenSlug = this.$route.params.token;
    return (
      store.state.tokens.find((t) => toAssetSlug(t) === tokenSlug) || null
    );
  }

  get defaultRemLiqStatus() {
    return "Remove Liquidity";
  }

  get account() {
    return getAccount();
  }

  get valid() {
    return (
      this.selectedToken &&
      this.dexAddress &&
      this.sharesToRemove &&
      +this.sharesToRemove > 0 &&
      this.inTokens
    );
  }

  get exchangeRate() {
    if (!this.selectedToken || !this.inTokens) {
      return null;
    }

    const price = new BigNumber(this.inTokens.tezos)
      .div(this.inTokens.token)
      .toFormat(6);

    return `1 ${this.selectedToken.name} = ${price} XTZ`;
  }

  formatNum(val: string, dec: number) {
    return new BigNumber(val).toFormat(dec);
  }

  @Watch("selectedToken")
  onSelectedTokenChange() {
    this.loadDex();
  }

  @Watch("dexAddress")
  onDexAddressChange() {
    this.loadPoolMetadata();
    this.loadMyShares();
  }

  @Watch("account")
  onAccountChange() {
    this.loadPoolMetadata();
    this.loadMyShares();
  }

  setActiveSlippagePercentage(percentage: number) {
    this.activeSlippagePercentage = percentage;
    this.customSlippagePercentage = "";
    this.lastValidCustomSlippagePercentage = "";
  }

  handleCustomSlippageChange(amount: string) {
    const numAmount = amount ? Number(amount) : undefined;
    const shouldRevertChange =
      numAmount !== undefined && (numAmount < 0 || numAmount > 30);

    if (
      numAmount !== undefined &&
      !Number.isNaN(numAmount) &&
      !shouldRevertChange
    ) {
      this.activeSlippagePercentage = numAmount;
    }

    if (shouldRevertChange) {
      this.customSlippagePercentage = this.lastValidCustomSlippagePercentage;
    } else {
      this.lastValidCustomSlippagePercentage = amount;
    }
  }

  async loadMyShares() {
    this.myShares = null;

    try {
      if (this.selectedToken && this.dexAddress && this.account.pkh) {
        const shares = await getDexShares(
          this.account.pkh,
          this.dexAddress
        );
        if (!shares) {
          this.myShares = "0";
        } else {
          this.myShares = sharesFromNat(shares.unfrozen).toFixed();
        }
      }
    } catch (err) {
      if (process.env.NODE_ENV === "development") {
        console.error(err);
      }
    }
  }

  async loadDex() {
    this.dexAddress = null;

    if (this.selectedToken) {
      this.tokenLoading = true;
      const dex = await findTezDex(this.selectedToken);
      if (dex) {
        this.dexAddress = dex.address;
      }
      this.tokenLoading = false;

      this.calcInTokens();
    }
  }

  async loadPoolMetadata() {
    this.poolMeta = null;

    if (this.selectedToken && this.dexAddress && this.account.pkh) {
      const myShares = await getDexShares(
        this.account.pkh,
        this.dexAddress
      );
      const dexStorage = await getDexStorage(this.dexAddress);

      const myShare =
        myShares &&
        new BigNumber(myShares.unfrozen).div(dexStorage.totalSupply);
      const myTokens =
        myShare &&
        fromNat(
          new BigNumber(dexStorage.tokenPool)
            .times(myShare)
            .integerValue(BigNumber.ROUND_DOWN),
          this.selectedToken
        );

      this.poolMeta = {
        tezFull: `${mutezToTz(dexStorage.tezPool)} XTZ`,
        tokenFull: `${fromNat(dexStorage.tokenPool, this.selectedToken)} ${
          this.selectedToken.name
        }`,
        myShare: myShare && !myShare.isNaN() ? `${myShare.times(100).toFormat(2)}%` : "-",
        myTokens: myTokens && !myTokens.isNaN() ? `${myTokens} ${this.selectedToken.name}` : "-",
      };
    }
  }

  async handleTokenSelect(token: QSAsset) {
    this.$router.replace(`/invest/remove-liquidity/${toAssetSlug(token)}`);

    if (!this.sharesToRemove) {
      this.sharesToRemove = "1";
    }
  }

  handleSharesToRemoveChange(amount: string) {
    this.sharesToRemove = amount;
    const isNum = /^[0-9.]*$/g.test(amount);
    if (isNum) {
      this.calcInTokens();
    } else {
      this.inTokens = null;
    }
  }

  async calcInTokens() {
    this.inTokens = null;
    if (!this.selectedToken || !this.dexAddress || !this.sharesToRemove) return;

    const dexStorage = await getDexStorage(this.dexAddress);

    const tezAmount = estimateInTezos(sharesToNat(this.sharesToRemove), dexStorage);
    const tokenAmount = estimateInTokens(
      sharesToNat(this.sharesToRemove),
      dexStorage,
      this.selectedToken
    );

    const tezos = toValidAmount(tezAmount);
    const token = toValidAmount(tokenAmount);
    this.inTokens = tezos && token ? { tezos, token } : null;
  }

  async removeLiquidity() {
    if (this.processing) return;
    this.processing = true;
    try {
      const tezos = await useWallet();

      const shares = sharesToNat(this.sharesToRemove!);
      const selTk = this.selectedToken!;
      const dexAddress = this.dexAddress!;

      const mySharesPure = await getDexShares(this.account.pkh, dexAddress);
      let myShares: string | undefined;
      if (mySharesPure) {
        myShares = mySharesPure.unfrozen.toFixed();
      }

      if (!myShares || shares.isGreaterThan(myShares)) {
        throw new Error("Not Enough Shares");
      }

      const slippage = new BigNumber(this.activeSlippagePercentage || 0).div(100);

      const minTezos = withSlippage(tzToMutez(this.inTokens!.tezos), slippage);
      const minToken = withSlippage(toNat(this.inTokens!.token, selTk), slippage);

      const dexContract = await tezos.wallet.at(dexAddress);
      const operation = await dexContract.methods
        .use(
          "divestLiquidity",
          minTezos.toFixed(),
          minToken.toFixed(),
          shares.toFixed()
        )
        .send();

      notifyConfirm(
        operation.confirmation()
          .then(() => this.refresh())
      );
    } catch (err) {
      console.error(err);
      const msg = err.message;
      this.remLiqStatus =
        msg && msg.length < 30
          ? msg.startsWith("Dex/")
            ? msg.replace("Dex/", "")
            : msg
          : "Something went wrong";
    }
    this.processing = false;

    await new Promise((res) => setTimeout(res, 5000));
    this.remLiqStatus = this.defaultRemLiqStatus;
  }

  refresh() {
    clearMem();
    this.loadMyShares();
    this.loadPoolMetadata();
    this.calcInTokens();
  }
}

function withSlippage(val: BigNumber.Value, tolerance: BigNumber.Value) {
  return new BigNumber(val)
    .times(new BigNumber(1).minus(tolerance))
    .integerValue(BigNumber.ROUND_DOWN);
}
</script>

<style lang="postcss" scoped>
.field {
  @apply px-3 flex items-stretch;
  min-height: 5rem;
  background: #2a3248;
}

.fieldval {
  @apply text-base font-light;
}

@screen xs {
  .field {
    @apply px-6 h-32;
  }

  .field-extend {
    @apply px-6 h-32;
  }

  .fieldval {
    @apply text-lg;
  }
}

.label {
  color: #f6cc5b;
}
</style>
