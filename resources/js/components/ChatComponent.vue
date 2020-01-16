<template>
    <div class="card">
        <div class="card-header">{{ otherUser.name }}</div>

        <div class="card-body">
            <div v-for="message in messages" v-bind:key="message.id">
                <div
                    class="mb-1"
                    :class="{ 'text-right': message.author === authUser.email }"
                >
                    <template v-if="message.type === 'media'">
                        <img
                            class="img-thumbnail"
                            :src="message.mediaUrl"
                            :alt="message.filename"
                            width="150px">
                    </template>
                    <template v-else>
                        {{ message.body }}
                    </template>
                </div>
            </div>
        </div>

        <div class="card-footer">
            <div class="form-row">
                <div class="form-group col-md-8">
                    <input
                        type="text"
                        v-model="newMessage"
                        class="form-control"
                        placeholder="Type your message..."
                        @keyup.enter="sendMessage"
                    />
                </div>
                <div class="form-group col-md-4">
                    <input type="file" accept="image/*" @change="sendMediaMessage">
                </div>
            </div>
        </div>
    </div>
</template>

<script>
export default {
    name: "ChatComponent",
    props: {
        authUser: {
            type: Object,
            required: true
        },
        otherUser: {
            type: Object,
            required: true
        }
    },
    data() {
        return {
            messages: [],
            newMessage: "",
            channel: "",
        };
    },
    async created() {
        const token = await this.fetchToken();
        await this.initializeClient(token);
        await this.fetchMessages();
    },
    methods: {
        async fetchToken() {
            const { data } = await axios.post("/api/token", {
                email: this.authUser.email
            });

            return data.token;
        },
        async initializeClient(token) {
            const client = await Twilio.Chat.Client.create(token);

            client.on("tokenAboutToExpire", async () => {
                const token = await this.fetchToken();

                client.updateToken(token);
            });

            this.channel = await client.getChannelByUniqueName(
                `${this.authUser.id}-${this.otherUser.id}`
            );

            this.channel.on("messageAdded", message => {
                this.pushToArray(message)
            });
        },
        async fetchMessages() {
            const messages = (await this.channel.getMessages()).items;

            for (const message of messages) {
                this.pushToArray(message)
            }
        },
        sendMessage() {
            this.channel.sendMessage(this.newMessage);

            this.newMessage = "";
        },
        sendMediaMessage({ target }) {
            const formData = new FormData();
            formData.append('file', target.files[0]);

            this.channel.sendMessage(formData);

            target.value = "";
        },
        async pushToArray (message) {
            if (message.type === 'media') {
                const mediaUrl = await message.media.getContentUrl()

                this.messages.push({
                    type: message.type,
                    author: message.author,
                    filename: message.media.filename,
                    mediaUrl
                })
            } else {
                this.messages.push({
                    type: message.type,
                    author: message.author,
                    body: message.body,
                })
            }
        }
    }
};
</script>
