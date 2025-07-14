<script>
  import { tweened } from 'svelte/motion'
  import { cubicOut } from 'svelte/easing'

  let terms = [
    { term: 'Arcane', definition: 'Understood by few; mysterious or secret.' },
    {
      term: 'Whimsical',
      definition:
        'Playfully quaint or fanciful, especially in an appealing and amusing way.',
    },
    {
      term: 'Ephemeral',
      definition: 'Lasting for a very short time; transient.',
    },
    {
      term: 'Luminous',
      definition:
        'Emitting or reflecting bright light, especially in the dark.',
    },
    {
      term: 'Serendipity',
      definition:
        'The occurrence of events by chance in a happy or beneficial way.',
    },
  ]

  let currentCardIndex = 0

  // Svelte Action to detect swipes
  function swipeable(node) {
    let touchX
    let touchY
    const SWIPE_THRESHOLD = 50 // Minimum pixels for a swipe

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
        if (deltaY < 0) {
          // Negative deltaY means swiping up
          node.dispatchEvent(new CustomEvent('swipeup'))
        } else if (deltaY > 0) {
          // Positive deltaY means swiping down
          node.dispatchEvent(new CustomEvent('swipedown'))
        }
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

  const rotation = tweened(0, {
    duration: 600,
    easing: cubicOut,
  })

  // Whenever the card changes, instantly flip it back to the front
  $: if (currentCardIndex !== undefined) {
    rotation.set(0, { duration: 0 })
  }

  // --- Navigation Logic ---
  function goToNext() {
    currentCardIndex = (currentCardIndex + 1) % terms.length
  }

  function goToPrevious() {
    currentCardIndex = (currentCardIndex - 1 + terms.length) % terms.length
  }

  function onCardClick() {
    const isFlipped = $rotation > 90
    rotation.set(isFlipped ? 0 : 180)
  }
</script>

<main>
  <div
    class="card-perspective-wrapper"
    use:swipeable
    on:swipeup={goToNext}
    on:swipedown={goToPrevious}
  >
    <button type="button" class="card-button" on:click={onCardClick}>
      <div class="card-visuals" style="transform: rotateY({$rotation}deg);">
        <div class="card-face card-front">
          <h1 class="term-text">{terms[currentCardIndex].term}</h1>
        </div>
        <div class="card-face card-back">
          <p class="definition-text">{terms[currentCardIndex].definition}</p>
        </div>
      </div>
    </button>
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
      Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    background-color: #111827;
    color: white;
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

  .card-perspective-wrapper {
    width: 100%;
    max-width: 384px;
    perspective: 1000px;
    position: relative;
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

  .card-button:focus-visible {
    outline: 3px solid #60a5fa;
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
  }

  .card-front {
    background-color: #1e3a8a;
  }

  .card-back {
    background-color: #374151;
    overflow-y: auto;
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
