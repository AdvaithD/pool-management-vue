<template>
  <UiModal :open="open" @close="$emit('close')" v-if="pool.id">
    <UiModalForm @submit="handleSubmit">
      <template slot="header">
        <h3 class="text-white">Add Liquidity</h3>
      </template>
      <div class="text-center m-4 mt-0">
        <Toggle
          :value="type"
          :options="toggleOptions"
          @select="handleSelectType"
        />
      </div>
      <div class="m-4 d-block d-sm-flex">
        <PoolOverview
          :pool="pool"
          :userShare="userLiquidity.relative"
          class="hide-sm hide-md col-3 float-left"
        />
        <div class="col-12 col-md-9 float-left pl-0 pl-md-4">
          <UiTable>
            <UiTableTh>
              <div class="column-lg flex-auto text-left">Asset</div>
              <div class="column">Wallet Balance</div>
              <div class="column-sm">Deposit Amount</div>
            </UiTableTh>
            <UiTableTr v-for="token in pool.tokens" :key="token.checksum">
              <div
                class="column-lg flex-auto d-flex flex-items-center text-left d-flex"
              >
                <UiRadio
                  class="mr-1"
                  v-if="!isMultiAsset"
                  :checked="activeToken === token.checksum"
                  :onChange="
                    e => {
                      onTokenSelect(token.checksum);
                    }
                  "
                />
                <Token :address="token.address" class="mr-2" size="20" />
                <div class="text-white">{{ _ticker(token.checksum) }}</div>
                <ButtonUnlock
                  class="button-primary ml-2"
                  :tokenAddress="token.checksum"
                  :amount="amounts[token.checksum]"
                />
              </div>
              <div class="column">
                {{
                  _trunc(
                    formatBalance(
                      web3.balances[token.checksum] || '0',
                      token.decimals
                    ),
                    2
                  )
                }}
                <a @click="handleMax(token)" class="ml-1">
                  <UiLabel v-text="'Max'" />
                </a>
              </div>
              <div class="column-sm">
                <div
                  class="flex-auto ml-3 text-left d-flex flex-items-center position-relative"
                >
                  <input
                    v-model="amounts[token.checksum]"
                    v-if="isMultiAsset || activeToken === token.checksum"
                    class="input flex-auto text-right"
                    :class="isInputValid(token) ? 'text-white' : 'text-red'"
                    placeholder="0.0"
                    @input="handleChange(amounts[token.checksum], token)"
                  />
                </div>
              </div>
            </UiTableTr>
          </UiTable>
          <UiTable class="mt-4">
            <UiTableTh class="text-left flex-items-center text-white">
              <div class="flex-auto">BPT amount</div>
              <div class="flex-auto text-right">
                {{ _num(userLiquidity.absolute.current) }}
                <span v-if="userLiquidity.absolute.future">
                  → {{ _num(userLiquidity.absolute.future) }}
                </span>
                BPT
              </div>
            </UiTableTh>
          </UiTable>
        </div>
      </div>
      <div class="mx-4">
        <MessageError v-if="tokenError" :text="tokenError" class="mb-4" />
        <MessageError
          v-if="validationError"
          :text="validationError"
          class="mb-4"
        />
        <MessageError
          v-if="lockedTokenError"
          :text="lockedTokenError"
          class="mb-4"
        />
        <MessageError v-if="transferError" :text="transferError" class="mb-4" />
        <MessageCheckbox
          v-if="!tokenError && !validationError"
          :custom="hasCustomToken"
          :accepted="checkboxAccept"
          @toggle="checkboxAccept = !checkboxAccept"
          class="mb-4 text-left"
        />
        <MessageWarningRateChange
          v-if="rateChangeWarning"
          @lower="lowerAmounts"
          class="mb-4"
        />
        <MessageSlippage
          v-if="slippage"
          :value="slippage"
          :isDeposit="true"
          class="mb-4"
        />
      </div>
      <template slot="footer">
        <UiButton
          class="button-primary"
          type="submit"
          :disabled="
            tokenError ||
              validationError ||
              lockedTokenError ||
              !checkboxAccept ||
              transactionFailed
          "
          :loading="loading"
        >
          Add Liquidity
        </UiButton>
      </template>
    </UiModalForm>
  </UiModal>
</template>

<script>
import { mapActions } from 'vuex';
import { getAddress } from '@ethersproject/address';

import BigNumber from '@/helpers/bignumber';
import {
  calcPoolTokensByRatio,
  bnum,
  normalizeBalance,
  denormalizeBalance,
  isTxReverted,
  getTokenBySymbol,
  toggleOptions,
  isLocked
} from '@/helpers/utils';
import { calcPoolOutGivenSingleIn } from '@/helpers/math';
import { validateNumberInput, formatError } from '@/helpers/validation';

const BALANCE_BUFFER = 0.01;

function hasToken(pool, symbol) {
  const token = getTokenBySymbol(symbol);
  if (!token) {
    return false;
  }
  const tokenAddress = token.address;
  return pool.tokensList.includes(tokenAddress);
}

export default {
  props: ['open', 'pool'],
  data() {
    return {
      toggleOptions,
      loading: false,
      poolTokens: null,
      amounts: {},
      type: 'MULTI_ASSET',
      activeToken: null,
      checkboxAccept: false,
      transactionFailed: false
    };
  },
  watch: {
    open() {
      this.loading = false;
      this.poolTokens = null;
      this.amounts = Object.fromEntries(
        this.pool.tokens.map(token => {
          return [token.checksum, ''];
        })
      );
      this.type = 'MULTI_ASSET';
      this.activeToken = this.pool.tokens[0].checksum;
      this.checkboxAccept = false;
      this.transactionFailed = false;
    }
  },
  computed: {
    poolTokenBalance() {
      const poolAddress = getAddress(this.pool.id);
      const balance = this.web3.balances[poolAddress] || 0;
      const poolBalanceNumber = normalizeBalance(balance, 18);
      return poolBalanceNumber.toString();
    },
    totalShares() {
      const poolAddress = getAddress(this.pool.id);
      const poolSupply = this.web3.supplies[poolAddress] || 0;
      const totalShareNumber = normalizeBalance(poolSupply, 18);
      return totalShareNumber.toString();
    },
    userLiquidity() {
      const poolSharesFrom = parseFloat(this.poolTokenBalance);
      const totalShares = parseFloat(this.totalShares);
      const current = poolSharesFrom / totalShares;
      if (this.validationError) {
        return {
          absolute: {
            current: poolSharesFrom
          },
          relative: {
            current
          }
        };
      }

      const poolTokens = this.poolTokens
        ? bnum(this.poolTokens)
            .div('1e18')
            .toNumber()
        : 0;
      const future = (poolSharesFrom + poolTokens) / (totalShares + poolTokens);
      const userLiquidity = {
        absolute: {
          current: poolSharesFrom,
          future: poolSharesFrom + poolTokens
        },
        relative: {
          current,
          future
        }
      };
      return userLiquidity;
    },
    tokenError() {
      if (
        this.pool.tokens.some(token =>
          this.config.untrusted.includes(token.checksum)
        )
      ) {
        return 'This pool contains untrusted token that may cause loss of funds. Do not deposit.';
      }
      return undefined;
    },
    validationError() {
      if (this.tokenError) {
        return undefined;
      }
      for (const token of this.pool.tokensList) {
        if (!this.isMultiAsset && this.activeToken !== token) {
          continue;
        }
        const amountError = validateNumberInput(this.amounts[token]);
        const amountErrorText = formatError(amountError);
        if (amountErrorText) return amountErrorText;
      }
      // Amount validation
      for (const token of this.pool.tokensList) {
        if (!this.isMultiAsset && this.activeToken !== token) {
          continue;
        }
        const amount = bnum(this.amounts[token]);
        const balance = normalizeBalance(
          this.web3.balances[token],
          this.web3.tokenMetadata[token].decimals
        );
        if (amount.gt(balance)) {
          return 'Token amount should not exceed balance';
        }
      }
      // Max in ratio validation
      if (!this.isMultiAsset) {
        const maxInRatio = 1 / 2;
        const amount = bnum(this.amounts[this.activeToken]);
        const tokenIn = this.pool.tokens.find(
          token => token.checksum === this.activeToken
        );
        if (amount.div(tokenIn.balance).gt(maxInRatio)) {
          return 'Insufficient pool liquidity';
        }
      }
      return undefined;
    },
    lockedTokenError() {
      if (this.tokenError || this.validationError) {
        return undefined;
      }
      for (const token of this.pool.tokensList) {
        if (
          isLocked(
            this.web3.allowances,
            token,
            this.web3.dsProxyAddress,
            this.amounts[token],
            this.web3.tokenMetadata[token].decimals
          )
        ) {
          const ticker = this._ticker(token);
          return `Unlock ${ticker} to continue`;
        }
      }
      return undefined;
    },
    transferError() {
      if (this.tokenError || this.validationError || this.lockedTokenError) {
        return undefined;
      }
      if (!this.transactionFailed) {
        return undefined;
      }
      if (hasToken(this.pool, 'SNX')) {
        return 'Adding liquidity failed as your SNX is locked in staking.';
      }
      const synths = ['sUSD', 'sBTC', 'sETH', 'sXAU', 'sXAG', 'sDEFI', 'sXMR'];
      if (synths.some(synth => hasToken(this.pool, synth))) {
        return 'Adding liquidity failed as your Synthetix position might go underwater. ';
      }
      const aTokens = [
        'aDAI',
        'aUSDT',
        'aUSDC',
        'aSUSD',
        'aTUSD',
        'aBUSD',
        'aBAT',
        'aETH',
        'aKNC',
        'aLEND',
        'aLINK',
        'aMANA',
        'aMKR',
        'aREP',
        'aSNX',
        'aWBTC',
        'aZRX'
      ];
      if (aTokens.some(aToken => hasToken(this.pool, aToken))) {
        return 'Adding liquidity failed as your Aave position might go underwater. ';
      }
      const cTokens = [
        'cUSDC',
        'cDAI',
        'cETH',
        'cUSDT',
        'cREP',
        'cZRX',
        'cBAT',
        'cWBTC'
      ];
      if (cTokens.some(cToken => hasToken(this.pool, cToken))) {
        return 'Adding liquidity failed as your Compound position might go underwater. ';
      }
      return 'Adding liquidity failed as one of the underlying tokens blocked the transfer. ';
    },
    hasCustomToken() {
      if (this.validationError || this.tokenError) {
        return false;
      }
      for (const token of this.pool.tokens) {
        const tokenMetadata = this.web3.tokenMetadata[token.checksum];
        if (!tokenMetadata || !tokenMetadata.whitelisted) {
          return true;
        }
      }
      return false;
    },
    rateChangeWarning() {
      if (this.validationError || this.tokenError) {
        return false;
      }
      if (!this.isMultiAsset) {
        return false;
      }
      const token = this.findFrontrunnableToken;
      if (!token) {
        return false;
      }
      const frontrunningThreshold = 1 - BALANCE_BUFFER;
      const address = token.checksum;
      const amount = bnum(this.amounts[address]);
      const denormAmount = denormalizeBalance(amount, token.decimals);
      const balance = this.web3.balances[address];
      const amountToBalanceRatio = denormAmount.div(balance);
      return (
        amountToBalanceRatio.gt(frontrunningThreshold) &&
        amountToBalanceRatio.lte(1)
      );
    },
    slippage() {
      if (this.validationError || this.tokenError) {
        return undefined;
      }
      if (this.isMultiAsset) {
        return undefined;
      }
      const tokenInAddress = this.activeToken;
      if (!this.amounts[tokenInAddress]) {
        return undefined;
      }
      const tokenIn = this.pool.tokens.find(
        token => token.checksum === tokenInAddress
      );
      const amount = bnum(this.amounts[tokenInAddress]);

      const tokenBalanceIn = denormalizeBalance(
        tokenIn.balance,
        tokenIn.decimals
      );
      const tokenWeightIn = bnum(tokenIn.denormWeight).times('1e18');
      const poolSupply = denormalizeBalance(this.totalShares, 18);
      const totalWeight = bnum(this.pool.totalWeight).times('1e18');
      const tokenAmountIn = denormalizeBalance(
        amount,
        tokenIn.decimals
      ).integerValue(BigNumber.ROUND_UP);
      const swapFee = bnum(this.pool.swapFee).times('1e18');

      const poolAmountOut = calcPoolOutGivenSingleIn(
        tokenBalanceIn,
        tokenWeightIn,
        poolSupply,
        totalWeight,
        tokenAmountIn,
        swapFee
      );
      const expectedPoolAmountOut = tokenAmountIn
        .times(tokenWeightIn)
        .times(poolSupply)
        .div(tokenBalanceIn)
        .div(totalWeight);
      const one = bnum(1);
      const slippage = one.minus(poolAmountOut.div(expectedPoolAmountOut));
      return slippage;
    },
    findFrontrunnableToken() {
      if (this.validationError) {
        return;
      }
      let maxAmountToBalanceRatio = bnum(0);
      let maxRatioToken = undefined;
      for (const token of this.pool.tokens) {
        const address = token.checksum;
        const amount = bnum(this.amounts[address]);
        const denormAmount = denormalizeBalance(amount, token.decimals);
        const balance = this.web3.balances[address];
        const amountToBalanceRatio = denormAmount.div(balance);
        if (amountToBalanceRatio.gt(maxAmountToBalanceRatio)) {
          maxAmountToBalanceRatio = amountToBalanceRatio;
          maxRatioToken = token;
        }
      }
      return maxRatioToken;
    },
    isMultiAsset() {
      return this.type === 'MULTI_ASSET';
    }
  },
  methods: {
    ...mapActions(['joinPool', 'joinswapExternAmountIn']),
    handleChange(changedAmount, changedToken) {
      const ratio = bnum(changedAmount).div(changedToken.balance);
      if (this.isMultiAsset) {
        this.poolTokens = calcPoolTokensByRatio(ratio, this.totalShares);
      } else {
        const tokenIn = this.pool.tokens.find(
          token => token.checksum === this.activeToken
        );
        const amount = new BigNumber(this.amounts[tokenIn.checksum]);

        const maxInRatio = 1 / 2;
        if (amount.div(tokenIn.balance).gt(maxInRatio)) {
          return;
        }

        const tokenBalanceIn = denormalizeBalance(
          tokenIn.balance,
          tokenIn.decimals
        );
        const tokenWeightIn = bnum(tokenIn.denormWeight).times('1e18');
        const poolSupply = denormalizeBalance(this.totalShares, 18);
        const totalWeight = bnum(this.pool.totalWeight).times('1e18');
        const tokenAmountIn = denormalizeBalance(
          amount,
          tokenIn.decimals
        ).integerValue(BigNumber.ROUND_UP);
        const swapFee = bnum(this.pool.swapFee).times('1e18');

        this.poolTokens = calcPoolOutGivenSingleIn(
          tokenBalanceIn,
          tokenWeightIn,
          poolSupply,
          totalWeight,
          tokenAmountIn,
          swapFee
        ).toString();
      }

      this.pool.tokens.forEach(token => {
        if (!this.isMultiAsset) {
          return;
        }
        if (token.checksum === changedToken.checksum) {
          return;
        }
        this.amounts[token.checksum] = ratio.isNaN()
          ? ''
          : ratio.times(token.balance).toString();
      });
    },
    handleMax(token) {
      const balance = this.web3.balances[token.checksum];
      const amount = normalizeBalance(balance, token.decimals);
      this.amounts[token.checksum] = amount.toString();
      this.handleChange(amount, token);
    },
    lowerAmounts() {
      const frontrunningThreshold = 1 - BALANCE_BUFFER;
      const token = this.findFrontrunnableToken;
      const address = token.checksum;
      const balance = this.web3.balances[address];
      const safeAmount = bnum(balance).times(frontrunningThreshold);
      const normalizedAmount = normalizeBalance(safeAmount, token.decimals);
      this.amounts[token.checksum] = normalizedAmount.toString();
      this.handleChange(normalizedAmount, token);
    },
    handleSelectType(type) {
      this.type = type;
      this.poolTokens = null;
      this.amounts = Object.fromEntries(
        this.pool.tokens.map(token => {
          return [token.checksum, ''];
        })
      );
    },
    onTokenSelect(token) {
      this.activeToken = token;
    },
    async handleSubmit() {
      this.loading = true;
      if (this.isMultiAsset) {
        const params = {
          poolAddress: this.pool.id,
          poolAmountOut: this.poolTokens,
          maxAmountsIn: this.pool.tokensList.map(tokenAddress => {
            const token = this.pool.tokens.find(
              token => token.checksum === tokenAddress
            );
            const amount = bnum(this.amounts[token.checksum]);
            const inputAmountIn = denormalizeBalance(amount, token.decimals)
              .div(1 - BALANCE_BUFFER)
              .integerValue(BigNumber.ROUND_UP);
            const balanceAmountIn = bnum(this.web3.balances[token.checksum]);
            const tokenAmountIn = BigNumber.min(inputAmountIn, balanceAmountIn);
            return tokenAmountIn.toString();
          })
        };
        const txResult = await this.joinPool(params);
        if (isTxReverted(txResult)) {
          this.transactionFailed = true;
        }
      } else {
        const tokenIn = this.pool.tokens.find(
          token => token.checksum === this.activeToken
        );
        const tokenAmountIn = denormalizeBalance(
          this.amounts[tokenIn.checksum],
          tokenIn.decimals
        )
          .integerValue(BigNumber.ROUND_UP)
          .toString();
        const minPoolAmountOut = bnum(this.poolTokens)
          .times(1 - BALANCE_BUFFER)
          .integerValue(BigNumber.ROUND_UP)
          .toString();
        const params = {
          poolAddress: this.pool.id,
          tokenInAddress: this.activeToken,
          tokenAmountIn,
          minPoolAmountOut
        };
        await this.joinswapExternAmountIn(params);
      }
      this.loading = false;
    },
    isInputValid(token) {
      const tokenAddress = token.checksum;
      if (!this.isMultiAsset && this.activeToken !== tokenAddress) {
        return true;
      }
      const amount = this.amounts[tokenAddress];
      if (!amount || isNaN(amount)) {
        return false;
      }
      const amountNumber = denormalizeBalance(amount, token.decimals);
      const balance = this.web3.balances[tokenAddress];
      return amountNumber.lte(balance);
    },
    formatBalance(balanceString, tokenDecimals) {
      return normalizeBalance(balanceString, tokenDecimals);
    }
  }
};
</script>
