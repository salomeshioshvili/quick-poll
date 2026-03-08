<script>
  import { page } from '$app/stores';
  import { db } from '$lib/firebase';
  import { ref, onValue, update, increment } from 'firebase/database';
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';

  const themes = {
    violet: 'linear-gradient(150deg, #2e1065 0%, #4c1d95 30%, #6d28d9 65%, #8b5cf6 100%)',
    ocean:  'linear-gradient(150deg, #0c2340 0%, #0c4a6e 30%, #0369a1 65%, #38bdf8 100%)',
    sunset: 'linear-gradient(150deg, #450a0a 0%, #991b1b 30%, #dc2626 65%, #fb923c 100%)',
    slate:  'linear-gradient(150deg, #0f172a 0%, #1e293b 30%, #334155 65%, #64748b 100%)',
  };

  let poll = null;
  let voted = false;
  let pollId = $page.params.id;
  let loading = true;
  let copied = false;

  onMount(() => {
    const pollRef = ref(db, `polls/${pollId}`);

    onValue(pollRef, (snapshot) => {
      if (snapshot.exists()) {
        poll = snapshot.val();
        document.body.style.background = themes[poll.theme] || themes.violet;
        loading = false;
      } else {
        loading = false;
      }
    });

    voted = localStorage.getItem(`voted_${pollId}`) === 'true';

    if (!sessionStorage.getItem(`viewed_${pollId}`)) {
      update(ref(db), { [`polls/${pollId}/views`]: increment(1) });
      sessionStorage.setItem(`viewed_${pollId}`, 'true');
    }
  });

  async function vote(optionIndex) {
    if (voted) return;
    const updates = {};
    updates[`polls/${pollId}/options/${optionIndex}/votes`] =
      (poll.options[optionIndex].votes || 0) + 1;
    await update(ref(db), updates);
    localStorage.setItem(`voted_${pollId}`, 'true');
    voted = true;
    setTimeout(() => goto(`/results/${pollId}`), 900);
  }

  async function copyLink() {
    await navigator.clipboard.writeText(window.location.href);
    await update(ref(db), { [`polls/${pollId}/shares`]: increment(1) });
    copied = true;
    setTimeout(() => copied = false, 2000);
  }
</script>

<svelte:head>
  <title>{poll ? poll.question : 'Poll'}</title>
</svelte:head>

<main>
  {#if loading}
    <div class="container">
      <div class="state-center">
        <div class="spinner"></div>
        <p>Loading...</p>
      </div>
    </div>
  {:else if !poll}
    <div class="container">
      <div class="state-center">
        <p class="state-title">Poll not found</p>
        <p class="state-sub">This poll may have been deleted.</p>
        <a href="/" class="btn-primary">Create a Poll</a>
      </div>
    </div>
  {:else}
    <div class="container">
      {#if poll.imageUrl}
        <div class="poll-image">
          <img src={poll.imageUrl} alt="" />
        </div>
      {/if}

      <h1>{poll.question}</h1>

      {#if poll.views || poll.shares}
        <p class="poll-meta">{poll.views || 0} views · {poll.shares || 0} shares</p>
      {/if}

      {#if !voted}
        <p class="instruction">Choose one</p>
        <div class="options">
          {#each poll.options as option, i}
            <button class="option" on:click={() => vote(i)}>
              <span class="option-text">{option.text}</span>
              <svg class="option-arrow" width="16" height="16" viewBox="0 0 16 16" fill="none">
                <path d="M3 8H13M9 4l4 4-4 4" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
            </button>
          {/each}
        </div>
      {:else}
        <div class="voted-state">
          <div class="voted-icon">
            <svg width="28" height="28" viewBox="0 0 28 28" fill="none">
              <path d="M6 14l6 6L22 8" stroke="white" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
            </svg>
          </div>
          <p class="voted-title">Vote recorded</p>
          <p class="voted-sub">Redirecting to results…</p>
        </div>
      {/if}

      <div class="actions">
        <button class="btn-secondary" class:btn-success={copied} on:click={copyLink}>
          {#if copied}
            <svg width="14" height="14" viewBox="0 0 14 14" fill="none">
              <path d="M2 7l4 4 6-7" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"/>
            </svg>
            Copied
          {:else}
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none">
              <path d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-2M8 5a2 2 0 012-2h4a2 2 0 012 2v0a2 2 0 01-2 2h-4a2 2 0 01-2-2z" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
            </svg>
            Copy link
          {/if}
        </button>
        <a href="/results/{pollId}" class="btn-secondary">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none">
            <path d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
          Results
        </a>
      </div>
    </div>
  {/if}
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background: linear-gradient(150deg, #2e1065 0%, #4c1d95 30%, #6d28d9 65%, #8b5cf6 100%);
    min-height: 100vh;
  }

  main {
    padding: 2.5rem 1rem;
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .container {
    width: 100%;
    max-width: 440px;
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(28px);
    -webkit-backdrop-filter: blur(28px);
    border: 1px solid rgba(255, 255, 255, 0.18);
    border-radius: 24px;
    padding: 2rem;
    box-shadow:
      0 4px 6px rgba(0, 0, 0, 0.07),
      0 20px 48px rgba(0, 0, 0, 0.2),
      inset 0 1px 0 rgba(255, 255, 255, 0.22);
  }

  .state-center {
    text-align: center;
    padding: 2.5rem 0;
  }

  .spinner {
    width: 36px;
    height: 36px;
    border: 3px solid rgba(255, 255, 255, 0.15);
    border-top-color: rgba(255, 255, 255, 0.8);
    border-radius: 50%;
    animation: spin 0.75s linear infinite;
    margin: 0 auto 1rem;
  }

  @keyframes spin {
    to { transform: rotate(360deg); }
  }

  .state-center p {
    color: rgba(255, 255, 255, 0.7);
    margin: 0;
    font-size: 0.9rem;
  }

  .state-title {
    color: white !important;
    font-size: 1.1rem !important;
    font-weight: 600;
    margin-bottom: 0.4rem !important;
  }

  .state-sub {
    margin-bottom: 1.5rem !important;
  }

  .poll-image {
    border-radius: 14px;
    overflow: hidden;
    margin-bottom: 1.25rem;
  }

  .poll-image img {
    width: 100%;
    max-height: 200px;
    object-fit: cover;
    display: block;
  }

  h1 {
    margin: 0 0 0.5rem;
    color: white;
    font-size: 1.35rem;
    font-weight: 700;
    line-height: 1.35;
    letter-spacing: -0.01em;
  }

  .poll-meta {
    margin: 0 0 1.25rem;
    color: rgba(255, 255, 255, 0.4);
    font-size: 0.78rem;
  }

  .instruction {
    margin: 1rem 0 0.75rem;
    color: rgba(255, 255, 255, 0.55);
    font-size: 0.78rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.07em;
  }

  .options {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    margin-bottom: 1.5rem;
  }

  .option {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.75rem;
    padding: 0.85rem 1rem;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.15);
    border-radius: 12px;
    cursor: pointer;
    color: white;
    font-size: 0.9rem;
    font-weight: 500;
    font-family: inherit;
    text-align: left;
    transition: background 0.15s, border-color 0.15s, transform 0.15s, box-shadow 0.15s;
  }

  .option:hover {
    background: rgba(255, 255, 255, 0.18);
    border-color: rgba(255, 255, 255, 0.35);
    transform: translateY(-2px);
    box-shadow: 0 6px 16px rgba(0, 0, 0, 0.18);
  }

  .option:active {
    transform: translateY(0);
  }

  .option-text {
    flex: 1;
  }

  .option-arrow {
    flex-shrink: 0;
    color: rgba(255, 255, 255, 0.4);
    transition: color 0.15s, transform 0.15s;
  }

  .option:hover .option-arrow {
    color: rgba(255, 255, 255, 0.9);
    transform: translateX(2px);
  }

  .voted-state {
    text-align: center;
    padding: 2.5rem 0 1.5rem;
  }

  .voted-icon {
    width: 60px;
    height: 60px;
    background: rgba(16, 185, 129, 0.25);
    border: 1px solid rgba(16, 185, 129, 0.4);
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 0 auto 1rem;
    animation: scaleIn 0.25s ease-out;
  }

  @keyframes scaleIn {
    from { transform: scale(0.5); opacity: 0; }
    to { transform: scale(1); opacity: 1; }
  }

  .voted-title {
    margin: 0 0 0.3rem;
    color: white;
    font-size: 1.1rem;
    font-weight: 600;
  }

  .voted-sub {
    margin: 0;
    color: rgba(255, 255, 255, 0.5);
    font-size: 0.82rem;
  }

  .actions {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0.6rem;
  }

  .btn-primary {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 0.4rem;
    padding: 0.7rem 1.25rem;
    background: rgba(255, 255, 255, 0.2);
    border: 1px solid rgba(255, 255, 255, 0.32);
    border-radius: 10px;
    color: white;
    font-size: 0.85rem;
    font-weight: 600;
    font-family: inherit;
    text-decoration: none;
    cursor: pointer;
    transition: background 0.15s, transform 0.15s, box-shadow 0.15s;
    box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.18);
  }

  .btn-primary:hover {
    background: rgba(255, 255, 255, 0.28);
    transform: translateY(-1px);
  }

  .btn-secondary {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 0.4rem;
    padding: 0.7rem 1rem;
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.14);
    border-radius: 10px;
    color: rgba(255, 255, 255, 0.8);
    font-size: 0.82rem;
    font-weight: 500;
    font-family: inherit;
    text-decoration: none;
    cursor: pointer;
    transition: background 0.15s, border-color 0.15s, color 0.15s, transform 0.15s;
  }

  .btn-secondary:hover {
    background: rgba(255, 255, 255, 0.14);
    border-color: rgba(255, 255, 255, 0.25);
    color: white;
    transform: translateY(-1px);
  }

  .btn-success {
    background: rgba(16, 185, 129, 0.2) !important;
    border-color: rgba(16, 185, 129, 0.35) !important;
    color: #6ee7b7 !important;
  }
</style>
