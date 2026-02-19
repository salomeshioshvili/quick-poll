<script>
  import { page } from '$app/stores';
  import { db } from '$lib/firebase';
  import { ref, onValue, update, increment } from 'firebase/database';
  import { onMount } from 'svelte';

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
    return `https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=${encodeURIComponent(getPollUrl())}&qzone=1`;
  }

  async function copyVoteLink() {
    const voteLink = getPollUrl();
    await navigator.clipboard.writeText(voteLink);
    await update(ref(db), { [`polls/${pollId}/shares`]: increment(1) });
    copied = true;
    setTimeout(() => copied = false, 2000);
  }

  function getWinner() {
    if (!poll || totalVotes === 0) return null;
    let maxVotes = 0;
    let winners = [];
    
    poll.options.forEach((option, index) => {
      if (option.votes > maxVotes) {
        maxVotes = option.votes;
        winners = [index];
      } else if (option.votes === maxVotes) {
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
      <div class="loading">
        <div class="spinner"></div>
        <p>Loading results...</p>
      </div>
    </div>
  {:else if !poll}
    <div class="container">
      <div class="error">
        <h1>Poll Not Found</h1>
        <p>This poll doesn't exist or may have been deleted.</p>
        <a href="/" class="btn-primary">Create Your Own Poll</a>
      </div>
    </div>
  {:else}
    <div class="container">
      {#if poll.imageUrl}
        <div class="poll-image">
          <img src={poll.imageUrl} alt="Poll illustration" />
        </div>
      {/if}
      <h1>{poll.question}</h1>
      <div class="poll-stats-bar">
        <div class="stat-item">
          <span class="stat-label">Views</span>
          <span class="stat-value">{poll.views || 0}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Shares</span>
          <span class="stat-value">{poll.shares || 0}</span>
        </div>
      </div>

      <div class="results">
        {#each poll.options as option, i}
          {@const votes = option.votes || 0}
          {@const percentage = totalVotes > 0 ? (votes / totalVotes * 100).toFixed(1) : 0}
          {@const isWinner = getWinner() === i}
          
          <div class="result-item" class:winner={isWinner}>
            <div class="result-header">
              <span class="option-text">
                {#if isWinner && totalVotes > 0}
                  <span class="trophy"></span>
                {/if}
                {option.text}
              </span>
              <span class="votes-info">
                <span class="vote-count">{votes}</span>
                <span class="percentage">{percentage}%</span>
              </span>
            </div>
            <div class="bar-container">
              <div 
                class="bar" 
                class:winner-bar={isWinner}
                style="width: {percentage}%"
              >
              </div>
            </div>
          </div>
        {/each}
      </div>

      {#if showQR}
        <div class="qr-section">
          <p class="qr-label">Scan to vote</p>
          <img class="qr-code" src={getQrUrl()} alt="QR Code for poll" />
          <p class="qr-url">{getPollUrl()}</p>
        </div>
      {/if}

      <div class="actions">
        <button class="btn-primary" class:copied on:click={copyVoteLink}>
          {copied ? 'Link Copied!' : ' Share Poll Link'}
        </button>
        <button class="btn-secondary qr-btn" on:click={() => showQR = !showQR}>
          {showQR ? 'Hide QR' : 'QR Code'}
        </button>
        <a href="/poll/{pollId}" class="btn-secondary">
          ← Back to Vote
        </a>
        <a href="/" class="btn-secondary">
          Create New Poll
        </a>
      </div>
    </div>
  {/if}
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
  }

  main {
    padding: 2rem 1rem;
    min-height: 70vh;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .container {
    max-width: 420px;
    width: 100%;
    background: rgba(255, 255, 255, 0.12);
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    padding: 2rem 1.5rem;
    border-radius: 20px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.15);
  }

  .loading, .error {
    text-align: center;
    padding: 2rem 0;
  }

  .loading p {
    color: white;
    font-weight: 500;
    margin-top: 1rem;
  }

  .spinner {
    border: 4px solid rgba(255, 255, 255, 0.2);
    border-top: 4px solid white;
    border-radius: 50%;
    width: 50px;
    height: 50px;
    animation: spin 0.8s linear infinite;
    margin: 0 auto;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }

  h1 {
    margin: 0 0 1.2rem 0;
    color: white;
    font-size: 1.4rem;
    text-align: center;
    font-weight: 700;
    line-height: 1.3;
  }

  .total-votes {
    display: none;
  }

  .votes-count {
    display: none;
  }

  .votes-label {
    display: none;
  }

  .live-indicator {
    display: none;
  }

  .results {
    display: flex;
    flex-direction: column;
    gap: 0.8rem;
    margin-bottom: 1.5rem;
  }

  .result-item {
    background: rgba(255, 255, 255, 0.1);
    padding: 1rem;
    border-radius: 12px;
    transition: background 0.3s ease;
    border: 1px solid rgba(255, 255, 255, 0.1);
  }

  .result-item.winner {
    background: rgba(251, 191, 36, 0.2);
    border: 1px solid rgba(251, 191, 36, 0.4);
  }

  .result-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 0.6rem;
    gap: 0.8rem;
  }

  .option-text {
    font-weight: 600;
    color: white;
    font-size: 0.95rem;
    flex: 1;
  }

  .trophy {
    margin-right: 0.5rem;
    font-size: 1rem;
  }

  .votes-info {
    display: flex;
    gap: 0.8rem;
    align-items: center;
    white-space: nowrap;
  }

  .vote-count {
    color: rgba(255, 255, 255, 0.8);
    font-weight: 600;
    font-size: 0.9rem;
  }

  .percentage {
    color: white;
    font-size: 0.95rem;
    font-weight: 700;
    min-width: 35px;
    text-align: right;
  }

  .bar-container {
    background: rgba(0, 0, 0, 0.2);
    height: 28px;
    border-radius: 6px;
    overflow: hidden;
  }

  .bar {
    background: rgba(255, 255, 255, 0.4);
    height: 100%;
    transition: width 0.6s ease-out;
  }

  .bar.winner-bar {
    background: rgba(251, 191, 36, 0.7);
  }

  .actions {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0.8rem;
  }

  .btn-primary {
    padding: 0.75rem 1.25rem;
    background: rgba(255, 255, 255, 0.2);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.3);
    border-radius: 12px;
    cursor: pointer;
    font-weight: 600;
    text-decoration: none;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s ease;
    font-size: 0.85rem;
    grid-column: 1 / -1;
  }

  .btn-primary:hover {
    background: rgba(255, 255, 255, 0.3);
    transform: translateY(-1px);
  }

  .btn-secondary {
    padding: 0.75rem 1.25rem;
    background: rgba(255, 255, 255, 0.1);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 12px;
    cursor: pointer;
    text-decoration: none;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s ease;
    font-size: 0.8rem;
    font-weight: 500;
  }

  .btn-secondary:hover {
    background: rgba(255, 255, 255, 0.15);
    transform: translateY(-1px);
  }

  .error {
    text-align: center;
  }

  .error h1 {
    margin-bottom: 1rem;
    color: white;
  }

  .error p {
    color: rgba(255, 255, 255, 0.85);
    margin-bottom: 2rem;
  }

  @media (max-width: 600px) {
    .container {
      padding: 2rem 1.5rem;
    }

    h1 {
      font-size: 1.4rem;
    }

    .result-header {
      flex-direction: column;
      align-items: flex-start;
    }

    .votes-info {
      width: 100%;
      justify-content: space-between;
    }

    .actions {
      grid-template-columns: 1fr;
    }

    .btn-primary {
      grid-column: auto;
    }
  }

  .poll-image {
    border-radius: 12px;
    overflow: hidden;
    margin-bottom: 1rem;
  }

  .poll-image img {
    width: 100%;
    max-height: 180px;
    object-fit: cover;
    display: block;
  }

  .poll-stats-bar {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0.8rem;
    margin: 1rem 0 1.5rem;
    padding: 1rem;
    background: rgba(255, 255, 255, 0.08);
    border-radius: 12px;
    border: 1px solid rgba(255, 255, 255, 0.1);
  }

  .stat-item {
    text-align: center;
  }

  .stat-label {
    display: block;
    font-size: 0.7rem;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: rgba(255, 255, 255, 0.5);
    margin-bottom: 0.35rem;
    font-weight: 600;
  }

  .stat-value {
    display: block;
    font-size: 1.6rem;
    font-weight: 700;
    color: white;
  }

  .qr-section {
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.15);
    border-radius: 12px;
    padding: 1.2rem 1rem;
    text-align: center;
    margin-bottom: 1rem;
    animation: fadeIn 0.25s ease-out;
  }

  @keyframes fadeIn {
    from { opacity: 0; transform: translateY(-6px); }
    to { opacity: 1; transform: translateY(0); }
  }

  .qr-label {
    color: rgba(255, 255, 255, 0.75);
    font-size: 0.75rem;
    margin: 0 0 0.75rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.07em;
  }

  .qr-code {
    border-radius: 10px;
    display: block;
    margin: 0 auto;
    width: 150px;
    height: 150px;
  }

  .qr-url {
    color: rgba(255, 255, 255, 0.45);
    font-size: 0.68rem;
    margin: 0.6rem 0 0;
    word-break: break-all;
  }

  .btn-primary.copied {
    background: rgba(16, 185, 129, 0.3);
    border-color: rgba(16, 185, 129, 0.5);
  }

  .qr-btn {
    grid-column: 1 / -1;
  }
</style>