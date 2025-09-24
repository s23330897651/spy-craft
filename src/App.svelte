<script lang="ts">
	interface CompanyData {
		companyName: string;
		companyWebsite: string;
	}

	interface ResearchEntry {
		id: string;
		companyName: string;
		category: string;
		title: string;
		summary: string;
		url: string;
	}

	let activeTab: 'research' | 'summaries' = 'research';
	let formData: CompanyData = {
		companyName: '',
		companyWebsite: ''
	};
	let summaries: ResearchEntry[] = [];
	let isLoading = false;
	let error = '';

	// Use your n8n production URL (workflow is active)
	const webhookUrl = 'https://n8n.intelligentresourcing.co/webhook/d95c8e70-5cb1-4323-838b-8a910fbecf65';

	let __uid = 0;
	function makeId(suffix: string): string {
		__uid++;
		return `${Date.now()}-${__uid}-${suffix}`;
	}

	async function submitResearch() {
	if (!formData.companyName.trim() || !formData.companyWebsite.trim()) {
		error = 'Company name and website are required';
		return;
	}

	isLoading = true;
	error = '';

	try {
		const payload = {
			companyName: formData.companyName,
			companyWebsite: formData.companyWebsite
		};
		console.log('Submitting research to webhook:', webhookUrl, payload);

		const response = await fetch(webhookUrl, {
			method: 'POST',
			mode: 'cors',
			credentials: 'omit',
			headers: {
				'Content-Type': 'application/json',
				'Accept': 'application/json'
			},
			body: JSON.stringify(payload)
		});

		console.log('Webhook response status:', response.status, response.statusText);

		const rawText = await response.text();
		console.log('Webhook raw body length:', rawText.length);
		console.log('Webhook raw body (preview 1k):', rawText.slice(0, 1000));

		if (!response.ok) {
			throw new Error(`HTTP ${response.status} ${response.statusText}${rawText ? ` - ${rawText}` : ''}`);
		}

		let data: unknown = null;
		try {
			data = JSON.parse(rawText);
		} catch {
			console.warn('Response was not valid JSON.');
			data = null;
		}

		console.log('Parsed payload type:', Array.isArray(data) ? 'array' : typeof data);

		const newEntries: ResearchEntry[] = [];
		if (Array.isArray(data)) {
			console.log('Array length:', data.length);
			data.forEach((item, idx) => {
				console.log(`Item[${idx}]`, item);

				if (!item || typeof item !== 'object' || Array.isArray(item)) {
					console.log(`Skipping non-object item[${idx}].`);
					return;
				}

				const rec = item as Record<string, unknown>;
				const title = typeof rec.title === 'string' ? rec.title : '';
				const summary = typeof rec.summary === 'string' ? rec.summary : '';
				const url = typeof rec.url === 'string' ? rec.url : '';
				const category = typeof rec.category === 'string' ? rec.category : 'Research';

				if (!title && !summary && !url) {
					console.log(`Skipping item[${idx}] due to missing title/summary/url.`, rec);
					return;
				}

				newEntries.push({
					id: makeId(category || 'item'),
					companyName: formData.companyName,
					category: category || 'Research',
					title,
					summary,
					url: url || '#'
				});
			});
		} else {
			console.warn('Expected an array payload; nothing to map.');
		}

		console.log('Mapped entries count:', newEntries.length);
		console.table(newEntries.map(e => ({ category: e.category, title: e.title, url: e.url })));

		if (newEntries.length === 0) {
			const placeholder: ResearchEntry = {
				id: makeId('no-results'),
				companyName: formData.companyName,
				category: 'Research',
				title: 'No articles found',
				summary: '',
				url: '#'
			};
			summaries = [placeholder, ...summaries];
		} else {
			summaries = [...newEntries, ...summaries];
		}

		formData = {
			companyName: '',
			companyWebsite: ''
		};
		activeTab = 'summaries';
	} catch (err) {
		console.error('Error submitting research:', err);
		const message = err instanceof Error ? err.message : 'Failed to submit research request';
		error = message.includes('Failed to fetch') ? 'Failed to fetch (likely CORS). See note below.' : message;
	} finally {
		isLoading = false;
	}
}

	function clearSummaries() {
		summaries = [];
	}
</script>

<main class="container">
  <header class="header">
    <h1 class="title">Company Research Portal</h1>
    <p class="subtitle">Powered by N8N Intelligence</p>
  </header>

  <nav class="tabs">
    <button 
      class="tab" 
      class:active={activeTab === 'research'}
      on:click={() => activeTab = 'research'}
    >
      Research Form
    </button>
    <button 
      class="tab" 
      class:active={activeTab === 'summaries'}
      on:click={() => activeTab = 'summaries'}
    >
      Research Summaries
      {#if summaries.length > 0}
        <span class="badge">{summaries.length}</span>
      {/if}
    </button>
  </nav>

  <div class="content">
    {#if activeTab === 'research'}
      <div class="research-section">
        <div class="form-card">
          <h2>Company Research Request</h2>
          
          {#if error}
            <div class="alert error">
              <svg class="alert-icon" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7 4a1 1 0 11-2 0 1 1 0 012 0zm-1-9a1 1 0 00-1 1v4a1 1 0 102 0V6a1 1 0 00-1-1z" clip-rule="evenodd" />
              </svg>
              {error}
            </div>
          {/if}

          <form on:submit|preventDefault={submitResearch} class="form">
            <div class="form-group">
              <label for="companyName" class="label">Company Name *</label>
              <input 
                id="companyName"
                type="text" 
                bind:value={formData.companyName}
                placeholder="Enter company name"
                class="input"
                disabled={isLoading}
                required
              />
            </div>

            <div class="form-group">
              <label for="companyWebsite" class="label">Company Website *</label>
              <input 
                id="companyWebsite"
                type="url"
                bind:value={formData.companyWebsite}
                placeholder="https://example.com"
                class="input"
                disabled={isLoading}
                required
              />
            </div>

            <button
              type="submit"
              class="submit-btn"
              disabled={isLoading || !formData.companyName.trim() || !formData.companyWebsite.trim()}
            >
              {#if isLoading}
                <svg class="spinner" viewBox="0 0 24 24">
                  <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" fill="none"/>
                  <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"/>
                </svg>
                Processing Research...
              {:else}
                Submit Research Request
              {/if}
            </button>
          </form>
        </div>
      </div>
    {:else if activeTab === 'summaries'}
      <div class="summaries-section">
        <div class="summaries-header">
          <h2>Research Summaries</h2>
          {#if summaries.length > 0}
            <button class="clear-btn" on:click={clearSummaries}>
              Clear All
            </button>
          {/if}
        </div>

        {#if summaries.length === 0}
          <div class="empty-state">
            <svg class="empty-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
            </svg>
            <h3>No research summaries yet</h3>
            <p>Submit a research request to see summaries here</p>
          </div>
        {:else}
          <div class="summaries-list">
            {#each summaries as entry (entry.id)}
              <div class="summary-card">
                <div class="summary-meta">
                  <span class="category-badge">{entry.category}</span>
                </div>

                <div class="summary-content">
                  <a class="summary-link" href={entry.url} target="_blank" rel="noopener noreferrer">
                    {entry.title}
                  </a>
                  <div class="summary-text">
                    {@html (entry.summary || '').replace(/\n/g, '<br>')}
                  </div>
                </div>
              </div>
            {/each}
          </div>
        {/if}
      </div>
    {/if}
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  }

  .container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 2rem;
    min-height: 100vh;
  }

  .header {
    text-align: center;
    margin-bottom: 3rem;
    color: white;
  }

  .title {
    font-size: 3rem;
    font-weight: 700;
    margin: 0;
    text-shadow: 0 2px 4px rgba(0,0,0,0.1);
  }

  .subtitle {
    font-size: 1.25rem;
    margin: 0.5rem 0 0 0;
    opacity: 0.9;
    font-weight: 400;
  }

  .tabs {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 2rem;
    background: rgba(255,255,255,0.1);
    border-radius: 12px;
    padding: 0.5rem;
    backdrop-filter: blur(10px);
  }

  .tab {
    flex: 1;
    background: transparent;
    border: none;
    color: white;
    padding: 1rem 2rem;
    border-radius: 8px;
    font-size: 1rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
  }

  .tab:hover {
    background: rgba(255,255,255,0.1);
  }

  .tab.active {
    background: white;
    color: #667eea;
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  }

  .badge {
    background: #667eea;
    color: white;
    border-radius: 50%;
    width: 24px;
    height: 24px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 0.75rem;
    font-weight: 600;
  }

  .tab.active .badge {
    background: #667eea;
    color: white;
  }

  .content {
    background: white;
    border-radius: 16px;
    box-shadow: 0 20px 40px rgba(0,0,0,0.1);
    overflow: hidden;
  }

  .research-section {
    padding: 3rem;
  }

  .form-card h2 {
    color: #374151;
    font-size: 1.875rem;
    font-weight: 600;
    margin: 0 0 2rem 0;
    text-align: center;
  }

  .alert {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    padding: 1rem;
    border-radius: 8px;
    margin-bottom: 1.5rem;
  }

  .alert.error {
    background: #fef2f2;
    color: #dc2626;
    border: 1px solid #fecaca;
  }

  .alert-icon {
    width: 20px;
    height: 20px;
    flex-shrink: 0;
  }

  .form {
    max-width: 500px;
    margin: 0 auto;
  }

  .form-group {
    margin-bottom: 1.5rem;
  }

  .label {
    display: block;
    font-weight: 500;
    color: #374151;
    margin-bottom: 0.5rem;
    font-size: 0.875rem;
  }

  .input {
    width: 100%;
    padding: 0.875rem 1rem;
    border: 2px solid #e5e7eb;
    border-radius: 8px;
    font-size: 1rem;
    transition: all 0.2s ease;
    background: white;
    color: #000;
  }

  .input:focus {
    outline: none;
    border-color: #667eea;
    box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
  }

  .input:disabled {
    background: #f9fafb;
    opacity: 0.6;
    cursor: not-allowed;
    color: #000;
  }

  .submit-btn {
    width: 100%;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border: none;
    padding: 1rem 2rem;
    border-radius: 8px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
    margin-top: 1rem;
  }

  .submit-btn:hover:not(:disabled) {
    transform: translateY(-2px);
    box-shadow: 0 8px 20px rgba(102, 126, 234, 0.3);
  }

  .submit-btn:disabled {
    opacity: 0.7;
    cursor: not-allowed;
    transform: none;
  }

  .spinner {
    width: 20px;
    height: 20px;
    animation: spin 1s linear infinite;
  }

  @keyframes spin {
    from { transform: rotate(0deg); }
    to { transform: rotate(360deg); }
  }

  .summaries-section {
    padding: 3rem;
  }

  .summaries-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
  }

  .summaries-header h2 {
    color: #374151;
    font-size: 1.875rem;
    font-weight: 600;
    margin: 0;
  }

  .clear-btn {
    background: #ef4444;
    color: white;
    border: none;
    padding: 0.5rem 1rem;
    border-radius: 6px;
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.2s ease;
  }

  .clear-btn:hover {
    background: #dc2626;
  }

  .empty-state {
    text-align: center;
    padding: 4rem 2rem;
    color: #6b7280;
  }

  .empty-icon {
    width: 64px;
    height: 64px;
    margin: 0 auto 1rem;
    opacity: 0.5;
  }

  .empty-state h3 {
    font-size: 1.25rem;
    font-weight: 600;
    margin: 0 0 0.5rem 0;
    color: #374151;
  }

  .empty-state p {
    margin: 0;
    font-size: 1rem;
  }

  .summaries-list {
    display: flex,;
    flex-direction: column;
    gap: 1.5rem;
  }

  .summary-card {
    background: #f9fafb;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    padding: 1.5rem;
    transition: all 0.2s ease;
  }

  .summary-card:hover {
    box-shadow: 0 4px 12px rgba(0,0,0,0.05);
    border-color: #d1d5db;
  }

  .summary-meta {
    margin-bottom: 0.5rem;
  }

  .category-badge {
    display: inline-block;
    background: #e0e7ff;
    color: #3730a3;
    border-radius: 9999px;
    padding: 0.125rem 0.5rem;
    font-size: 0.75rem;
    font-weight: 600;
  }

  .summary-content {
    color: #4b5563;
    line-height: 1.6;
    font-size: 0.95rem;
  }

  .summary-link {
    display: inline-block;
    color: #1d4ed8;
    font-weight: 600;
    text-decoration: none;
    margin-bottom: 0.25rem;
  }

  .summary-link:hover {
    text-decoration: underline;
  }

  .summary-text {
    margin-top: 0.25rem;
  }

  @media (max-width: 768px) {
    .container {
      padding: 1rem;
    }

    .title {
      font-size: 2rem;
    }

    .research-section,
    .summaries-section {
      padding: 2rem 1.5rem;
    }

    .tabs {
      flex-direction: column;
    }

    .tab {
      padding: 0.875rem;
    }

    .summaries-header {
      flex-direction: column;
      align-items: flex-start;
      gap: 1rem;
    }
  }
</style>