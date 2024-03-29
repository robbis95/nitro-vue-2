
<script lang="ts">
import { Nitro, NitroPoint, RoomDragEvent } from "@nitrots/nitro-renderer";
import store from "../../../../../utils/store";
import { RoomWidgetUpdateChatEvent } from "../../events";
import { ChatBubbleMessage } from "./utils/ChatBubbleMessage";
import ChatBubble from "./chatBubble/ChatBubble.vue";

export default {
    props: ["roomSession"],
    components: { ChatBubble },
    data() {
        return {
            interval: null,
            chatInt: null,
            chatMessages: []
        };
    },
    mounted() {
        store.state.room.widgetHandler.eventDispatcher.addEventListener(
            RoomWidgetUpdateChatEvent.CHAT_EVENT,
            this.onRoomWidgetUpdateChatEvent.bind(this)
        );

        this.interval = setInterval(() => this.moveAllChatsUp(15), 6000);

        Nitro.instance.roomEngine.events.addEventListener(
            RoomDragEvent.ROOM_DRAG,
            this.onDrag.bind(this)
        );
        store.state.room.chat = [];
        store.state.room.history = false;
        store.state.room.historyDragged = false;
    },
    beforeUnmount() {
        clearInterval(this.interval);

        store.state.room.widgetHandler.eventDispatcher.removeEventListener(
            RoomWidgetUpdateChatEvent.CHAT_EVENT,
            this.onRoomWidgetUpdateChatEvent.bind(this)
        );

        Nitro.instance.roomEngine.events.removeEventListener(
            RoomDragEvent.ROOM_DRAG,
            this.onDrag.bind(this)
        );
    },
    methods: {
        onDrag(event: RoomDragEvent) {
            if (!this.chatMessages.length) return;

            if (event.roomId !== this.roomSession.roomId) return;

            this.chatMessages.forEach(chat => {
                if (!chat.elementRef) return;

                chat.left += event.offsetX;
            });
        },
        removeHiddenChats() {
            if (!this.chatMessages.length) return;

            const newMessages = this.chatMessages.filter(
                chat => chat.top > -chat.height * 2
            );

            if (newMessages.length !== this.chatMessages.length)
                this.chatMessages = newMessages;
        },
        moveChatUp(chat: ChatBubbleMessage, amount: number) {
            chat.top -= amount;
        },
        moveAllChatsUp(amount) {
            this.chatMessages.forEach(chat => this.moveChatUp(chat, amount));
            this.removeHiddenChats();
        },
        makeRoom(chat: ChatBubbleMessage) {
            const lowestPoint = chat.top + chat.height - 1;
            const requiredSpace = chat.height + 1;
            const spaceAvailable =
                this.$refs["roomchat"].offsetHeight - lowestPoint;

            if (spaceAvailable < requiredSpace) {
                const amount = requiredSpace - spaceAvailable;

                this.chatMessages.forEach(existingChat => {
                    if (existingChat === chat) return;

                    this.moveChatUp(existingChat, amount);
                });
            }
        },
        onRoomWidgetUpdateChatEvent(event: RoomWidgetUpdateChatEvent) {
            this.chatMessages.push(this.newChatMessage(event));

            if (this.chatInt) clearInterval(this.chatInt);

            if (!store.state.room.historyDragged) {
                store.state.room.history = false;

                this.chatInt = setTimeout(() => {
                    store.state.room.history = true;
                }, parseInt(Nitro.instance.getConfiguration("chat.history.delay.seconds")) * 1000);
            }

            store.state.room.chat.push(
                Object.assign(this.newChatMessage(event))
            );
        },
        newChatMessage(event: RoomWidgetUpdateChatEvent) {
            return new ChatBubbleMessage(
                event.userId,
                event.userCategory,
                event.roomId,
                event.text,
                event.userName,
                new NitroPoint(event.userX, event.userY),
                event.chatType,
                event.styleId,
                event.userImage,
                event.userColor &&
                    ("#" + event.userColor.toString(16).padStart(6, "0") ||
                        null)
            );
        }
    }
};
</script>

<template>
    <div
        class="room-chat"
        ref="roomchat"
        :style="$store.state.room.historyDragged ? `visibility:hidden;` : ``"
    >
        <ChatBubble
            v-for="(r,i) in chatMessages"
            :key="i"
            :chat="r"
            :fixed="false"
            @makeroom="makeRoom"
        />
    </div>
</template>