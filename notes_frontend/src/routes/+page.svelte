<script lang="ts">
  import { onMount } from 'svelte';
  import { writable, derived } from 'svelte/store';
  import Modal from '../lib/Modal.svelte';
  import NoteEditor from '../lib/NoteEditor.svelte';
  import { getNotes, createNote, updateNote, deleteNote } from '../lib/api';
  import type { Note } from '../lib/types';

  // App State
  const notes = writable<Note[]>([]);
  const loading = writable(true);
  const selectedId = writable<string | null>(null);
  const searchInput = writable('');
  const showModal = writable(false);
  const editingNote = writable<Note | null>(null);

  // Derived filtered notes, real time search
  const filteredNotes = derived([notes, searchInput], ([$notes, $searchInput]) => {
    const lower = $searchInput.toLowerCase();
    return $notes.filter(
      (note) => note.title.toLowerCase().includes(lower) || note.content.toLowerCase().includes(lower)
    );
  });

  // Load notes on mount
  onMount(async () => {
    loading.set(true);
    notes.set(await getNotes());
    loading.set(false);
  });

  // Select a note from list
  function selectNote(id: string) {
    selectedId.set(id);
    editingNote.set(null);
    showModal.set(false);
  }

  // Open modal or editor for new note
  function onAddNote() {
    editingNote.set({ id: '', title: '', content: '', created_at: '', updated_at: '' });
    showModal.set(true);
  }

  // Open modal for editing
  function onEditNote(note: Note) {
    editingNote.set({ ...note });
    showModal.set(true);
  }

  // Save note (add or update)
  async function onSaveNote(note: Note) {
    loading.set(true);
    let updated;
    if (note.id) {
      updated = await updateNote(note);
    } else {
      updated = await createNote(note);
    }
    notes.set(await getNotes());
    selectedId.set(updated.id ?? updated._id ?? '');
    showModal.set(false);
    editingNote.set(null);
    loading.set(false);
  }

  // Delete note
  async function onDeleteNote(note: Note) {
    if (confirm('Delete this note? This cannot be undone.')) {
      loading.set(true);
      await deleteNote(note.id);
      notes.set(await getNotes());
      selectedId.set(null);
      showModal.set(false);
      editingNote.set(null);
      loading.set(false);
    }
  }
</script>

<svelte:head>
  <title>Notes</title>
</svelte:head>

<div class="app-shell">
  <aside class="sidebar">
    <div class="branding">Notes</div>
    <button class="accent add-btn" on:click={onAddNote}>+ New Note</button>
    <input
      class="search"
      type="text"
      placeholder="Search notes"
      bind:value={$searchInput}
      autocomplete="off"
    />
    <div class="note-list">
      {#if $loading}
        <div class="loading">Loading...</div>
      {:else if $filteredNotes.length === 0}
        <div class="empty-state">No notes found.</div>
      {:else}
        {#each $filteredNotes as note (note.id)}
        <button
          class="note-item {note.id === $selectedId ? 'active' : ''}"
          type="button"
          aria-pressed={note.id === $selectedId}
          on:click={() => selectNote(note.id)}
          title={note.title}
          role="option"
          tabindex="0"
        >
          <span class="note-title">
            {#if note.title && note.title.trim().length}
              {note.title}
            {:else}
              <em>Untitled</em>
            {/if}
          </span>
          <span class="note-date">{note.updated_at?.slice(0, 10)}</span>
        </button>
        {/each}
      {/if}
    </div>
  </aside>
  <main class="main">
    {#if $selectedId}
      {#if $loading}
        <div class="loading">Loading...</div>
      {:else}
        {#each $notes.filter(n => n.id === $selectedId) as note (note.id)}
          <section class="note-view">
            <header>
              <h1>
                {#if note.title && note.title.trim().length}
                  {note.title}
                {:else}
                  <em>Untitled</em>
                {/if}
              </h1>
              <div class="meta">Last updated: {note.updated_at?.slice(0, 16).replace('T', ' ')}</div>
            </header>
            <article>{note.content}</article>
            <div class="note-actions">
              <button class="accent" on:click={() => onEditNote(note)}>Edit</button>
              <button class="danger" on:click={() => onDeleteNote(note)}>Delete</button>
            </div>
          </section>
        {/each}
      {/if}
    {:else}
      <div class="prompt">Select a note or create a new one.</div>
    {/if}
  </main>
</div>

{#if $showModal}
  <Modal on:close={() => showModal.set(false)}>
    <NoteEditor
      note={$editingNote}
      on:save={e => onSaveNote(e.detail)}
      on:delete={e => onDeleteNote(e.detail)}
      on:cancel={() => showModal.set(false)}
    />
  </Modal>
{/if}

<style>
.app-shell {
  display: flex;
  min-height: 100vh;
  background: var(--color-background);
}

.sidebar {
  background: var(--color-secondary, #f5f5f5);
  width: 320px;
  min-width: 220px;
  max-width: 400px;
  display: flex;
  flex-direction: column;
  border-right: 1px solid #ececec;
  padding: 2rem 1rem 1rem 1.5rem;
  box-sizing: border-box;
}

.branding {
  font-weight: 700;
  font-size: 1.5rem;
  margin-bottom: 1rem;
  color: var(--color-primary, #1e90ff);
  letter-spacing: 2px;
}

.add-btn {
  padding: 0.5rem 0.75rem;
  margin-bottom: 1rem;
  background: var(--color-accent, #ffab00);
  color: #fff;
  border: none;
  border-radius: 6px;
  font-size: 1rem;
  cursor: pointer;
  transition: background 0.15s;
}
.add-btn:hover { background: #ff9800; }

.search {
  margin-bottom: 1.25rem;
  padding: 0.35rem 0.75rem;
  border-radius: 5px;
  border: 1px solid #ddd;
  font-size: 1rem;
  outline: none;
  background: #fff;
  color: #111;
}

.note-list {
  flex: 1 1 0;
  overflow-y: auto;
  margin-top: 0.25rem;
  margin-right: 0.5rem;
}
.note-item {
  border-radius: 5px;
  padding: 0.75rem 0.5rem 0.75rem 0.75rem;
  margin-bottom: 0.25rem;
  background: transparent;
  cursor: pointer;
  font-size: 1.04rem;
  display: flex;
  flex-direction: column;
  gap: 0.15rem;
  transition: background 0.13s;
}
.note-item.active, .note-item:hover {
  background: var(--color-primary, #1e90ff);
  color: #fff;
}
.note-title {
  font-weight: 500;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}
.note-date {
  font-size: 0.86em;
  color: var(--color-text-secondary, #666);
  opacity: 0.9;
}
.loading,
.empty-state,
.prompt {
  color: #aaa;
  margin: 1.5rem 0 0 1.2rem;
  font-size: 1.1rem;
}
.main {
  flex: 1 1 0;
  display: flex;
  flex-direction: column;
  padding: 3rem 3.5rem;
  min-width: 0;
}
.note-view header {
  margin-bottom: 1.5rem;
}
.note-view h1 {
  margin: 0;
  font-size: 2.2rem;
  font-weight: 600;
}
.note-view .meta {
  color: #99a;
  font-size: 0.98rem;
  margin-top: 0.2rem;
}
.note-view article {
  white-space: pre-wrap;
  margin: 1.3rem 0 2rem 0;
  font-size: 1.14rem;
}
.note-actions {
  display: flex;
  gap: 1rem;
}
.note-actions .accent {
  background: var(--color-primary, #1e90ff);
  color: #fff;
  border: none;
}
.note-actions .accent:hover { background: #176dc1; }
.note-actions .danger {
  background: #f04d54;
  color: white;
  border: none;
}
.note-actions .danger:hover { background: #da2121; }
@media (max-width: 650px) {
  .main { padding: 1.5rem 0.5rem; }
  .sidebar { width: 95vw; }
}
</style>
