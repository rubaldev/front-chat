<script>
  import axios from "axios";
  import { io } from "socket.io-client";
  import { afterUpdate, onDestroy } from "svelte";

  const API_URL = "http://localhost:4000";

  let isLoggedIn = false;
  let phoneNumber = "";
  let password = "";
  let currentMessage = "";
  let messages = [];
  let socket;
  let conversation_list = [];
  let userId = null;
  let selectedConversation = null;
  let selectedConversationName = "";
  let selectedConversationAvatar = "";
  let selectedConversationStatus = "En ligne";
  let showConversations = true;
  let messagesContainer;
  let unreadConversations = new Set();

  // état menu flottant
  let showMenu = false;
  let allUsers = [];
  let selectedUsers = [];
  let groupName = "";
  let showGroupModal = false;
  let showPrivateModal = false;
  let selectedPrivateUser = null;
  let showAddUserToGroup = false;
  let idUserAddToGroup = "";
  // ---------------- LOGIN ----------------
  async function handleLogin() {
    try {
      const response = await axios.post(`${API_URL}/api/auth/login`, {
        phoneNumber,
        password,
      });
      const { token, userId: id } = response.data;
      userId = id;

      socket = io(API_URL, { auth: { token } });

      isLoggedIn = true;
      // récupérer tous les utilisateurs (sauf moi)
      const usersRes = await axios.get(`${API_URL}/api/auth/users`);
      allUsers = usersRes.data.filter((u) => u.id_user !== userId);
      // récupérer conversations
      const list = await axios.post(`${API_URL}/api/auth/getconversationlist`, {
        userId,
      });
      conversation_list = list.data;

      socket.on("receiveMessage", (messageData) => {
        if (messageData.conversationId === selectedConversation) {
          messages = [...messages, messageData];
          scrollToBottom();
        } else {
          unreadConversations.add(messageData.conversationId);
        }
      });
    } catch (err) {
      console.error(err);
      alert("Erreur de connexion. Vérifiez vos identifiants.");
    }
  }

  // ---------------- Sélection conversation ----------------
  async function selectConversation(conversation) {
    selectedConversation = conversation.conversation_id;
    selectedConversationName = conversation.name;
    selectedConversationAvatar =
      conversation.avatar || "https://i.pravatar.cc/40";
    showConversations = false;
    unreadConversations.delete(selectedConversation);

    socket.emit("joinGroup", selectedConversation);

    try {
      const res = await axios.post(`${API_URL}/api/auth/getMessages`, {
        conversationId: selectedConversation,
        userId,
      });
      messages = res.data;
      scrollToBottom();
    } catch (err) {
      console.error(err);
      messages = [];
    }
  }

  function backToConversations() {
    selectedConversation = null;
    selectedConversationName = "";
    selectedConversationAvatar = "";
    showConversations = true;
    messages = [];
  }

  // ---------------- Création conversation privée ----------------
  
  function openPrivateModal() {
    selectedPrivateUser = null;
    showPrivateModal = true;
  }

  async function createPrivateConversation() {
    if (!selectedPrivateUser) return;

    try {
      const res = await axios.post(
        `${API_URL}/api/auth/createPrivateConversation`,
        { userId, phoneNumber: selectedPrivateUser.phone_number }
      );

      // Récupérer la conversation complète depuis le back
      const convRes = await axios.post(
        `${API_URL}/api/auth/getconversationlist`,
        { userId }
      );
      conversation_list = convRes.data; // maintenant toutes les données sont là
      showPrivateModal = false;
    } catch (err) {
      console.error(err);
      alert("Impossible de créer la conversation privée.");
    }
  }

  // ---------------- Création groupe ----------------
  function openGroupModal() {
    groupName = "";
    selectedUsers = [];
    showGroupModal = true;
  }

  async function createGroupConversation() {
    try {
      const res = await axios.post(
        `${API_URL}/api/auth/createGroupConversation`,
        {
          userId,
          name: groupName,
          selectedUsers: selectedUsers.map((u) => u.id_user),
        }
      );

      // Récupérer la conversation complète
      const convRes = await axios.post(
        `${API_URL}/api/auth/getconversationlist`,
        { userId }
      );
      conversation_list = convRes.data;
      showGroupModal = false;
    } catch (err) {
      console.error(err);
      alert("Impossible de créer le groupe.");
    }
  }

  // ---------------- Envoyer message ----------------
  function sendMessageto() {
    if (!currentMessage || !selectedConversation) return;

    const messageData = {
      conversationId: selectedConversation,
      content: currentMessage,
      sender_id: userId,
      username: "Moi",
      timestamp: new Date().toISOString(),
    };

    socket.emit("sendMessage", messageData);
    currentMessage = "";
    scrollToBottom();
  }

  function scrollToBottom() {
    if (messagesContainer)
      messagesContainer.scrollTop = messagesContainer.scrollHeight;
  }

  afterUpdate(() => scrollToBottom());

  onDestroy(() => {
    if (socket) socket.disconnect();
  });

  function formatTime(timestamp) {
    const date = new Date(timestamp);
    return date.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" });
  }
  function openAddtoGroupeModal() {
    selectedUsers = [];
    showAddUserToGroup = true;
  }

  async function addToGroup() {
    try {
      const res = await axios.post(`${API_URL}/api/auth/addUserToGroup`, {
        conversationId: selectedConversation,
        idUserAddToGroup: idUserAddToGroup,
      });
      console.log(res)
      showAddUserToGroup = false;
    } catch (error) {
      console.log("erreur lors de l'ajout dans le groupe", error);
      showAddUserToGroup = false;
    }
  }
</script>

<h1 class="text-4xl font-bold text-center mt-5 mb-6">Rubaldev-Chat</h1>
<div class="p-5 max-w-xl mx-auto">
  {#if !isLoggedIn}
    <!-- Login Form -->
    <div class="flex flex-col space-y-4 w-full max-w-md mx-auto p-4">
      <input
        type="text"
        placeholder="Numéro de téléphone"
        bind:value={phoneNumber}
        class="w-full p-2 border border-gray-300 rounded-md"
      />
      <input
        type="password"
        placeholder="Mot de passe"
        bind:value={password}
        class="w-full p-2 border border-gray-300 rounded-md"
      />
      <button
        on:click={handleLogin}
        class="w-full bg-blue-500 text-white font-bold p-2 rounded-md hover:bg-blue-600"
        >Se connecter</button
      >
    </div>
  {:else}
    <!-- Conversations List -->
    {#if showConversations}
      <div class="flex justify-between items-center mb-3">
        <h2 class="text-xl font-semibold">Conversations</h2>
      </div>
      <div class="space-y-2">
        {#each conversation_list as conversation}
          <div
            class="relative p-3 rounded-lg bg-gray-100 cursor-pointer hover:bg-gray-200 transition"
            on:click={() => selectConversation(conversation)}
          >
            <p class="font-semibold">{conversation.name}</p>
            <p class="text-sm text-gray-600">
              ID: {conversation.conversation_id}
            </p>
            {#if unreadConversations.has(conversation.conversation_id)}
              <span
                class="absolute top-2 right-2 w-4 h-4 bg-red-500 rounded-full"
              ></span>
            {/if}
          </div>
        {/each}
      </div>

      <!-- Floating Menu Button -->
      <div class="fab" on:click={() => (showMenu = !showMenu)}>+</div>
    {:else}
      <!-- Chat Window -->
      <div
        class="flex items-center justify-between p-3 border-b bg-gray-50 rounded-t-lg"
      >
        <button on:click={backToConversations} class="text-blue-500 font-bold"
          >← retour</button
        >
        <div class="flex items-center flex-1 justify-center space-x-3">
          <img
            src={selectedConversationAvatar}
            alt="Avatar"
            class="w-10 h-10 rounded-full"
          />
          <div class="flex flex-col items-start">
            <p class="font-semibold">{selectedConversationName}</p>
            <p class="text-xs text-green-500">{selectedConversationStatus}</p>
          </div>
          <button
            class="r-10 size-8 bg-green-400 rounded-2xl"
            on:click={openAddtoGroupeModal}>+</button
          >
        </div>
        <div class="w-6"></div>
      </div>

      <div
        bind:this={messagesContainer}
        class="h-96 border-x border-b rounded-b-lg p-2 space-y-2 overflow-y-auto bg-gray-50"
      >
        {#each messages as msg}
          <div
            class="flex {msg.sender_id === userId
              ? 'justify-end'
              : 'justify-start'}"
          >
            <div
              class="max-w-[70%] p-3 rounded-2xl shadow {msg.sender_id ===
              userId
                ? 'bg-gradient-to-br from-blue-400 to-blue-600 text-white'
                : 'bg-gray-200 text-gray-900'}"
            >
              <p class="break-words">{msg.content}</p>
              <span class="text-xs text-gray-700 float-right mt-1"
                >{formatTime(msg.messages_send_at)}</span
              >
            </div>
          </div>
        {/each}
      </div>

      <div class="flex mt-2 items-center space-x-2">
        <input
          placeholder="Écrire un message..."
          bind:value={currentMessage}
          on:keydown={(e) => e.key === "Enter" && sendMessageto()}
          class="flex-1 p-3 border rounded-full focus:outline-none focus:ring-2 focus:ring-blue-500"
        />
        <button
          on:click={sendMessageto}
          class="p-3 bg-blue-500 text-white rounded-full hover:bg-blue-600 flex items-center justify-center"
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            class="h-6 w-6 rotate-45"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M3 10l9-7 9 7-9 7-9-7z"
            />
          </svg>
        </button>
      </div>
    {/if}
  {/if}

  <!-- Floating Menu -->
  {#if showMenu}
    <div class="menu">
      <button on:click={openPrivateModal}>Nouvelle conversation privée</button>
      <button on:click={openGroupModal}>Nouveau groupe</button>
    </div>
  {/if}

  <!-- Private Modal -->
  {#if showPrivateModal}
    <div class="modal-backdrop">
      <div class="modal">
        <h3 class="font-bold mb-2">Choisir un utilisateur</h3>
        {#each allUsers as u}
          <div>
            <input
              type="radio"
              bind:group={selectedPrivateUser}
              value={u}
              id={"private" + u.id_user}
            />
            <label for={"private" + u.id_user}
              >{u.username} ({u.phone_number})</label
            >
          </div>
        {/each}
        <button
          on:click={createPrivateConversation}
          class="mt-3 bg-blue-500 text-white px-4 py-2 rounded">Créer</button
        >
        <button
          on:click={() => (showPrivateModal = false)}
          class="mt-2 px-4 py-2 rounded border">Annuler</button
        >
      </div>
    </div>
  {/if}

  <!-- Group Modal -->
  {#if showGroupModal}
    <div class="modal-backdrop">
      <div class="modal">
        <h3 class="font-bold mb-2">Nom du groupe</h3>
        <input
          type="text"
          placeholder="Nom du groupe"
          bind:value={groupName}
          class="w-full p-2 border rounded mb-2"
        />
        <h3 class="font-bold mb-2">Sélectionnez les membres</h3>
        {#each allUsers as u}
          <div>
            <input
              type="checkbox"
              bind:group={selectedUsers}
              value={u}
              id={"group" + u.id_user}
            />
            <label for={"group" + u.id_user}
              >{u.username} ({u.phone_number})</label
            >
          </div>
        {/each}
        <button
          on:click={createGroupConversation}
          class="mt-3 bg-blue-500 text-white px-4 py-2 rounded">Créer</button
        >
        <button
          on:click={() => (showGroupModal = false)}
          class="mt-2 px-4 py-2 rounded border">Annuler</button
        >
      </div>
    </div>
  {/if}

  {#if showAddUserToGroup}
    <div class="modal-backdrop">
      <div class="modal">
        <h3 class="font-bold mb-2">Sélectionnez le membre</h3>
        {#each allUsers as u}
          <div>
            <input
              type="radio"
              bind:group={idUserAddToGroup}
              value={u}
              id={"group" + u.id_user}
            />
            <label for={"group" + u.id_user}
              >{u.username} ({u.phone_number})</label
            >
          </div>
        {/each}
        <button
          on:click={addToGroup}
          class="mt-3 bg-blue-500 text-white px-4 py-2 rounded"
          >Ajouter dans le groupe</button
        >
        <button
          on:click={() => (showAddUserToGroup = false)}
          class="mt-2 px-4 py-2 rounded border">Annuler</button
        >
      </div>
    </div>
  {/if}
</div>

<style>
  .fab {
    position: fixed;
    bottom: 20px;
    right: 20px;
    background: #25d366;
    color: white;
    width: 56px;
    height: 56px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 28px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
    cursor: pointer;
  }

  .menu {
    position: fixed;
    bottom: 80px;
    right: 20px;
    background: white;
    border: 1px solid #ddd;
    border-radius: 12px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
    padding: 10px;
  }

  .menu button {
    display: block;
    width: 100%;
    padding: 8px 12px;
    text-align: left;
    border: none;
    background: none;
    cursor: pointer;
  }

  .menu button:hover {
    background: #f5f5f5;
  }

  .modal-backdrop {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.3);
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .modal {
    background: white;
    padding: 20px;
    border-radius: 12px;
    width: 300px;
    max-height: 80%;
    overflow-y: auto;
  }
</style>
