<svelte:options immutable={true} />

<script lang="ts">
  import type { VFile } from 'vfile'
  import type { BytemdPlugin, ViewerProps } from './types'
  import {
    tick,
    onDestroy,
    onMount,
    createEventDispatcher,
    afterUpdate,
  } from 'svelte'
  import { getProcessor } from './utils'

  const dispatch = createEventDispatcher()

  export let value: ViewerProps['value'] = ''
  export let plugins: NonNullable<ViewerProps['plugins']> = []
  export let sanitize: ViewerProps['sanitize']

  let markdownBody: HTMLElement
  let cbs: ReturnType<NonNullable<BytemdPlugin['viewerEffect']>>[] = []

  function on() {
    // console.log('von')
    cbs = plugins.map((p) => p.viewerEffect?.({ markdownBody, file }))
  }
  function off() {
    // console.log('voff')
    cbs.forEach((cb) => cb?.())
  }

  onMount(() => {
    markdownBody.addEventListener('click', (e) => {
      const $ = e.target as HTMLElement
      if ($.tagName !== 'A') return

      const href = $.getAttribute('href')
      if (!href?.startsWith('#')) return

      markdownBody
        .querySelector('#user-content-' + href.slice(1))
        ?.scrollIntoView()
    })
  })

  onDestroy(off)

  let file: VFile
  let i = 0

  $: try {
    file = getProcessor({
      sanitize,
      plugins: [
        ...plugins,
        {
          // remark: (p) =>
          //   p.use(() => (tree) => {
          //     console.log(tree)
          //   }),
          rehype: (p) =>
            p.use(() => (tree, file) => {
              tick().then(() => {
                // console.log(tree);
                dispatch('hast', { hast: tree, file })
              })
            }),
        },
      ],
    }).processSync(value)
    i++
  } catch (err) {
    console.error(err)
  }

  afterUpdate(() => {
    // TODO: `off` should be called before DOM update
    // https://github.com/sveltejs/svelte/issues/6016
    off()
    on()
  })

  $: html = `${file}<!--${i}-->`
</script>

<div bind:this={markdownBody} class="markdown-body">
  {@html html}
</div>
