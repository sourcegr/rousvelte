<script context="module">
  import { writable } from 'svelte/store';

  let breadCrumbArray;


  const removeTrailingSlashes = url => {
    let found = url;
    while (found.endsWith('/') && !found.includes('#')) {
      found = found.slice(0, -1);
    }
    return found;
  };

  const makeStoreValue = function(url) {
    const urlObj = new URL(url);

    const hash = urlObj.hash || null;
    const cleanPath = urlObj.pathname.substring(1);
    const noSlashUrl = removeTrailingSlashes(cleanPath);
    if (noSlashUrl !== cleanPath) {
      window.history.replaceState(null, '', `/${noSlashUrl}`);
    }

    const parts = cleanPath.length > 0 ? cleanPath.split('/') : [];
    const urlSearchParams = new URLSearchParams(urlObj.search);
    const query = Object.fromEntries(urlSearchParams.entries());

    breadCrumbArray = [];
    return { parts, hash, query, url: urlObj.pathname };
  };

  let st = makeStoreValue(document.location.href);
  const { subscribe, set } = writable(st);

  const router = {
    url: document.location.pathname,
    subscribe,
    go:
      (where, params = {}) =>
        (event = null) => {
          params.preventDefault && event?.preventDefault();
          params.stopPropagation && event?.stopPropagation();
          go(where);
        },
    goNow: (where, params = {}) => {
      router.go(where, params)();
    },
  };


  const getANode = event => {
    const hasA = event.target.closest('a[href]:not([data-sre-ignore])');
    const key = event.ctrlKey || event.metaKey || event.altKey || event.shiftKey;
    if (!hasA || key) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();
    window.history.pushState(null, '', hasA.href);
    set(makeStoreValue(hasA.href));
  };

  window.addEventListener('click', getANode);
  window.addEventListener('popstate', e => set(makeStoreValue(window.location.href)));

  export { router };
</script>

<script>
  import { getContext, setContext } from 'svelte';

  export let match = '';
  export let exact = false;
  export let redirect = null;

  let matches = false;

  let meta = {
    wildcard: null
  };

  const checkMatch = url => {
    const parts = $router.parts;

    const levelMatch = parts[level - 1] || '';

    if (levelMatch === match) {
      if (!exact) {
        return true;
      } else if (parts.length === level || parts.length === 0) {
        matches = true;
        return true;
      }
    }

    if (match === '*' && parts.length >= level) {
      if (exact && parts.length !== level) {
        return false;
      }

      meta.wildcard = parts[level - 1];
      return true;
    }

    return false;
  };

  const getLevelFunction = () => level;

  const getLevelCallback = getContext('getLevel') || getLevelFunction;

  let level = -1;
  level = getLevelCallback() + 1;

  setContext('getLevel', getLevelFunction);


  $: {
    matches = checkMatch(document.location.pathname.substring(1), $router, match, exact);
    meta = {...meta, ...$router}
    if (matches && redirect) {
      matches = false;
      window.history.replaceState(null, '', `${redirect}`);
      setTimeout(() => {
        set(makeStoreValue(`/${redirect}`));
      });
    }
  }
</script>

{#if matches}
  <slot {meta}/>
{/if}