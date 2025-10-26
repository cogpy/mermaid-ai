<script lang="ts">
  import { Button } from '$/components/ui/button';
  import * as Dialog from '$/components/ui/dialog';
  import { Input } from '$/components/ui/input';
  import { updateCode } from '$/util/state';
  import { logEvent } from '$/util/stats';
  import SparklesIcon from '~icons/material-symbols/auto-awesome-rounded';

  let open = $state(false);
  let description = $state('');
  let apiKey = $state('');
  let loading = $state(false);
  let error = $state('');
  let diagramType = $state('flowchart');

  // Load API key from localStorage on mount
  $effect(() => {
    if (typeof window !== 'undefined') {
      const saved = localStorage.getItem('mermaid-ai-api-key');
      if (saved) {
        apiKey = saved;
      }
    }
  });

  const saveApiKey = () => {
    if (typeof window !== 'undefined') {
      localStorage.setItem('mermaid-ai-api-key', apiKey);
    }
  };

  const generateDiagram = async () => {
    if (!description.trim()) {
      error = 'Please enter a description';
      return;
    }

    if (!apiKey.trim()) {
      error = 'Please enter your OpenAI API key';
      return;
    }

    loading = true;
    error = '';

    try {
      const response = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apiKey}`
        },
        body: JSON.stringify({
          model: 'gpt-4o-mini',
          messages: [
            {
              role: 'system',
              content: `You are a Mermaid diagram expert. Generate valid Mermaid ${diagramType} syntax based on user descriptions. Return ONLY the Mermaid code without markdown code blocks or explanations.`
            },
            {
              role: 'user',
              content: description
            }
          ],
          temperature: 0.7,
          max_tokens: 2000
        })
      });

      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}));
        throw new Error(errorData.error?.message || `API request failed: ${response.statusText}`);
      }

      const data = await response.json();
      const generatedCode = data.choices[0]?.message?.content?.trim() || '';

      // Remove markdown code blocks if present
      const cleanCode = generatedCode
        .replace(/^```mermaid\n?/i, '')
        .replace(/^```\n?/i, '')
        .replace(/\n?```$/i, '')
        .trim();

      if (cleanCode) {
        updateCode(cleanCode);
        logEvent('aiGenerate', { diagramType });
        open = false;
        description = '';
      } else {
        error = 'Failed to generate diagram code';
      }
    } catch (err) {
      error = err instanceof Error ? err.message : 'Failed to generate diagram';
      console.error('AI generation error:', err);
    } finally {
      loading = false;
    }
  };

  const handleKeyDown = (e: KeyboardEvent) => {
    if (e.key === 'Enter' && (e.metaKey || e.ctrlKey)) {
      void generateDiagram();
    }
  };
</script>

<Dialog.Root bind:open>
  <Dialog.Trigger asChild let:builder>
    <Button builders={[builder]} variant="accent" size="sm">
      <SparklesIcon />
      AI Assistant
    </Button>
  </Dialog.Trigger>
  <Dialog.Content class="max-w-2xl">
    <Dialog.Header>
      <Dialog.Title>AI Diagram Generator</Dialog.Title>
      <Dialog.Description>
        Describe your diagram in natural language and let AI generate Mermaid code for you.
      </Dialog.Description>
    </Dialog.Header>

    <div class="flex flex-col gap-4 py-4">
      <div class="flex flex-col gap-2">
        <label for="diagram-type" class="text-sm font-medium">Diagram Type</label>
        <select
          id="diagram-type"
          bind:value={diagramType}
          class="flex h-9 w-full rounded-md border border-input bg-transparent px-3 py-1 text-sm shadow-sm transition-colors focus-visible:ring-1 focus-visible:ring-ring focus-visible:outline-none">
          <option value="flowchart">Flowchart</option>
          <option value="sequence">Sequence Diagram</option>
          <option value="class">Class Diagram</option>
          <option value="state">State Diagram</option>
          <option value="erDiagram">Entity Relationship Diagram</option>
          <option value="journey">User Journey</option>
          <option value="gantt">Gantt Chart</option>
          <option value="pie">Pie Chart</option>
          <option value="mindmap">Mind Map</option>
          <option value="timeline">Timeline</option>
        </select>
      </div>

      <div class="flex flex-col gap-2">
        <label for="description" class="text-sm font-medium">Description</label>
        <textarea
          id="description"
          bind:value={description}
          onkeydown={handleKeyDown}
          placeholder="E.g., Create a flowchart showing the process of user login, with steps for entering credentials, validation, and successful/failed login scenarios"
          rows="4"
          class="flex min-h-[80px] w-full rounded-md border border-input bg-transparent px-3 py-2 text-sm shadow-sm placeholder:text-muted-foreground focus-visible:ring-1 focus-visible:ring-ring focus-visible:outline-none disabled:cursor-not-allowed disabled:opacity-50"
        ></textarea>
      </div>

      <div class="flex flex-col gap-2">
        <label for="api-key" class="text-sm font-medium">
          OpenAI API Key
          <span class="ml-2 text-xs text-muted-foreground">(stored locally in your browser)</span>
        </label>
        <div class="flex gap-2">
          <Input
            id="api-key"
            type="password"
            bind:value={apiKey}
            placeholder="sk-..."
            class="flex-1" />
          <Button variant="outline" onclick={saveApiKey} disabled={!apiKey.trim()}>Save</Button>
        </div>
        <p class="text-xs text-muted-foreground">
          Get your API key from <a
            href="https://platform.openai.com/api-keys"
            target="_blank"
            rel="noopener noreferrer"
            class="underline">OpenAI Platform</a>
        </p>
      </div>

      {#if error}
        <div class="rounded-md bg-destructive/10 p-3 text-sm text-destructive">
          {error}
        </div>
      {/if}
    </div>

    <Dialog.Footer>
      <Button
        variant="outline"
        onclick={() => {
          open = false;
        }}>
        Cancel
      </Button>
      <Button onclick={generateDiagram} disabled={loading || !description.trim() || !apiKey.trim()}>
        {loading ? 'Generating...' : 'Generate Diagram'}
      </Button>
    </Dialog.Footer>
  </Dialog.Content>
</Dialog.Root>
