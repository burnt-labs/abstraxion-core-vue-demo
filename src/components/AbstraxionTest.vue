<script lang="ts">
import { onMounted, onUnmounted, ref } from "vue";
import { AbstraxionAuth } from "@burnt-labs/abstraxion-core";

const abstraxion = new AbstraxionAuth(
  "https://testnet-rpc.xion-api.com:443",
  [
    "xion1z70cvc08qv5764zeg3dykcyymj5z6nu4sqr7x8vl4zjef2gyp69s9mmdka",
    {
      address:
        "xion1z70cvc08qv5764zeg3dykcyymj5z6nu4sqr7x8vl4zjef2gyp69s9mmdka",
      amounts: [{ denom: "uxion", amount: "1000000" }],
    },
  ],
  true,
  [
    {
      denom: "uxion",
      amount: "1000000",
    },
  ]
);

export default {
  setup() {
    const isLoading = ref(false);
    const isPolling = ref(false);
    const isLoggedIn = ref(false);

    // Login
    const triggerStartAbstraxion = async () => {
      isLoading.value = true;
      await abstraxion.login();
      isLoading.value = false;
    };

    async function handleSign(): Promise<void> {
      const signerClient = await abstraxion.getSigner();
      if (!signerClient) {
        throw new Error("No signer client");
      }
      if (!abstraxion.signArbWallet) {
        throw new Error("No sign arb wallet");
      }
      const response = await abstraxion.signArbWallet?.signArb?.(
        signerClient.granteeAddress,
        "FOOBAR"
      );
      // eslint-disable-next-line no-console -- We log this for testing purposes.
      console.log(response);
    }

    function getTimestampInSeconds(date: Date | null): number {
      if (!date) return 0;
      const d = new Date(date);
      return Math.floor(d.getTime() / 1000);
    }

    const now = new Date();
    now.setSeconds(now.getSeconds() + 15);
    const oneYearFromNow = new Date();
    oneYearFromNow.setFullYear(oneYearFromNow.getFullYear() + 1);

    async function claimSeat(): Promise<void> {
      isLoading.value = true;
      const bech32Address = await abstraxion.getAccountAddress();
      const msg = {
        sales: {
          claim_item: {
            token_id: String(getTimestampInSeconds(now)),
            owner: bech32Address,
            token_uri: "",
            extension: {},
          },
        },
      };

      try {
        const client = await abstraxion.getSigner();
        const claimRes = await client?.execute(
          bech32Address,
          "xion1z70cvc08qv5764zeg3dykcyymj5z6nu4sqr7x8vl4zjef2gyp69s9mmdka",
          msg,
          {
            amount: [{ amount: "0", denom: "uxion" }],
            gas: "500000",
          },
          "",
          []
        );

        console.log(claimRes);
      } catch (error) {
        console.log(error);
      } finally {
        isLoading.value = false;
      }
    }

    // Subscribe to authentication state changes
    const unsubscribe = abstraxion.subscribeToAuthStateChange((newState) => {
      isLoggedIn.value = newState;
    });

    // Persistent State
    onMounted(async () => {
      isLoading.value = true;
      await abstraxion.authenticate();
      isLoading.value = false;
    });

    // Redirect Case
    onMounted(async () => {
      const searchParams = new URLSearchParams(window.location.search);
      const isGranted = searchParams.get("granted");
      if (isGranted) {
        isPolling.value = true;
        await abstraxion.pollForGrants();
        isPolling.value = false;
      }
    });

    // Cleanup on component unmount
    onUnmounted(() => {
      unsubscribe();
    });

    return {
      isLoading,
      isPolling,
      isLoggedIn,
      triggerStartAbstraxion,
      handleSign,
      claimSeat,
      abstraxion,
    };
  },
};
</script>

<template>
  <div class="abstraxion-demo">
    <img
      alt="Vue logo"
      class="logo"
      src="../assets/logo.svg"
      width="50"
      height="50"
    />
    <div v-if="isLoading || isPolling">LOADING...</div>
    <div v-else>
      <div class="logged-in-block" v-if="isLoggedIn">
        <button @click="claimSeat()">CLAIM SEAT</button>
        <button @click="handleSign()">SIGN ARB</button>
        <button @click="abstraxion.logout">LOG OUT</button>
      </div>
      <div v-else><button @click="triggerStartAbstraxion">CONNECT</button></div>
    </div>
  </div>
</template>

<style scoped>
.abstraxion-demo {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 1rem;
  color: white;
}

.logged-in-block {
  display: flex;
  flex-direction: column;
  justify-content: center;
  max-width: 20rem;
  gap: 1rem;
}

button {
  background-color: #fff;
  color: black;
  border: none;
  padding: 1rem 4rem;
  border-radius: 4px;

  &:hover {
    cursor: pointer;
    opacity: 90%;
  }
}

.logo {
  display: block;
}
</style>
