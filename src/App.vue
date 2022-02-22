<script setup>
import { computed, ref, watchEffect } from 'vue';
import { useStorage, useClipboard } from '@vueuse/core';

const { copy, copied } = useClipboard();

const uidStr = useStorage('uids', '58692, 48017, 52717');
const pidStr = useStorage('pids', 'a001, a002, a003');

const uids = ref([])
const pids = ref([])

watchEffect(() => {
  uids.value = uidStr.value.split(', ')
})
watchEffect(() => {
  pids.value = pidStr.value.split(', ')
})

const fetcher = computed(() => {
  return `
  const uids = [${uids.value.map(u => `"${u}"`).join(', ')}];\n
  const pids = [${pids.value.map(p => `"${p}"`).join(', ')}];\n
  const res = [];
  const reqs = uids.map(
    u => fetch('https://zerojudge.tw/UserStatistic?id=' + u)
      .then(res => res.text())
      .then(text => {
        const parser = new DOMParser();
        const doc = parser.parseFromString(text, 'text/html');
        const name = doc.querySelector('span > a').innerText;
        const acs = doc.querySelectorAll('.acstyle');
        const pid = []
        for ( let i=0; i<acs.length; i++ ) {
          pid.push(acs[i].innerText);
        }
        const pres = {}
        for ( let i=0; i<pids.length; i++ ) {
          if ( pid.includes(pids[i]) ) {
            pres[pids[i]] = true;
          } else {
            pres[pids[i]] = false;
          }
        }

        res.push({
          uid: u,
          name,
          ...pres,
        })
      })
  )
  Promise.all(reqs).then(() => console.log('done')).then(() => console.log(res))
  `.trim();
});

const res = ref('');
const data = computed(() => {
  return res.value ? JSON.parse(res.value) : '';
});

</script>

<template>
  <div style="padding: 20px">
    <div>
      uids:
      <textarea v-model="uidStr" style="width: 100%" />
    </div>
    <div>
      pid:
      <textarea v-model="pidStr" style="width: 100%" />
    </div>

    <span>login zerojudge and run this -> </span>
    <button @click="copy(fetcher)">
      <span v-if='!copied'>Copy</span>
      <span v-else>Copied!</span>
    </button>
    <span> &lt;- in console</span>

    <div style="padding-top: 10px">
      copy object and paste it here:
      <textarea v-model="res" style="width: 100%" />
    </div>

    <table v-if="data" style="padding-top: 20px;">
      <thead>
        <tr>
          <th>id</th>
          <th>name</th>
          <th v-for="p in pids" :key="p">{{ p }}</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="d in data" :key="d.no">
          <td>{{ d.uid }}</td>
          <td>{{ d.name }}</td>
          <td
            v-for="p in pids"
            :key="p"
            :style="{ backgroundColor: d[p] ? 'green' : 'transparent', textAlign: 'center' }"
          >
            {{ d[p] ? 'v' : 'x' }}
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>
