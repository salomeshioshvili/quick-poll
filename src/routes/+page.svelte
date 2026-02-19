<script>
  import { db } from '$lib/firebase';
  import { ref, push, set } from 'firebase/database';
  import { goto } from '$app/navigation';

  let question = '';
  let options = ['', ''];
  let creating = false;
  let imagePreview = null;
  let imageError = '';

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
      imageError = 'Image must be under 500KB';
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
    if (!question || options.filter(o => o.trim()).length < 2) {
      alert('Need a question and at least 2 options!');
      return;
    }

    creating = true;
    const pollsRef = ref(db, 'polls');
    const newPollRef = push(pollsRef);
    
    await set(newPollRef, {
      question,
      options: options.filter(o => o.trim()).map(text => ({
        text,
        votes: 0
      })),
      createdAt: Date.now(),
      views: 0,
      shares: 0,
      ...(imagePreview ? { imageUrl: imagePreview } : {})
    });

    goto(`/poll/${newPollRef.key}`);
  }
</script>

<svelte:head>
  <title>Create Quick Poll</title>
</svelte:head>

<main>
  <div class="container">
    <h1>Quick Poll Generator</h1>
    <p class="subtitle">Create instant polls with real-time results</p>
    
    <form on:submit|preventDefault={createPoll}>
      <div class="form-group">
        <label for="question">Your Question</label>
        <input 
          id="question"
          type="text" 
          bind:value={question} 
          placeholder="What's your favorite programming language?" 
          required 
        />
      </div>

      <div class="form-group">
        <label>Poll Options</label>
        {#each options as option, i}
          <div class="option-row">
            <input 
              type="text" 
              bind:value={options[i]} 
              placeholder="Option {i + 1}" 
              required 
            />
            {#if options.length > 2}
              <button 
                type="button" 
                class="remove-btn" 
                on:click={() => removeOption(i)}
                title="Remove option"
              >
                ×
              </button>
            {/if}
          </div>
        {/each}
        <button type="button" class="add-btn" on:click={addOption}>
          + Add Another Option
        </button>
      </div>

      <div class="form-group">
        <label>Question Image <span class="optional">(optional)</span></label>
        {#if imagePreview}
          <div class="image-preview">
            <img src={imagePreview} alt="Preview" />
            <button type="button" class="remove-image-btn" on:click={removeImage}>✕ Remove</button>
          </div>
        {:else}
          <label class="image-upload-area">
            <input type="file" accept="image/*" on:change={handleImageUpload} />
            <span>Click to upload image</span>
            <span class="upload-hint">PNG, JPG, GIF up to 500KB</span>
          </label>
        {/if}
        {#if imageError}
          <p class="image-error">{imageError}</p>
        {/if}
      </div>

      <button type="submit" class="create-btn" disabled={creating}>
        {creating ? ' Creating...' : ' Create Poll'}
      </button>
    </form>
  </div>
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

  h1 {
    margin: 0 0 0.5rem 0;
    color: white;
    font-size: 2rem;
    text-align: center;
    font-weight: 700;
    text-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
  }

  .subtitle {
    text-align: center;
    color: rgba(255, 255, 255, 0.9);
    margin: 0 0 2rem 0;
    text-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
  }

  .form-group {
    margin-bottom: 1.5rem;
  }

  label {
    display: block;
    font-weight: 600;
    color: white;
    margin-bottom: 0.5rem;
    text-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
  }

  input[type="text"] {
    width: 100%;
    padding: 0.75rem;
    background: rgba(255, 255, 255, 0.2);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.3);
    border-radius: 12px;
    font-size: 1rem;
    transition: all 0.2s;
    box-sizing: border-box;
    color: white;
  }

  input[type="text"]::placeholder {
    color: rgba(255, 255, 255, 0.6);
  }

  input[type="text"]:focus {
    outline: none;
    background: rgba(255, 255, 255, 0.25);
    border-color: rgba(255, 255, 255, 0.5);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  }

  .option-row {
    display: flex;
    gap: 0.5rem;
    align-items: center;
    margin-bottom: 0.75rem;
  }

  .remove-btn {
    background: #ef4444;
    color: white;
    border: none;
    width: 36px;
    height: 36px;
    border-radius: 50%;
    cursor: pointer;
    font-size: 1.5rem;
    line-height: 1;
    transition: background 0.2s;
    flex-shrink: 0;
  }

  .remove-btn:hover {
    background: #dc2626;
  }

  .add-btn {
    background: rgba(255, 255, 255, 0.15);
    color: white;
    padding: 0.75rem 1rem;
    border: 1px dashed rgba(255, 255, 255, 0.4);
    border-radius: 12px;
    cursor: pointer;
    font-size: 0.95rem;
    width: 100%;
    transition: all 0.2s;
    backdrop-filter: blur(10px);
  }

  .add-btn:hover {
    background: rgba(255, 255, 255, 0.25);
    border-color: rgba(255, 255, 255, 0.6);
  }

  .create-btn {
    background: rgba(255, 255, 255, 0.3);
    backdrop-filter: blur(10px);
    color: white;
    padding: 1rem 2rem;
    border: 1px solid rgba(255, 255, 255, 0.4);
    border-radius: 16px;
    cursor: pointer;
    font-size: 1.1rem;
    font-weight: 600;
    width: 100%;
    margin-top: 1rem;
    transition: all 0.2s;
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.1);
  }

  .create-btn:hover:not(:disabled) {
    background: rgba(255, 255, 255, 0.4);
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(0, 0, 0, 0.15);
  }

  .create-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .optional {
    font-weight: 400;
    font-size: 0.8rem;
    opacity: 0.7;
  }

  .image-upload-area {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.4rem;
    padding: 1.5rem;
    border: 2px dashed rgba(255, 255, 255, 0.4);
    border-radius: 12px;
    cursor: pointer;
    transition: all 0.2s;
    text-align: center;
    color: rgba(255, 255, 255, 0.8);
    font-size: 0.9rem;
    font-weight: normal;
  }

  .image-upload-area:hover {
    border-color: rgba(255, 255, 255, 0.7);
    background: rgba(255, 255, 255, 0.1);
  }

  .image-upload-area input[type="file"] {
    display: none;
  }


  .upload-hint {
    font-size: 0.75rem;
    opacity: 0.6;
  }

  .image-preview {
    position: relative;
    border-radius: 12px;
    overflow: hidden;
  }

  .image-preview img {
    width: 100%;
    max-height: 200px;
    object-fit: cover;
    display: block;
    border-radius: 12px;
  }

  .remove-image-btn {
    position: absolute;
    top: 0.5rem;
    right: 0.5rem;
    background: rgba(0, 0, 0, 0.6);
    color: white;
    border: none;
    border-radius: 8px;
    padding: 0.3rem 0.6rem;
    cursor: pointer;
    font-size: 0.8rem;
    transition: background 0.2s;
  }

  .remove-image-btn:hover {
    background: rgba(239, 68, 68, 0.8);
  }

  .image-error {
    color: #fca5a5;
    font-size: 0.85rem;
    margin: 0.4rem 0 0;
  }
</style>