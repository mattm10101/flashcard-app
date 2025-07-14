<script>
  import { tweened } from 'svelte/motion'
  import { cubicOut } from 'svelte/easing'
  import { onMount } from 'svelte'

  // --- Credentials from .env file ---
  const API_KEY = import.meta.env.VITE_GOOGLE_SHEETS_API_KEY
  const SPREADSHEET_ID = import.meta.env.VITE_SPREADSHEET_ID

  // --- State Management ---
  let selectedDeckName = null
  let activeTerms = []
  let currentCardIndex = 0
  let isLoading = false
  let loadingMessage = ''
  let error = null
  let availableDecks = [] // This will be populated from the API

  // --- Data Fetching Logic ---
  // This now runs once when the app starts to get the list of decks
  onMount(async () => {
    isLoading = true
    loadingMessage = 'Finding decks...'
    error = null

    const url = `https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}?fields=sheets.properties.title&key=${API_KEY}`

    try {
      const response = await fetch(url)
      if (!response.ok)
        throw new Error(`API responded with status: ${response.status}`)
      const data = await response.json()

      // Extract the title from each sheet's properties
      availableDecks = data.sheets.map((sheet) => sheet.properties.title)
    } catch (e) {
      error =
        'Could not find decks. Please check your Spreadsheet ID and API Key.'
      console.error(e)
    } finally {
      isLoading = false
    }
  })

  // This function is called when a user clicks a deck button
  async function selectDeck(deckName) {
    isLoading = true
    loadingMessage = 'Summoning cards...'
    selectedDeckName = deckName
    error = null

    const url = `https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values/${deckName}?key=${API_KEY}`

    try {
      const response = await fetch(url)
      if (!response.ok)
        throw new Error(`API responded with status: ${response.status}`)
      const data = await response.json()

      activeTerms = data.values.slice(1).map((row) => ({
        term: row[0],
        definition: row[1],
      }))
      currentCardIndex = 0
    } catch (e) {
      error =
        'Could not load deck. Please check your sheet name and sharing settings.'
      console.error(e)
    } finally {
      isLoading = false
    }
  }

  // --- Swipe Action (no changes) ---
  function swipeable(node) {
    let touchX, touchY
    const SWIPE_THRESHOLD = 50
    function handleTouchStart(event) {
      touchX = event.touches[0].clientX
      touchY = event.touches[0].clientY
    }
    function handleTouchEnd(event) {
      if (!touchX || !touchY) return
      const deltaX = event.changedTouches[0].clientX - touchX
      const deltaY = event.changedTouches[0].clientY - touchY
      if (
        Math.abs(deltaY) > SWIPE_THRESHOLD &&
        Math.abs(deltaY) > Math.abs(deltaX)
      ) {
        if (deltaY < 0) node.dispatchEvent(new CustomEvent('swipeup'))
        else if (deltaY > 0) node.dispatchEvent(new CustomEvent('swipedown'))
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

  // --- Animation & Navigation (no changes to logic) ---
  const rotation = tweened(0, { duration: 600, easing: cubicOut })
  $: if (currentCardIndex !== undefined) rotation.set(0, { duration: 0 })
  function goToNext() {
    currentCardIndex = (currentCardIndex + 1) % activeTerms.length
  }
  function goToPrevious() {
    currentCardIndex =
      (currentCardIndex - 1 + activeTerms.length) % activeTerms.length
  }
  function onCardClick() {
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
  {:else}
    <div
      class="card-perspective-wrapper"
      use:swipeable
      on:swipeup={goToNext}
      on:swipedown={goToPrevious}
    >
      <button type="button" class="card-button" on:click={onCardClick}>
        <div class="card-visuals" style="transform: rotateY({$rotation}deg);">
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

  /* --- Initial Screens Styles --- */
  .deck-selector,
  .error-message {
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

  .error-message button {
    border-color: #9ca3af;
  }

  /* These classes are now dynamically applied but will only work if your sheet names match */
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
  } /* Added a style for "Super Hard" */

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

  /* --- Flashcard Styles --- */
  .card-perspective-wrapper {
    width: 100%;
    max-width: 384px;
    perspective: 1000px;
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
