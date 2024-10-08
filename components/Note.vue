<script setup lang="ts">
import {
    type GraffitiObjectTyped,
    useGraffitiSession,
} from "@graffiti-garden/client-vue";
import Name from "./Name.vue";
import Notes from "./Notes.vue";
import { noteSchema } from "./schemas";
import { ref, computed } from "vue";

const session = useGraffitiSession();

const props = withDefaults(
    defineProps<{
        note: GraffitiObjectTyped<ReturnType<typeof noteSchema>>;
        follows: string[];
        inReplyToContentAddress: string;
    }>(),
    {
        follows: () => [],
        inReplyToContentAddress: "",
    },
);

const formattedTimestamp = computed(() =>
    props.note
        ? new Date(props.note.value.createdAt).toLocaleDateString(undefined, {
              month: "long",
              day: "numeric",
              year: "numeric",
              hour: "numeric",
              minute: "numeric",
          })
        : "",
);

const lookup: { [key: string]: string } = {
    "&": "&amp;",
    '"': "&quot;",
    "'": "&apos;",
    "<": "&lt;",
    ">": "&gt;",
};
const sanitizedContent = computed(() =>
    props.note.value.content
        // Remove all included tags
        .replace(/[&"'<>]/g, (c) => lookup[c])
        // Replace urls with <a>
        .replace(
            /(https?:\/\/[^\s]+)/g,
            (url) => `<a target="_blank" href="${url}">${url}</a>`,
        ),
);

const commentsOpen = ref(false);
const editMenuOpen = ref(false);
const editing = ref(false);
const editText = ref("");
</script>

<template>
    <h1>
        <Name :webId="note.webId" />
        <template v-if="note.value.at">
            <template v-for="webId in note.value.at" :key="webId">
                @<Name :webId="webId" />
            </template>
        </template>
    </h1>

    <h2>
        {{ formattedTimestamp }}
    </h2>

    <div class="modifiers" v-click-away="() => (editMenuOpen = false)">
        <input
            type="checkbox"
            :id="'menu' + $graffiti.locationToUrl(note)"
            :checked="editMenuOpen"
            @click="editMenuOpen = !editMenuOpen"
        />
        <label :for="'menu' + $graffiti.locationToUrl(note)">⚙️</label>

        <menu v-if="editMenuOpen" @click="editMenuOpen = false">
            <template v-if="note.webId === session.webId">
                <li v-if="!editing">
                    <button
                        @click="
                            editing = true;
                            editText = note.value.content;
                        "
                    >
                        edit
                    </button>
                </li>
                <li>
                    <button @click="$graffiti.delete(note, session)">
                        delete
                    </button>
                </li>
            </template>
            <li>
                <a
                    target="_blank"
                    class="button"
                    :href="$graffiti.locationToUrl(note)"
                    >link</a
                >
            </li>
        </menu>
    </div>

    <form
        v-if="editing && session.webId"
        @submit.prevent="
            $graffiti
                .patch(
                    {
                        value: [
                            {
                                op: 'replace',
                                path: '/content',
                                value: editText,
                            },
                        ],
                    },
                    note,
                    session,
                )
                .then(() => (editing = false))
        "
    >
        <textarea v-model="editText" v-focus> </textarea>
        <div class="edit-buttons">
            <input type="button" value="cancel" @click="editing = false" />
            <input type="submit" value="save" />
        </div>
    </form>
    <p v-else v-html="sanitizedContent"></p>

    <div class="post-annotators">
        <input
            type="checkbox"
            :id="'comments' + $graffiti.locationToUrl(note)"
            :checked="commentsOpen"
            @click="commentsOpen = !commentsOpen"
        />
        <label :for="'comments' + $graffiti.locationToUrl(note)">
            comments
        </label>
    </div>

    <Notes
        v-if="commentsOpen"
        :inReplyTo="$graffiti.locationToUrl(note)"
        prompt="write a comment..."
    />
</template>
