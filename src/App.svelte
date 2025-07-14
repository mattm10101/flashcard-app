<script>
  import { tweened } from 'svelte/motion'
  import { cubicOut, cubicInOut } from 'svelte/easing'
  import { onMount } from 'svelte'

  // --- Credentials from .env file ---
  const API_KEY = import.meta.env.VITE_GOOGLE_SHEETS_API_KEY
  const SPREADSHEET_ID = import.meta.env.VITE_SPREADSHEET_ID

  // --- State Management ---
  let selectedDeckName = null
  let allTerms = []
  let activeTerms = []
  let currentCardIndex = 0
  let isLoading = false
  let loadingMessage = ''
  let error = null
  let availableDecks = []
  let showInfoPanel = false // To track if the info panel is visible

  // --- Animation Stores ---
  const rotation = tweened(0, { duration: 600, easing: cubicInOut })
  const cardPosition = tweened({ x: 0 }, { duration: 200, easing: cubicOut })

  // --- Data Fetching & Deck Selection ---
  onMount(async () => {
    isLoading = true
    loadingMessage = 'Finding decks...'
    const url = `https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}?fields=sheets.properties.title&key=${API_KEY}`
    try {
      const response = await fetch(url)
      if (!response.ok)
        throw new Error(`API responded with status: ${response.status}`)
      const data = await response.json()
      availableDecks = data.sheets.map((sheet) => sheet.properties.title)
    } catch (e) {
      error = 'Could not find decks. Check Spreadsheet ID and API Key.'
      console.error(e)
    } finally {
      isLoading = false
    }
  })

  async function selectDeck(deckName) {
    isLoading = true
    loadingMessage = 'Summoning cards...'
    selectedDeckName = deckName
    const url = `https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values/${deckName}?key=${API_KEY}`
    try {
      const response = await fetch(url)
      if (!response.ok)
        throw new Error(`API responded with status: ${response.status}`)
      const data = await response.json()
      allTerms = data.values.slice(1).map((row) => ({
        term: row[0],
        definition: row[1],
        status: 'active',
      }))
      activeTerms = [...allTerms]
      currentCardIndex = 0
    } catch (e) {
      error = 'Could not load deck. Check sheet name and sharing settings.'
      console.error(e)
    } finally {
      isLoading = false
    }
  }

  // --- Swipe Action ---
  function swipeable(node) {
    let touchX, touchY
    const SWIPE_THRESHOLD = 50
    function handleTouchStart(e) {
      touchX = e.touches[0].clientX
      touchY = e.touches[0].clientY
    }
    function handleTouchEnd(e) {
      if (!touchX || !touchY) return
      const dX = e.changedTouches[0].clientX - touchX
      const dY = e.changedTouches[0].clientY - touchY
      if (Math.abs(dX) > Math.abs(dY) && Math.abs(dX) > SWIPE_THRESHOLD) {
        node.dispatchEvent(new CustomEvent(dX < 0 ? 'swipeleft' : 'swiperight'))
      } else if (
        Math.abs(dY) > Math.abs(dX) &&
        Math.abs(dY) > SWIPE_THRESHOLD
      ) {
        node.dispatchEvent(new CustomEvent(dY < 0 ? 'swipeup' : 'swipedown'))
      }
      touchX = null
      touchY = null
    }
    node.addEventListener('touchstart', handleTouchStart, { passive: true })
    node.addEventListener('touchend', handleTouchEnd, { passive: true })
    return {
      destroy() {
        node.removeEventListener('touchstart', handleTouchStart)
        node.removeEventListener('touchend', handleTouchEnd)
      },
    }
  }

  // --- Navigation & Card Logic ---
  $: if (currentCardIndex !== undefined && activeTerms.length > 0) {
    rotation.set(0, { duration: 0 })
    if (showInfoPanel) {
      showInfoPanel = false
      cardPosition.set({ x: 0 }, { duration: 200 })
    }
  }

  async function handleSwipeLeft() {
    if (showInfoPanel) {
      showInfoPanel = false
      cardPosition.set({ x: 0 })
      return
    }
    await cardPosition.set({ x: -600 })
    activeTerms[currentCardIndex].status = 'learned'
    activeTerms = activeTerms.filter((term) => term.status === 'active')
    if (activeTerms.length === 0) return
    currentCardIndex = currentCardIndex % activeTerms.length
    cardPosition.set({ x: 0 }, { duration: 0 })
  }

  function handleSwipeRight() {
    showInfoPanel = !showInfoPanel
    cardPosition.set({ x: showInfoPanel ? 250 : 0 })
  }

  function handleNextCard() {
    if (activeTerms.length === 0) return
    currentCardIndex = (currentCardIndex + 1) % activeTerms.length
  }

  function handlePreviousCard() {
    if (activeTerms.length === 0) return
    currentCardIndex =
      (currentCardIndex - 1 + activeTerms.length) % activeTerms.length
  }

  function resetDeck() {
    activeTerms = [...allTerms]
    currentCardIndex = 0
  }

  function onCardClick() {
    if (showInfoPanel) return
    rotation.set($rotation > 90 ? 0 : 180)
  }
</script>

<main>
  {#if isLoading}
    <p class="loading-text">{loadingMessage}</p>
  {:else if error}
    <div class="error-message">
      <p><strong>Oops! Something went wrong.</strong></p>
      <p>{error}</p>
      <button on:click={() => window.location.reload()}>Try Again</button>
    </div>
  {:else if !selectedDeckName}
    <div class="deck-selector">
      <h1 class="selector-title">Choose a Deck</h1>
      {#each availableDecks as deckName}
        <button
          class="deck-button {deckName.toLowerCase()}"
          on:click={() => selectDeck(deckName)}
        >
          {deckName}
        </button>
      {/each}
    </div>
  {:else if activeTerms.length === 0}
    <div class="completion-message">
      <h2 class="selector-title">✨ Deck Complete! ✨</h2>
      <button class="deck-button" on:click={resetDeck}>Start Over</button>
    </div>
  {:else}
    <div
      class="card-perspective-wrapper"
      use:swipeable
      on:swipeup={handleNextCard}
      on:swipedown={handlePreviousCard}
      on:swipeleft={handleSwipeLeft}
      on:swiperight={handleSwipeRight}
    >
      <div class="info-panel">
        <div class="card-counter">
          <span>{currentCardIndex + 1}</span>
          <span class="divider">/</span>
          <span>{activeTerms.length}</span>
        </div>
      </div>

      <button type="button" class="card-button" on:click={onCardClick}>
        <div
          class="card-visuals"
          style="transform: translateX({$cardPosition.x}px) rotateY({$rotation}deg);"
        >
          <div class="card-face card-front">
            <h1 class="term-text">{activeTerms[currentCardIndex].term}</h1>
          </div>
          <div class="card-face card-back">
            <p class="definition-text">
              {activeTerms[currentCardIndex].definition}
            </p>
          </div>
        </div>
      </button>
    </div>
  {/if}
</main>

<style>
  :global(body) {
    background-color: #111827;
    color: white;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
      Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    margin: 0;
    overscroll-behavior-y: contain;
  }
  main {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 1rem;
    box-sizing: border-box;
  }
  .deck-selector,
  .error-message,
  .completion-message {
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;
    gap: 1.5rem;
  }
  .selector-title,
  .loading-text {
    font-size: 3rem;
    font-weight: bold;
    color: #9ca3af;
  }
  .error-message {
    background-color: #374151;
    padding: 2rem;
    border-radius: 1rem;
  }
  .deck-button,
  .error-message button {
    width: 250px;
    padding: 1rem;
    border: 2px solid;
    border-radius: 0.75rem;
    font-size: 1.5rem;
    font-weight: bold;
    color: white;
    background-color: transparent;
    cursor: pointer;
    transition: all 0.2s ease-in-out;
  }
  .completion-message .deck-button {
    border-color: #a78bfa;
  }
  .completion-message .deck-button:hover {
    background-color: #a78bfa;
  }
  .error-message button {
    border-color: #9ca3af;
  }
  .deck-button.easy {
    border-color: #34d399;
  }
  .deck-button.medium {
    border-color: #f59e0b;
  }
  .deck-button.hard {
    border-color: #ef4444;
  }
  .deck-button.super-hard {
    border-color: #a78bfa;
  }
  .deck-button:hover,
  .error-message button:hover {
    transform: scale(1.05);
  }
  .deck-button.easy:hover {
    background-color: #34d399;
  }
  .deck-button.medium:hover {
    background-color: #f59e0b;
  }
  .deck-button.hard:hover {
    background-color: #ef4444;
  }
  .deck-button.super-hard:hover {
    background-color: #a78bfa;
  }

  .card-perspective-wrapper {
    width: 100%;
    max-width: 384px;
    perspective: 1000px;
    position: relative; /* Needed for positioning info panel */
  }

  /* ✨ UPDATED: Info Panel & Card Counter Styles ✨ */
  .info-panel {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #1f2937;
    border-radius: 1.25rem;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    color: #9ca3af;
    padding: 2rem;
    box-sizing: border-box;
  }

  .card-counter {
    font-size: 3rem;
    font-weight: 900;
    color: #4b5563;
  }
  .card-counter .divider {
    font-size: 2rem;
    margin: 0 0.5rem;
  }

  .card-button {
    background: none;
    border: none;
    padding: 0;
    cursor: pointer;
    display: block;
    width: 100%;
    border-radius: 1.25rem;
    -webkit-tap-highlight-color: transparent;
    position: relative; /* Make sure button is on top of info panel */
    z-index: 10;
  }

  @keyframes pulse-glow {
    0%,
    100% {
      box-shadow:
        0 0 8px #0ff,
        0 0 16px #0ff,
        0 0 24px #f0f;
    }
    50% {
      box-shadow:
        0 0 16px #0ff,
        0 0 32px #f0f,
        0 0 48px #f0f;
    }
  }

  .card-visuals {
    transform-style: preserve-3d;
    height: 450px;
    position: relative;
    pointer-events: none;
    border-radius: 1.25rem;
    animation: pulse-glow 5s ease-in-out infinite;
  }
  .card-face {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    border-radius: 1.25rem;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 1.5rem 1rem;
    text-align: center;
    backface-visibility: hidden;
    -webkit-backface-visibility: hidden;
    box-sizing: border-box;
    overflow-y: auto;
  }
  .card-front {
    background-color: #1e3a8a;
  }
  .card-back {
    background-color: #374151;
  }

  .term-text {
    color: #ffffff;
    font-size: clamp(3rem, 15vw, 5rem);
    font-weight: bold;
    line-height: 1.1;
    text-shadow: 0 2px 5px rgba(0, 0, 0, 0.5);
    overflow-wrap: break-word;
    word-break: break-word;
  }
  .definition-text {
    color: #ffffff;
    font-size: clamp(2rem, 9vw, 3rem);
    line-height: 1.4;
    text-shadow: 0 2px 5px rgba(0, 0, 0, 0.5);
    overflow-wrap: break-word;
    word-break: break-word;
  }
</style>
