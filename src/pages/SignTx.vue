<template>
  <h2>Confirm tx content and sign tx</h2>
  <div>
    <h3>Wallet info</h3>
    <Multisig />
  </div>
  <div>
    <h3>Transaction info</h3>
    <ImportUnsignedTx />
  </div>
  <div>
    <h3>Signer info</h3>
    <Signer />
  </div>
  <button :disabled="!hasSignerAddress" @click="signTx">Sign tx</button>
  <div v-if="!hasSignerAddress">
    <div>
      Current signer address is not in multisig wallet public key.
    </div>
    <div>
      Please check the multisig wallet definition, and also your signer (Keplr) account.
    </div>
  </div>
  <div>
    Signature: <code>{{ displaySignature }}</code>
  </div>
  <div v-if="signature">
    If download was not start automatically, <button @click="exportSignature">click here</button> to download the signature
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue';

import Signer from '@/components/Signer.vue';
import Multisig from '@/components/Multisig.vue';
import ImportUnsignedTx from '@/components/tx/ImportUnsignedTx.vue';

import { readAccountChainInfo } from '@/cosmos/client';
import { signTxAmino, SingleSignature } from '@/cosmos/signing';

import { useSignerStore, useMultisigStore, useTxStore } from '@/stores';
import { encodeBase64, generateFileAndDownload } from '@/utils/utils';

const signerStore = useSignerStore();
const multisigStore = useMultisigStore();
const txStore = useTxStore();

const signature = ref(null as SingleSignature | null);
const displaySignature = computed(() => {
  if (!signature.value) {
    return '-';
  }
  return encodeBase64(signature.value.signature);
});
const hasSignerAddress = computed(() => multisigStore.hasAddress(signerStore.address));

async function signTx() {
  await signerStore.init();
  if (!multisigStore.hasAddress(signerStore.address)) {
    throw new Error(`Signer address (${signerStore.address}) not in multisig wallet (${multisigStore.address})`);
  }
  const { accountNumber, sequence } = await readAccountChainInfo(multisigStore.address);
  const aminoSignDoc = txStore.aminoSignDoc(accountNumber, sequence);
  signature.value = await signTxAmino(signerStore.offlineSigner!, signerStore.address, aminoSignDoc);
  exportSignature();
}

function exportSignature() {
  const json = signature.value!.toJSON();
  const exported = JSON.stringify(json, null, 2);
  generateFileAndDownload(exported, 'signature.json');
}
</script>
