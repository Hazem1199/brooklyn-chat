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
          <div class="user-icon">
            <img
              class="user-icon"
              v-if="state.img"
              :src="state.img"
              alt="User profile image"
            />
          </div>
          <h2>{{ state.username }}</h2>
        </div>
        <div class="header-logout" @click="logout">Logout</div>
      </header>
      <div class="main-box">
        <main class="user-box">
          <div
            class="message-box"
            v-for="user in state.users"
            :key="user.id"
            style="position: relative"
          >
            <span
              class="isLoggin"
              v-if="user.username === state.username"
            ></span>
            <img
              class="user-icon"
              v-if="user.img"
              :src="user.img"
              alt="User profile image"
            />
            <div class="message-box">{{ user.username }}</div>
          </div>
        </main>
        <main>
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
          />
          <button type="submit">Send</button>
        </form>
      </footer>
    </div>
  </div>
</template>

<script>
import { ref, reactive, onMounted } from "vue";
// import firebase from "firebase/compat/app";
import { getAuth, GoogleAuthProvider, signInWithPopup } from "firebase/auth";
// import { ref } from "firebase/compat/database";

import db from "./db";
export default {
  setup() {
    const inputLogin = ref("");
    const inputMessage = ref("");
    const isLoggedIn = ref(true);

    // const UserInfo = getAuth().currentUser;

    // onMounted(() => {
    //   UserInfo.then((user) => {
    //     if (user) {
    //       console.log(user);
    //     } else {
    //       console.log("No user is signed in");
    //     }
    //   });
    // });

    const state = reactive({
      username: "",
      messages: [],
      users: [],
      img: null, // Default to null
    });

    const login = () => {
      if (inputLogin.value) {
        state.username = inputLogin.value;
        isLoggedIn.value = true;
        inputLogin.value = "";
      }
    };

    const logout = () => {
      const auth = getAuth();
      auth
        .signOut()
        .then((result) => {
          console.log(result);
          state.username = "";
          state.img = null;
          isLoggedIn.value = false;
        })
        .catch((error) => {
          console.error("Logout Failed:", error);
        });
    };

    const googleLogin = async () => {
      const provider = new GoogleAuthProvider();
      const auth = getAuth();
      // const current = auth.currentUser;
      try {
        const result = await signInWithPopup(auth, provider);
        const user = result.user;
        console.log(user);
        state.username = user.displayName;
        state.img = user.photoURL;
        // console.log(current);
        const usersRef = db.database().ref("users");
        const userRef = usersRef.child(user.uid);
        userRef.set({
          username: user.displayName,
          img: user.photoURL,
          isLoggedIn: isLoggedIn.value,
        });
      } catch (error) {
        console.error("Login Failed:", error);
      }
    };

    const sendMessage = () => {
      const messagesRef = db.database().ref("messages");

      if (inputMessage.value === "" || inputMessage.value === null) {
        return;
      }
      const message = {
        username: state.username,
        img: state.img,
        body: inputMessage.value,
      };

      messagesRef.push(message);
      inputMessage.value = "";
    };

    onMounted(() => {
      const messagesRef = db.database().ref("messages");
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

      messagesRef.on("value", (snapshot) => {
        const data = snapshot.val();
        let messages = [];
        Object.keys(data || {}).forEach((key) => {
          messages.push({
            id: key,
            username: data[key].username,
            img: data[key].img,
            body: data[key].body,
          });
        });
        state.messages = messages;
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
      // UserInfo,
      isLoggedIn,
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
  font-family: sans-serif;
  width: 100vw;
  min-height: 95vh;
  overflow-x: hidden;

  background-color: #ffeded;
  background-image: linear-gradient(62deg, #f8d4d6 0%, #ffeded 100%);
}

.container {
  margin: 5vh auto 0;
  min-height: 90vh;
  background-color: #92829e;

  width: 60%;
  border-radius: 20px;
  box-shadow: 3px 3px 8px rgba(0, 0, 0, 0.1);

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
  background-color: #f1f0f4;
  border-radius: 20px;
  padding: 3rem 1rem;
  box-shadow: 1px 1px 6px rgba(31, 32, 61, 0.1);
  color: #5d5a72;
}

.login h1 {
  text-align: center;
  font-size: 1.5rem;

  margin-bottom: 1.5rem;
}

.login input {
  width: 100%;
  border-radius: 10rem;
  background-color: #fff;
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
  padding: 0.5rem;
  border: none;
  outline: none;
  color: #fff;
  transition: all 0.3s;
}

.login button:hover {
  background-color: #ec9e9e;
}

.isLoggin {
  position: absolute;
  top: -10px;
  right: -10px;
  padding: 5px 5%;
  border-radius: 50%;
  background-color: lawngreen;
  /* color: white; */
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
  color: #e7e6f0;
}

header h2 {
  font-size: 1.5rem;
}

.header-user {
  display: flex;
  align-items: center;
}

.user-icon {
  border-radius: 10rem;
  width: 1.5rem;
  height: 1.5rem;
  display: flex;
  justify-content: center;
  align-items: center;

  font-size: 0.75rem;
}

.header-user .user-icon {
  background-color: #fff;
  color: #7d799b;
  margin-right: 0.5rem;
  font-weight: bold;
}

.header-logout {
  transition: all 0.3s;
  padding: 0.5rem;
  border-radius: 5px;
  cursor: pointer;
  color: #f5e1e1;
}

.header-logout:hover {
  background-color: #7d799a;

  box-shadow: 0 0 6px rgba(0, 0, 0, 0.1);
}

.header-logout:active {
  background-color: #5d5a72;
  box-shadow: 0 0 4px rgba(0, 0, 0, 0.1);
}

.main-box {
  display: flex;
  flex-grow: 1;
}

.user-box {
  background-color: #7d799b;
  color: #fff;
  border-radius: 20px;
  padding: 1rem;
  /* width: 20%; */
  max-height: calc(90vh - 130px);
  border-top-left-radius: 20px;
  width: 5%;
  /* overflow-y: auto; */
  margin-right: 7px;
}

main {
  background-color: #fff;
  flex-grow: 1;
  /* border-radius:  20px; */
  /* box-shadow: -6px 0 6px rgba(0, 0, 0, 0.2); */
  padding: 1rem;
  display: flex;
  flex-direction: column;
  border-top-right-radius: 20px;
  border-top-left-radius: 20px;
  /* max-height: calc(90vh - 130px); */
  overflow-y: auto;
}

main::-webkit-scrollbar-track {
  border-radius: 10px;
  background-color: transparent;
}

main::-webkit-scrollbar {
  width: 4px;
  background-color: transparent;
}

main::-webkit-scrollbar-thumb {
  border-radius: 10px;
  background-color: rgba(66, 69, 129, 0.3);
}

.message-box {
  max-width: 80%;
  display: flex;
  margin-bottom: 1rem;
  position: relative;
  flex-shrink: 1;
}

.message-box .message {
  background-color: #f1f0f5;
  color: #666380;
  padding: 0.5rem;
  border-radius: 7px;
}

.message-box .user-icon {
  background-color: #f1f0f5;
  color: #7d799b;

  flex-shrink: 0;
  margin-right: 0.5rem;
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
  background: #7d799b;
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
  background: #fea9aa;
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
</style>
