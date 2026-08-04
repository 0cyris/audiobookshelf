<template>
  <div class="w-full h-full relative overflow-hidden">
    <!-- Page viewer -->
    <div ref="viewer" class="pdf-viewer absolute top-14 bottom-14 right-0 overflow-auto" :class="[sidebarOpen ? 'left-0 md:left-80' : 'left-0', { 'pdf-invert': invertColors }]" @scroll.passive="onScroll">
      <div ref="pagesContainer" class="relative mx-auto" :style="containerStyle">
        <div v-for="pageLayout in activeLayouts" :key="pageLayout.page" :data-page="pageLayout.page" class="pdf-page absolute bg-white shadow-md" :style="{ top: pageLayout.top + 'px', left: pageLayout.left + 'px', width: pageLayout.width + 'px', height: pageLayout.height + 'px' }"></div>
      </div>
    </div>

    <div v-if="loading" class="absolute inset-0 flex items-center justify-center z-10">
      <ui-loading-indicator />
    </div>
    <div v-else-if="loadError" class="absolute inset-0 flex items-center justify-center z-10 px-8">
      <p class="text-center text-lg text-error">{{ loadError }}</p>
    </div>

    <!-- Sidebar backdrop (mobile only, the sidebar is inline on md+) -->
    <div v-if="sidebarOpen" class="absolute top-14 bottom-0 left-0 right-0 bg-black/40 z-20 md:hidden" @click="sidebarOpen = false"></div>

    <!-- Outline / search / thumbnails sidebar -->
    <!-- Starts below the reader's own top bar so the title and close button stay reachable -->
    <div class="absolute top-14 bottom-0 left-0 w-80 max-w-full z-30 flex flex-col bg-bg border-r border-t border-gray-500 shadow-xl transition-transform" :class="sidebarOpen ? 'translate-x-0' : '-translate-x-full'">
      <div class="flex items-center px-2 h-14 shrink-0 border-b border-gray-700">
        <button v-for="tab in sidebarTabs" :key="tab.value" type="button" class="flex-1 h-9 text-sm rounded-md mx-0.5 transition-colors" :class="sidebarTab === tab.value ? 'bg-primary text-white' : 'text-gray-300 hover:bg-primary/50'" @click="selectTab(tab.value)">
          {{ tab.text }}
        </button>
        <button type="button" aria-label="Close sidebar" class="ml-1 inline-flex opacity-70 hover:opacity-100" @click="sidebarOpen = false">
          <span class="material-symbols text-xl">chevron_left</span>
        </button>
      </div>

      <!-- Outline -->
      <div v-show="sidebarTab === 'outline'" class="flex-1 overflow-y-auto px-2 py-2">
        <p v-if="!outlineItems.length" class="text-sm text-gray-300 text-center py-4">{{ $strings.MessagePdfNoOutline }}</p>
        <ul v-else>
          <li v-for="item in visibleOutlineItems" :key="item.id">
            <div class="flex items-center rounded-md relative" :class="{ 'bg-yellow-400/20': item.id === activeOutlineId }" :style="{ paddingLeft: item.depth * 12 + 'px' }">
              <div v-if="item.id === activeOutlineId" class="w-0.5 h-full absolute top-0 left-0 bg-yellow-400" />
              <button v-if="item.children.length" type="button" aria-label="Expand outline section" class="shrink-0 inline-flex opacity-70 hover:opacity-100" @click="toggleOutlineItem(item)">
                <span class="material-symbols text-lg">{{ item.expanded ? 'expand_more' : 'chevron_right' }}</span>
              </button>
              <span v-else class="w-5 shrink-0" />
              <button type="button" class="flex-1 min-w-0 text-left text-sm py-1 pr-1 opacity-90 hover:opacity-100" @click="clickOutlineItem(item)">
                <span class="line-clamp-2">{{ item.title }}</span>
              </button>
              <span v-if="item.page" class="shrink-0 text-xs font-mono text-gray-400 pl-1">{{ item.page }}</span>
            </div>
          </li>
        </ul>
      </div>

      <!-- Search -->
      <div v-show="sidebarTab === 'search'" class="flex-1 flex flex-col min-h-0 px-2 py-2">
        <form @submit.prevent="runSearch">
          <ui-text-input ref="searchInput" v-model="searchQuery" clearable :placeholder="$strings.PlaceholderSearch" class="h-8 w-full text-sm flex" @clear="clearSearch" />
        </form>
        <div class="flex items-center h-8 shrink-0 text-xs text-gray-300">
          <p v-if="isSearching" class="flex-1 truncate">{{ $getString('MessagePdfSearching', [searchProgress, numPages]) }}</p>
          <p v-else-if="searchRan" class="flex-1 truncate">{{ $getString('MessagePdfSearchResults', [searchResults.length]) }}</p>
          <p v-else class="flex-1" />
          <template v-if="searchResults.length">
            <ui-icon-btn icon="keyboard_arrow_up" :size="7" borderless :aria-label="$strings.LabelPdfPreviousMatch" @click="stepMatch(-1)" />
            <ui-icon-btn icon="keyboard_arrow_down" :size="7" borderless :aria-label="$strings.LabelPdfNextMatch" @click="stepMatch(1)" />
          </template>
        </div>
        <div class="flex-1 overflow-y-auto">
          <button v-for="(result, index) in searchResults" :key="index" type="button" class="w-full text-left px-2 py-1.5 rounded-md relative" :class="index === activeMatchIndex ? 'bg-yellow-400/20' : 'hover:bg-primary/50'" @click="goToMatch(index)">
            <div v-if="index === activeMatchIndex" class="w-0.5 h-full absolute top-0 left-0 bg-yellow-400" />
            <p class="text-xs font-mono text-gray-400">{{ $getString('LabelPdfPageNumber', [result.page]) }}</p>
            <p class="text-sm text-gray-200">{{ result.excerpt }}</p>
          </button>
        </div>
      </div>

      <!-- Thumbnails -->
      <div v-show="sidebarTab === 'thumbnails'" ref="thumbnailPanel" class="flex-1 overflow-y-auto px-2 py-2">
        <div v-for="pageNum in numPages" :key="pageNum" :data-thumb-page="pageNum" class="pdf-thumb mx-auto mb-3 cursor-pointer rounded-xs" :class="{ 'ring-2 ring-yellow-400': pageNum === page }" :style="{ width: thumbnailWidth + 'px' }" @click="clickThumbnail(pageNum)">
          <div class="bg-white/10" :style="{ height: thumbnailHeight(pageNum) + 'px' }" />
          <p class="text-xs font-mono text-center text-gray-400">{{ pageNum }}</p>
        </div>
      </div>
    </div>

    <!-- Bottom toolbar -->
    <div class="absolute bottom-2 left-1/2 -translate-x-1/2 z-20 flex items-center px-1.5 h-11 rounded-md bg-bg/90 border border-gray-400 shadow-lg">
      <ui-icon-btn :icon="sidebarOpen ? 'menu_open' : 'menu'" :size="8" borderless :aria-label="$strings.HeaderTableOfContents" @click="sidebarOpen = !sidebarOpen" />
      <ui-icon-btn icon="search" :size="8" borderless :aria-label="$strings.PlaceholderSearch" @click="openSearch" />

      <div class="w-px h-6 bg-gray-500 mx-1" />

      <ui-icon-btn icon="keyboard_arrow_up" :size="8" borderless :disabled="!canGoPrev" :aria-label="$strings.LabelPdfPreviousPage" @click="prev" />
      <div class="flex items-center text-sm font-mono">
        <input v-model="pageInput" type="text" inputmode="numeric" class="w-10 h-7 text-center rounded-sm bg-primary text-gray-200 border border-gray-600 focus:bg-bg focus:outline-hidden" :aria-label="$strings.LabelPdfPage" @keydown.enter="commitPageInput" @blur="commitPageInput" @focus="$event.target.select()" />
        <span class="px-1 text-gray-400">/ {{ numPages || '–' }}</span>
      </div>
      <ui-icon-btn icon="keyboard_arrow_down" :size="8" borderless :disabled="!canGoNext" :aria-label="$strings.LabelPdfNextPage" @click="next" />

      <div class="w-px h-6 bg-gray-500 mx-1" />

      <ui-icon-btn icon="zoom_out" :size="8" borderless :disabled="!canScaleDown" :aria-label="$strings.LabelPdfZoomOut" @click="zoomOut" />
      <button type="button" class="hidden sm:block w-12 text-xs font-mono text-gray-300 hover:text-white" :aria-label="$strings.LabelPdfZoom" @click="resetZoom">{{ Math.round(scale * 100) }}%</button>
      <ui-icon-btn icon="zoom_in" :size="8" borderless :disabled="!canScaleUp" :aria-label="$strings.LabelPdfZoomIn" @click="zoomIn" />
      <ui-icon-btn icon="fit_width" :size="8" borderless :aria-label="$strings.LabelPdfFitWidth" :class="{ 'text-yellow-400': fitMode === 'width' }" @click="setFitMode('width')" />
      <ui-icon-btn icon="fit_screen" :size="8" borderless :aria-label="$strings.LabelPdfFitPage" :class="{ 'text-yellow-400': fitMode === 'page' }" @click="setFitMode('page')" />
      <ui-icon-btn icon="rotate_right" :size="8" borderless class="hidden sm:flex" :aria-label="$strings.LabelPdfRotate" @click="rotateClockwise" />
    </div>
  </div>
</template>

<script>
import * as pdfjsLib from 'pdfjs-dist/legacy/build/pdf'

const PAGE_GAP = 12
const RENDER_BUFFER_ROWS = 1
const ZOOM_LEVELS = [0.25, 0.5, 0.75, 1, 1.25, 1.5, 2, 3, 4, 6]
const THUMBNAIL_WIDTH = 130
const SEARCH_PAGES_PER_TICK = 8
const EXCERPT_PADDING = 45

const LIGATURES = {
  ﬀ: 'ff',
  ﬁ: 'fi',
  ﬂ: 'fl',
  ﬃ: 'ffi',
  ﬄ: 'ffl',
  ﬅ: 'st',
  ﬆ: 'st'
}

/**
 * Lowercase, drop soft hyphens, expand ligatures and collapse whitespace runs.
 * Returns the normalized text plus a map from each normalized character index
 * back to its index in the raw text, so excerpts can be cut from the raw text.
 *
 * @param {string} raw
 * @returns {{ text: string, map: number[] }}
 */
function normalizeText(raw) {
  let text = ''
  const map = []
  let prevWasSpace = false
  for (let i = 0; i < raw.length; i++) {
    const char = raw[i]
    if (char === '­') continue // soft hyphen
    if (/\s/.test(char)) {
      if (prevWasSpace) continue
      prevWasSpace = true
      text += ' '
      map.push(i)
      continue
    }
    prevWasSpace = false
    const replacement = LIGATURES[char] || char.toLowerCase()
    for (let c = 0; c < replacement.length; c++) {
      text += replacement[c]
      map.push(i)
    }
  }
  map.push(raw.length)
  return { text, map }
}

function escapeRegExp(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}

function escapeHtml(str) {
  return str.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
}

export default {
  props: {
    libraryItem: {
      type: Object,
      default: () => {}
    },
    playerOpen: Boolean,
    keepProgress: Boolean,
    fileId: String
  },
  data() {
    return {
      loading: true,
      loadError: null,
      isRefreshing: false,
      numPages: 0,
      page: 1,
      pageInput: '1',
      // Intrinsic page dimensions at scale 1: { width, height, rotate }
      pageSizes: [],
      scale: 1,
      fitMode: 'width',
      rotation: 0,
      renderMode: 'scroll',
      invertColors: false,
      layoutWidth: 0,
      layoutHeight: 0,
      // [{ page, top, left, width, height }] for every page in the document
      pageLayouts: [],
      // [{ pages: [pageNum], top, height, width }]
      rows: [],
      containerWidth: 0,
      containerHeight: 0,
      sidebarOpen: false,
      sidebarTab: 'outline',
      outlineItems: [],
      searchQuery: '',
      searchResults: [],
      searchProgress: 0,
      isSearching: false,
      searchRan: false,
      activeMatchIndex: -1,
      thumbnailWidth: THUMBNAIL_WIDTH
    }
  },
  computed: {
    userToken() {
      return this.$store.getters['user/getToken']
    },
    libraryItemId() {
      return this.libraryItem?.id
    },
    ebookUrl() {
      if (this.fileId) {
        return `/api/items/${this.libraryItemId}/ebook/${this.fileId}`
      }
      return `/api/items/${this.libraryItemId}/ebook`
    },
    userMediaProgress() {
      if (!this.libraryItemId) return null
      return this.$store.getters['user/getUserMediaProgress'](this.libraryItemId)
    },
    savedPage() {
      if (!this.keepProgress) return 0
      // Validate ebookLocation is a number
      if (!this.userMediaProgress?.ebookLocation || isNaN(this.userMediaProgress.ebookLocation)) return 0
      return Number(this.userMediaProgress.ebookLocation)
    },
    sidebarTabs() {
      return [
        { text: this.$strings.HeaderPdfOutline, value: 'outline' },
        { text: this.$strings.PlaceholderSearch, value: 'search' },
        { text: this.$strings.HeaderPdfThumbnails, value: 'thumbnails' }
      ]
    },
    /** Only the pages that currently have a layout box mounted in the DOM */
    activeLayouts() {
      if (this.renderMode === 'single') {
        // Single page mode shows one page pinned to the top-left of the container
        const layout = this.pageLayouts.find((l) => l.page === this.page)
        if (!layout) return []
        return [{ page: layout.page, top: 0, left: 0, width: layout.width, height: layout.height }]
      }
      return this.pageLayouts
    },
    containerStyle() {
      if (this.renderMode === 'single') {
        const layout = this.activeLayouts[0]
        return { width: `${layout?.width || 0}px`, height: `${layout?.height || 0}px` }
      }
      return { width: `${this.layoutWidth}px`, height: `${this.layoutHeight}px` }
    },
    visibleOutlineItems() {
      return this.outlineItems.filter((item) => item.visible)
    },
    activeOutlineId() {
      let activeId = null
      for (const item of this.outlineItems) {
        if (item.page && item.page <= this.page) activeId = item.id
      }
      return activeId
    },
    canGoNext() {
      return this.page < this.numPages
    },
    canGoPrev() {
      return this.page > 1
    },
    canScaleUp() {
      return this.scale < ZOOM_LEVELS[ZOOM_LEVELS.length - 1]
    },
    canScaleDown() {
      return this.scale > ZOOM_LEVELS[0]
    }
  },
  watch: {
    page(newVal) {
      this.pageInput = String(newVal)
      this.updateProgress()
    },
    renderMode() {
      this.relayout({ preservePage: true })
    },
    rotation() {
      this.thumbnailCache = {}
      this.thumbQueue = []
      if (this.thumbObserver) {
        this.thumbObserver.disconnect()
        this.thumbObserver = null
      }
      if (this.sidebarTab === 'thumbnails') this.$nextTick(() => this.observeThumbnails())
      this.relayout({ preservePage: true })
    }
  },
  methods: {
    //
    // Document loading
    //
    async init() {
      this.loading = true
      this.loadError = null
      pdfjsLib.GlobalWorkerOptions.workerSrc = `${this.$config.routerBasePath}/pdfjs/pdf.worker.min.js`
      await this.loadDocument(this.userToken)
    },
    async loadDocument(token) {
      await this.destroyDocument()
      try {
        this.loadingTask = pdfjsLib.getDocument({
          url: this.ebookUrl,
          httpHeaders: {
            Authorization: `Bearer ${token}`
          },
          // Stream only what is needed - rulebooks can be hundreds of MB
          disableAutoFetch: true,
          rangeChunkSize: 65536
        })
        this.pdfDoc = await this.loadingTask.promise
      } catch (error) {
        this.loadingTask = null
        if (error?.status === 401 && !this.isRefreshing) {
          console.log('Received 401 error, refreshing token')
          return this.refreshToken()
        }
        console.error('PdfReader failed to load document', error)
        this.loadError = error?.message || 'Failed to load PDF'
        this.loading = false
        return
      }
      if (this.destroyed) return

      this.numPages = this.pdfDoc.numPages
      await this.loadPageSizes()
      if (this.destroyed) return
      this.loading = false

      this.measureContainer()
      const startPage = this.savedPage > 0 && this.savedPage <= this.numPages ? this.savedPage : 1
      this.page = startPage
      this.pageInput = String(startPage)
      this.relayout({ page: startPage })

      this.loadOutline()
    },
    /**
     * Page dimensions are needed up front so the scroll height is correct from
     * the start. Rulebooks mix portrait text pages with landscape maps, so the
     * sizes cannot be assumed uniform.
     */
    async loadPageSizes() {
      const sizes = new Array(this.numPages)
      const chunkSize = 20
      for (let start = 0; start < this.numPages; start += chunkSize) {
        const pageNums = []
        for (let pageNum = start + 1; pageNum <= Math.min(start + chunkSize, this.numPages); pageNum++) {
          pageNums.push(pageNum)
        }
        await Promise.all(
          pageNums.map(async (pageNum) => {
            const pdfPage = await this.pdfDoc.getPage(pageNum)
            const viewport = pdfPage.getViewport({ scale: 1 })
            sizes[pageNum - 1] = { width: viewport.width, height: viewport.height, rotate: pdfPage.rotate }
          })
        )
        if (this.destroyed || !this.pdfDoc) return
      }
      this.pageSizes = sizes
    },
    async refreshToken() {
      if (this.isRefreshing) return
      this.isRefreshing = true
      const newAccessToken = await this.$store.dispatch('user/refreshToken').catch((error) => {
        console.error('Failed to refresh token', error)
        return null
      })
      if (!newAccessToken) {
        // Redirect to login on failed refresh
        this.$router.push('/login')
        return
      }
      await this.loadDocument(newAccessToken)
      this.isRefreshing = false
    },
    async destroyDocument() {
      this.teardownAllPageViews()
      this.thumbnailCache = {}
      this.pageTextCache = {}
      this.thumbQueue = []
      if (this.pdfDoc) {
        const doc = this.pdfDoc
        this.pdfDoc = null
        await doc.destroy().catch(() => {})
      }
      this.loadingTask = null
    },

    //
    // Layout
    //
    measureContainer() {
      const viewer = this.$refs.viewer
      if (!viewer) return
      this.containerWidth = viewer.clientWidth
      this.containerHeight = viewer.clientHeight
    },
    /** Intrinsic size of a page at scale 1, accounting for user rotation */
    sizeForPage(pageNum) {
      const size = this.pageSizes[pageNum - 1]
      if (!size) return { width: 612, height: 792 }
      if (this.rotation % 180 === 90) return { width: size.height, height: size.width }
      return { width: size.width, height: size.height }
    },
    pagesPerRow() {
      return this.renderMode === 'spread' ? 2 : 1
    },
    /** Scale implied by the current fit mode, based on the largest page */
    computeFitScale() {
      if (!this.pageSizes.length || !this.containerWidth) return this.scale
      let maxWidth = 0
      let maxHeight = 0
      for (let pageNum = 1; pageNum <= this.numPages; pageNum++) {
        const size = this.sizeForPage(pageNum)
        if (size.width > maxWidth) maxWidth = size.width
        if (size.height > maxHeight) maxHeight = size.height
      }
      const perRow = this.pagesPerRow()
      const availableWidth = this.containerWidth - PAGE_GAP * (perRow + 1)
      const widthScale = availableWidth / (maxWidth * perRow)
      if (this.fitMode === 'page') {
        const heightScale = (this.containerHeight - PAGE_GAP * 2) / maxHeight
        return Math.max(0.1, Math.min(widthScale, heightScale))
      }
      return Math.max(0.1, widthScale)
    },
    /**
     * Rebuild the row/page geometry. Every page gets a positioned box sized from
     * pageSizes so the scroll height is right from the start, but only pages
     * inside the render window get a canvas.
     */
    relayout({ page, preservePage } = {}) {
      if (!this.pageSizes.length) return

      const targetPage = page || (preservePage ? this.page : null)
      if (this.fitMode !== 'custom') {
        this.scale = this.computeFitScale()
      }

      const perRow = this.pagesPerRow()
      const rows = []
      const layouts = []
      let top = PAGE_GAP
      let widest = 0

      let pageNum = 1
      while (pageNum <= this.numPages) {
        // Classic book pairing: page 1 stands alone, then 2-3, 4-5, ...
        const rowPages = []
        if (perRow === 2 && pageNum === 1) {
          rowPages.push(1)
        } else {
          for (let i = 0; i < perRow && pageNum + i <= this.numPages; i++) {
            rowPages.push(pageNum + i)
          }
        }
        pageNum += rowPages.length

        let rowWidth = 0
        let rowHeight = 0
        for (let i = 0; i < rowPages.length; i++) {
          const size = this.sizeForPage(rowPages[i])
          const width = Math.floor(size.width * this.scale)
          const height = Math.floor(size.height * this.scale)
          rowWidth += width + (i > 0 ? PAGE_GAP : 0)
          if (height > rowHeight) rowHeight = height
        }
        if (rowWidth > widest) widest = rowWidth
        rows.push({ pages: rowPages, top, height: rowHeight, width: rowWidth })
        top += rowHeight + PAGE_GAP
      }

      // Center every row within the same content width so pages do not shift
      // horizontally while scrolling past a differently sized page
      const contentWidth = Math.max(widest, this.containerWidth - PAGE_GAP * 2)
      for (const row of rows) {
        let left = Math.floor((contentWidth - row.width) / 2)
        for (const num of row.pages) {
          const size = this.sizeForPage(num)
          const width = Math.floor(size.width * this.scale)
          const height = Math.floor(size.height * this.scale)
          layouts.push({ page: num, top: row.top, left, width, height })
          left += width + PAGE_GAP
        }
      }

      this.rows = rows
      this.pageLayouts = layouts
      this.layoutWidth = contentWidth
      this.layoutHeight = top

      // Page boxes moved or resized, so every rendered layer is stale
      this.teardownAllPageViews()

      this.$nextTick(() => {
        if (this.renderMode === 'single') {
          if (this.$refs.viewer) this.$refs.viewer.scrollTop = 0
        } else if (targetPage) {
          this.scrollToPage(targetPage)
        }
        this.renderVisiblePages()
      })
    },
    pageElement(pageNum) {
      if (!this.$refs.pagesContainer) return null
      return this.$refs.pagesContainer.querySelector(`.pdf-page[data-page="${pageNum}"]`)
    },

    //
    // Scrolling / navigation
    //
    onScroll() {
      if (this.scrollRaf) return
      this.scrollRaf = requestAnimationFrame(() => {
        this.scrollRaf = null
        this.updateCurrentPageFromScroll()
        this.renderVisiblePages()
      })
    },
    updateCurrentPageFromScroll() {
      if (this.renderMode === 'single' || !this.rows.length) return
      const viewer = this.$refs.viewer
      if (!viewer) return
      const midpoint = viewer.scrollTop + viewer.clientHeight / 2
      let current = this.rows[0]
      for (const row of this.rows) {
        if (row.top <= midpoint) current = row
        else break
      }
      const pageNum = current.pages[0]
      if (pageNum !== this.page) this.page = pageNum
    },
    goToPage(pageNum, yFromPageBottom = null) {
      const clamped = Math.max(1, Math.min(this.numPages, Math.round(Number(pageNum) || 1)))
      this.page = clamped
      if (this.renderMode === 'single') {
        this.$nextTick(() => {
          if (this.$refs.viewer) this.$refs.viewer.scrollTop = 0
          this.renderVisiblePages()
        })
        return
      }
      this.scrollToPage(clamped, yFromPageBottom)
      this.renderVisiblePages()
    },
    scrollToPage(pageNum, yFromPageBottom = null) {
      const viewer = this.$refs.viewer
      const layout = this.pageLayouts.find((l) => l.page === pageNum)
      if (!viewer || !layout) return
      let offset = layout.top - PAGE_GAP
      if (yFromPageBottom != null) {
        const size = this.sizeForPage(pageNum)
        // PDF destinations measure y from the bottom of the page
        offset += Math.max(0, (size.height - yFromPageBottom) * this.scale - PAGE_GAP)
      }
      viewer.scrollTop = Math.max(0, offset)
    },
    next() {
      if (this.page >= this.numPages) return
      this.goToPage(this.page + this.pagesPerRow())
    },
    prev() {
      if (this.page <= 1) return
      this.goToPage(this.page - this.pagesPerRow())
    },
    firstPage() {
      this.goToPage(1)
    },
    lastPage() {
      this.goToPage(this.numPages)
    },
    pageDown() {
      this.scrollByViewport(1)
    },
    pageUp() {
      this.scrollByViewport(-1)
    },
    scrollDown() {
      this.scrollBy(120)
    },
    scrollUp() {
      this.scrollBy(-120)
    },
    scrollByViewport(direction) {
      if (this.renderMode === 'single') {
        if (direction > 0) this.next()
        else this.prev()
        return
      }
      const viewer = this.$refs.viewer
      if (!viewer) return
      this.scrollBy(direction * (viewer.clientHeight - 40))
    },
    scrollBy(amount) {
      const viewer = this.$refs.viewer
      if (!viewer || this.renderMode === 'single') return
      viewer.scrollTop += amount
    },
    commitPageInput() {
      const parsed = parseInt(this.pageInput, 10)
      if (isNaN(parsed)) {
        this.pageInput = String(this.page)
        return
      }
      const clamped = Math.max(1, Math.min(this.numPages, parsed))
      this.pageInput = String(clamped)
      this.goToPage(clamped)
    },

    //
    // Zoom / rotation / display mode
    //
    setScale(scale) {
      this.fitMode = 'custom'
      this.scale = scale
      this.relayout({ preservePage: true })
    },
    zoomIn() {
      const next = ZOOM_LEVELS.find((level) => level > this.scale + 0.001)
      if (next) this.setScale(next)
    },
    zoomOut() {
      const levels = ZOOM_LEVELS.filter((level) => level < this.scale - 0.001)
      if (levels.length) this.setScale(levels[levels.length - 1])
    },
    resetZoom() {
      this.setScale(1)
    },
    setFitMode(fitMode) {
      this.fitMode = fitMode
      this.relayout({ preservePage: true })
    },
    rotateClockwise() {
      this.rotation = (this.rotation + 90) % 360
    },
    updateSettings(settings) {
      if (!settings) return
      this.invertColors = !!settings.pdfInvert
      const renderMode = settings.pdfRenderMode || 'scroll'
      const zoomMode = settings.pdfZoomMode || 'width'
      // renderMode is watched, so only touch the fit mode when it did not change
      if (renderMode !== this.renderMode) {
        this.fitMode = zoomMode === 'actual' ? 'custom' : zoomMode
        if (zoomMode === 'actual') this.scale = 1
        this.renderMode = renderMode
      } else if (zoomMode === 'actual') {
        if (this.fitMode !== 'custom') this.setScale(1)
      } else if (zoomMode !== this.fitMode) {
        this.setFitMode(zoomMode)
      }
    },

    //
    // Page rendering
    //
    /** Page numbers that should have a canvas right now */
    visiblePageNumbers() {
      if (this.renderMode === 'single') {
        return [this.page]
      }
      const viewer = this.$refs.viewer
      if (!viewer || !this.rows.length) return []
      const top = viewer.scrollTop
      const bottom = top + viewer.clientHeight
      let firstIndex = -1
      let lastIndex = -1
      for (let i = 0; i < this.rows.length; i++) {
        const row = this.rows[i]
        if (row.top + row.height < top) continue
        if (row.top > bottom) break
        if (firstIndex === -1) firstIndex = i
        lastIndex = i
      }
      if (firstIndex === -1) {
        firstIndex = 0
        lastIndex = 0
      }
      const from = Math.max(0, firstIndex - RENDER_BUFFER_ROWS)
      const to = Math.min(this.rows.length - 1, lastIndex + RENDER_BUFFER_ROWS)
      const pageNums = []
      for (let i = from; i <= to; i++) {
        pageNums.push(...this.rows[i].pages)
      }
      return pageNums
    },
    renderVisiblePages() {
      if (!this.pdfDoc || this.loading) return
      const wanted = this.visiblePageNumbers()
      const wantedSet = new Set(wanted)

      // Release everything outside the window - canvases are the memory hog
      for (const key of Object.keys(this.pageViews)) {
        if (!wantedSet.has(Number(key))) this.teardownPageView(Number(key))
      }
      for (const pageNum of wanted) {
        const view = this.pageViews[pageNum]
        if (view && view.scale === this.scale && view.rotation === this.rotation) continue
        this.renderPage(pageNum)
      }
    },
    async renderPage(pageNum) {
      const el = this.pageElement(pageNum)
      if (!el || !this.pdfDoc) return
      this.teardownPageView(pageNum)

      const scale = this.scale
      const rotation = this.rotation
      const view = { scale, rotation, canvas: null, renderTask: null, textLayerTask: null, textDivs: [] }
      this.pageViews[pageNum] = view

      let pdfPage
      try {
        pdfPage = await this.pdfDoc.getPage(pageNum)
      } catch (error) {
        console.error('PdfReader failed to get page', pageNum, error)
        return
      }
      if (this.pageViews[pageNum] !== view || this.destroyed) return

      const viewport = pdfPage.getViewport({ scale, rotation: (pdfPage.rotate + rotation) % 360 })
      // pdf.js positions text spans relative to this variable
      el.style.setProperty('--scale-factor', String(scale))

      const dpr = Math.min(window.devicePixelRatio || 1, 2)
      const canvas = document.createElement('canvas')
      canvas.className = 'pdf-canvas'
      canvas.width = Math.floor(viewport.width * dpr)
      canvas.height = Math.floor(viewport.height * dpr)
      view.canvas = canvas
      el.appendChild(canvas)

      view.renderTask = pdfPage.render({
        canvasContext: canvas.getContext('2d', { alpha: false }),
        viewport,
        transform: dpr !== 1 ? [dpr, 0, 0, dpr, 0, 0] : null
      })
      try {
        await view.renderTask.promise
      } catch (error) {
        if (error?.name !== 'RenderingCancelledException') console.error('PdfReader render failed', pageNum, error)
        return
      }
      if (this.pageViews[pageNum] !== view || this.destroyed) return

      this.buildTextLayer(pdfPage, viewport, el, view, pageNum)
      this.buildLinkLayer(pdfPage, viewport, el, view, pageNum)
    },
    async buildTextLayer(pdfPage, viewport, el, view, pageNum) {
      let textContent
      try {
        textContent = await pdfPage.getTextContent()
      } catch (error) {
        return
      }
      if (this.pageViews[pageNum] !== view || this.destroyed) return

      const textLayerDiv = document.createElement('div')
      textLayerDiv.className = 'textLayer'
      el.appendChild(textLayerDiv)
      view.textLayerDiv = textLayerDiv

      view.textLayerTask = pdfjsLib.renderTextLayer({
        textContentSource: textContent,
        container: textLayerDiv,
        viewport,
        textDivs: view.textDivs
      })
      try {
        await view.textLayerTask.promise
      } catch (error) {
        return
      }
      if (this.pageViews[pageNum] !== view) return
      this.highlightMatchesOnPage(pageNum)
    },
    async buildLinkLayer(pdfPage, viewport, el, view, pageNum) {
      let annotations
      try {
        annotations = await pdfPage.getAnnotations({ intent: 'display' })
      } catch (error) {
        return
      }
      if (this.pageViews[pageNum] !== view || this.destroyed) return

      const links = annotations.filter((annotation) => annotation.subtype === 'Link' && (annotation.url || annotation.dest))
      if (!links.length) return

      const linkLayerDiv = document.createElement('div')
      linkLayerDiv.className = 'pdf-link-layer'
      linkLayerDiv.style.width = `${viewport.width}px`
      linkLayerDiv.style.height = `${viewport.height}px`

      for (const link of links) {
        const rect = viewport.convertToViewportRectangle(link.rect)
        const anchor = document.createElement('a')
        anchor.style.left = `${Math.min(rect[0], rect[2])}px`
        anchor.style.top = `${Math.min(rect[1], rect[3])}px`
        anchor.style.width = `${Math.abs(rect[2] - rect[0])}px`
        anchor.style.height = `${Math.abs(rect[3] - rect[1])}px`
        if (link.url) {
          anchor.href = link.url
          anchor.target = '_blank'
          anchor.rel = 'noopener noreferrer'
        } else {
          anchor.href = '#'
          anchor.addEventListener('click', (e) => {
            e.preventDefault()
            this.goToDest(link.dest)
          })
        }
        linkLayerDiv.appendChild(anchor)
      }
      el.appendChild(linkLayerDiv)
      view.linkLayerDiv = linkLayerDiv
    },
    teardownPageView(pageNum) {
      const view = this.pageViews[pageNum]
      if (!view) return
      delete this.pageViews[pageNum]
      if (view.renderTask) {
        try {
          view.renderTask.cancel()
        } catch (error) {}
      }
      if (view.textLayerTask) {
        try {
          view.textLayerTask.cancel()
        } catch (error) {}
      }
      if (view.canvas) {
        // Zeroing the backing store is what actually frees the memory
        view.canvas.width = 0
        view.canvas.height = 0
        view.canvas.remove()
        view.canvas = null
      }
      if (view.textLayerDiv) view.textLayerDiv.remove()
      if (view.linkLayerDiv) view.linkLayerDiv.remove()
    },
    teardownAllPageViews() {
      for (const key of Object.keys(this.pageViews || {})) {
        this.teardownPageView(Number(key))
      }
      this.pageViews = {}
    },

    //
    // Outline
    //
    async loadOutline() {
      let outline
      try {
        outline = await this.pdfDoc.getOutline()
      } catch (error) {
        console.error('PdfReader failed to load outline', error)
        return
      }
      if (!outline?.length || this.destroyed) return

      const items = []
      let nextId = 0
      const walk = (nodes, depth, parentExpanded) => {
        return nodes.map((node) => {
          const item = {
            id: nextId++,
            title: (node.title || '').replace(/\s+/g, ' ').trim() || '—',
            depth,
            dest: node.dest,
            url: node.url,
            page: null,
            y: null,
            expanded: depth === 0,
            visible: parentExpanded,
            children: []
          }
          items.push(item)
          item.children = walk(node.items || [], depth + 1, parentExpanded && item.expanded)
          return item
        })
      }
      walk(outline, 0, true)
      this.outlineItems = items
      this.resolveOutlineDestinations()
    },
    /**
     * Resolving a destination is one or two async pdf.js calls per entry, and a
     * rulebook outline can have hundreds. Fill page numbers in progressively so
     * the panel is usable immediately.
     */
    async resolveOutlineDestinations() {
      const items = this.outlineItems
      let cursor = 0
      const worker = async () => {
        while (cursor < items.length && !this.destroyed && this.pdfDoc) {
          const item = items[cursor++]
          const resolved = await this.resolveDest(item.dest)
          if (resolved) {
            item.page = resolved.page
            item.y = resolved.y
          }
        }
      }
      await Promise.all(new Array(8).fill(null).map(worker))
    },
    async resolveDest(dest) {
      if (!dest || !this.pdfDoc) return null
      try {
        const explicit = typeof dest === 'string' ? await this.pdfDoc.getDestination(dest) : dest
        if (!Array.isArray(explicit) || !explicit.length) return null
        const ref = explicit[0]
        const pageIndex = typeof ref === 'number' ? ref : await this.pdfDoc.getPageIndex(ref)
        // Explicit destinations are [ref, { name }, ...args]; XYZ and FitH carry a y
        const destName = explicit[1]?.name
        let y = null
        if (destName === 'XYZ') y = explicit[3]
        else if (destName === 'FitH' || destName === 'FitBH') y = explicit[2]
        return { page: pageIndex + 1, y: typeof y === 'number' ? y : null }
      } catch (error) {
        return null
      }
    },
    async goToDest(dest) {
      const resolved = await this.resolveDest(dest)
      if (resolved) this.goToPage(resolved.page, resolved.y)
    },
    toggleOutlineItem(item) {
      item.expanded = !item.expanded
      const setVisible = (children, visible) => {
        for (const child of children) {
          child.visible = visible
          setVisible(child.children, visible && child.expanded)
        }
      }
      setVisible(item.children, item.expanded)
    },
    clickOutlineItem(item) {
      if (item.url) {
        window.open(item.url, '_blank', 'noopener,noreferrer')
        return
      }
      if (item.page) this.goToPage(item.page, item.y)
      else this.goToDest(item.dest)
      this.closeSidebarOnMobile()
    },
    clickThumbnail(pageNum) {
      this.goToPage(pageNum)
      this.closeSidebarOnMobile()
    },
    closeSidebarOnMobile() {
      if (window.innerWidth < 768) this.sidebarOpen = false
    },

    //
    // Sidebar
    //
    selectTab(tab) {
      this.sidebarTab = tab
      if (tab === 'thumbnails') this.$nextTick(() => this.observeThumbnails())
      else if (tab === 'search') this.$nextTick(() => this.$refs.searchInput?.setFocus?.())
    },
    openSearch() {
      this.sidebarOpen = true
      this.selectTab('search')
    },

    //
    // Search
    //
    clearSearch() {
      this.searchToken++
      this.searchQuery = ''
      this.searchResults = []
      this.searchRan = false
      this.isSearching = false
      this.activeMatchIndex = -1
      this.refreshHighlights()
    },
    async getPageText(pageNum) {
      if (this.pageTextCache[pageNum]) return this.pageTextCache[pageNum]
      const pdfPage = await this.pdfDoc.getPage(pageNum)
      const textContent = await pdfPage.getTextContent()
      let raw = ''
      for (const item of textContent.items) {
        if (item.str === undefined) continue
        raw += item.str
        if (item.hasEOL) raw += '\n'
      }
      const normalized = normalizeText(raw)
      const entry = { raw, text: normalized.text, map: normalized.map }
      this.pageTextCache[pageNum] = entry
      return entry
    },
    async runSearch() {
      const query = this.searchQuery.trim()
      const token = ++this.searchToken
      this.activeMatchIndex = -1
      this.searchResults = []
      if (query.length < 2 || !this.pdfDoc) {
        this.isSearching = false
        this.searchRan = false
        this.refreshHighlights()
        return
      }
      const needle = normalizeText(query).text
      this.isSearching = true
      this.searchRan = true
      this.searchProgress = 0

      const results = []
      for (let pageNum = 1; pageNum <= this.numPages; pageNum++) {
        if (token !== this.searchToken || this.destroyed || !this.pdfDoc) return
        let entry
        try {
          entry = await this.getPageText(pageNum)
        } catch (error) {
          continue
        }
        let index = entry.text.indexOf(needle)
        while (index !== -1) {
          const rawStart = entry.map[index]
          const rawEnd = entry.map[Math.min(index + needle.length, entry.map.length - 1)]
          const excerpt = entry.raw
            .slice(Math.max(0, rawStart - EXCERPT_PADDING), rawEnd + EXCERPT_PADDING)
            .replace(/\s+/g, ' ')
            .trim()
          results.push({ page: pageNum, excerpt })
          index = entry.text.indexOf(needle, index + needle.length)
        }
        this.searchProgress = pageNum
        if (pageNum % SEARCH_PAGES_PER_TICK === 0) {
          this.searchResults = results.slice()
          // Yield so the panel stays responsive on a 500 page book
          await new Promise((resolve) => setTimeout(resolve))
        }
      }
      if (token !== this.searchToken) return
      this.searchResults = results
      this.isSearching = false
      this.refreshHighlights()
    },
    goToMatch(index) {
      if (index < 0 || index >= this.searchResults.length) return
      this.activeMatchIndex = index
      this.goToPage(this.searchResults[index].page)
      this.$nextTick(() => this.refreshHighlights())
    },
    stepMatch(direction) {
      if (!this.searchResults.length) return
      let index = this.activeMatchIndex + direction
      if (index < 0) index = this.searchResults.length - 1
      if (index >= this.searchResults.length) index = 0
      this.goToMatch(index)
    },
    /** Regex that tolerates the whitespace and soft-hyphen noise inside text spans */
    searchRegex() {
      const query = this.searchQuery.trim()
      if (query.length < 2 || !this.searchRan) return null
      const pattern = escapeRegExp(query).replace(/\s+/g, '[\\s\\u00ad]+')
      return new RegExp(pattern, 'gi')
    },
    refreshHighlights() {
      for (const key of Object.keys(this.pageViews)) {
        this.highlightMatchesOnPage(Number(key))
      }
    },
    /**
     * Wrap matches inside each text span. A match spanning two spans highlights
     * both spans rather than the exact glyph range.
     */
    highlightMatchesOnPage(pageNum) {
      const view = this.pageViews[pageNum]
      if (!view?.textDivs?.length) return
      const regex = this.searchRegex()
      const activeResult = this.activeMatchIndex >= 0 ? this.searchResults[this.activeMatchIndex] : null
      const isActivePage = activeResult?.page === pageNum

      for (const textDiv of view.textDivs) {
        const text = textDiv.textContent
        if (regex) regex.lastIndex = 0
        if (!regex || !regex.test(text)) {
          if (textDiv.dataset.highlighted) {
            textDiv.textContent = text
            delete textDiv.dataset.highlighted
          }
          continue
        }
        regex.lastIndex = 0
        const highlightClass = isActivePage ? 'pdf-highlight pdf-highlight-active' : 'pdf-highlight'
        textDiv.innerHTML = escapeHtml(text).replace(regex, (match) => `<span class="${highlightClass}">${match}</span>`)
        textDiv.dataset.highlighted = '1'
      }
    },

    //
    // Thumbnails
    //
    thumbnailHeight(pageNum) {
      const size = this.sizeForPage(pageNum)
      return Math.round((THUMBNAIL_WIDTH / size.width) * size.height)
    },
    observeThumbnails() {
      const panel = this.$refs.thumbnailPanel
      if (!panel || !this.pdfDoc) return
      if (!this.thumbObserver) {
        this.thumbObserver = new IntersectionObserver(
          (entries) => {
            for (const entry of entries) {
              if (entry.isIntersecting) this.queueThumbnail(Number(entry.target.dataset.thumbPage))
            }
          },
          { root: panel, rootMargin: '200px' }
        )
      }
      for (const el of panel.querySelectorAll('.pdf-thumb')) {
        this.thumbObserver.observe(el)
      }
    },
    queueThumbnail(pageNum) {
      if (!pageNum || this.thumbnailCache[pageNum]) return
      this.thumbnailCache[pageNum] = true
      this.thumbQueue.push(pageNum)
      this.drainThumbQueue()
    },
    async drainThumbQueue() {
      if (this.thumbRendering >= 2) return
      const pageNum = this.thumbQueue.shift()
      if (!pageNum) return
      this.thumbRendering++
      try {
        await this.renderThumbnail(pageNum)
      } catch (error) {
        this.thumbnailCache[pageNum] = false
      }
      this.thumbRendering--
      this.drainThumbQueue()
    },
    async renderThumbnail(pageNum) {
      if (!this.pdfDoc) return
      const pdfPage = await this.pdfDoc.getPage(pageNum)
      const size = this.sizeForPage(pageNum)
      const scale = THUMBNAIL_WIDTH / size.width
      const viewport = pdfPage.getViewport({ scale, rotation: (pdfPage.rotate + this.rotation) % 360 })
      const canvas = document.createElement('canvas')
      canvas.width = Math.floor(viewport.width)
      canvas.height = Math.floor(viewport.height)
      await pdfPage.render({ canvasContext: canvas.getContext('2d', { alpha: false }), viewport }).promise
      if (this.destroyed || !this.pdfDoc) return
      const el = this.$refs.thumbnailPanel?.querySelector(`.pdf-thumb[data-thumb-page="${pageNum}"]`)
      if (!el) return
      el.replaceChild(canvas, el.firstElementChild)
    },

    //
    // Progress
    //
    updateProgress() {
      if (!this.keepProgress || !this.numPages) return
      if (this.savedPage === this.page) return
      clearTimeout(this.progressTimeout)
      this.progressTimeout = setTimeout(() => {
        const payload = {
          ebookLocation: String(this.page),
          ebookProgress: Math.max(0, Math.min(1, (Number(this.page) - 1) / Number(this.numPages)))
        }
        this.$axios.$patch(`/api/me/progress/${this.libraryItemId}`, payload, { progress: false }).catch((error) => {
          console.error('PdfReader.updateProgress failed:', error)
        })
      }, 500)
    },

    //
    // Input handling
    //
    onKeyDown(e) {
      const modifier = e.ctrlKey || e.metaKey
      const targetTag = e.target?.tagName?.toLowerCase()
      const inInput = targetTag === 'input' || targetTag === 'textarea'

      if (modifier && (e.key === 'f' || e.key === 'F')) {
        e.preventDefault()
        this.openSearch()
        return
      }
      if (modifier && (e.key === '+' || e.key === '=')) {
        e.preventDefault()
        this.zoomIn()
        return
      }
      if (modifier && e.key === '-') {
        e.preventDefault()
        this.zoomOut()
        return
      }
      if (modifier && e.key === '0') {
        e.preventDefault()
        this.setFitMode('width')
        return
      }
      if (inInput) return
      if (e.key === 'Enter' && this.searchResults.length) {
        e.preventDefault()
        this.stepMatch(e.shiftKey ? -1 : 1)
      }
    },
    onTouchStart(e) {
      if (e.touches.length !== 2) return
      this.pinchStartDistance = this.touchDistance(e.touches)
      this.pinchStartScale = this.scale
    },
    onTouchMove(e) {
      if (e.touches.length !== 2 || !this.pinchStartDistance) return
      e.preventDefault()
      const ratio = this.touchDistance(e.touches) / this.pinchStartDistance
      const scale = Math.max(ZOOM_LEVELS[0], Math.min(ZOOM_LEVELS[ZOOM_LEVELS.length - 1], this.pinchStartScale * ratio))
      if (Math.abs(scale - this.scale) < 0.02) return
      this.fitMode = 'custom'
      this.scale = scale
      if (this.pinchRaf) return
      this.pinchRaf = requestAnimationFrame(() => {
        this.pinchRaf = null
        this.relayout({ preservePage: true })
      })
    },
    onTouchEnd() {
      this.pinchStartDistance = 0
    },
    touchDistance(touches) {
      return Math.hypot(touches[0].clientX - touches[1].clientX, touches[0].clientY - touches[1].clientY)
    },
    /**
     * Covers window resize, the audio player opening and the sidebar sliding in,
     * all of which change the width available to the pages
     */
    onContainerResize() {
      const viewer = this.$refs.viewer
      if (!viewer) return
      if (viewer.clientWidth === this.containerWidth && viewer.clientHeight === this.containerHeight) return
      this.measureContainer()
      clearTimeout(this.resizeTimeout)
      this.resizeTimeout = setTimeout(() => {
        this.relayout({ preservePage: true })
      }, 150)
    }
  },
  created() {
    // Non-reactive render state - Vue must not touch canvases or DOM layers
    this.pdfDoc = null
    this.loadingTask = null
    this.pageViews = {}
    this.pageTextCache = {}
    this.thumbnailCache = {}
    this.thumbQueue = []
    this.thumbRendering = 0
    this.searchToken = 0
    this.destroyed = false
    this.pinchStartDistance = 0
    this.pinchStartScale = 1
  },
  mounted() {
    window.addEventListener('keydown', this.onKeyDown)
    const viewer = this.$refs.viewer
    viewer.addEventListener('touchstart', this.onTouchStart, { passive: true })
    viewer.addEventListener('touchmove', this.onTouchMove, { passive: false })
    viewer.addEventListener('touchend', this.onTouchEnd, { passive: true })

    this.resizeObserver = new ResizeObserver(this.onContainerResize)
    this.resizeObserver.observe(viewer)

    this.init()
  },
  beforeDestroy() {
    this.destroyed = true
    window.removeEventListener('keydown', this.onKeyDown)
    const viewer = this.$refs.viewer
    if (viewer) {
      viewer.removeEventListener('touchstart', this.onTouchStart)
      viewer.removeEventListener('touchmove', this.onTouchMove)
      viewer.removeEventListener('touchend', this.onTouchEnd)
    }
    this.resizeObserver?.disconnect()
    this.thumbObserver?.disconnect()
    if (this.scrollRaf) cancelAnimationFrame(this.scrollRaf)
    if (this.pinchRaf) cancelAnimationFrame(this.pinchRaf)
    clearTimeout(this.progressTimeout)
    clearTimeout(this.resizeTimeout)
    this.destroyDocument()
  }
}
</script>

<style>
.pdf-viewer {
  transition: left 0.15s ease-in-out;
  -webkit-overflow-scrolling: touch;
}
.pdf-page .pdf-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
}
.pdf-invert .pdf-page {
  filter: invert(1) hue-rotate(180deg);
}

/* pdf.js text layer - span positions derive from --scale-factor on .pdf-page */
.pdf-page .textLayer {
  position: absolute;
  top: 0;
  left: 0;
  text-align: initial;
  overflow: hidden;
  line-height: 1;
  text-size-adjust: none;
  forced-color-adjust: none;
  transform-origin: 0 0;
  z-index: 2;
}
.pdf-page .textLayer span,
.pdf-page .textLayer br {
  color: transparent;
  position: absolute;
  white-space: pre;
  cursor: text;
  transform-origin: 0% 0%;
}
.pdf-page .textLayer span.pdf-highlight {
  position: static;
  background-color: rgba(250, 204, 21, 0.4);
  border-radius: 2px;
}
.pdf-page .textLayer span.pdf-highlight-active {
  background-color: rgba(250, 204, 21, 0.75);
}
.pdf-page .textLayer ::selection {
  background: rgba(37, 99, 235, 0.3);
}
.pdf-page .textLayer[data-main-rotation='90'] {
  transform: rotate(90deg) translateY(-100%);
}
.pdf-page .textLayer[data-main-rotation='180'] {
  transform: rotate(180deg) translate(-100%, -100%);
}
.pdf-page .textLayer[data-main-rotation='270'] {
  transform: rotate(270deg) translateX(-100%);
}

.pdf-page .pdf-link-layer {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 3;
}
.pdf-page .pdf-link-layer a {
  position: absolute;
  display: block;
  cursor: pointer;
  border-radius: 2px;
}
.pdf-page .pdf-link-layer a:hover {
  background-color: rgba(250, 204, 21, 0.25);
}

.pdf-thumb canvas {
  display: block;
  width: 100%;
  height: auto;
  background-color: white;
}
</style>
