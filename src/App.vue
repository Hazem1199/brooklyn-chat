<template>
  <div id="app" class="container">
    <div class="login" v-if="!state.username">
      <form @submit.prevent="login">
        <h1>VueChat</h1>
        <input
          type="text"
          placeholder="Please enter your name ..."
          v-model="inputLogin"
        />
        <button type="submit">Login</button>
        <button type="button" @click="googleLogin">
          Login with Google account
        </button>
      </form>
    </div>
    <div class="chat" v-else>
      <header>
        <div class="header-user">
          <button class="sidebar-toggle" @click="toggleSidebar">☰</button>
          <div class="user-icon">
            <img
              class="user-icon"
              v-if="state.img"
              :src="state.img"
              alt="User profile image"
            />
          </div>
          <h2>{{ state.username }}</h2>
          <div v-if="otherUserTyping" class="typing-indicator">
            Someone is typing...
          </div>
        </div>
        <div class="header-logout" @click="logout">Logout</div>
      </header>
      <div class="main-box">
        <main class="user-box" :class="{ collapsed: isSidebarCollapsed }">
          <div
            class="message-box"
            v-for="user in filteredUsers"
            :key="user.id"
            @click="createPrivateRoom(user)"
            style="position: relative; cursor: pointer"
          >
            <span class="isLoggin" v-show="user.isLoggedIn == true"></span>
            <img
              v-if="user.img"
              class="user-icon"
              v-show="user.img"
              :src="user.img"
              alt="User profile image"
            />
            <div
              v-if="!isSidebarCollapsed && user.username"
              class="message-box"
            >
              {{ user.username.split(" ")[0] }}
            </div>
          </div>
        </main>
        <main class="chat-box">
          <div
            class="message-box"
            :class="state.username === message.username ? 'current-user' : ''"
            v-for="message in state.messages"
            :key="message.id"
          >
            <div class="user-icon">
              <img
                class="user-icon"
                v-if="message.img"
                :src="message.img"
                alt="User profile image"
              />
            </div>
            <div class="message">{{ message.body }}</div>
          </div>
        </main>
      </div>
      <footer>
        <form @submit.prevent="sendMessage">
          <input
            type="text"
            placeholder="Write a message ..."
            v-model="inputMessage"
            @input="handleTypingStart"
            @blur="handleTypingStop"
          />
          <button type="submit">Send</button>
        </form>
      </footer>
    </div>
  </div>
</template>

<script>
import { ref, reactive, computed, onMounted } from "vue";
import { getAuth, GoogleAuthProvider, signInWithPopup } from "firebase/auth";
import db from "./db";

export default {
  setup() {
    const inputLogin = ref("");
    const inputMessage = ref("");

    const state = reactive({
      username: "",
      messages: [],
      users: [],
      img: null,
      isLoggedIn: null,
      typing: null,
      to: null,
      currentRoom: null,
    });

    const isSidebarCollapsed = ref(false);

    const login = () => {
      if (inputLogin.value) {
        state.username = inputLogin.value;
        state.isLoggedIn = true; // Set isLoggedIn to true on manual login
        updateUserStatus(state.isLoggedIn); // Update status in database
        inputLogin.value = "";
      }
    };

    const logout = () => {
      const auth = getAuth();
      const user = auth.currentUser;
      auth
        .signOut()
        .then(() => {
          state.username = "";
          state.img = null;
          state.isLoggedIn = false; // Set isLoggedIn to false on logout
          updateUserStatus(state.isLoggedIn, user.uid); // Update status in database
          console.log(state.users);
        })
        .catch((error) => {
          console.error("Logout Failed:", error);
        });
    };

    const googleLogin = async () => {
      const provider = new GoogleAuthProvider();
      const auth = getAuth();
      try {
        const result = await signInWithPopup(auth, provider);
        const user = result.user;
        state.username = user.displayName;
        state.img = user.photoURL;
        state.to = user.uid;
        state.isLoggedIn = true; // Set isLoggedIn to true on Google login
        updateUserStatus(state.isLoggedIn, user.uid); // Update status in database
        console.log(state.users);
      } catch (error) {
        console.error("Login Failed:", error);
      }
    };

    const updateUserStatus = (
      isLoggedIn,
      userId = getAuth().currentUser?.uid
    ) => {
      const usersRef = db.database().ref("users/" + userId);
      usersRef.update({
        username: state.username,
        img: state.img,
        isLoggedIn,
      });
    };

    const sendMessage = () => {
      const messagesRef = db.database().ref("messages/" + state.currentRoom);
      if (inputMessage.value === "" || inputMessage.value === null) {
        return;
      }
      const message = {
        username: state.username,
        img: state.img,
        to: state.to,
        body: inputMessage.value,
      };
      messagesRef.push(message);
      inputMessage.value = "";
    };

    const createPrivateRoom = (user) => {
      const auth = getAuth();
      const currentUser = auth.currentUser;
      if (currentUser) {
        const roomId = [currentUser.uid, user.id].sort().join("-");
        state.currentRoom = roomId;
        state.to = user.id;
        loadMessages(roomId);
      }
    };

    const loadMessages = (roomId) => {
      const messagesRef = db.database().ref("messages/" + roomId);
      messagesRef.on("value", (snapshot) => {
        const data = snapshot.val();
        let messages = [];
        Object.keys(data || {}).forEach((key) => {
          messages.push({
            id: key,
            username: data[key].username,
            img: data[key].img,
            to: data[key].to,
            body: data[key].body,
          });
        });
        state.messages = messages;
      });
    };

    const toggleSidebar = () => {
      isSidebarCollapsed.value = !isSidebarCollapsed.value;
    };

    const filteredUsers = computed(() =>
      state.users.filter((user) => user.username !== state.username)
    );

    onMounted(() => {
      const usersRef = db.database().ref("users");

      usersRef.on("child_added", (snapshot) => {
        const data = snapshot.val();
        state.users.push({
          id: snapshot.key,
          username: data.username,
          img: data.img,
          isLoggedIn: data.isLoggedIn,
        });
      });

      // Listen for changes in each user's data
      usersRef.on("child_changed", (snapshot) => {
        const data = snapshot.val();
        const index = state.users.findIndex((u) => u.id === snapshot.key);
        if (index !== -1) {
          state.users[index] = {
            id: snapshot.key,
            username: data.username,
            img: data.img,
            isLoggedIn: data.isLoggedIn,
          };
        }
      });
    });

    return {
      inputLogin,
      inputMessage,
      login,
      logout,
      state,
      sendMessage,
      googleLogin,
      createPrivateRoom,
      toggleSidebar,
      filteredUsers,
      isSidebarCollapsed,
      isLoggedIn: state.isLoggedIn,
    };
  },
};
</script>

<style>
*,
*:after,
*:before {
  padding: 0;
  margin: 0;
  box-sizing: border-box;
}

body {
  font-family: "Roboto", sans-serif;
  width: 100vw;
  min-height: 95vh;
  overflow-x: hidden;
  background-color: #f0f2f5;
  background-image: linear-gradient(62deg, #f8d4d6 0%, #ffeded 100%);
}

.container {
  margin: 5vh auto 0;
  min-height: 90vh;
  background-color: #ffffff;
  width: 70%;
  border-radius: 20px;
  box-shadow: 3px 3px 10px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.login {
  height: 90vh;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 20px;
}

.login form {
  width: 90%;
  background-color: #ffffff;
  border-radius: 20px;
  padding: 3rem 1rem;
  box-shadow: 1px 1px 8px rgba(31, 32, 61, 0.1);
  color: #5d5a72;
}

.login h1 {
  text-align: center;
  font-size: 1.8rem;
  margin-bottom: 2rem;
  color: #333;
}

.login input {
  width: 100%;
  border-radius: 10rem;
  background-color: #f9f9f9;
  padding: 0.8rem;
  border: 1px solid #ddd;
  margin-bottom: 1rem;
}

.login button {
  display: block;
  width: 100%;
  text-align: center;
  font-size: 1rem;
  font-weight: bold;
  margin-top: 1rem;
  background-color: #feaaaa;
  cursor: pointer;
  border-radius: 10rem;
  padding: 0.8rem;
  border: none;
  outline: none;
  color: #fff;
  transition: background-color 0.3s;
}

.login button:hover {
  background-color: #ec9e9e;
}

.isLoggin {
  position: absolute;
  bottom: 5px;
  right: 5px;
  width: 10px;
  height: 10px;
  border-radius: 50%;
  padding: 0;
  background-color: lawngreen;
}

input {
  background-color: #f3f4f6;
  border: none;
  outline: none;
  padding: 0.5rem 1rem;
}

.chat {
  border-radius: 20px;
  display: flex;
  flex-direction: column;
  min-height: 90vh;
}

header {
  padding: 1rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #333;
  background-color: #feaaaa;
  border-bottom: 1px solid #ec9e9e;
}

header h2 {
  font-size: 1.5rem;
  color: #fff;
}

.header-user {
  display: flex;
  align-items: center;
}

.sidebar-toggle {
  font-size: 1.5rem;
  margin-right: 1rem;
  background: none;
  border: none;
  color: #fff;
  cursor: pointer;
}

.user-icon {
  border-radius: 50%;
  width: 2.5rem;
  height: 2.5rem;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 1rem;
  position: relative;
}

.header-user .user-icon {
  background-color: #fff;
  color: #feaaaa;
  margin-right: 0.5rem;
  font-weight: bold;
}

.header-logout {
  transition: background-color 0.3s;
  padding: 0.5rem 1rem;
  border-radius: 5px;
  cursor: pointer;
  color: #fff;
}

.header-logout:hover {
  background-color: #ec9e9e;
}

.main-box {
  display: flex;
  flex-grow: 1;
}

.user-box {
  background-color: #feaaaa;
  color: #fff;
  font-weight: bold;
  padding: 1rem;
  transition: width 0.3s, background-color 0.3s;
  width: 20%;
  max-height: calc(90vh - 130px);
  overflow-y: auto;
  margin-right: 7px;
}

.user-box.collapsed {
  width: 10%;
}

.user-box .message-box {
  display: flex;
  align-items: center;
  padding: 0.5rem;
  margin-bottom: 1rem;
  background-color: #ec9e9e;
  border-radius: 10px;
  transition: background-color 0.3s;
}

.user-box .message-box:hover {
  background-color: #e58e8e;
}

.user-box .message-box .user-icon {
  width: 2.5rem;
  height: 2.5rem;
  margin-right: 0.5rem;
  position: relative;
}

.user-box .message-box .isLoggin {
  bottom: 5px;
  right: 5px;
}

.user-box.collapsed .message-box .user-icon {
  margin: auto;
}

.user-box.collapsed .message-box .message-box {
  display: none;
}

.chat-box {
  background-color: #fff;
  flex-grow: 1;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  border-top-right-radius: 20px;
  border-top-left-radius: 20px;
  overflow-y: auto;
}

.chat-box::-webkit-scrollbar-track {
  border-radius: 10px;
  background-color: transparent;
}

.chat-box::-webkit-scrollbar {
  width: 4px;
  background-color: transparent;
}

.chat-box::-webkit-scrollbar-thumb {
  border-radius: 10px;
  background-color: rgba(66, 69, 129, 0.3);
}

.message-box {
  /* max-width: 80%; */
  display: flex;
  margin-bottom: 1rem;
  position: relative;
  flex-shrink: 1;
}

.message-box .message {
  background-color: #f1f0f5;
  color: #666380;
  padding: 0.5rem 1rem;
  border-radius: 7px;
}

.message-box .user-icon {
  background-color: #f1f0f5;
  color: #7d799b;
  flex-shrink: 0;
  margin-right: 0.5rem;
  position: relative;
}

.message-box.current-user {
  align-self: flex-end;
}

.message-box.current-user .user-icon {
  background-color: #7d799b;
  color: #fff;
  order: 2;
  margin-right: 0;
  margin-left: 0.5rem;
}

.message-box.current-user .message {
  background-color: #fea9aa;
  color: #ffffff;
}

footer {
  background: #fff;
  border-top-left-radius: 20px;
}

footer form {
  background: #feaaaa;
  padding: 1rem;
  border-top-right-radius: 20px;
  border-top-left-radius: 20px;
  display: flex;
}

footer button {
  border: none;
  outline: none;
  padding: 0.5rem 1rem;
  font-weight: bold;
  background: #ec9e9e;
  border-top-right-radius: 10rem;
  border-bottom-right-radius: 10rem;
  color: #fff;
  cursor: pointer;
}

footer input {
  flex-grow: 1;
  border-top-left-radius: 10rem;
  border-bottom-left-radius: 10rem;
}

/* Responsive Styles */
@media (max-width: 768px) {
  .container {
    width: 90%;
  }

  .user-box {
    width: 40%;
  }

  .user-box.collapsed {
    width: 10%;
  }
}
</style>
