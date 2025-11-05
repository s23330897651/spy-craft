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
		postedDate?: string;
	}

	interface Debrief {
		title: string;
		fullBody?: string;
		bulletPoints?: string[];
		totals?: Record<string, number>;
		bulletPointCount?: number;
	}

	let activeTab: 'research' | 'summaries' = 'research';

	let formData: CompanyData = {
		companyName: '',
		companyWebsite: ''
	};

	let summaries: ResearchEntry[] = [];
	let debrief: Debrief | null = null;

	let isLoading = false;
	let error = '';

	// Use your n8n production URL (workflow is active)
	const webhookUrl = 'https://n8n.intelligentresourcing.co/webhook/d95c8e70-5cb1-4323-838b-8a910fbecf65';

	let __uid = 0;

	function makeId(suffix: string): string {
		__uid++;
		return `${Date.now()}-${__uid}-${suffix}`;
	}

	function formatDate(value?: string): string {
		if (!value) return '';
		const trimmed = value.trim();
	 const parsed = Date.parse(trimmed);
		if (!Number.isNaN(parsed)) {
			const d = new Date(parsed);
			return d.toLocaleDateString(undefined, { year: 'numeric', month: 'short', day: 'numeric' });
		}
		const tIdx = trimmed.indexOf('T');
		if (tIdx > 0) return trimmed.slice(0, tIdx);
		const spaceIdx = trimmed.indexOf(' ');
		if (spaceIdx > 0) return trimmed.slice(0, spaceIdx);
		return trimmed;
	}

	function normalizeBulletPoint(text: string): string {
		// Remove accidental leading/trailing bold markers and whitespace
		return text.replace(/^\s*\*\*\s*/, '').replace(/\s*\*\*\s*$/, '').trim();
	}

	function prettyKey(key: string): string {
		if (!key) return key;
		return key.charAt(0).toUpperCase() + key.slice(1);
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
			}

			const isNonEmpty = (v: unknown) => typeof v === 'string' && v.trim().length > 0;
			const safeParse = (v: unknown): unknown => {
				if (typeof v !== 'string') return v;
				try { return JSON.parse(v); } catch { return v; }
			};
			const peel = (v: unknown, maxDepth = 2): unknown => {
				let cur: unknown = v;
				let d = 0;
				while (typeof cur === 'string' && d < maxDepth) {
					const p = safeParse(cur);
					if (p === cur) break;
					cur = p;
					d++;
				}
				return cur;
			};

			const pushIfValid = (bucket: ResearchEntry[], obj: Record<string, unknown>, categoryHint?: string) => {
				const err = typeof obj.error === 'string' ? obj.error : '';
				const title = typeof obj.title === 'string' ? obj.title : '';
				const summary = typeof obj.summary === 'string' ? obj.summary : '';
				const url = typeof obj.url === 'string' ? obj.url : '';
				const category = typeof obj.category === 'string' ? obj.category : (categoryHint || 'Research');
				const postedDate = typeof obj.postedDate === 'string' ? obj.postedDate : '';

				// Skip if error-only or all fields empty
				if (isNonEmpty(err) && !isNonEmpty(title) && !isNonEmpty(summary) && !isNonEmpty(url)) return;
				if (!isNonEmpty(title) && !isNonEmpty(summary) && !isNonEmpty(url)) return;

				bucket.push({
					id: makeId(category || 'item'),
					companyName: formData.companyName,
					category: category || 'Research',
					title: title || (isNonEmpty(err) ? `${category} Error` : ''),
					summary: summary || err || '',
					url: isNonEmpty(url) ? url : '#',
					postedDate: postedDate || undefined
				});
			};

			// Normalize to list and extract debrief first if present
			const baseList: unknown[] = Array.isArray(data)
				? data
				: (data && typeof data === 'object' ? [data] : []);

			console.log('Normalized list length:', baseList.length);

			let remainingList: unknown[] = baseList;
			// First element is always a debrief now
			if (baseList.length > 0) {
				const first = peel(baseList[0]);
				if (first && typeof first === 'object' && !Array.isArray(first)) {
					const rec = first as Record<string, unknown>;
					// Heuristic: presence of fullBody or bulletPoints marks a debrief
					if ('fullBody' in rec || 'bulletPoints' in rec) {
						const d: Debrief = {
							title: typeof rec.title === 'string' ? rec.title : 'Content Debrief',
							fullBody: typeof rec.fullBody === 'string' ? rec.fullBody : undefined,
							bulletPoints: Array.isArray(rec.bulletPoints) ? rec.bulletPoints.filter((s: unknown) => typeof s === 'string' && s.trim().length > 0) as string[] : undefined,
							totals: (rec.totals && typeof rec.totals === 'object') ? rec.totals as Record<string, number> : undefined,
							bulletPointCount: typeof rec.bulletPointCount === 'number' ? rec.bulletPointCount : undefined
						};
						debrief = d;
						remainingList = baseList.slice(1);
						console.log('Debrief extracted:', d.title);
					}
				}
			}

			const newEntries: ResearchEntry[] = [];

			remainingList.forEach((item, idx) => {
				console.log(`Item[${idx}]`, item);
				if (!item || typeof item !== 'object' || Array.isArray(item)) return;

				const rec = item as Record<string, unknown>;

				const hasDirectShape = ('title' in rec) || ('summary' in rec) || ('url' in rec) || ('category' in rec) || ('postedDate' in rec);
				if (hasDirectShape) {
					pushIfValid(newEntries, rec);
					return;
				}

				for (const [categoryKey, rawVal] of Object.entries(rec)) {
					const category = categoryKey.toString();
					let v = peel(rawVal);

					if (Array.isArray(v)) {
						v.forEach((child) => {
							const obj = peel(child);
							if (obj && typeof obj === 'object' && !Array.isArray(obj)) {
								pushIfValid(newEntries, obj as Record<string, unknown>, category);
							}
						});
					} else if (v && typeof v === 'object') {
						pushIfValid(newEntries, v as Record<string, unknown>, category);
					} else {
						// Primitive/empty -> skip
					}
				}
			});

			// Deduplicate by URL (keep first occurrence)
			const seen = new Set<string>();
			const deduped = newEntries.filter((e) => {
				const key = (e.url || '').trim().toLowerCase();
				if (!key) return true;
				if (seen.has(key)) return false;
				seen.add(key);
				return true;
			});

			console.log('Mapped entries count (deduped):', deduped.length);
			if (deduped.length > 0) {
				console.table(deduped.map(e => ({ category: e.category, title: e.title, url: e.url, postedDate: e.postedDate || '' })));
			}

			if (deduped.length === 0 && !debrief) {
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
				summaries = [...deduped, ...summaries];
			}

			console.log('Summaries after update (length):', summaries.length);

			formData = {
				companyName: '',
				companyWebsite: ''
			};
			activeTab = 'summaries';
			console.log('Switched to tab:', activeTab);

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
		debrief = null;
	}

</script>

<main class="container">
  <header class="header">
    <h1 class="title">Company Research Portal</h1>
    <p class="subtitle">Powered by N8N Intelligence</p>
  </header>

  <nav class="tabs" role="tablist" aria-label="Research navigation">
    <button
      id="tab-research"
      class="tab"
      class:active={activeTab === 'research'}
      role="tab"
      aria-selected={activeTab === 'research'}
      aria-controls="panel-research"
      on:click={() => activeTab = 'research'}
    >
      Research Form
    </button>

    <button
      id="tab-summaries"
      class="tab"
      class:active={activeTab === 'summaries'}
      role="tab"
      aria-selected={activeTab === 'summaries'}
      aria-controls="panel-summaries"
      on:click={() => activeTab = 'summaries'}
    >
      Research Summaries
      {#if summaries.length > 0}
        <span class="badge" aria-label="Number of summaries">{summaries.length}</span>
      {/if}
    </button>
  </nav>

  <div class="content">
    {#if activeTab === 'research'}
      <section id="panel-research" role="tabpanel" aria-labelledby="tab-research" class="research-section">
        <div class="form-card">
          <h2>Company Research Request</h2>

          {#if error}
            <div class="alert error" role="alert" aria-live="polite">
              <svg class="alert-icon" viewBox="0 0 20 20" fill="currentColor" aria-hidden="true">
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
                <svg class="spinner" viewBox="0 0 24 24" aria-hidden="true">
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
      </section>

    {:else if activeTab === 'summaries'}
      <section id="panel-summaries" role="tabpanel" aria-labelledby="tab-summaries" class="summaries-section">
        <div class="summaries-header">
          <h2>{debrief ? 'Sources' : 'Research Summaries'}</h2>
          {#if summaries.length > 0 || debrief}
            <button class="clear-btn" on:click={clearSummaries}>
              Clear All
            </button>
          {/if}
        </div>

        {#if !debrief && summaries.length === 0}
          <div class="empty-state">
            <svg class="empty-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24" aria-hidden="true">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
            </svg>
            <h3>No research summaries yet</h3>
            <p>Submit a research request to see summaries here</p>
          </div>
        {:else}
          <div class="summaries-list">
            {#if debrief}
              <article class="debrief-card">
                <div class="debrief-header">
                  <span class="debrief-badge">Debrief</span>
                  <h3 class="debrief-title">{debrief.title || 'Content Debrief'}</h3>
                </div>

                {#if debrief.bulletPoints && debrief.bulletPoints.length > 0}
                  <ul class="debrief-list">
                    {#each debrief.bulletPoints as bp}
                      <li><strong>{normalizeBulletPoint(bp)}</strong></li>
                    {/each}
                  </ul>
                {:else if debrief.fullBody}
                  <div class="debrief-body">
                    {#each debrief.fullBody.split('\n') as line}
                      {#if line.trim().startsWith('- ')}
                        <div class="debrief-line"><span>•</span> <strong>{line.trim().slice(2)}</strong></div>
                      {:else if line.trim().length === 0}
                        <div class="debrief-spacer"></div>
                      {:else}
                        <div class="debrief-text">{line}</div>
                      {/if}
                    {/each}
                  </div>
                {/if}

                {#if debrief.totals}
                  <div class="debrief-totals">
                    {#if 'uncategorised' in debrief.totals}
                      <span>Totals: {prettyKey('uncategorised')}: {debrief.totals['uncategorised']}</span>
                      <span class="divider">|</span>
                    {/if}
                    {#if 'overall' in debrief.totals}
                      <span>Overall: {debrief.totals['overall']}</span>
                    {/if}
                  </div>
                {/if}
              </article>

              <div class="section-divider" aria-hidden="true"></div>
            {/if}

            {#each summaries as entry (entry.id)}
              <article class="summary-card">
                <div class="summary-meta">
                  <span class="category-badge">{entry.category}</span>
                  {#if entry.postedDate}
                    <span class="date-badge" title={entry.postedDate}>{formatDate(entry.postedDate)}</span>
                  {/if}
                </div>

                <div class="summary-content">
                  {#if entry.url && entry.url !== '#'}
                    <a class="summary-link" href={entry.url} target="_blank" rel="noopener noreferrer nofollow">
                      {entry.title}
                    </a>
                  {:else}
                    <div class="summary-link disabled">{entry.title}</div>
                  {/if}

                  <div class="summary-text">
                    {@html (entry.summary || '').replace(/\n/g, '<br>')}
                  </div>
                </div>
              </article>
            {/each}
          </div>
        {/if}
      </section>
    {/if}
  </div>
</main>

<style>
  :root {
    --bg: #ffffff;
    --text: #111827;
    --muted: #6b7280;
    --primary: #667eea;
    --primary-2: #764ba2;
    --card: #ffffff;
    --card-muted: #f9fafb;
    --border: #e5e7eb;
    --border-strong: #d1d5db;
    --danger: #ef4444;
    --danger-2: #dc2626;
  }

  :global(body) {
    margin: 0;
    padding: 0;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    color-scheme: light;
    color: var(--text);
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
    color: #ffffff;
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
    opacity: 0.95;
    font-weight: 500;
  }

  .tabs {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 2rem;
    background: rgba(255,255,255,0.12);
    border-radius: 12px;
    padding: 0.5rem;
    backdrop-filter: blur(10px);
  }

  .tab {
    flex: 1;
    background: transparent;
    border: none;
    color: #ffffff;
    padding: 1rem 2rem;
    border-radius: 8px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
    outline: none;
  }

  .tab:hover {
    background: rgba(255,255,255,0.18);
  }

  .tab:focus-visible {
    box-shadow: 0 0 0 3px rgba(255,255,255,0.6);
  }

  .tab.active {
    background: #ffffff;
    color: var(--primary);
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  }

  .badge {
    background: var(--primary);
    color: #ffffff;
    border-radius: 9999px;
    min-width: 24px;
    height: 24px;
    padding: 0 8px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-size: 0.75rem;
    font-weight: 700;
    line-height: 1;
  }

  .content {
    background: var(--card);
    color: var(--text);
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
    color: var(--danger-2);
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
    font-weight: 600;
    color: #374151;
    margin-bottom: 0.5rem;
    font-size: 0.9rem;
  }

  .input {
    width: 100%;
    padding: 0.875rem 1rem;
    border: 2px solid var(--border);
    border-radius: 8px;
    font-size: 1rem;
    transition: all 0.2s ease;
    background: #ffffff;
    color: #111827;
  }

  .input::placeholder {
    color: #9ca3af;
  }

  .input:focus {
    outline: none;
    border-color: var(--primary);
    box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.15);
  }

  .input:disabled {
    background: #f3f4f6;
    opacity: 0.75;
    cursor: not-allowed;
    color: #111827;
  }

  .submit-btn {
    width: 100%;
    background: linear-gradient(135deg, var(--primary) 0%, var(--primary-2) 100%);
    color: #ffffff;
    border: none;
    padding: 1rem 2rem;
    border-radius: 8px;
    font-size: 1rem;
    font-weight: 700;
    cursor: pointer;
    transition: all 0.2s ease;
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
    opacity: 0.75;
    cursor: not-allowed;
    transform: none;
    box-shadow: none;
  }

  .submit-btn:focus-visible {
    outline: 3px solid rgba(102, 126, 234, 0.4);
    outline-offset: 2px;
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
    background: var(--danger);
    color: #ffffff;
    border: none;
    padding: 0.5rem 1rem;
    border-radius: 6px;
    font-size: 0.9rem;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.15s ease, transform 0.15s ease;
  }

  .clear-btn:hover {
    background: var(--danger-2);
    transform: translateY(-1px);
  }

  .clear-btn:focus-visible {
    outline: 3px solid rgba(239, 68, 68, 0.35);
    outline-offset: 2px;
  }

  .empty-state {
    text-align: center;
    padding: 4rem 2rem;
    color: var(--muted);
  }

  .empty-icon {
    width: 64px;
    height: 64px;
    margin: 0 auto 1rem;
    opacity: 0.6;
  }

  .empty-state h3 {
    font-size: 1.25rem;
    font-weight: 700;
    margin: 0 0 0.5rem 0;
    color: #374151;
  }

  .empty-state p {
    margin: 0;
    font-size: 1rem;
  }

  .summaries-list {
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
  }

  .debrief-card {
    background: #ffffff;
    border: 2px solid var(--border);
    border-radius: 14px;
    padding: 1.5rem;
    box-shadow: 0 8px 20px rgba(0,0,0,0.06);
  }

  .debrief-header {
    display: flex;
    align-items: baseline;
    gap: 0.5rem;
    margin-bottom: 0.75rem;
  }

  .debrief-badge {
    display: inline-block;
    background: #e0e7ff;
    color: #3730a3;
    border-radius: 9999px;
    padding: 0.125rem 0.5rem;
    font-size: 0.75rem;
    font-weight: 700;
  }

  .debrief-title {
    margin: 0;
    font-size: 1.25rem;
    color: #111827;
    font-weight: 700;
  }

  .debrief-list {
    margin: 0.5rem 0 0 0;
    padding-left: 1.25rem;
    color: #374151;
  }

  .debrief-list li {
    margin: 0.4rem 0;
    line-height: 1.5;
  }

  .debrief-body {
    color: #374151;
    line-height: 1.6;
  }

  .debrief-line {
    display: flex;
    gap: 0.5rem;
    margin: 0.35rem 0;
  }

  .debrief-text {
    margin: 0.35rem 0;
  }

  .debrief-spacer {
    height: 0.5rem;
  }

  .debrief-totals {
    margin-top: 0.75rem;
    color: #4b5563;
    font-weight: 600;
  }

  .debrief-totals .divider {
    margin: 0 0.5rem;
    color: #9ca3af;
  }

  .section-divider {
    height: 1px;
    background: var(--border);
    margin: 0.5rem 0 0.25rem 0;
  }

  .summary-card {
    background: var(--card-muted);
    border: 1px solid var(--border);
    border-radius: 12px;
    padding: 1.5rem;
    transition: all 0.2s ease;
  }

  .summary-card:hover {
    box-shadow: 0 4px 12px rgba(0,0,0,0.06);
    border-color: var(--border-strong);
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
    font-weight: 700;
  }

  .date-badge {
    display: inline-block;
    background: #eef2ff;
    color: #374151;
    border-radius: 9999px;
    padding: 0.125rem 0.5rem;
    font-size: 0.75rem;
    font-weight: 600;
    margin-left: 0.5rem;
  }

  .summary-content {
    color: #374151;
    line-height: 1.6;
    font-size: 0.95rem;
  }

  .summary-link {
    display: inline-block;
    color: #1d4ed8;
    font-weight: 700;
    text-decoration: none;
    margin-bottom: 0.25rem;
  }

  .summary-link.disabled {
    color: #374151;
    font-weight: 700;
    margin-bottom: 0.25rem;
  }

  .summary-link:hover {
    text-decoration: underline;
  }

  .summary-text {
    margin-top: 0.25rem;
    color: #4b5563;
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
      padding: 2rem 1.25rem;
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