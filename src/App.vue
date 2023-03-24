<script setup>
import { CdxLookup } from '@wikimedia/codex';
import { ref } from 'vue';
const options = [
  {
    value: 'item',
    label: 'Items (Q:)',
    shortcut: 'Q:',
  },
  {
    value: 'property',
    label: 'Properties (P:)',
    shortcut: 'P:',
  },
  {
    value: 'lexeme',
    label: 'Lexemes (L:)',
    shortcut: 'L:',
  },
];
const namespace = ref('item');
const searchInput = ref('');
const menuItems = ref([]);
const currentSearchTerm = ref('');
async function fetchResults(searchTerm, offset) {
  const params = new URLSearchParams({
    origin: '*',
    action: 'wbsearchentities',
    format: 'json',
    limit: '10',
    props: 'url',
    language: 'en',
    type: namespace.value,
    search: searchTerm,
  });
  if (offset) {
    params.set('continue', `${offset}`);
  }
  const response = await fetch(`https://www.wikidata.org/w/api.php?${params.toString()}`);
  return response.json();
}
async function onInput(value) {
  if (value.length >= 2) {
    const namespaceOption = options.find((option) => option.shortcut === value.substring(0, 2));
    if (namespaceOption) {
      namespace.value = namespaceOption.value;
      value = value.substring(2);
    }
  }

  // Internally track the current search term.
  currentSearchTerm.value = value;

  // Do nothing if we have no input.
  if (!value || value === '') {
    menuItems.value = [];
    return;
  }

  try {
    const data = await fetchResults(value);

    // Make sure this data is still relevant first.
    if (currentSearchTerm.value !== value) {
      return;
    }

    // Reset the menu items if there are no results.
    if (!data.search || data.search.length === 0) {
      menuItems.value = [];
      return;
    }

    // Build an array of menu items.
    const results = data.search.map((result) => {
      return {
        label: result.label,
        value: result.id,
        description: result.description,
      };
    });

    // Update menuItems.
    menuItems.value = results;
  } catch (e) {
    // On error, set results to empty.
    menuItems.value = [];
  }
}
function deduplicateResults(results) {
  const seen = new Set(menuItems.value.map((result) => result.value));
  return results.filter((result) => !seen.has(result.value));
}

async function onLoadMore() {
  if (!currentSearchTerm.value) {
    return;
  }

  const data = await fetchResults(currentSearchTerm.value, menuItems.value.length);

  if (!data.search || data.search.length === 0) {
    return;
  }

  const results = data.search.map((result) => {
    return {
      label: result.label,
      value: result.id,
      description: result.description,
    };
  });
  // Update menuItems.
  const deduplicatedResults = deduplicateResults(results);
  menuItems.value.push(...deduplicatedResults);
}
const menuConfig = {
  visibleItemLimit: 6,
};
function onNamespaceChange(event) {
  const namespaceOption = options.find((option) => option.value === event.target.value);
  const allShortcuts = options.map((option) => option.shortcut);
  if (allShortcuts.includes(searchInput.value.substring(0, 2))) {
    searchInput.value = namespaceOption.shortcut + searchInput.value.substring(2);
  } else {
    searchInput.value = namespaceOption.shortcut + searchInput.value;
  }
  if (searchInput.value.length >= 2) {
    onInput(searchInput.value);
  }
}
function onSelectionChange(entityId) {
  const url = `https://www.wikidata.org/wiki/Special:EntityData/${entityId}`;
  window.location = url;
}
</script>

<template>
  <div>
    <select v-model="namespace" @change="onNamespaceChange">
      <option v-for="option in options" :key="option.value" :value="option.value">
        {{ option.label }}
      </option>
    </select>
    <div>
      <cdx-lookup
        v-model="searchInput"
        v-model:selected="selection"
        @update:selected="onSelectionChange"
        :menu-items="menuItems"
        :menu-config="menuConfig"
        @input="onInput"
        @load-more="onLoadMore"
        placeholder="Search Wikidata"
      >
        <template #no-results> No results found. </template>
      </cdx-lookup>
    </div>
  </div>
</template>

<style scoped></style>
