<script>
  import { page } from '$app/stores';
  import { db } from '$lib/firebase';
  import { ref, onValue, update } from 'firebase/database';
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';

  let poll = null;
  let voted = false;
  let pollId = $page.params.id;
  let loading = true;

  onMount(() => {
    const pollRef = ref(db, `polls/${pollId}`);
    
    onValue(pollRef, (snapshot) => {
      if (snapshot.exists()) {
        poll = snapshot.val();
        loading = false;
      } else {
        loading = false;
      }
    });

    // Check if already voted
    voted = localStorage.getItem(`voted_${pollId}`) === 'true';
  });

  async function vote(optionIndex) {
    if (voted) return;

    const updates = {};
    updates[`polls/${pollId}/options/${optionIndex}/votes`] = 
      (poll.options[optionIndex].votes || 0) + 1;

    await update(ref(db), updates);
    localStorage.setItem(`voted_${pollId}`, 'true');
    voted = true;
    
    // Redirect to results after voting
    setTimeout(() => goto(`/results/${pollId}`), 800);
  }

  function copyLink() {
    navigator.clipboard.writeText(window.location.href);
    alert('Link copied to clipboard!');
  }
</script>

<svelte:head>
  <title>{poll ? poll.question : 'Loading Poll...'}</title>
</svelte:head>

<main>
  {#if loading}
    <div class="container">
      <div class="loading">
        <div class="spinner"></div>
        <p>Loading poll...</p>
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
      <h1>{poll.question}</h1>
      
      {#if !voted}
        <p class="instruction">Choose your answer:</p>
        <div class="options">
          {#each poll.options as option, i}
            <button class="option" on:click={() => vote(i)}>
              <span class="option-text">{option.text}</span>
              <span class="arrow">→</span>
            </button>
          {/each}
        </div>
      {:else}
        <div class="voted">
          <div class="check">✓</div>
          <p>Vote recorded!</p>
          <p class="sub">Redirecting to results...</p>
        </div>
      {/if}

      <div class="actions">
        <button class="btn-secondary" on:click={copyLink}>
          Copy Poll Link
        </button>
        <a href="/results/{pollId}" class="btn-secondary">
          View Results
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
    min-height: 100vh;
  }

  .container {
    max-width: 420px;
    margin: 0 auto;
    background: rgba(255, 255, 255, 0.15);
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    padding: 2.5rem;
    border-radius: 24px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.18);
  }

  .loading, .error {
    text-align: center;
    padding: 2rem 0;
  }

  .loading p {
    color: white;
    font-weight: 500;
  }

  .spinner {
    border: 4px solid rgba(255, 255, 255, 0.2);
    border-top: 4px solid white;
    border-radius: 50%;
    width: 50px;
    height: 50px;
    animation: spin 1s linear infinite;
    margin: 0 auto 1rem;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }

  h1 {
    margin: 0 0 0.5rem 0;
    color: white;
    font-size: 2rem;
    text-align: center;
    font-weight: 700;
    text-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
  }

  .instruction {
    text-align: center;
    color: rgba(255, 255, 255, 0.9);
    margin: 1rem 0 2rem;
    font-size: 1.1rem;
    text-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
  }

  .options {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    margin: 2rem 0;
  }

  .option {
    padding: 1.5rem;
    background: rgba(255, 255, 255, 0.2);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.3);
    border-radius: 16px;
    cursor: pointer;
    font-size: 1.1rem;
    transition: all 0.3s;
    text-align: left;
    display: flex;
    justify-content: space-between;
    align-items: center;
    color: white;
    font-weight: 500;
  }

  .option:hover {
    background: rgba(255, 255, 255, 0.3);
    border-color: rgba(255, 255, 255, 0.5);
    transform: translateY(-3px);
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15);
  }

  .option-text {
    flex: 1;
  }

  .arrow {
    opacity: 0;
    transition: opacity 0.2s;
    color: white;
    font-weight: bold;
    font-size: 1.5rem;
  }

  .option:hover .arrow {
    opacity: 1;
  }

  .voted {
    text-align: center;
    padding: 3rem 0;
  }

  .check {
    width: 80px;
    height: 80px;
    background: #10b981;
    color: white;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 3rem;
    margin: 0 auto 1rem;
    animation: scaleIn 0.3s ease-out;
  }

  @keyframes scaleIn {
    0% { transform: scale(0); }
    50% { transform: scale(1.1); }
    100% { transform: scale(1); }
  }

  .voted p {
    margin: 0.5rem 0;
    font-size: 1.5rem;
    color: white;
    font-weight: 600;
  }

  .voted .sub {
    color: rgba(255, 255, 255, 0.8);
    font-size: 1rem;
  }

  .actions {
    display: flex;
    gap: 1rem;
    justify-content: center;
    margin-top: 2rem;
    flex-wrap: wrap;
  }

  .btn-secondary {
    padding: 0.75rem 1.5rem;
    background: rgba(255, 255, 255, 0.2);
    backdrop-filter: blur(10px);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.3);
    border-radius: 12px;
    cursor: pointer;
    text-decoration: none;
    display: inline-block;
    transition: all 0.2s;
    font-size: 0.95rem;
    font-weight: 500;
  }

  .btn-secondary:hover {
    background: rgba(255, 255, 255, 0.3);
    border-color: rgba(255, 255, 255, 0.5);
    transform: translateY(-2px);
  }

  .btn-primary {
    padding: 1rem 2rem;
    background: rgba(255, 255, 255, 0.3);
    backdrop-filter: blur(10px);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.4);
    border-radius: 12px;
    cursor: pointer;
    text-decoration: none;
    display: inline-block;
    font-weight: 600;
    transition: all 0.2s;
  }

  .btn-primary:hover {
    background: rgba(255, 255, 255, 0.4);
    transform: translateY(-2px);
  }
</style>