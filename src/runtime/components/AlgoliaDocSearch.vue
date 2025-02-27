<template>
  <div id="docsearch">
    <button type="button" class="DocSearch DocSearch-Button" aria-label="Search" />
  </div>
</template>

<script setup lang="ts">
import type { PropType } from 'vue'
import { withoutTrailingSlash } from 'ufo'
import type { DocSearchOptions } from '../../types'
// @ts-ignore - These are Nuxt3 aliases
import { useRuntimeConfig, useRoute, useRouter, onMounted, watch, computed } from '#imports'

const route = useRoute()

const router = useRouter()

const props = defineProps({
  options: {
    type: [Object, undefined] as PropType<DocSearchOptions | undefined>,
    required: false,
    default: undefined
  }
})

/**
 * Try to grab options from runtimeConfig.
 *
 * If not found, fallback on props.
 */
const options = computed<DocSearchOptions>(
  () => {
    if (props.options) { return props.options }

    const { algolia } = useRuntimeConfig()

    if (algolia && algolia.docSearch) { return algolia.docSearch }

    return {}
  }
)

/**
 * Check if event is special click to avoid closing the DocSearch too soon.
 */
const isSpecialClick = (event: MouseEvent) => event.button === 1 || event.altKey || event.ctrlKey || event.metaKey || event.shiftKey

/**
 * Gets the relative path from an absolute URL provided by the DocSearch instance.
 */
const getRelativePath = (absoluteUrl: string) => {
  const { pathname, hash } = new URL(absoluteUrl)
  const url = window.location.origin
  const relativeUrl = pathname.replace(url, '/') + hash
  return withoutTrailingSlash(relativeUrl)
}

/**
 * Initialize the DocSearch instance.
 * @param userOptions
 */
const initialize = async (userOptions: DocSearchOptions) => {
  // Import @docsearch at runtime
  const docsearch = await Promise.all([
    // @ts-ignore
    import(/* webpackChunkName: "docsearch" */ '@docsearch/js'),
    // @ts-ignore
    process.client && import(/* webpackChunkName: "docsearch" */ '@docsearch/css')
  ]).then(([docsearch]) => docsearch.default)

  // TODO: Maybe bind this with @nuxt/i18n ?
  // Resolve lang
  const lang = userOptions?.lang || 'en'
  // Generate lang prefix
  const langPrefix = `${userOptions.langAttribute || 'language'}:${lang}`
  // Get facet filters
  const userFacetFilters = userOptions.facetFilters || []

  // Create DocSearch instance
  docsearch({
    /**
     * Local implementation of this DocSearch box uses a local element with an `docsearch` id.
     */
    container: '#docsearch',
    appId: userOptions.applicationId,
    apiKey: userOptions.apiKey,
    indexName: userOptions.indexName,
    searchParameters: {
      ...(
        // Prefix facetFilters with langAttribute, otherwise use raw facetFilters
        (!lang)
          ? {
              facetFilters: userFacetFilters
            }
          : {
              facetFilters: [langPrefix].concat(userFacetFilters)
            }
      ),
      ...userOptions.searchParameters
    },
    /**
     * Transform items into relative URL format (compatibiltiy with Vue Router).
     */
    transformItems: userOptions.transformItems
      ? userOptions.transformItems
      : (items) => {
          return items.map((item) => {
            return {
              ...item,
              url: getRelativePath(item.url)
            }
          })
        },
    navigator: userOptions.navigator
      ? userOptions.navigator
      : {
          navigate: ({ itemUrl }) => {
            const { pathname: hitPathname } = new URL(window.location.origin + itemUrl)
            // Vue Router doesn't handle same-page navigation so we use
            // the native browser location API for anchor navigation.
            if (route.path === hitPathname) {
              window.location.assign(window.location.origin + itemUrl)
            } else {
              router.push(itemUrl)
            }
          }
        },
    hitComponent: userOptions.hitComponent
      ? userOptions.hitComponent
      : ({ hit, children }) => {
          return {
            type: 'a',
            constructor: undefined,
            __v: 1,
            props: {
              href: hit.url,
              children,
              onClick: (event: MouseEvent) => {
                if (isSpecialClick(event)) { return }

                // We rely on the native link scrolling when user is
                // already on the right anchor because Vue Router doesn't
                // support duplicated history entries.
                if (route.fullPath === hit.url) { return }

                const { pathname: hitPathname } = new URL(window.location.origin + hit.url)

                // If the hits goes to another page, we prevent the native link behavior
                // to leverage the Vue Router loading feature.
                if (route.path !== hitPathname) { event.preventDefault() }

                router.push(hit.url)
              }
            }
          }
        },
    // Spread user options, except the ones that are already used in the instance.
    ...Object.entries(userOptions)
      // Skip already used keys
      .filter(
        ([key]) => !['applicationId', 'apiKey', 'indexName', 'transformItems', 'navigator', 'hitComponent', 'facetFilters', 'langAttribute', 'lang'].includes(key)
      )
      // Recompose options
      .reduce(
        (acc, [key, value]) => {
          acc[key] = value
          return acc
        },
        {}
      )
  })
}

// Initialize when mounted.
onMounted(() => initialize(options.value))

// Watch options and restart the instance if needed.
watch(options, (n: any) => initialize(n))
</script>
