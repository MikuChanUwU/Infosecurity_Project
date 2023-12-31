<script context="module">
export const encryption = {};
</script>


<script>
import GDrive, { service } from "$lib/google/GDrive.svelte";
import { keysInited, masterKey, roomKeys, wrapKey } from "$lib/stores/key";
import { user_id } from "$lib/stores/user";
import { e2ee } from "./e2ee";

const MASTER_KEY_FILE_NAME = "master_key.json";
const ROOM_KEYS_FILE_NAME = "room_keys.json";

encryption.handleSignoutClick = () => {
  service?.handleSignoutClick();
};

/**
 * Initialise keys from localStorage if possible,
 * else download keys from google drive.
 */
async function initKeys() {
  // Ensure only initialise once
  if ($keysInited) return;

  let file, error;

  // If masterKey failed to init from localStorage, download from google drive
  if (masterKey.init()) {
    ({ file, error } = await service.downloadFile(MASTER_KEY_FILE_NAME));

    // Init masterKey from downloaded file, else create new masterKey
    if (error && error.code === 403) {
      // If user unauthorised, authenticate user before initialising keys
      service.authUser(initKeys);
      return;

    } else if (error && error.code === 404) {
      // Create new keys
      const newMasterKey = await getNewMasterKey();

      // Upload publicKey to server
      uploadPublicKey(newMasterKey.pubKey);

      // Upload masterKey file to gdrive
      service.uploadJSONFile(MASTER_KEY_FILE_NAME, newMasterKey);

      // Save masterKey to stores and localStorage
      masterKey.saveMasterKey(newMasterKey);

    } else {
      masterKey.initFromJson(file.body, true);
    }
  }

  // If roomKeys failed to init from localStorage, download from google drive
  if (roomKeys.init()) {
    ({ file, error } = await service.downloadFile(ROOM_KEYS_FILE_NAME));

    // Init roomKeys from downloaded file, else create new empty object
    if (error && error.code === 403) {
      // If user is unauthorised, authenticate user before initialising keys
      service.authUser(initKeys);
      return;

    } else if (error && error.code === 404) {
      // Upload roomKeys file to gdrive
      service.uploadJSONFile(ROOM_KEYS_FILE_NAME, {});

      // Save roomKeys to stores and localStorage
      roomKeys.initFromJson("{}", true);

    } else {
      roomKeys.initFromJson(file.body, true);
    }
  }

  // When all keys initiated successfully,
  // set flag to true to prevent multiple calls
  keysInited.set(true);
}


encryption.initKeys = initKeys;


async function uploadPublicKey(public_key) {
  try {
    const response = await fetch(
      "https://localhost:8443/api/chat/upload-public-key", {
        method: "POST",
        headers: {
          "Content-Type": "application/json"
        },
        credentials: "include",
        body: JSON.stringify({ public_key })
      }
    );
    if (!response.ok) {
      setTimeout(() => uploadPublicKey(public_key), 100);
      return;
    }
  } catch (err) {
    setTimeout(() => uploadPublicKey(public_key), 100);
    return;
  }
}


encryption.encryptMessage = async (message, room_id) => {
  const key = await getRoomKey(room_id);
  if (key === undefined) return;
  return await e2ee.encryptMessage(message, key);
};


encryption.decryptMessage = async (message, room_id) => {
  const key = await getRoomKey(room_id);
  if (key === undefined) return;
  return await e2ee.decryptMessage(message, key);
};


encryption.encryptFile = async (image, room_id) => {
  const key = await getRoomKey(room_id);
  if (key === undefined) return;
  return await e2ee.encryptFile(image, key);
}


encryption.decryptFile = async (image, iv, room_id) => {
  const key = await getRoomKey(room_id);
  if (key === undefined) return;
  return await e2ee.decryptFile(image, key, iv);
}


encryption.decryptImage = async (image, iv, room_id) => {
  const key = await getRoomKey(room_id);
  if (key === undefined) return;
  return await e2ee.decryptImage(image, key, iv);
}


encryption.generateSecurityCode = async (room_id) => {
  const key = await getRoomKey(room_id);
  if (key === undefined) return;
  return await e2ee.generateSecurityCode(key);
}


async function getRoomKey(room_id) {
  const wrap_key = await getWrapKey($user_id);
  let key = $roomKeys[room_id];
  if (key === undefined) {
    key = await createAndSaveRoomKey(room_id, $user_id);
    if (key === undefined) return;
  }
  return await e2ee.importRoomKey(key.roomKey, wrap_key, key.iv);
}


/**
 * Get and returns the wrap key
 * @param {string} user_id
 * @returns {Promise<CryptoKey>}
 */
async function getWrapKey(user_id) {
  if ($wrapKey.key) {
    return await e2ee.importWrapKey($wrapKey.key);
  }

  const response = await fetch(
    `https://localhost:8443/api/user/wrap-key/${user_id}`, {
      method: "GET",
      credentials: "include"
    }
  );
  const { wrap_key, message } = await response.json();
  if (message) throw new Error(message);

  if (!wrap_key) {
    return await createAndSaveWrapKey();
  }

  // Store value into stores
  wrapKey.set({ key: wrap_key });

  return await e2ee.importWrapKey(wrap_key);
}


async function createAndSaveWrapKey() {
  const wrap_key = await e2ee.generateWrapKey();
  uploadWrapKey(await e2ee.exportWrapKey(wrap_key));
  return wrap_key;
}


async function uploadWrapKey(wrap_key) {
  try {
    const response = await fetch(
      "https://localhost:8443/api/chat/upload-wrap-key", {
        method: "POST",
        headers: {
          "Content-Type": "application/json"
        },
        credentials: "include",
        body: JSON.stringify({ wrap_key })
      }
    );
    if (!response.ok) {
      setTimeout(() => uploadWrapKey(wrap_key), 300);
      return;
    }
  } catch (err) {
    setTimeout(() => uploadWrapKey(wrap_key), 300);
    return;
  }
}


/**
 * Creates and save the room key
 * @param {string} room_id current room id
 * @param {string} user_id current user id
 * @returns {Promise<string>} Base64 encoded room key
 */
async function createAndSaveRoomKey(room_id, user_id) {
  // Get other user public key from server
  const response = await fetch(`https://localhost:8443/api/user/public-key/${room_id}/${user_id}`);
  const { public_key: pubKey, message } = await response.json();
  if (message) throw new Error(message);

  // Derive room key
  const key = await deriveRoomKey(pubKey);

  if (key === undefined) {
    service.authUser(() => createAndSaveRoomKey(room_id, user_id));
  }

  // Upload room key to google drive
  saveRoomKey(room_id, key);

  return key;
}


/**
 * Creates and save the room key
 * @param {string} room_id room id
 * @param {string} key Base64 encoded room key
 */
async function saveRoomKey(room_id, key) {
  // Upload room key to google drive
  const { content, error } = await service.updateJSONFile(ROOM_KEYS_FILE_NAME, room_id, key);

  // If user is not authenticated yet
  if (error && error.code === 403) {
    service.authUser(() => saveRoomKey(room_id, key));

  // If file is not created yet
  } else if (error && error.code === 404) {
    const data = {};
    data[room_id] = key;
    service.uploadJSONFile(ROOM_KEYS_FILE_NAME, data);
    roomKeys.saveNewKey(room_id, key);
  } else {
    roomKeys.initFromJson(content, true);
  }
}


async function getNewMasterKey() {
  const wrap_key = await getWrapKey($user_id);
  const key = await e2ee.generateKeyPair();
  const { privKey, iv } = await e2ee.exportPrivateKey(key.privateKey, wrap_key);
  const pubKey = await e2ee.exportPublicKey(key.publicKey);
  return { privKey, pubKey, iv };
}


async function deriveRoomKey(userPubKey) {
  const wrap_key = await getWrapKey($user_id);
  const storedKey = $masterKey.privKey;
  const iv = $masterKey.iv;
  if (storedKey === undefined) return;
  const privKey = await e2ee.importPrivateKey(storedKey, wrap_key, iv);
  const pubKey = await e2ee.importPublickey(userPubKey);
  const roomKey = await e2ee.deriveSecretKey(privKey, pubKey);;
  return await e2ee.exportRoomKey(roomKey, wrap_key);
}
</script>


<!-- TODO(high)(SpeedFox198) Change from null to something -->
<GDrive on:load={initKeys}/>
