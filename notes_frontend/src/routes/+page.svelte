<script lang="ts">
    import { onMount } from 'svelte';
    import { notes, selectedNoteId, fetchNotes, createNote, updateNote, deleteNote } from '$lib/api';
    import type { Note } from '$lib/api';
    import NotesList from '$lib/NotesList.svelte';
    import NoteEditor from '$lib/NoteEditor.svelte';
    import FloatingAddButton from '$lib/FloatingAddButton.svelte';
    import { get } from 'svelte/store';

    let loading = false, error = '', initialLoad = true;

    let selectedNote: Note|null = null;
    let isCreating = false;

    // Fetch initial notes on mount
    onMount(async () => {
        loading = true; error = '';
        try {
            notes.set(await fetchNotes());
        } catch (err) {
            error = err?.message ?? "Could not load notes.";
        }
        loading = false; initialLoad = false;
    });

    $: selectedNote =
        isCreating
            ? null
            : ($notes.find((n) => n.id === $selectedNoteId) ?? null);

    // Select note for viewing/editing
    function selectNote(id: number) {
        selectedNoteId.set(id);
        isCreating = false;
    }

    // Start creating a new note
    function beginNewNote() {
        isCreating = true;
        selectedNoteId.set(null);
    }

    // Save note (create or update)
    async function handleSaveNote(note) {
        try {
            loading = true; error = '';
            if (isCreating) {
                const newNote = await createNote({title: note.title, content: note.content});
                notes.update(nlist => [...nlist, newNote]);
                selectedNoteId.set(newNote.id);
                isCreating = false;
            } else if (selectedNote) {
                const updNote = await updateNote(selectedNote.id, note);
                notes.update(nlist =>
                    nlist.map(n => n.id === selectedNote.id ? updNote : n)
                );
            }
        } catch (err) {
            error = err?.message || "Save failed";
        }
        loading = false;
    }

	// Delete a note
    async function handleDeleteNote(id:number) {
        let doDelete = confirm('Delete this note?');
        if (!doDelete) return;
        try {
            loading = true; error = '';
            await deleteNote(id);
            notes.update(nlist => nlist.filter(n => n.id !== id));
            // If the deleted note is selected, deselect or fallback to another note
            if ($selectedNoteId === id) {
                const remaining = get(notes);
                if (remaining.length) selectedNoteId.set(remaining[0].id);
                else selectedNoteId.set(null);
            }
        } catch (err) {
            error = err?.message || "Delete failed";
        }
        loading = false;
    }

</script>

<svelte:head>
	<title>Notes</title>
	<meta name="description" content="Minimal notes app" />
</svelte:head>

<div class="notes-app">
    <aside>
        <NotesList
            onSelect={selectNote}
            onDelete={handleDeleteNote}
        />
    </aside>
    <main>
        {#if loading && initialLoad}
            <div class="center-msg">Loading notes...</div>
        {:else if error}
            <div class="center-msg error">{error}</div>
        {:else if isCreating}
            <NoteEditor note={null} isNew={true} onSave={handleSaveNote} />
        {:else if selectedNote}
            <NoteEditor note={selectedNote} onSave={handleSaveNote} />
        {:else}
            <div class="center-msg">Select or add a note to get started.</div>
        {/if}
    </main>
    <FloatingAddButton onClick={beginNewNote} />
</div>

<style>
.notes-app {
    width: 100%;
    height: 100%;
    min-height: 70vh;
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: stretch;
    margin: 0 auto;
    position: relative;
    gap: 1.3rem;
    box-sizing: border-box;
    background: transparent;
    padding: 0.3rem 0.35rem 2rem 0.35rem;
    max-width: 80rem;
}

aside {
    width: 100%;
    max-width: 300px;
    margin-right: 0.15rem;
}

main {
    flex: 1 1 auto;
    min-width: 0;
    display: flex;
    align-items: flex-start;
    justify-content: flex-start;
    background: var(--color-bg-1, #f8f9fa);
    border-radius: 14px;
    padding: 1.1rem 1.55rem;
    box-shadow: 0 0 16px #dde7fa33;
    min-height: 70vh;
    position: relative;
    margin-left: 0.08rem;
}
.center-msg {
    width: 100%;
    color: #888;
    font-size: 1.09rem;
    text-align: center;
    margin-top: 2.4rem;
}
.center-msg.error {
    color: #d52222;
    margin-top: 1rem;
}
@media (max-width: 900px) {
    .notes-app {
        flex-direction: column;
        gap: 0.2rem;
        padding: 0;
    }
    aside {
        max-width: none; width: 100%;
        margin: 0;
    }
    main {
        min-height: 260px;
        padding: 1rem 0.4rem;
    }
}
</style>
