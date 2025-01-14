<script setup>
  import { ref, watch, reactive, computed } from 'vue'
  import TranslatableTargets from './translatable-targets.vue'
  import ContentReference from'./content-reference.vue'
  import { useRouter } from 'vue-router'
  import languageCodes from './language-codes.js'

  const router = useRouter()
  const selected = computed(() => router.currentRoute?.value?.params?.translatableItemId)

  const [
    env,
    appStateNonReactive,
    itemDomain,
  ] = await Promise.all([
    Agent.environment(),
    Agent.state('app'),
    selected.value ? Agent.metadata(selected.value).then(md => md.domain) : window.location.host
  ])


  if (!appStateNonReactive.domains  ) appStateNonReactive.domains   = [itemDomain]
  if (!appStateNonReactive.languages) appStateNonReactive.languages = []

  const appState = reactive(appStateNonReactive)
  const domain = ref(itemDomain)
  const drawer = ref(true)
  const translatableItems = await Agent.query('translatable-items', [domain.value])
  const translatableItemIds = reactive(translatableItems.map(i => i.translatable_item))
  const editing = ref(false)
  const domainEntry = ref(itemDomain)
  const search = ref('')
  const enteredSearch = ref('')
  const searchResults = ref(null)

  console.log(appState.languages)

  const loggedIn = env.auth.provider !== 'anonymous'

  async function createNewItem() {
    const id = Agent.uuid()
    const state = await Agent.state(id)
    state.name = 'Wooo?'
    state.translations = {
      source_language: 'en-us',
      paths: [
        ['name']
      ]
    }
    selected.value = id
    translatableItemIds.unshift(id)
  }

  watch(() => domain.value, async () => {
    const translatableItems = await Agent.query('translatable-items', [domain.value])
    console.log('translatable items', translatableItems)
    translatableItemIds.splice(0, translatableItemIds.length)
    translatableItemIds.push(...translatableItems.map(i => i.translatable_item))
    router.push('/')
  })

  watch(() => appState.languages, () => {
    console.log(appState.languages)
  })

  async function addDomain(e) {
    const d = domainEntry.value.trim()
    // TODO: verfiy domain name

    if (!appState.domains.includes(d)) appState.domains.push(d)

    domain.value = d
  }

  function addDomainOnUpdate() {
    const d = domainEntry.value

    if (appState.domains.includes(d)) domain.value = d
  }

  function logOut() {
    Agent.logout()
  }

  function logIn() {
    Agent.login()
  }

  function removeDomain(domain) {
    const index = appState.domains.indexOf(domain)
    console.log(index)
    if (index > -1) appState.domains.splice(index, 1)
  }

  async function fetchSearchResults(query) {
    enteredSearch.value = query
    searchResults.value = await Agent.query('search-translations', [domain.value, query])
  }

  window.appState = appState
</script>

<template>
  <v-app v-if="loggedIn">
    <v-main>
      <v-navigation-drawer
        v-model="drawer"
      >
        <template v-slot:prepend>
          <v-toolbar
            color="primary"
          >
            <v-menu location="bottom">
              <template v-slot:activator="slot">
                <v-btn
                  icon="fa-solid fa-gear"
                  v-bind="slot.props"
                />
              </template>
                <v-list>
                  <v-dialog
                    max-width="700"
                    style="
                      margin-top: 32px !important;
                      align-items: flex-start !important;
                    "
                  >
                    <template v-slot:activator="{ props: activatorProps }">
                      <v-list-item
                        title="Select Languages"
                        prepend-icon="fa-solid fa-language"
                        v-bind="activatorProps"
                      />
                    </template>
                    <template v-slot:default="{ isActive }">
                      <v-autocomplete
                        v-if="isActive"
                        autofocus
                        v-model="appState.languages"
                        label="Select languages to show"
                        variant="solo"
                        multiple
                        chips
                        closable-chips
                        :items="Object.entries(languageCodes).map(([ code, name ]) => ({ title: name, subtitle: code, value: code }))"
                      />
                    </template>
                  </v-dialog>
                  <v-dialog
                    max-width="700"
                    style="
                      margin-top: 32px !important;
                      align-items: flex-start !important;
                    "
                  >
                    <template v-slot:activator="{ props: activatorProps }">
                      <v-list-item
                        title="Change Domain"
                        prepend-icon="fa-solid fa-globe"
                        v-bind="activatorProps"
                      />
                    </template>
                    <template v-slot:default="{ isActive }">
                      <v-combobox
                        v-if="isActive"
                        autofocus
                        v-model="domainEntry"
                        label="Select Domain or Enter a New One"
                        variant="solo"
                        :items="appState.domains"
                        @keydown.enter="addDomain"
                        @update:modelValue="addDomainOnUpdate"
                      >
                        <template v-slot:item="{ props, item }">
                          <v-list-item
                            v-bind="props"
                            :title="item.value"
                          >
                            <template v-slot:append>
                              <v-btn
                                @click.stop="removeDomain(item.value)"
                                variant="plain"
                                icon="fa-solid fa-close"
                              />
                            </template>
                          </v-list-item>
                        </template>
                      </v-combobox>
                    </template>
                  </v-dialog>
                  <v-list-item
                    title="Log Out"
                    prepend-icon="fa-solid fa-sign-out"
                    @click="logOut()"
                  />
                </v-list>
            </v-menu>
            <v-spacer />
            <v-btn
              variant="plain"
              icon="fa-regular fa-pen-to-square"
              @click="createNewItem"
            />
          </v-toolbar>
        </template>
        <v-list>
          <v-list-item
            v-for="id in translatableItemIds"
            :active="id === selected"
            @click="router.push(`/${id}`)"
          >
            <v-list-item-title>
              <ContentReference
                :key="id"
                :id="id"
              />
            </v-list-item-title>
          </v-list-item>
        </v-list>
      </v-navigation-drawer>
      <v-app-bar
        scroll-behavior="elevate"
      >
        <template v-slot:prepend>
          <v-btn
            variant="plain"
            :icon="drawer ? 'fa-solid fa-chevron-left' : 'fa-solid fa-bars'"
            @click="drawer = !drawer"
          />
        </template>
        <template v-slot:title>
          {{ domain }}
        </template>
        <template v-slot:append>
          <v-btn
            icon="fa-solid fa-search"
            class="mr-3"
            @click="() => {
              if (selected) router.push('/')
            }"
          />
          <v-switch
            class="mr-4"
            v-model="editing"
            color="primary"
            hide-details
            label="edit"
          />
          <v-avatar
            class="mr-2"
            :image="env.auth.info.picture"
          />
        </template>
      </v-app-bar>
      <TranslatableTargets
        v-if="selected"
        :key="selected"
        :editing="editing"
        :translatableItemId="selected"
        :languages="appState.languages"
      />
      <v-container v-else>
        <v-text-field
          variant="outlined"
          autofocus
          clearable
          v-model="search"
          @keypress.enter="fetchSearchResults(search)"
        >
          <template v-slot:append-inner>
            <v-btn
              text="Search"
              @click="() => fetchSearchResults(search)"
            />
          </template>
        </v-text-field>
        <v-list>
          <v-list-item v-if="searchResults && searchResults.length === 0">
            No results for "{{ enteredSearch }}"
          </v-list-item>
          <v-list-item
            v-for="result in searchResults"
            @click="router.push(`/${result.translatable_item}`)"
          >
            <v-list-item-title>
              <ContentReference :id="result.translatable_item" />
              {{ result.path.slice(1) }}
            </v-list-item-title>
            <v-list-item-subtitle>
              <b>{{ languageCodes[result.language] }}:</b> <i>{{ result.match }}</i>
            </v-list-item-subtitle>
          </v-list-item>
        </v-list>
      </v-container>
    </v-main>
  </v-app>
  <v-app v-else>
    <v-btn
      text="Log in"
      @click="logIn"
    />
  </v-app>
</template>

<style scoped>
  .active { background-color: chartreuse; }
</style>