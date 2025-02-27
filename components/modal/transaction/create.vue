<template>
  <modal-base :title="$t('modals.newTransaction.title')" :opened="opened" :name="'transaction-create-modal'">
    <div class="w-full flex flex-col gap-2">
      <div class="w-full flex gap-2">
        <div class="form-control w-full">
          <label class="label">
            <span class="label-text">{{ $t('modals.newTransaction.placeholders.transactionAccount') }}</span>
          </label>

          <select-account class="w-full"
                          v-model="account">

          </select-account>
        </div>
        <div class="form-control w-full">
          <label class="label">
            <span class="label-text">{{ $t('modals.newTransaction.placeholders.transactionTag') }}</span>
          </label>

          <select-transaction-tag class="w-full"
                                        v-model.number="parentTag"
                                        :searchable="true"
                                        :can-be-without-parent="false"
                                        :tags-tree="tagsTree">
          </select-transaction-tag>
        </div>

      </div>

      <div class="form-control w-full">
        <label class="label">
          <span class="label-text">{{ $t('modals.newTransaction.placeholders.transactionAmount') }}</span>
        </label>

        <div class="join w-full">
          <div v-if="!signChoice" class="join-item flex justify-center items-center px-4 bg-base-200">
            <svg v-if="sign > 0" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="w-5 h-5">
              <path stroke-linecap="round" stroke-linejoin="round" d="M12 6v12m6-6H6" />
            </svg>
            <svg v-else xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="w-5 h-5">
              <path stroke-linecap="round" stroke-linejoin="round" d="M18 12H6" />
            </svg>
          </div>
          <select v-else class="select select-bordered join-item" v-model.number="sign">
            <option :value="-1"> - </option>
            <option :value="1"> + </option>
          </select>

          <input type="number" class="input input-bordered join-item w-full" v-model.number="amount" @change="amountChanged">
          <div v-if="currency !== undefined" class="join-item flex justify-center items-center px-4 bg-base-200">
            <p class="font-bold">
              {{ currency.symbol }}
            </p>
          </div>
        </div>
      </div>

      <div class="form-control w-full">
        <label class="label">
          <span class="label-text">{{ $t('modals.newTransaction.placeholders.transactionDate') }}</span>
        </label>

        <Datepicker class="input-bordered dp-h-12" v-model="date" :teleport="true" teleport-center/>
      </div>

      <div class="form-control w-full">
        <label class="label">
          <span class="label-text">{{ $t('modals.newTransaction.placeholders.transactionDescription') }}</span>
        </label>
        <textarea class="textarea input-bordered"
                  :placeholder="$t('modals.newTransaction.placeholders.transactionDescription')"
                  v-model.trim="description"
                  :maxlength="configs.maxDescriptionLength">
        </textarea>
      </div>
    </div>

    <div class="modal-action">
      <button @click="close" class="btn btn-sm btn-ghost">{{ $t('modals.buttons.cancel') }}</button>
      <button @click="create" class="btn btn-sm btn-success">{{ $t('modals.buttons.create') }}</button>
    </div>
  </modal-base>
</template>

<script setup>
import Datepicker from '@vuepic/vue-datepicker';

const props = defineProps({
  opened: {
    required: true,
    type: Boolean
  }
})

const {$serverConfigs, $transactionsTagsApi, $transactionsApi, $currenciesApi, $accountsApi, $toastsManager} = useNuxtApp();
const configs = $serverConfigs.configs.transactions;

const emit = defineEmits(['close', 'reloadTransactions']);
const { t } = useI18n();

const accounts = $accountsApi.getAccounts();
const accountsMap = $accountsApi.getAccountsMap();
const currenciesMap = $currenciesApi.getCurrenciesMap();
const tagsTree = $transactionsTagsApi.getTagsTree();
const tagsMap = $transactionsTagsApi.getTagsTreeMap();

const account = ref();
const amount = ref();
const description = ref("");
const parentTag = ref();
const sign = ref(1);
const signChoice = ref(true);
const date = ref(new Date());

watch(parentTag, () => {
  if (parentTag.value === undefined) {
    signChoice.value = true;
    return;
  }

  const parentTagObject = tagsMap.value.get(parentTag.value);

  if (parentTagObject === undefined || parentTagObject.tag === undefined) {
    signChoice.value = true;
    return;
  }

  if (parentTagObject.tag.type !== 0)
    sign.value = parentTagObject.tag.type

  signChoice.value = parentTagObject.tag.type === 0;
});

const currency = computed(() => {
  if (account.value === undefined)
    return undefined;

  const accountObject = accountsMap.value.get(account.value);

  if (accountObject === undefined)
    return undefined;

  let currencyId = accountObject.currencyId;

  return currenciesMap.value.get(currencyId);
});

const amountChanged = () => {
  if (signChoice.value)
    sign.value = Math.sign(amount.value);

  amount.value = Math.abs(amount.value);
}

const close = () => {
  emit('close')
}

const create = () => {
  if (account.value === undefined || amount.value === undefined || amount.value === 0 || parentTag.value === undefined || !date.value)
    return;

  close();

  $transactionsApi.newTransaction(parentTag.value, account.value, date.value, amount.value * sign.value, description.value.length > 0 ? description.value : null).then((s) => {
    if (s) {
      $toastsManager.pushToast(t("modals.newTransaction.messages.success"), 2500, "success");
      emit('reloadTransactions');
      $accountsApi.reloadAccounts();
    }else {
      $toastsManager.pushToast(t("modals.newTransaction.messages.error"), 3000,"error");
    }
  });

}
</script>

<style scoped>

</style>