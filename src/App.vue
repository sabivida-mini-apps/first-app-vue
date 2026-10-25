<script setup lang="ts">
import { onMounted, ref } from 'vue';
import { sabivida, signIn, type Member, type MiniAppContext, type TaskOutcome } from './sabivida';

type State =
  | { kind: 'loading' }
  | { kind: 'outside' }
  | { kind: 'failed' }
  | { kind: 'ready'; member: Member; context: MiniAppContext };

const OUTCOME: Record<TaskOutcome, string> = {
  added: 'Added to the household plans.',
  declined: 'Not added: the member declined.',
  unavailable: 'SabiVida cannot add steps right now.',
};

const state = ref<State>({ kind: 'loading' });
const outcome = ref<TaskOutcome | null>(null);
const asking = ref(false);
const sdk = sabivida();

// A launch token is accepted once: sign in once, when the page mounts.
onMounted(async () => {
  try {
    const { member, context } = await signIn();
    state.value = { kind: 'ready', member, context };
  } catch (error) {
    state.value = { kind: (error as Error).message === 'not_embedded' ? 'outside' : 'failed' };
  }
});

async function addStep() {
  asking.value = true;
  // SabiVida shows the member this exact text before anything is added.
  outcome.value = (await sdk?.addTask({ title: 'Book a consular appointment', dueDate: null })) ?? 'unavailable';
  asking.value = false;
}
</script>

<template>
  <main>
    <p v-if="state.kind === 'loading'" class="muted">Signing in with SabiVida…</p>
    <template v-else-if="state.kind === 'outside'">
      <h1>Open this app from SabiVida</h1>
      <p class="muted">It signs members in through SabiVida, so it only works inside SabiVida.</p>
    </template>
    <template v-else-if="state.kind === 'failed'">
      <h1>Sign-in failed</h1>
      <p class="muted">Your server refused the launch token. Check SABIVIDA_HOST and MINI_APP_ID.</p>
    </template>
    <template v-else>
      <h1>Hello, {{ state.member.name ?? 'SabiVida member' }}</h1>
      <div class="card">
        <p>Your member id for this app: <code>{{ state.member.subject.slice(0, 12) }}…</code></p>
        <p class="muted">
          Language {{ state.context.locale }} · on {{ state.context.platform }}
          <template v-if="state.context.startParam"> · opened with “{{ state.context.startParam }}”</template>
        </p>
      </div>
      <div class="actions">
        <button v-if="sdk?.supports('add_task')" class="primary" :disabled="asking" @click="addStep">
          Add a step to my plans
        </button>
        <button @click="sdk?.close()">Close</button>
      </div>
      <p v-if="outcome" role="status">{{ OUTCOME[outcome] }}</p>
    </template>
  </main>
</template>
