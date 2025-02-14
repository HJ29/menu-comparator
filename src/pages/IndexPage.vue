<template>
  <q-page class="row items-center justify-center">
    <div class="column q-gutter-y-sm" :style="{ minWidth: '240px' }">
      <q-file outlined v-model="newFile" label="New File" dense accept=".csv" />
      <q-file outlined v-model="oldFile" label="Old File" dense accept=".csv" />
      <q-btn color="white" text-color="black" label="Download" @click="onClickDownload" />
    </div>
  </q-page>
</template>

<script setup lang="ts">
import { ref } from "vue";
const newFile  = ref()
const oldFile  = ref()

function fileToJson(file): Promise<string[][]> {
  return new Promise((resolve) => {
    if (!file) throw new Error('No File');
    const reader = new FileReader();
    reader.onload = (event) => {
      const csv = event.target?.result;
      const json = csvToJson(csv);
      resolve(json);
    };
    reader.readAsText(file);
  })
};

function csvToJson(csv) {
  const rows = csv.split("\n").map(row => row.trim()).filter(row => row);
  const headers = rows.shift().split(",");
  return rows.map(row => {
    const values = row.split(",");
    return headers.reduce((acc, header, index) => {
      acc[header] = values[index];
      return acc;
    }, {});
  });
};

async function onClickDownload() {
  const newJson = await fileToJson(newFile.value)
  const oldJson = await fileToJson(oldFile.value)
  compareJson({newJson, oldJson})
}

function groupMenu(menu) {
  const products = menu.reduce((products, row) => {
    const productId = `${row.product_id}|${row.product_sku}|${row.product_name}`.replaceAll('"', '');
    const groupId = `${row.addon_category_id}|${row.addon_category}`.replaceAll('"', '');
    const addonId = `${row.addon_id}|${row.addon_sku}|${row.addon_name}`.replaceAll('"', '');
    if (!products[productId]) {
      products[productId] = {}
    }
    if (!products[productId][groupId]) {
      products[productId][groupId] = []
    }
    if (!products[productId][groupId].includes(addonId)) {
      products[productId][groupId].push(addonId)
      products[productId][groupId].sort((a, b) => a.localeCompare(b))
    }
    return products
  }, {})
  return Object.keys(products).reduce((groups, productId) => {
    const product = products[productId];
    Object.keys(product).forEach((groupId) => {
      if (!groups[groupId]) {
        groups[groupId] = []
      }
      const groupUniqueId = [groupId, ...product[groupId]].join('|');
      let index = groups[groupId].findIndex((group) => group.uniqueId === groupUniqueId)
      if (index < 0) {
        index = groups[groupId].length
        groups[groupId].push({
          uniqueId: groupUniqueId,
          id: groupId,
          addons: product[groupId],
          products: []
        })
      }
      groups[groupId][index].products.push(productId);
      groups[groupId].sort((a, b) => a.uniqueId.localeCompare(b.uniqueId))
    })
    return groups;
  }, {})
}

function compareJson({ newJson, oldJson }: { newJson: string[][]; oldJson: string[][] }) {
  const oldGroups = groupMenu(oldJson)
  const newGroups = groupMenu(newJson)
  const actions: string[][] = [['status', 'group', 'addons', 'addon count', 'products', 'product count']]
  const remainUniqueIds: string[] = []
  Object.keys(newGroups).forEach(newGroupId => {
    const newGroup = newGroups[newGroupId];
    const oldGroup = oldGroups[newGroupId];
    newGroup.forEach(newUniqueGroup => {
      const existingOldUniqueGroup = oldGroup?.find((oldUniqueGroup) => oldUniqueGroup.uniqueId === newUniqueGroup.uniqueId)
      const action = [newUniqueGroup.id, `"${newUniqueGroup.addons.join('\n')}"`, newUniqueGroup.addons.length, `"${newUniqueGroup.products.join('\n')}"`, newUniqueGroup.products.length];
      if (!existingOldUniqueGroup) {
        action.unshift('UPDATE');
      } else if (!remainUniqueIds.includes(existingOldUniqueGroup.uniqueId)) {
        remainUniqueIds.push(existingOldUniqueGroup.uniqueId)
        action.unshift('REMAIN');
      }
      actions.push(action);
    })
  })
  Object.keys(oldGroups).forEach(oldGroupId => {
    const oldGroup = oldGroups[oldGroupId];
    oldGroup.forEach(oldUniqueGroup => {
      if (!remainUniqueIds.includes(oldUniqueGroup.uniqueId)) {
        actions.push(['DELETE', oldUniqueGroup.id, `"${oldUniqueGroup.addons.join('\n')}"`, oldUniqueGroup.addons.length, `"${oldUniqueGroup.products.join('\n')}"`, oldUniqueGroup.products.length])
      }
    })
  })
  const actionOrders = ['UPDATE', 'DELETE', 'REMAIN']
  actions.sort((a, b) => actionOrders.indexOf(`${a[0]}`) - actionOrders.indexOf(`${b[0]}`))
  const rows = actions.map(row => Object.values(row).join(",")).join("\n");
  downloadCsvFile(rows)
}

function downloadCsvFile(csv) {
  const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.setAttribute("href", url);
  link.setAttribute("download", "compare-result.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}
</script>
