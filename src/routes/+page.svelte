<script>
  import { db } from '$lib/firebase';
  import { ref, push, set } from 'firebase/database';
  import { goto } from '$app/navigation';
  import { onMount } from 'svelte';

  const themes = {
    violet: { bg: 'linear-gradient(150deg, #2e1065 0%, #4c1d95 30%, #6d28d9 65%, #8b5cf6 100%)', label: 'Violet' },
    ocean:  { bg: 'linear-gradient(150deg, #0c2340 0%, #0c4a6e 30%, #0369a1 65%, #38bdf8 100%)', label: 'Ocean' },
    sunset: { bg: 'linear-gradient(150deg, #450a0a 0%, #991b1b 30%, #dc2626 65%, #fb923c 100%)', label: 'Sunset' },
    slate:  { bg: 'linear-gradient(150deg, #0f172a 0%, #1e293b 30%, #334155 65%, #64748b 100%)', label: 'Slate' },
  };

  let question = '';
  let options = ['', ''];
  let creating = false;
  let imagePreview = null;
  let imageError = '';
  let selectedTheme = 'violet';

  onMount(() => {
    applyTheme(selectedTheme);
  });

  function applyTheme(key) {
    document.body.style.background = themes[key].bg;
  }

  function selectTheme(key) {
    selectedTheme = key;
    applyTheme(key);
  }

  function addOption() {
    options = [...options, ''];
  }

  function removeOption(index) {
    options = options.filter((_, i) => i !== index);
  }

  function handleImageUpload(e) {
    const file = e.target.files[0];
    if (!file) return;
    if (file.size > 500 * 1024) {
      imageError = 'Image must be under 500 KB';
      return;
    }
    imageError = '';
    const reader = new FileReader();
    reader.onload = (ev) => { imagePreview = ev.target.result; };
    reader.readAsDataURL(file);
  }

  function removeImage() {
    imagePreview = null;
    imageError = '';
  }

  async function createPoll() {
    if (!question.trim() || options.filter(o => o.trim()).length < 2) {
      alert('Please add a question and at least 2 options.');
      return;
    }
    creating = true;
    const pollsRef = ref(db, 'polls');
    const newPollRef = push(pollsRef);
    await set(newPollRef, {
      question: question.trim(),
      options: options.filter(o => o.trim()).map(text => ({ text: text.trim(), votes: 0 })),
      createdAt: Date.now(),
      views: 0,
      shares: 0,
      theme: selectedTheme,
      ...(imagePreview ? { imageUrl: imagePreview } : {})
    });
    goto(`/poll/${newPollRef.key}`);
  }
</script>

<svelte:head>
  <title>Create Poll</title>
</svelte:head>

<main>
  <div class="container">
    <div class="header">
      <h1>New Poll</h1>
      <p class="subtitle">Create and share in seconds</p>
    </div>

    <form on:submit|preventDefault={createPoll}>

      <div class="form-group">
        <label for="question">Question</label>
        <input
          id="question"
          type="text"
          bind:value={question}
          placeholder="Ask something..."
          autocomplete="off"
          required
        />
      </div>

      <div class="form-group">
        <p class="field-label">Options</p>
        {#each options as _, i}
          <div class="option-row">
            <span class="option-index">{i + 1}</span>
            <input
              type="text"
              bind:value={options[i]}
              placeholder="Option {i + 1}"
              autocomplete="off"
              required
            />
            {#if options.length > 2}
              <button
                type="button"
                class="remove-btn"
                on:click={() => removeOption(i)}
                aria-label="Remove option"
              >
                <svg width="12" height="12" viewBox="0 0 12 12" fill="none" class="remove-icon">
                  <path d="M1 1L11 11M11 1L1 11" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
                </svg>
              </button>
            {/if}
          </div>
        {/each}
        <button type="button" class="add-btn" on:click={addOption}>
          <svg width="13" height="13" viewBox="0 0 13 13" fill="none" class="add-icon">
            <path d="M6.5 1V12M1 6.5H12" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
          </svg>
          Add option
        </button>
      </div>

      <div class="form-row">
        <div class="form-group theme-group">
          <p class="field-label">Theme</p>
          <div class="theme-swatches">
            {#each Object.entries(themes) as [key, t]}
              <button
                type="button"
                class="swatch"
                class:active={selectedTheme === key}
                style="background: {t.bg}"
                title={t.label}
                on:click={() => selectTheme(key)}
                aria-label={t.label}
              ></button>
            {/each}
          </div>
        </div>

        <div class="form-group image-group">
          <p class="field-label">Image <span class="optional">optional</span></p>
          {#if imagePreview}
            <div class="image-preview">
              <img src={imagePreview} alt="Preview" />
              <button type="button" class="remove-image-btn" on:click={removeImage} aria-label="Remove image">
                <svg width="11" height="11" viewBox="0 0 11 11" fill="none">
                  <path d="M1 1L10 10M10 1L1 10" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
                </svg>
              </button>
            </div>
          {:else}
            <label class="image-upload-area">
              <input type="file" accept="image/*" on:change={handleImageUpload} />
              <svg width="18" height="18" viewBox="0 0 24 24" fill="none" class="upload-icon">
                <rect x="3" y="3" width="18" height="18" rx="2" stroke="currentColor" stroke-width="1.5"/>
                <path d="M3 16l5-5 4 4 3-4 6 5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
                <circle cx="8.5" cy="8.5" r="1.5" fill="currentColor"/>
              </svg>
              <span>Upload image</span>
              <span class="upload-hint">Max 500 KB</span>
            </label>
          {/if}
          {#if imageError}
            <p class="image-error">{imageError}</p>
          {/if}
        </div>
      </div>

      <button type="submit" class="create-btn" disabled={creating}>
        {creating ? 'Creating poll...' : 'Create Poll'}
      </button>
    </form>
  </div>
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
    padding: 2.5rem 1rem 3rem;
    min-height: 100vh;
    display: flex;
    align-items: flex-start;
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
    padding: 2rem 2rem 2.5rem;
    box-shadow:
      0 4px 6px rgba(0, 0, 0, 0.07),
      0 20px 48px rgba(0, 0, 0, 0.2),
      inset 0 1px 0 rgba(255, 255, 255, 0.22);
  }

  .header {
    text-align: center;
    margin-bottom: 1.75rem;
  }

  h1 {
    margin: 0 0 0.25rem;
    color: white;
    font-size: 1.7rem;
    font-weight: 700;
    letter-spacing: -0.02em;
  }

  .subtitle {
    margin: 0;
    color: rgba(255, 255, 255, 0.55);
    font-size: 0.875rem;
  }

  .form-group {
    margin-bottom: 1.25rem;
  }

  label {
    display: block;
    font-size: 0.75rem;
    font-weight: 600;
    color: rgba(255, 255, 255, 0.65);
    text-transform: uppercase;
    letter-spacing: 0.07em;
    margin-bottom: 0.5rem;
  }

  .field-label {
    display: block;
    font-size: 0.75rem;
    font-weight: 600;
    color: rgba(255, 255, 255, 0.65);
    text-transform: uppercase;
    letter-spacing: 0.07em;
    margin: 0 0 0.5rem;
    padding: 0;
  }

  .optional {
    font-weight: 400;
    text-transform: none;
    letter-spacing: 0;
    font-size: 0.72rem;
    opacity: 0.6;
  }

  input[type="text"] {
    width: 100%;
    padding: 0.7rem 0.875rem;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.18);
    border-radius: 10px;
    font-size: 0.9rem;
    color: white;
    transition: border-color 0.15s, background 0.15s, box-shadow 0.15s;
    box-sizing: border-box;
    font-family: inherit;
  }

  input[type="text"]::placeholder {
    color: rgba(255, 255, 255, 0.3);
  }

  input[type="text"]:focus {
    outline: none;
    background: rgba(255, 255, 255, 0.16);
    border-color: rgba(255, 255, 255, 0.4);
    box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.07);
  }

  .option-row {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-bottom: 0.45rem;
  }

  .option-index {
    width: 20px;
    flex-shrink: 0;
    font-size: 0.72rem;
    font-weight: 600;
    color: rgba(255, 255, 255, 0.35);
    text-align: right;
  }

  .remove-btn {
    width: 30px;
    height: 30px;
    min-width: 30px;
    background: rgba(239, 68, 68, 0.08);
    border: 1px solid rgba(239, 68, 68, 0.18);
    border-radius: 8px;
    color: rgba(252, 165, 165, 0.55);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background 0.15s, border-color 0.15s, color 0.15s;
  }

  .remove-btn:hover {
    background: rgba(239, 68, 68, 0.22);
    border-color: rgba(239, 68, 68, 0.4);
    color: #fca5a5;
  }

  .add-btn {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-top: 0.35rem;
    padding: 0.55rem 0.875rem;
    background: transparent;
    border: 1px dashed rgba(110, 231, 183, 0.28);
    border-radius: 10px;
    color: rgba(255, 255, 255, 0.45);
    font-size: 0.82rem;
    font-family: inherit;
    cursor: pointer;
    transition: border-color 0.15s, color 0.15s, background 0.15s;
    width: 100%;
    box-sizing: border-box;
  }

  .add-btn:hover {
    border-color: rgba(110, 231, 183, 0.5);
    color: rgba(255, 255, 255, 0.85);
    background: rgba(110, 231, 183, 0.06);
  }

  .add-icon {
    color: rgba(110, 231, 183, 0.7);
    flex-shrink: 0;
  }

  .add-btn:hover .add-icon {
    color: #6ee7b7;
  }

  .upload-icon {
    color: rgba(147, 197, 253, 0.7);
  }

  .image-upload-area:hover .upload-icon {
    color: #93c5fd;
  }

  .form-row {
    display: grid;
    grid-template-columns: auto 1fr;
    gap: 1rem;
    align-items: start;
    margin-bottom: 1.25rem;
  }

  .theme-group,
  .image-group {
    margin-bottom: 0;
  }

  .theme-swatches {
    display: flex;
    gap: 0.5rem;
    align-items: center;
  }

  .swatch {
    width: 28px;
    height: 28px;
    border-radius: 50%;
    border: 2px solid transparent;
    cursor: pointer;
    transition: transform 0.15s, box-shadow 0.15s, border-color 0.15s;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.35);
    flex-shrink: 0;
    padding: 0;
  }

  .swatch:hover {
    transform: scale(1.18);
    box-shadow: 0 3px 10px rgba(0, 0, 0, 0.45);
  }

  .swatch.active {
    border-color: rgba(255, 255, 255, 0.9);
    box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.25), 0 3px 10px rgba(0, 0, 0, 0.4);
    transform: scale(1.12);
  }

  .image-upload-area {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 0.25rem;
    height: 72px;
    border: 1px dashed rgba(255, 255, 255, 0.22);
    border-radius: 10px;
    cursor: pointer;
    color: rgba(255, 255, 255, 0.4);
    font-size: 0.76rem;
    text-align: center;
    transition: border-color 0.15s, background 0.15s, color 0.15s;
    font-weight: normal;
    box-sizing: border-box;
  }

  .image-upload-area:hover {
    border-color: rgba(255, 255, 255, 0.45);
    background: rgba(255, 255, 255, 0.05);
    color: rgba(255, 255, 255, 0.7);
  }

  .image-upload-area input[type="file"] {
    display: none;
  }

  .upload-hint {
    font-size: 0.68rem;
    opacity: 0.5;
  }

  .image-preview {
    position: relative;
    border-radius: 10px;
    overflow: hidden;
    height: 72px;
  }

  .image-preview img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }

  .remove-image-btn {
    position: absolute;
    top: 0.35rem;
    right: 0.35rem;
    background: rgba(0, 0, 0, 0.5);
    backdrop-filter: blur(4px);
    color: white;
    border: none;
    border-radius: 6px;
    width: 24px;
    height: 24px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background 0.15s;
  }

  .remove-image-btn:hover {
    background: rgba(220, 38, 38, 0.65);
  }

  .image-error {
    color: #fca5a5;
    font-size: 0.76rem;
    margin: 0.3rem 0 0;
  }

  .create-btn {
    width: 100%;
    padding: 0.8rem;
    background: rgba(255, 255, 255, 0.18);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.3);
    border-radius: 12px;
    color: white;
    font-size: 0.9rem;
    font-weight: 600;
    font-family: inherit;
    cursor: pointer;
    letter-spacing: 0.01em;
    transition: background 0.2s, transform 0.15s, box-shadow 0.2s;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15), inset 0 1px 0 rgba(255, 255, 255, 0.18);
  }

  .create-btn:hover:not(:disabled) {
    background: rgba(255, 255, 255, 0.26);
    transform: translateY(-1px);
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.22), inset 0 1px 0 rgba(255, 255, 255, 0.22);
  }

  .create-btn:active:not(:disabled) {
    transform: translateY(0);
  }

  .create-btn:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
</style>