# UI Mockup: Modular Agent Skills Loader

Five-tab interface. Tab switching uses `data-tab` attribute selection — no JavaScript class toggling (Lesson 26).

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Modular Agent Skills Loader</title>
  <style>
    :root {
      --bg: #0f0f1a;
      --surface: #1a1a2e;
      --border: #4a4a8a;
      --text: #e0e0e0;
      --text-muted: #9090b0;
      --accent: #6a6aaa;
      --success: #4a8a4a;
      --reject: #8a4a4a;
      --pending: #8a8a4a;
    }
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { background: var(--bg); color: var(--text); font-family: system-ui, sans-serif; font-size: 14px; }
    header { background: var(--surface); border-bottom: 1px solid var(--border); padding: 12px 24px; }
    header h1 { font-size: 16px; font-weight: 600; color: var(--text); }
    header p { color: var(--text-muted); font-size: 12px; margin-top: 2px; }

    /* Tab nav */
    .tab-nav { display: flex; gap: 0; border-bottom: 1px solid var(--border); background: var(--surface); padding: 0 24px; }
    .tab-nav button {
      background: none; border: none; color: var(--text-muted); cursor: pointer;
      padding: 10px 16px; font-size: 13px; border-bottom: 2px solid transparent;
      transition: color 0.15s, border-color 0.15s;
    }
    .tab-nav button[data-active="true"] { color: var(--text); border-bottom-color: var(--accent); }
    .tab-nav button:hover { color: var(--text); }

    /* Tab panels */
    .tab-panel { display: none; padding: 24px; }
    .tab-panel[data-visible="true"] { display: block; }

    /* Cards */
    .card { background: var(--surface); border: 1px solid var(--border); border-radius: 6px; padding: 16px; margin-bottom: 16px; }
    .card h3 { font-size: 13px; color: var(--text-muted); margin-bottom: 8px; }
    .stat { font-size: 28px; font-weight: 700; color: var(--text); }

    /* Status badges */
    .badge { display: inline-block; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: 600; }
    .badge[data-status="COMPLETED"] { background: var(--success); color: #e0e0e0; }
    .badge[data-status="REJECTED"] { background: var(--reject); color: #e0e0e0; }
    .badge[data-status="PENDING"] { background: var(--pending); color: #1a1a2e; }
    .badge[data-status="EXECUTING"] { background: var(--accent); color: #e0e0e0; }
    .badge[data-status="GUARDRAIL_PASSED"] { background: #4a6a8a; color: #e0e0e0; }
    .badge[data-status="SKILL_SELECTED"] { background: #6a4a8a; color: #e0e0e0; }

    /* Table */
    table { width: 100%; border-collapse: collapse; }
    th { text-align: left; font-size: 11px; color: var(--text-muted); padding: 6px 8px; border-bottom: 1px solid var(--border); }
    td { padding: 8px; border-bottom: 1px solid #2a2a3e; font-size: 13px; }
    tr:last-child td { border-bottom: none; }

    /* Form */
    .form-row { margin-bottom: 12px; }
    label { display: block; font-size: 12px; color: var(--text-muted); margin-bottom: 4px; }
    input, textarea, select {
      width: 100%; background: #0f0f1a; border: 1px solid var(--border);
      color: var(--text); border-radius: 4px; padding: 8px; font-size: 13px;
    }
    input:focus, textarea:focus, select:focus { outline: 1px solid var(--accent); }
    button.primary {
      background: var(--accent); color: var(--text); border: none;
      border-radius: 4px; padding: 8px 16px; cursor: pointer; font-size: 13px;
    }
    button.primary:hover { background: #8a8acf; }

    /* Skill toggle */
    .skill-row { display: flex; align-items: center; justify-content: space-between; padding: 10px 0; border-bottom: 1px solid #2a2a3e; }
    .skill-meta { font-size: 12px; color: var(--text-muted); margin-top: 2px; }
    .toggle { background: none; border: 1px solid var(--border); color: var(--text-muted); border-radius: 3px; padding: 3px 10px; cursor: pointer; font-size: 11px; }
    .toggle[data-enabled="true"] { border-color: var(--success); color: #6aaa6a; }
  </style>
</head>
<body>

<header>
  <h1>Modular Agent Skills Loader</h1>
  <p>Runtime skill registry · before-tool-invocation guardrail · port 9958</p>
</header>

<nav class="tab-nav" role="tablist">
  <button role="tab" data-tab="overview" data-active="true" onclick="switchTab('overview')">Overview</button>
  <button role="tab" data-tab="architecture" data-active="false" onclick="switchTab('architecture')">Architecture</button>
  <button role="tab" data-tab="risk-survey" data-active="false" onclick="switchTab('risk-survey')">Risk Survey</button>
  <button role="tab" data-tab="eval-matrix" data-active="false" onclick="switchTab('eval-matrix')">Eval Matrix</button>
  <button role="tab" data-tab="app-ui" data-active="false" onclick="switchTab('app-ui')">App UI</button>
</nav>

<!-- Tab 1: Overview -->
<section class="tab-panel" data-tab="overview" data-visible="true" role="tabpanel">
  <div style="display:grid;grid-template-columns:repeat(3,1fr);gap:16px;margin-bottom:24px;">
    <div class="card"><h3>Active Skills</h3><div class="stat">4</div></div>
    <div class="card"><h3>Tasks Today</h3><div class="stat">127</div></div>
    <div class="card"><h3>Rejection Rate</h3><div class="stat">6%</div></div>
  </div>
  <div class="card">
    <h3>What this system does</h3>
    <p style="margin-top:8px;line-height:1.6;color:var(--text-muted);">
      Incoming tasks declare a required capability. The TaskDispatchAgent queries the skill
      registry to find an enabled skill that provides that capability, then selects the
      highest-version match. Before the skill tool executes, a guardrail checks that the
      skill is still enabled and the capability is registered. Disabled or unregistered
      capabilities are blocked and the task moves to REJECTED.
    </p>
  </div>
</section>

<!-- Tab 2: Architecture -->
<section class="tab-panel" data-tab="architecture" data-visible="false" role="tabpanel">
  <div class="card">
    <h3>Component overview</h3>
    <p style="margin-top:8px;line-height:1.6;color:var(--text-muted);">
      See <strong>PLAN.md</strong> for the four Mermaid diagrams. Summary:
    </p>
    <ul style="margin-top:12px;padding-left:20px;line-height:2;color:var(--text-muted);">
      <li><strong>AgentOrchestrationEndpoint</strong> — REST + SSE entry point</li>
      <li><strong>TaskDispatchAgent</strong> — AI agent for skill selection and invocation</li>
      <li><strong>Guardrail</strong> — before-tool-invocation hook; blocks disabled/unregistered capabilities</li>
      <li><strong>SkillLoaderWorkflow</strong> — durable 4-step validate/load/verify/emit lifecycle</li>
      <li><strong>SkillRegistryEntity</strong> — authoritative Key-Value Entity for skill state</li>
      <li><strong>SkillExecutionView</strong> — read model over the event log; powers task queries</li>
    </ul>
  </div>
  <div class="card">
    <h3>Task state machine</h3>
    <p style="color:var(--text-muted);margin-top:8px;line-height:1.6;">
      PENDING → SKILL_SELECTED → GUARDRAIL_PASSED → EXECUTING → COMPLETED<br/>
      Any state can transition to REJECTED via the guardrail or a skill error.
    </p>
  </div>
</section>

<!-- Tab 3: Risk Survey -->
<section class="tab-panel" data-tab="risk-survey" data-visible="false" role="tabpanel">
  <div class="card">
    <h3>Deployer fields — complete before production use</h3>
    <form style="margin-top:12px;">
      <div class="form-row">
        <label for="rs-pii">Does this deployment process PII?</label>
        <select id="rs-pii">
          <option value="">— select —</option>
          <option>Yes</option>
          <option>No</option>
        </select>
      </div>
      <div class="form-row">
        <label for="rs-cross-border">Is cross-border data transfer in scope?</label>
        <select id="rs-cross-border">
          <option value="">— select —</option>
          <option>Yes</option>
          <option>No</option>
        </select>
      </div>
      <div class="form-row">
        <label for="rs-model-family">AI model family</label>
        <input id="rs-model-family" type="text" placeholder="e.g. claude-opus-4, gpt-4o" />
      </div>
      <div class="form-row">
        <label for="rs-review-threshold">Human review threshold (tasks per hour before alert)</label>
        <input id="rs-review-threshold" type="number" placeholder="e.g. 500" />
      </div>
      <button type="button" class="primary">Save survey answers</button>
    </form>
  </div>
</section>

<!-- Tab 4: Eval Matrix -->
<section class="tab-panel" data-tab="eval-matrix" data-visible="false" role="tabpanel">
  <div class="card">
    <table>
      <thead>
        <tr>
          <th>ID</th>
          <th>Control name</th>
          <th>Mechanism</th>
          <th>Enforcement</th>
          <th>Integration tier</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>G-001</td>
          <td>Validate skill capability before tool invocation</td>
          <td>guardrail · before-tool-invocation</td>
          <td>blocking</td>
          <td>runs-out-of-the-box</td>
        </tr>
      </tbody>
    </table>
  </div>
  <div class="card">
    <h3>G-001 — what it does</h3>
    <p style="color:var(--text-muted);margin-top:8px;line-height:1.6;">
      Blocks any tool invocation where the selected skill is disabled or the requested
      capability is not in the registry. Fires before every tool call inside
      TaskDispatchAgent. On block: task moves to REJECTED; HTTP 422 returned to caller.
    </p>
  </div>
</section>

<!-- Tab 5: App UI -->
<section class="tab-panel" data-tab="app-ui" data-visible="false" role="tabpanel">
  <div style="display:grid;grid-template-columns:1fr 320px;gap:16px;">

    <!-- Left: task submission + list -->
    <div>
      <div class="card">
        <h3>Submit task</h3>
        <form style="margin-top:12px;">
          <div class="form-row">
            <label for="task-desc">Task description</label>
            <textarea id="task-desc" rows="3" placeholder="Describe what the agent should do"></textarea>
          </div>
          <div class="form-row">
            <label for="task-cap">Required capability</label>
            <input id="task-cap" type="text" placeholder="e.g. web-search" />
          </div>
          <button type="button" class="primary">Submit task</button>
        </form>
      </div>

      <div class="card">
        <h3>Recent tasks</h3>
        <table style="margin-top:8px;">
          <thead>
            <tr><th>Task ID</th><th>Capability</th><th>Skill</th><th>Status</th><th>Dispatched</th></tr>
          </thead>
          <tbody>
            <tr>
              <td>task-8f3a1c</td><td>web-search</td><td>web-search-v2</td>
              <td><span class="badge" data-status="COMPLETED">COMPLETED</span></td>
              <td>10:00:00</td>
            </tr>
            <tr>
              <td>task-7b2c9d</td><td>data-extract</td><td>—</td>
              <td><span class="badge" data-status="REJECTED">REJECTED</span></td>
              <td>09:58:12</td>
            </tr>
            <tr>
              <td>task-5e1f0a</td><td>web-search</td><td>web-search-v2</td>
              <td><span class="badge" data-status="EXECUTING">EXECUTING</span></td>
              <td>10:00:45</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Right: skill registry panel -->
    <div>
      <div class="card">
        <h3>Skill registry</h3>
        <div style="margin-top:12px;">
          <div class="skill-row">
            <div>
              <div>web-search-v2</div>
              <div class="skill-meta">v2.1.0 · web-search, url-fetch</div>
            </div>
            <button class="toggle" data-enabled="true">enabled</button>
          </div>
          <div class="skill-row">
            <div>
              <div>data-extract-v1</div>
              <div class="skill-meta">v1.0.0 · data-extract, csv-parse</div>
            </div>
            <button class="toggle" data-enabled="false">disabled</button>
          </div>
          <div class="skill-row">
            <div>
              <div>image-classify-v3</div>
              <div class="skill-meta">v3.0.1 · image-classify</div>
            </div>
            <button class="toggle" data-enabled="true">enabled</button>
          </div>
          <div class="skill-row">
            <div>
              <div>summarize-v1</div>
              <div class="skill-meta">v1.2.0 · summarize, text-extract</div>
            </div>
            <button class="toggle" data-enabled="true">enabled</button>
          </div>
        </div>
      </div>
    </div>

  </div>
</section>

<script>
  function switchTab(name) {
    // Update buttons using data-tab attribute selection
    document.querySelectorAll('.tab-nav button[data-tab]').forEach(function(btn) {
      btn.dataset.active = btn.dataset.tab === name ? 'true' : 'false';
    });
    // Update panels using data-tab attribute selection
    document.querySelectorAll('.tab-panel[data-tab]').forEach(function(panel) {
      panel.dataset.visible = panel.dataset.tab === name ? 'true' : 'false';
    });
  }
</script>

</body>
</html>
```
