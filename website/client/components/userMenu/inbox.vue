<template lang="pug">
  b-modal#inbox-modal(title="", :hide-footer="true", size='lg', @shown="onModalShown", @hide="onModalHide")
    .header-wrap.container.align-items-center(slot="modal-header")
      .row.align-items-center
        .col-4
          .row.align-items-center
            .col-2
              .svg-icon.envelope(v-html="icons.messageIcon")
            .col-6
              h2.text-center(v-once) {{ $t('messages') }}
        .col-4.offset-3
          toggle-switch.float-right(
            :label="optTextSet.switchDescription",
            :checked="!this.user.inbox.optOut"
            :hoverText="optTextSet.popoverText",
            @change="toggleOpt()"
          )
        .col-1
          .close
            span.svg-icon.inline.icon-10(aria-hidden="true", v-html="icons.svgClose", @click="close()")
    .row
      .col-4.sidebar
        .search-section
          b-form-input(:placeholder="$t('search')", v-model='search')
        .empty-messages.text-center(v-if='filtersConversations.length === 0')
          .svg-icon.envelope(v-html="icons.messageIcon")
          h4(v-once) {{$t('emptyMessagesLine1')}}
          p(v-if="!user.flags.chatRevoked") {{$t('emptyMessagesLine2')}}
        .conversations(v-if='filtersConversations.length > 0')
          .conversation(v-for='conversation in filtersConversations', @click='selectConversation(conversation.key)',
            :class="{active: selectedConversation.key === conversation.key}")
            div
             span(:class="userLevelStyle(conversation)") {{conversation.name}}
             span.timeago {{conversation.date | timeAgo}}
            div {{conversation.lastMessageText ? conversation.lastMessageText.substring(0, 30) : ''}}
      .col-8.messages.d-flex.flex-column.justify-content-between
        .empty-messages.text-center(v-if='!selectedConversation.key')
          .svg-icon.envelope(v-html="icons.messageIcon")
          h4 {{placeholderTexts.title}}
          p(v-html="placeholderTexts.description")
        .empty-messages.text-center(v-if='selectedConversation.key && selectedConversationMessages.length === 0')
          p {{ $t('beginningOfConversation', {userName: selectedConversation.name})}}
        chat-messages.message-scroll(
          v-if="selectedConversation.messages && selectedConversationMessages.length > 0",
          :chat='selectedConversationMessages',
          :inbox='true',
          @message-removed='messageRemoved',
          ref="chatscroll"
        )
        .pm-disabled-caption.text-center(v-if="user.inbox.optOut && selectedConversation.key")
          h4 {{$t('PMDisabledCaptionTitle')}}
          p {{$t('PMDisabledCaptionText')}}
        .new-message-row(v-if='selectedConversation.key && !user.flags.chatRevoked')
          textarea(
            v-model='newMessage',
            @keyup.ctrl.enter='sendPrivateMessage()',
            @focus='checkUser()',
            :disabled='textAreaDisabled',
            maxlength='3000'
          )
          button.btn.btn-secondary(
            @click='sendPrivateMessage()',
            :disabled='sendBtnDisabled'
          ) {{$t('send')}}
          .row
            span.ml-3 {{ currentLength }} / 3000
</template>

<style lang="scss" scoped>
  @import '~client/assets/scss/colors.scss';

  .header-wrap {
    padding: 0.5em;

    h2 {
      margin: 0;
      line-height: 1;
    }
  }

  .envelope {
    color: $gray-400 !important;
    margin: 0;
  }

  .sidebar {
    background-color: $gray-700;
    min-height: 600px;
    padding: 0;

    .search-section {
      padding: 1em;
      box-shadow: 0 1px 2px 0 rgba(26, 24, 29, 0.24);
    }
  }

  .messages {
    position: relative;
    padding-left: 0;
    padding-bottom: 6em;
  }

  .message-scroll {
    max-height: 500px;
    overflow-x: scroll;

    @media (min-width: 992px) {
      overflow-x: hidden;
      overflow-y: scroll;
    }
  }

  .to-form input {
    width: 60%;
    display: inline-block;
    margin-left: 1em;
  }

  .empty-messages {
    margin-top: 10em;
    color: $gray-400;
    padding: 1em;

    h4 {
      color: $gray-400;
      margin-top: 1em;
    }

    .envelope {
      width: 30px;
      margin: 0 auto;
    }
  }

  .pm-disabled-caption {

    padding-top: 1em;
    background-color: $gray-700;
    z-index: 2;

    h4, p {
      color: $gray-300;
    }

    h4 {
      margin-top: 0;
      margin-bottom: 0.4em;
    }

    p {
      font-size: 12px;
      margin-bottom: 0;
    }
  }

  .new-message-row {
    background-color: $gray-700;
    position: absolute;
    bottom: 0;
    height: 88px;
    width: 100%;
    padding: 1em;

    textarea {
      height: 80%;
      display: inline-block;
      vertical-align: bottom;
      width: 80%;
    }

    button {
      vertical-align: bottom;
      display: inline-block;
      box-shadow: none;
      margin-left: 1em;
    }
  }

  .conversations {
    max-height: 400px;
    overflow-x: hidden;
    overflow-y: scroll;
  }

  .conversation {
    padding: 1.5em;
    background: $white;
    height: 80px;

    .timeago {
      margin-left: 1em;
    }
  }

  .conversation.active {
    border: 1px solid $purple-400;
  }

  .conversation:hover {
    cursor: pointer;
  }
</style>

<script>
import Vue from 'vue';
import moment from 'moment';
import filter from 'lodash/filter';
import sortBy from 'lodash/sortBy';
import groupBy from 'lodash/groupBy';
import { mapState } from 'client/libs/store';
import styleHelper from 'client/mixins/styleHelper';
import toggleSwitch from 'client/components/ui/toggleSwitch';
import axios from 'axios';

import messageIcon from 'assets/svg/message.svg';
import chatMessages from '../chat/chatMessages';
import svgClose from 'assets/svg/close.svg';

export default {
  mixins: [styleHelper],
  components: {
    chatMessages,
    toggleSwitch,
  },
  mounted () {
    this.$root.$on('habitica::new-inbox-message', (data) => {
      this.$root.$emit('bv::show::modal', 'inbox-modal');

      // Wait for messages to be loaded
      const unwatchLoaded = this.$watch('loaded', (loaded) => {
        if (!loaded) return;

        const conversation = this.conversations.find(convo => {
          return convo.key === data.userIdToMessage;
        });
        if (loaded) setImmediate(() => unwatchLoaded());

        if (conversation) {
          this.selectConversation(data.userIdToMessage);
          return;
        }

        this.initiatedConversation = {
          user: data.userName,
          uuid: data.userIdToMessage,
        };

        this.selectConversation(data.userIdToMessage);
      }, {immediate: true});
    });
  },
  destroyed () {
    this.$root.$off('habitica::new-inbox-message');
  },
  data () {
    return {
      icons: Object.freeze({
        messageIcon,
        svgClose,
      }),
      displayCreate: true,
      selectedConversation: {},
      search: '',
      newMessage: '',
      showPopover: false,
      messages: [],
      loaded: false,
      initiatedConversation: null,
      sendBtnDisabled: false,
      textAreaDisabled: false,
    };
  },
  filters: {
    timeAgo (value) {
      return moment(new Date(value)).fromNow();
    },
  },
  computed: {
    ...mapState({user: 'user.data'}),
    conversations () {
      const inboxGroup = groupBy(this.messages, 'uuid');

      // Add placeholder for new conversations
      if (this.initiatedConversation && this.initiatedConversation.uuid) {
        inboxGroup[this.initiatedConversation.uuid] = [{
          id: '',
          text: '',
          user: this.initiatedConversation.user,
          uuid: this.initiatedConversation.uuid,
          timestamp: new Date(),
        }];
      }
      // Create conversation objects
      const convos = [];
      for (let key in inboxGroup) {
        const convoSorted = sortBy(inboxGroup[key], [(o) => {
          return (new Date(o.timestamp)).getTime();
        }]);

        // Fix poor inbox chat models
        const newChatModels = convoSorted.map(chat => {
          let newChat = Object.assign({}, chat);
          if (newChat.sent) {
            newChat.toUUID = newChat.uuid;
            newChat.toUser = newChat.user;
            newChat.uuid = this.user._id;
            newChat.user = this.user.profile.name;
            newChat.contributor = this.user.contributor;
            newChat.backer = this.user.backer;
          }
          return newChat;
        });

        // In case the last message is a placeholder, remove it
        const recentMessage = newChatModels[newChatModels.length - 1];
        if (!recentMessage.text) newChatModels.splice(newChatModels.length - 1, 1);

        const convoModel = {
          name: recentMessage.toUser ? recentMessage.toUser : recentMessage.user, // Handles case where from user sent the only message or the to user sent the only message
          key: recentMessage.toUUID ? recentMessage.toUUID : recentMessage.uuid,
          messages: newChatModels,
          lastMessageText: recentMessage.text,
          date: recentMessage.timestamp,
        };

        convos.push(convoModel);
      }

      // Sort models by most recent
      const conversations = sortBy(convos, [(o) => {
        return moment(o.date).toDate();
      }]);

      return conversations.reverse();
    },
    // Separate from selectedConversation which is not coputed so messages don't update automatically
    selectedConversationMessages () {
      const selectedConversationKey = this.selectedConversation.key;
      const selectedConversation = this.conversations.find(c => c.key === selectedConversationKey);
      return selectedConversation ? selectedConversation.messages : [];
    },
    filtersConversations () {
      if (!this.search) return this.conversations;
      return filter(this.conversations, (conversation) => {
        return conversation.name.toLowerCase().indexOf(this.search.toLowerCase()) !== -1;
      });
    },
    currentLength () {
      return this.newMessage.length;
    },
    placeholderTexts () {
      if (this.user.flags.chatRevoked) {
        return {
          title: this.$t('PMPlaceholderTitleRevoked'),
          description: this.$t('PMPlaceholderDescriptionRevoked'),
        };
      }
      return {
        title: this.$t('PMPlaceholderTitle'),
        description: this.$t('PMPlaceholderDescription'),
      };
    },
    optTextSet () {
      if (!this.user.inbox.optOut) {
        return {
          switchDescription: this.$t('PMReceive'),
          popoverText: this.$t('PMEnabledOptPopoverText'),
        };
      }
      return {
        switchDescription: this.$t('PMReceive'),
        popoverText: this.$t('PMDisabledOptPopoverText'),
      };
    },
  },
  methods: {
    async onModalShown () {
      this.loaded = false;
      const res = await axios.get('/api/v4/inbox/messages');
      this.messages = res.data.data;
      this.loaded = true;
    },
    onModalHide () {
      this.messages = [];
      this.loaded = false;
      this.initiatedConversation = null;
    },
    messageRemoved (message) {
      const messageIndex = this.messages.findIndex(msg => msg.id === message.id);
      if (messageIndex !== -1) this.messages.splice(messageIndex, 1);
      if (this.selectedConversationMessages.length === 0) this.initiatedConversation = {
        user: this.selectedConversation.name,
        uuid: this.selectedConversation.key,
      };
    },
    toggleClick () {
      this.displayCreate = !this.displayCreate;
    },
    toggleOpt () {
      this.$store.dispatch('user:togglePrivateMessagesOpt');
    },
    selectConversation (key) {
      let convoFound = this.conversations.find((conversation) => {
        return conversation.key === key;
      });

      this.selectedConversation = convoFound || {};

      Vue.nextTick(() => {
        if (!this.$refs.chatscroll) return;
        let chatscroll = this.$refs.chatscroll.$el;
        chatscroll.scrollTop = chatscroll.scrollHeight;
      });
      this.textAreaDisabled = false;
    },

    // Checks if user is able to send msg to intended recipiant
    checkUser () {
      this.$store.dispatch('members:getObjectionsToInteraction', {
        interaction: 'send-private-message',
        toUserId: this.selectedConversation.key,
      }).then(response => {
        if (response.request.status === 500) {
          this.sendBtnDisabled = true;
          this.textAreaDisabled = true;
          this.$store.dispatch('snackbars:add', {
            title: 'Habitica',
            text: this.$t('cannotSendMessageToDeletedUser'),
            type: 'error',
            timeout: true,
          });
        } else {
          const objections = response.data.data;
          if (objections.length === 1) {
            this.sendBtnDisabled = true;
            this.textAreaDisabled = true;
            this.$store.dispatch('snackbars:add', {
              title: 'Habitica',
              text: this.$t('notAuthorizedToSendMessageToThisUser'),
              type: 'error',
              timeout: true,
            });
          } else {
            this.sendBtnDisabled = false;
            this.textAreaDisabled = false;
          }
        }
      });
    },

    sendPrivateMessage () {
      if (!this.newMessage) return;

      this.messages.push({
        sent: true,
        text: this.newMessage,
        timestamp: new Date(),
        user: this.selectedConversation.name,
        uuid: this.selectedConversation.key,
        contributor: this.user.contributor,
      });

      // Remove the placeholder message
      if (this.initiatedConversation && this.initiatedConversation.uuid === this.selectedConversation.key) {
        this.initiatedConversation = null;
      }

      this.selectedConversation.lastMessageText = this.newMessage;
      this.selectedConversation.date = new Date();

      Vue.nextTick(() => {
        if (!this.$refs.chatscroll) return;
        let chatscroll = this.$refs.chatscroll.$el;
        chatscroll.scrollTop = chatscroll.scrollHeight;
      });

      this.$store.dispatch('members:sendPrivateMessage', {
        toUserId: this.selectedConversation.key,
        message: this.newMessage,
      }).then(response => {
        const newMessage = response.data.data.message;
        Object.assign(this.messages[this.messages.length - 1], newMessage);
      });

      this.newMessage = '';
    },
    close () {
      this.$root.$emit('bv::hide::modal', 'inbox-modal');
    },
  },
};
</script>
