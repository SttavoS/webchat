<template>
    <app-layout>
        <template #header>
            <h2 class="font-semibold text-xl text-gray-800 leading-tight">
                Chat
            </h2>
        </template>

        <div class="py-12">
            <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
                <div class="chat-container bg-white overflow-hidden shadow-xl sm:rounded-lg flex">
                    <!-- User List -->
                    <div class="user-list-wrapper bg-scroll w-3/12 bg-gray-200 bg-opacity-25 border-r border-gray-200 overflow-y-scroll">
                        <ul>
                            <li
                                v-for="user in users"
                                :key="user.id"
                                @click="() => {loadMessages(user.id)}"
                                :class="(activeUser && activeUser.id == user.id) ? 'bg-gray-200 bg-opacity-50' : ''"
                                class="p-6 text-lg text-gray-600 leading-7 font-semibold border-b border-gray-200 hover:bg-gray-200 hover:bg-opacity-50 hover:cursor-pointer"
                            >
                                <p class="flex items-center">
                                    {{ user.name }}
                                    <span v-if="user.notification" class="ml-2 w-2 h-2 bg-blue-500 rounded-full"></span>
                                </p>
                            </li>
                        </ul>
                    </div>

                    <!-- Message Box -->
                    <div class="w-9/12 flex flex-col justify-between">
                        <!-- Message -->
                        <div class="w-full p-6 flex flex-col overflow-y-scroll">
                            <div
                                v-for="message in messages"
                                :key="message.id"
                                :class="(message.from == $page.auth.user.id) ? 'text-right' : ''"
                                class="message w-full mb-3"
                            >
                                <p
                                    class="inline-block p-2 rounded-md messageFromMe"
                                    :class="(message.from == $page.auth.user.id) ? 'messageFromMe' : 'messageToMe'"
                                >
                                    {{ message.content }}
                                </p>
                                    <span class="block mt-1 text-xs text-gray-500">
                                        {{ message.created_at | formatDate }}
                                    </span>
                            </div>
                        </div>

                        <div v-if="activeUser" class="w-full bg-gray-200 bg-opacity-25 p-6 border-t border-gray-200">
                            <form @submit.prevent="sendMessage">
                                <div class="flex rounded-md overflow-hidden border border-gray-300">
                                    <input
                                        v-model="message"
                                        type="text"
                                        placeholder="Digite a sua mensagem"
                                        class="flex-1 px-4 py-2 text-sm focus:outline-none"
                                    >
                                    <button type="submit" class="bg-indigo-500 hover:bg-indigo-600 text-white px-4 py-2">Enviar</button>
                                </div>
                            </form>
                        </div>
                    </div>

                </div>
            </div>
        </div>
    </app-layout>
</template>

<script>
    import Vue from 'vue';
    import AppLayout from '@/Layouts/AppLayout';
    import store from '@/store';

    export default {
        components: {
            AppLayout
        },
        data() {
            return {
                users: [],
                activeUser: null,
                messages: [],
                message: ''
            }
        },
        computed: {
            user() {
                return store.state.user;
            }
        },
        methods: {
            scrollToBottom() {
                if (this.messages.length) {
                    document.
                        querySelectorAll('.message:last-child')[0].
                        scrollIntoView({
                            behavior: 'smooth',
                            block: 'start'
                        });
                }
            },
            async loadMessages(userId) {
                axios.get(`api/users/${userId}`)
                    .then(response => {
                        this.activeUser = response.data.user;
                    });

                await axios.get(`api/messages/${userId}`)
                    .then(response => {
                        this.messages = response.data.messages;
                    });

                const user = this.users.filter((user) => {
                    if (user.id === userId) {
                        return user;
                    }
                });

                if (user) {
                    Vue.set(user[0], 'notification', false);
                }

                this.scrollToBottom();
            },
            async sendMessage() {
                await axios.post('api/messages/store', {
                    'content': this.message,
                    'to': this.activeUser.id
                }).then(response => {
                    this.messages.push({
                        'from': this.user.id,
                        'to': this.activeUser.id,
                        'content': this.message,
                        'created_at': new Date().toISOString(),
                        'updated_at': new Date().toISOString(),
                    });
                    this.message = '';
                });

                this.scrollToBottom();
            },
        },
        mounted() {
            axios.get('api/users')
                .then(response => {
                    this.users = response.data.users;
                });

            Echo.private(`user.${this.user.id}`)
                .listen('.SendMessage', async (event) => {
                    if (this.activeUser && this.activeUser.id === event.message.from) {
                        await this.messages.push(event.message);
                        this.scrollToBottom();
                    } else {
                        const user = this.users.filter((user) => {
                            if (user.id === event.message.from) {
                                return user;
                            }
                        });

                        if (user) {
                            Vue.set(user[0], 'notification', true);
                        }
                    }
                });
        }
    }
</script>

<style scoped>
    .chat-container {
        min-height: 400px;
        max-height: 400px;
    }

    .messageFromMe {
        max-width: 75%;
        background-color: rgba(180, 198, 252, 0.25);
    }

    .messageToMe {
        max-width: 75%;
        background-color: rgba(210, 214, 220, 0.25);
    }

    ::-webkit-scrollbar {
        width: 8px;
    }

    ::-webkit-scrollbar-button {
        display:none;
    }

    ::-webkit-scrollbar-thumb {
        background: #babac0;
        border-radius: 4px;
    }
</style>
