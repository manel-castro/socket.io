<template>
  <div>
    <h1>chat</h1>
    <div class="left-panel">
      <User
        v-for="user in users"
        :key="user.userID"
        :user="user"
        :selected="selectedUser === user"
        @select="onSelectUser(user)"
      />
    </div>
    <MessagePanel
      v-if="selectedUser"
      :user="selectedUser"
      @textInput="onMessage"
      class="right-panel"
    />
  </div>
</template>

<script>
import socket from "../socket";
import User from "./User.vue";
import MessagePanel from "./MessagePanel.vue";

export default {
  name: "Chat",
  components: {
    User,
    MessagePanel
  },
  data() {
    return {
      selectedUser: null,
      users: []
    };
  },
  methods: {
    onMessage(content) {
      console.log("onMessage", content);
      if (this.selectedUser) {
        socket.emit("private message", {
          content,
          to: this.selectedUser.userID
        });
        this.selectedUser.messages.push({
          content,
          fromSelf: true
        });
      }
    },
    onSelectUser(user) {
      console.log("selected user: ", user.userID);
      this.selectedUser = user;
      user.hasNewMessages = false;
      console.log("selected user data: ", this.selectedUser);
    }
  },
  created() {
    socket.on("connect", () => {
      this.users.forEach(user => {
        if (user.self) {
          user.connected = true;
        }
      });
    });

    socket.on("disconnect", () => {
      this.users.forEach(user => {
        if (user.self) {
          user.connected = false;
        }
      });
    });

    const initReactiveProperties = user => {
      user.connected = true;
      user.messages = [];
      user.hasNewMessages = false;
    };

    socket.on("users", users => {
      console.log("remote users", users)
      users.forEach(user => {

        console.log("remoteUsermessage: ", user.messages)
        // initReactiveProperties(user);
        const messages = JSON.parse(user.messages);
        if(! messages) return 

        
        user.messages = messages;
        console.log("messages is:", messages)
        console.log("user.messages is:", user.messages)
        user.messages.forEach((message)=>{
          message.fromSelf = message.from === socket.userID;
        });

        console.log("this.users", this.users)

        for(let i = 0; i< this.users.length; i++) {
          const existingUser = this.users[i];
          console.log("existingUsers", existingUser)
          if(existingUser.userID === user.userID) {
              existingUser.connected = user.connected
              existingUser.messages = user.messages
              console.log("existingUser.connected", existingUser.connected)
              console.log("existingUser.message", existingUser.messages)
            }
        }
        });


      this.users = users.sort((a, b) => {
        if (a.self) return -1;
        if (b.self) return 1;
        if (a.username < b.username) return -1;
        return a.username > b.username ? 1 : 0;
      });

      // user.self = user.userID === socket.id; //Varies by session (?)
        console.log("user.self: ", user.self);
      // Put the current (yourself) user first, and then sort by username
      
    });

    socket.on("user connected", user => {
      console.log("user connected: ", user);
      //checks wether the user reconnecting (or from other tab) is already in the list
      for (let i = 0; i < this.users.length; i++) {
        const existingUser = this.users[i];
        console.log("user ", i, ": ", existingUser);
        if (existingUser.userID === user.userID) {
          console.log("happened 1");
          existingUser.connected = true;
          return;
        }
      }
      console.log("happened 2");
      initReactiveProperties(user);
      this.users.push(user);
    });

    socket.on("user disconnected", id => {
      for (let i = 0; i < this.users.length; i++) {
        const user = this.users[i];
        if (user.userID === id) {
          user.connected = false;
          break;
        }
      }
    });

    socket.on("private message", ({ content, from }) => {
      for (let i = 0; i < this.users.length; i++) {
        const user = this.users[i];
        if (user.userID === from) {
          user.messages.push({
            content,
            fromSelf: false
          });
          if (user !== this.selectedUser) {
            user.hasNewMessages = true;
          }
          console.log("user me: ", user);
          break;
        }
      }
    });
  },
  unmounted() {
    socket.off("connect");
    socket.off("disconnect");
    socket.off("users");
    socket.off("user connected");
    socket.off("user disconnected");
    socket.off("private message");
  }
};
</script>

<style scoped>
.left-panel {
  position: fixed;
  left: 0;
  top: 0;
  bottom: 0;
  width: 260px;
  overflow-x: hidden;
  background-color: #3f0e40;
  color: white;
}

.right-panel {
  margin-left: 260px;
}
</style>
