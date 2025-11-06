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
		executiveSummary?: string;
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
	let lastCompanyName = '';

	let isLoading = false;
	let error = '';

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

	const hasText = (s?: unknown) => typeof s === 'string' && s.trim().length > 0;
	const stripBold = (s: string) => s.replace(/^\s*\*\*\s*|\s*\*\*\s*$/g, '');
	const normalizePoint = (s: string) => stripBold(s.replace(/^\s*-\s*/, '').trim());

	function extractBulletsFromFullBody(fullBody?: string): string[] {
		if (!fullBody) return [];
		return fullBody
			.split('\n')
			.filter((l) => l.trim().startsWith('- ') || l.trim().startsWith('• '))
			.map((l) => normalizePoint(l.replace(/^•\s*/, '')))
			.filter((t) => t.length > 0);
	}

	function extractParagraphFromFullBody(fullBody?: string): string | undefined {
		if (!fullBody) return undefined;
		const paras = fullBody
			.split('\n')
			.map((l) => l.trim())
			.filter((l) => l.length > 0 && !l.startsWith('#') && !l.startsWith('- ') && !l.startsWith('• '));
		return paras.length ? paras.join(' ') : undefined;
	}

	function deriveExecutiveSummary(existing?: string, bullets?: string[], fullBody?: string): string | undefined {
		if (hasText(existing)) return (existing as string).trim();
		const fromBody = extractParagraphFromFullBody(fullBody);
		if (fromBody) return fromBody;
		if (bullets && bullets.length) {
			const take = bullets.slice(0, 3);
			return take.join(' ');
		}
		return undefined;
	}

	function getDisplayCompany(): string {
		const fromSummaries = summaries.find(s => (s.companyName || '').trim().length > 0)?.companyName || '';
		return fromSummaries || lastCompanyName || formData.companyName || 'Company';
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

			lastCompanyName = formData.companyName;

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

			const rawText = await response.text();
			if (!response.ok) {
				throw new Error(`HTTP ${response.status} ${response.statusText}${rawText ? ` - ${rawText}` : ''}`);
			}

			let data: unknown = null;
			try {
				data = JSON.parse(rawText);
			} catch {}

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
				const title = typeof obj.title === 'string' ? obj.title : '';
				const summary = typeof obj.summary === 'string' ? obj.summary : '';
				const url = typeof obj.url === 'string' ? obj.url : '';
				const category = typeof obj.category === 'string' ? obj.category : (categoryHint || 'Research');
				const postedDate = typeof obj.postedDate === 'string' ? obj.postedDate : '';

				if (!hasText(summary)) return;

				bucket.push({
					id: makeId(category || 'item'),
					companyName: payload.companyName,
					category: category || 'Research',
					title: hasText(title) ? title : 'Untitled',
					summary: summary,
					url: hasText(url) ? url : '#',
					postedDate: hasText(postedDate) ? postedDate : undefined
				});
			};

			const baseList: unknown[] = Array.isArray(data)
				? data
				: (data && typeof data === 'object' ? [data] : []);

			let remainingList: unknown[] = baseList;

			if (baseList.length > 0) {
				const first = peel(baseList[0]);
				if (first && typeof first === 'object' && !Array.isArray(first)) {
					const rec = first as Record<string, unknown>;

					const title = typeof rec['title'] === 'string' ? rec['title'] : 'Content Debrief';
					const fullBody = typeof rec['fullBody'] === 'string' ? rec['fullBody'] : undefined;

					const exec_v2 = typeof rec['executive_summary'] === 'string' ? rec['executive_summary'] : undefined;

					const bpA = Array.isArray(rec['bulletPoints']) ? rec['bulletPoints'] as unknown[] : undefined;
					const bpB = Array.isArray(rec['supporting_points']) ? rec['supporting_points'] as unknown[] : undefined;

					let bullets: string[] = [];
					if (bpA) bullets = bpA.filter((s) => typeof s === 'string').map((s: any) => normalizePoint(String(s)));
					else if (bpB) bullets = bpB.filter((s) => typeof s === 'string').map((s: any) => normalizePoint(String(s)));
					else bullets = extractBulletsFromFullBody(fullBody);

					const executiveSummary = deriveExecutiveSummary(exec_v2, bullets, fullBody);

					debrief = {
						title,
						executiveSummary,
						fullBody,
						bulletPoints: bullets,
						totals: (rec['totals'] && typeof rec['totals'] === 'object') ? rec['totals'] as Record<string, number> : undefined,
						bulletPointCount: typeof rec['bulletPointCount'] === 'number' ? rec['bulletPointCount'] : undefined
					};
					remainingList = baseList.slice(1);
				}
			}

			const newEntries: ResearchEntry[] = [];

			remainingList.forEach((item) => {
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
					}
				}
			});

			const seen = new Set<string>();
			const deduped = newEntries.filter((e) => {
				const key = (e.url || '').trim().toLowerCase();
				if (!key) return true;
				if (seen.has(key)) return false;
				seen.add(key);
				return true;
			});

			if (deduped.length === 0 && !debrief) {
				const placeholder: ResearchEntry = {
					id: makeId('no-results'),
					companyName: payload.companyName,
					category: 'Research',
					title: 'No articles found',
					summary: '',
					url: '#'
				};
				summaries = [placeholder, ...summaries];
			} else {
				summaries = [...deduped, ...summaries];
			}

			formData = {
				companyName: '',
				companyWebsite: ''
			};
			activeTab = 'summaries';

		} catch (err) {
			const message = err instanceof Error ? err.message : 'Failed to submit research request';
			error = message.includes('Failed to fetch') ? 'Failed to fetch (likely CORS). See note below.' : message;
		} finally {
			isLoading = false;
		}
	}

	function clearSummaries() {
		summaries = [];
		debrief = null;
		lastCompanyName = '';
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
              <article class="debrief-card" aria-labelledby="debrief-heading">
                <header class="debrief-header">
                  <div class="debrief-rule" aria-hidden="true"></div>
                  <h2 id="debrief-heading" class="debrief-label">Debrief</h2>
                  <div class="debrief-rule" aria-hidden="true"></div>
                </header>

                <h3 class="debrief-title">{debrief.title}</h3>
                <h4 class="debrief-company">{getDisplayCompany()}</h4>

                <section class="debrief-exec" aria-labelledby="exec-summary-title">
                  <h5 id="exec-summary-title" class="debrief-section-title">Executive Summary</h5>

                  {#if hasText(debrief?.executiveSummary)}
                    <p class="debrief-paragraph">{debrief?.executiveSummary}</p>
                  {/if}

                  {#if debrief?.bulletPoints && debrief.bulletPoints.length}
                    <ul class="debrief-bullets">
                      {#each debrief.bulletPoints as bp}
                        <li><strong>{bp}</strong></li>
                      {/each}
                    </ul>
                  {/if}
                </section>

                {#if debrief?.totals}
                  <footer class="debrief-totals">
                    {#if 'uncategorised' in debrief.totals}
                      <span>Totals: Uncategorised: {debrief.totals['uncategorised']}</span>
                      <span class="divider">|</span>
                    {/if}
                    {#if 'overall' in debrief.totals}
                      <span>Overall: {debrief.totals['overall']}</span>
                    {/if}
                  </footer>
                {/if}
              </article>

              <div class="sources-title" role="heading" aria-level="3">Sources</div>
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
    --bg: #31160a;
    --bg-2: #5a2a12;
    --text: #0b1b2b;
    --muted: #5b6b7a;
    --primary: #f97316;
    --primary-2: #ea580c;
    --card: #ffffff;
    --card-muted: #f6f6f4;
    --border: #f0e5db;
    --border-strong: #e1cdbb;
    --danger: #ef4444;
    --danger-2: #dc2626;
    --ink-strong: #0b1b2b;
    --ink: #1f2a37;
    --ink-soft: #4b5563;
  }

  :global(body) {
    margin: 0;
    padding: 0;
    background: linear-gradient(135deg, var(--bg) 0%, var(--bg-2) 100%);
    min-height: 100vh;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    color-scheme: light;
    color: var(--text);
  }

  .container { max-width: 1100px; margin: 0 auto; padding: 2rem; min-height: 100vh; }

  .header { text-align: center; margin-bottom: 2.25rem; color: #ffe9db; }

  .title { font-size: 2.5rem; font-weight: 700; margin: 0; text-shadow: 0 2px 4px rgba(0,0,0,0.2); }

  .subtitle { font-size: 1.05rem; margin: 0.5rem 0 0 0; opacity: 0.9; font-weight: 500; }

  .tabs {
    display: flex; gap: 0.5rem; margin-bottom: 1.25rem;
    background: rgba(255,255,255,0.08); border-radius: 12px; padding: 0.4rem; backdrop-filter: blur(10px);
  }

  .tab {
    flex: 1; background: transparent; border: none; color: #fff7ed;
    padding: 0.9rem 1.25rem; border-radius: 9px; font-size: 0.98rem; font-weight: 600; cursor: pointer;
    transition: all 0.2s ease; display: flex; align-items: center; justify-content: center; gap: 0.5rem; outline: none;
  }
  .tab:hover { background: rgba(255,255,255,0.16); }
  .tab:focus-visible { box-shadow: 0 0 0 3px rgba(249, 115, 22, 0.55); }
  .tab.active { background: #ffffff; color: var(--primary-2); box-shadow: 0 6px 18px rgba(0,0,0,0.12); }

  .badge {
    background: var(--primary); color: #3a1f04; border-radius: 9999px; min-width: 24px; height: 24px; padding: 0 8px;
    display: inline-flex; align-items: center; justify-content: center; font-size: 0.75rem; font-weight: 800; line-height: 1;
  }

  .content { background: var(--card); color: var(--text); border-radius: 16px; box-shadow: 0 22px 44px rgba(0,0,0,0.14); overflow: hidden; }

  .research-section { padding: 2.25rem; }

  .form-card h2 { color: var(--ink-strong); font-size: 1.6rem; font-weight: 700; margin: 0 0 1.5rem 0; text-align: center; }

  .alert { display: flex; align-items: center; gap: 0.75rem; padding: 1rem; border-radius: 8px; margin-bottom: 1.25rem; }
  .alert.error { background: #ffefef; color: var(--danger-2); border: 1px solid #ffd0d0; }
  .alert-icon { width: 20px; height: 20px; flex-shrink: 0; }

  .form { max-width: 520px; margin: 0 auto; }
  .form-group { margin-bottom: 1.25rem; }
  .label { display: block; font-weight: 600; color: var(--ink); margin-bottom: 0.5rem; font-size: 0.92rem; }

  .input {
    width: 100%; padding: 0.85rem 1rem; border: 2px solid var(--border); border-radius: 10px; font-size: 1rem;
    transition: all 0.2s ease; background: #ffffff; color: #0b1b2b;
  }
  .input::placeholder { color: #99a7b3; }
  .input:focus { outline: none; border-color: var(--primary); box-shadow: 0 0 0 4px rgba(249, 115, 22, 0.18); }
  .input:disabled { background: #f1f1ef; opacity: 0.75; cursor: not-allowed; color: #0b1b2b; }

  .submit-btn {
    width: 100%; background: linear-gradient(135deg, var(--primary) 0%, var(--primary-2) 100%); color: #3a1f04;
    border: none; padding: 0.95rem 2rem; border-radius: 10px; font-size: 1rem; font-weight: 800; cursor: pointer;
    transition: all 0.2s ease; display: flex; align-items: center; justify-content: center; gap: 0.5rem; margin-top: 0.5rem;
  }
  .submit-btn:hover:not(:disabled) { transform: translateY(-2px); box-shadow: 0 10px 24px rgba(249, 115, 22, 0.28); }
  .submit-btn:disabled { opacity: 0.75; cursor: not-allowed; transform: none; box-shadow: none; }
  .submit-btn:focus-visible { outline: 3px solid rgba(249, 115, 22, 0.35); outline-offset: 2px; }

  .spinner { width: 20px; height: 20px; animation: spin 1s linear infinite; }
  @keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }

  .summaries-section { padding: 2.25rem; }

  .summaries-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1.25rem; }
  .summaries-header h2 { color: var(--ink-strong); font-size: 1.6rem; font-weight: 700; margin: 0; }

  .clear-btn {
    background: var(--danger); color: #ffffff; border: none; padding: 0.55rem 0.95rem; border-radius: 8px; font-size: 0.9rem; font-weight: 700;
    cursor: pointer; transition: background 0.15s ease, transform 0.15s ease;
  }
  .clear-btn:hover { background: var(--danger-2); transform: translateY(-1px); }
  .clear-btn:focus-visible { outline: 3px solid rgba(239, 68, 68, 0.35); outline-offset: 2px; }

  .empty-state { text-align: center; padding: 3.25rem 1.5rem; color: var(--muted); }
  .empty-icon { width: 60px; height: 60px; margin: 0 auto 1rem; opacity: 0.7; }
  .empty-state h3 { font-size: 1.15rem; font-weight: 800; margin: 0 0 0.5rem 0; color: var(--ink-strong); }
  .empty-state p { margin: 0; font-size: 0.98rem; }

  .summaries-list { display: flex; flex-direction: column; gap: 1.25rem; }

  .debrief-card {
    background: #ffffff; border: 1px solid var(--border); border-radius: 14px;
    padding: 1.25rem 1.25rem 1rem 1.25rem; box-shadow: 0 10px 24px rgba(0,0,0,0.06);
  }

  .debrief-header {
    display: grid; grid-template-columns: 1fr auto 1fr; gap: 0.75rem; align-items: center; margin-bottom: 0.7rem;
  }
  .debrief-rule { height: 1px; background: var(--border); }
  .debrief-label { color: var(--ink-strong); font-weight: 800; letter-spacing: 0.02em; }

  .debrief-title { margin: 0.25rem 0 0.25rem 0; color: var(--ink-strong); font-size: 1.15rem; font-weight: 800; }
  .debrief-company { color: var(--ink); font-size: 1rem; font-weight: 700; margin: 0 0 0.5rem 0; }

  .debrief-section-title {
    color: var(--ink); font-weight: 800; font-size: 1rem; margin: 0.25rem 0 0.4rem 0; padding-bottom: 0.35rem; border-bottom: 1px solid var(--border);
  }

  .debrief-paragraph { color: var(--ink); line-height: 1.65; margin: 0.25rem 0 0.5rem 0; }

  .debrief-bullets { margin: 0.35rem 0 0.25rem 1.25rem; color: var(--ink); }
  .debrief-bullets li { margin: 0.38rem 0; line-height: 1.55; }

  .debrief-totals { margin-top: 0.6rem; color: var(--ink-soft); font-weight: 700; }
  .debrief-totals .divider { margin: 0 0.5rem; color: #b88f6d; }

  .sources-title {
    margin: 0.8rem 0 -0.2rem 0; color: var(--ink-strong); font-weight: 800; font-size: 1rem;
    border-top: 1px solid var(--border); padding-top: 0.75rem;
  }

  .summary-card {
    background: var(--card-muted); border: 1px solid var(--border); border-radius: 12px; padding: 1.1rem; transition: all 0.2s ease;
  }
  .summary-card:hover { box-shadow: 0 6px 16px rgba(0,0,0,0.06); border-color: var(--border-strong); }

  .summary-meta { margin-bottom: 0.45rem; }

  .category-badge {
    display: inline-block; background: rgba(249, 115, 22, 0.14); color: var(--primary-2);
    border: 1px solid rgba(249, 115, 22, 0.34); border-radius: 9999px; padding: 0.12rem 0.5rem; font-size: 0.72rem; font-weight: 800;
  }

  .date-badge {
    display: inline-block; background: #fff1e6; color: var(--ink); border-radius: 9999px;
    padding: 0.12rem 0.5rem; font-size: 0.72rem; font-weight: 700; margin-left: 0.5rem;
  }

  .summary-content { color: var(--ink); line-height: 1.6; font-size: 0.96rem; }
  .summary-link { display: inline-block; color: var(--primary-2); font-weight: 800; text-decoration: none; margin-bottom: 0.25rem; }
  .summary-link:hover { text-decoration: underline; }
  .summary-link.disabled { color: var(--ink); font-weight: 800; margin-bottom: 0.25rem; }
  .summary-text { margin-top: 0.25rem; color: var(--ink-soft); }

  @media (max-width: 768px) {
    .container { padding: 1rem; }
    .title { font-size: 2rem; }
    .research-section, .summaries-section { padding: 1.5rem 1rem; }
    .tabs { flex-direction: column; }
    .tab { padding: 0.85rem; }
    .summaries-header { flex-direction: column; align-items: flex-start; gap: 0.8rem; }
  }
</style>