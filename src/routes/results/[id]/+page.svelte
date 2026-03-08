<script>
  import { page } from '$app/stores';
  import { db } from '$lib/firebase';
  import { ref, onValue, update, increment } from 'firebase/database';
  import { onMount } from 'svelte';

  const themes = {
    violet: 'linear-gradient(150deg, #2e1065 0%, #4c1d95 30%, #6d28d9 65%, #8b5cf6 100%)',
    ocean:  'linear-gradient(150deg, #0c2340 0%, #0c4a6e 30%, #0369a1 65%, #38bdf8 100%)',
    sunset: 'linear-gradient(150deg, #450a0a 0%, #991b1b 30%, #dc2626 65%, #fb923c 100%)',
    slate:  'linear-gradient(150deg, #0f172a 0%, #1e293b 30%, #334155 65%, #64748b 100%)',
  };

  let poll = null;
  let pollId = $page.params.id;
  let totalVotes = 0;
  let loading = true;
  let showQR = false;
  let copied = false;

  onMount(() => {
    const pollRef = ref(db, `polls/${pollId}`);

    onValue(pollRef, (snapshot) => {
      if (snapshot.exists()) {
        poll = snapshot.val();
        totalVotes = poll.options.reduce((sum, opt) => sum + (opt.votes || 0), 0);
        document.body.style.background = themes[poll.theme] || themes.violet;
        loading = false;
      } else {
        loading = false;
      }
    });
  });

  function getPollUrl() {
    return `${window.location.origin}/poll/${pollId}`;
  }

  function getQrUrl() {
    return `https://api.qrserver.com/v1/create-qr-code/?size=300x300&data=${encodeURIComponent(getPollUrl())}&qzone=1`;
  }

  async function copyVoteLink() {
    await navigator.clipboard.writeText(getPollUrl());
    await update(ref(db), { [`polls/${pollId}/shares`]: increment(1) });
    copied = true;
    setTimeout(() => copied = false, 2000);
  }

  function getWinner() {
    if (!poll || totalVotes === 0) return null;
    let maxVotes = 0;
    let winners = [];
    poll.options.forEach((option, index) => {
      if ((option.votes || 0) > maxVotes) {
        maxVotes = option.votes || 0;
        winners = [index];
      } else if ((option.votes || 0) === maxVotes) {
        winners.push(index);
      }
    });
    return winners.length === 1 ? winners[0] : null;
  }
</script>

<svelte:head>
  <title>{poll ? `Results: ${poll.question}` : 'Poll Results'}</title>
</svelte:head>

<main>
  {#if loading}
    <div class="container">
      <div class="state-center">
        <div class="spinner"></div>
        <p>Loading results…</p>
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
        <div class="stats-bar">
          <div class="stat-item">
            <span class="stat-label">Views</span>
            <span class="stat-value">{poll.views || 0}</span>
          </div>
          <div class="stat-divider"></div>
          <div class="stat-item">
            <span class="stat-label">Shares</span>
            <span class="stat-value">{poll.shares || 0}</span>
          </div>
          <div class="stat-divider"></div>
          <div class="stat-item">
            <span class="stat-label">Votes</span>
            <span class="stat-value">{totalVotes}</span>
          </div>
        </div>
      {/if}

      <div class="results">
        {#each poll.options as option, i}
          {@const votes = option.votes || 0}
          {@const percentage = totalVotes > 0 ? (votes / totalVotes * 100).toFixed(1) : 0}
          {@const isWinner = getWinner() === i}

          <div class="result-item" class:winner={isWinner}>
            <div class="result-header">
              <span class="option-text">
                {#if isWinner && totalVotes > 0}
                  <svg class="trophy-icon" width="14" height="14" viewBox="0 0 24 24" fill="none">
                    <path d="M12 15c-3.314 0-6-2.686-6-6V4h12v5c0 3.314-2.686 6-6 6zM8.5 21h7M12 15v6M6 4H3v3a3 3 0 003 3M18 4h3v3a3 3 0 01-3 3" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"/>
                  </svg>
                {/if}
                {option.text}
              </span>
              <span class="votes-info">
                <span class="vote-count">{votes}</span>
                <span class="percentage">{percentage}%</span>
              </span>
            </div>
            <div class="bar-track">
              <div class="bar" class:winner-bar={isWinner} style="width: {percentage}%"></div>
            </div>
          </div>
        {/each}
      </div>

      {#if showQR}
        <div class="qr-section">
          <p class="qr-label">Scan to vote</p>
          <img class="qr-code" src={getQrUrl()} alt="QR Code" />
          <p class="qr-url">{getPollUrl()}</p>
        </div>
      {/if}

      <div class="actions">
        <button class="btn-primary" class:btn-success={copied} on:click={copyVoteLink}>
          {#if copied}
            <svg width="14" height="14" viewBox="0 0 14 14" fill="none">
              <path d="M2 7l4 4 6-7" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"/>
            </svg>
            Copied
          {:else}
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none">
              <path d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-2M8 5a2 2 0 012-2h4a2 2 0 012 2v0a2 2 0 01-2 2h-4a2 2 0 01-2-2z" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
            </svg>
            Share poll link
          {/if}
        </button>

        <button class="btn-secondary" on:click={() => showQR = !showQR}>
          {#if showQR}
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none">
              <path d="M19 9l-7 7-7-7" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"/>
            </svg>
            Hide QR
          {:else}
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none">
              <rect x="3" y="3" width="7" height="7" rx="1" stroke="currentColor" stroke-width="1.8"/>
              <rect x="14" y="3" width="7" height="7" rx="1" stroke="currentColor" stroke-width="1.8"/>
              <rect x="3" y="14" width="7" height="7" rx="1" stroke="currentColor" stroke-width="1.8"/>
              <path d="M14 14h2v2h-2zM18 14h3M14 18h3M18 18h3v3" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
            </svg>
            QR Code
          {/if}
        </button>

        <a href="/poll/{pollId}" class="btn-secondary">
          <svg width="14" height="14" viewBox="0 0 16 16" fill="none">
            <path d="M13 8H3M7 4L3 8l4 4" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
          Back to vote
        </a>

        <a href="/" class="btn-secondary">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none">
            <path d="M12 5v14M5 12h14" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
          </svg>
          New poll
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
    margin: 0 0 0.75rem;
    color: white;
    font-size: 1.35rem;
    font-weight: 700;
    line-height: 1.35;
    letter-spacing: -0.01em;
  }

  .stats-bar {
    display: flex;
    align-items: center;
    gap: 0;
    background: rgba(255, 255, 255, 0.07);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 12px;
    margin-bottom: 1.25rem;
    overflow: hidden;
  }

  .stat-item {
    flex: 1;
    text-align: center;
    padding: 0.7rem 0.5rem;
  }

  .stat-divider {
    width: 1px;
    height: 32px;
    background: rgba(255, 255, 255, 0.12);
    flex-shrink: 0;
  }

  .stat-label {
    display: block;
    font-size: 0.68rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.07em;
    color: rgba(255, 255, 255, 0.45);
    margin-bottom: 0.2rem;
  }

  .stat-value {
    display: block;
    font-size: 1.25rem;
    font-weight: 700;
    color: white;
    line-height: 1;
  }

  .results {
    display: flex;
    flex-direction: column;
    gap: 0.55rem;
    margin-bottom: 1.5rem;
  }

  .result-item {
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.12);
    padding: 0.85rem 1rem;
    border-radius: 12px;
  }

  .result-item.winner {
    background: rgba(251, 191, 36, 0.14);
    border-color: rgba(251, 191, 36, 0.32);
  }

  .result-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 0.75rem;
    margin-bottom: 0.55rem;
  }

  .option-text {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    font-weight: 600;
    color: white;
    font-size: 0.88rem;
    flex: 1;
    line-height: 1.35;
  }

  .trophy-icon {
    flex-shrink: 0;
    color: #fbbf24;
  }

  .votes-info {
    display: flex;
    gap: 0.6rem;
    align-items: baseline;
    white-space: nowrap;
  }

  .vote-count {
    color: rgba(255, 255, 255, 0.55);
    font-size: 0.78rem;
    font-weight: 500;
  }

  .percentage {
    color: white;
    font-size: 0.9rem;
    font-weight: 700;
    min-width: 38px;
    text-align: right;
  }

  .bar-track {
    background: rgba(0, 0, 0, 0.2);
    height: 6px;
    border-radius: 99px;
    overflow: hidden;
  }

  .bar {
    background: rgba(255, 255, 255, 0.45);
    height: 100%;
    border-radius: 99px;
    transition: width 0.7s cubic-bezier(0.4, 0, 0.2, 1);
    min-width: 2px;
  }

  .bar.winner-bar {
    background: rgba(251, 191, 36, 0.75);
  }

  .qr-section {
    background: rgba(255, 255, 255, 0.07);
    border: 1px solid rgba(255, 255, 255, 0.14);
    border-radius: 14px;
    padding: 1.25rem 1rem;
    text-align: center;
    margin-bottom: 1rem;
    animation: fadeSlide 0.22s ease-out;
  }

  @keyframes fadeSlide {
    from { opacity: 0; transform: translateY(-6px); }
    to { opacity: 1; transform: translateY(0); }
  }

  .qr-label {
    color: rgba(255, 255, 255, 0.55);
    font-size: 0.72rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.07em;
    margin: 0 0 0.75rem;
  }

  .qr-code {
    border-radius: 10px;
    display: block;
    margin: 0 auto;
    width: 148px;
    height: 148px;
  }

  .qr-url {
    color: rgba(255, 255, 255, 0.35);
    font-size: 0.67rem;
    margin: 0.6rem 0 0;
    word-break: break-all;
  }

  .actions {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0.6rem;
  }

  .btn-primary {
    grid-column: 1 / -1;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 0.4rem;
    padding: 0.75rem 1.25rem;
    background: rgba(255, 255, 255, 0.18);
    border: 1px solid rgba(255, 255, 255, 0.32);
    border-radius: 10px;
    color: white;
    font-size: 0.85rem;
    font-weight: 600;
    font-family: inherit;
    cursor: pointer;
    text-decoration: none;
    transition: background 0.15s, transform 0.15s;
    box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.18);
  }

  .btn-primary:hover {
    background: rgba(255, 255, 255, 0.26);
    transform: translateY(-1px);
  }

  .btn-success {
    background: rgba(16, 185, 129, 0.2) !important;
    border-color: rgba(16, 185, 129, 0.35) !important;
    color: #6ee7b7 !important;
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
</style>