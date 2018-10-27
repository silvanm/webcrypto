<template>
    <div id="app" class="container">
        <h1>Client side encryption Demo</h1>
        <p>Demonstrates a end2end encryption with server side storage on Firestore.</p>
        <ol>
            <li>Select a file</li>
            <li>enter a password</li>
            <li>encrypt</li>
            <li>save to Firestore</li>
            <li>load from Firestore</li>
            <li>decrypt</li>
        <form>
            <div class="form-group form-check">
                <input type="file" class="custom-file-input" id="customFile" @change="filechange($event)">
                <label class="custom-file-label" for="customFile">Choose file</label>
            </div>
            <div class="form-group">
                <label for="password">Password</label>
                <input @keyup="updateKey()" class="form-control" id="password" v-model="password"
                       PLACEHOLDER="Enter encryption/decryption password here">
            </div>

            <div class="btn-group" role="group" aria-label="Basic example">
                <button type="button" class="btn btn-primary" @click="encrypt" :disabled="!this.password">Encrypt</button>
                <button type="button" class="btn btn-primary" @click="save">Save</button>
                <button type="button" class="btn btn-primary" @click="load">Load</button>
                <button type="button" class="btn btn-primary" @click="decrypt" :disabled="!this.password">Decrypt</button>
            </div>
        </form>
        <br>
        <div v-show="fileoutput.length>0">
        <h4>File content:</h4>
        <pre id="fileDisplayArea" style="height: 200px;padding:5px;overflow: auto;background-color: #eee">{{fileoutput}}</pre>
            </div>
        <div class="alert alert-primary" role="alert" v-show="status.length>0">
            <b>Status:</b> {{status}}
        </div>

    </div>
</template>

<script>
import Firebase from "firebase";

let app = Firebase.initializeApp({
  projectId: process.env.VUE_APP_PROJECTID,
  apiKey: process.env.VUE_APP_APIKEY,
  databaseURL: process.env.VUE_APP_DATABASEURL
});

let db = app.firestore();
const settings = { timestampsInSnapshots: true };
db.settings(settings);
let reader = new FileReader();

function convertStringToArrayBuffer(str) {
  let encoder = new TextEncoder("utf-8");
  return encoder.encode(str);
}

export default {
  name: "app",
  components: {},
  data() {
    return {
      fileoutput: "",
      password: "",
      status: ""
    };
  },
  mounted() {
    this.iv = new Uint8Array([43, 43, 23, 12, 43, 33, 22, 16, 65, 33, 27, 75]);
  },
  methods: {
    async updateKey() {
      let importedPassword = await window.crypto.subtle.importKey(
        "raw",
        convertStringToArrayBuffer(this.password),
        { name: "PBKDF2" },
        false,
        ["deriveKey"]
      );
      let key = await window.crypto.subtle.deriveKey(
        {
          name: "PBKDF2",
          salt: convertStringToArrayBuffer("the salt is this random string"),
          iterations: 100000,
          hash: "SHA-256"
        },
        importedPassword,
        {
          name: "AES-GCM",
          length: 128
        },
        false,
        ["encrypt", "decrypt"]
      );
      this.key = key;
      console.log("Key updated");
    },
    async encrypt() {
      let encrypted = await window.crypto.subtle.encrypt(
        {
          name: "AES-GCM",
          iv: this.iv
        },
        this.key,
        reader.result //ArrayBuffer of data you want to encrypt
      );
      this.encrypted = encrypted;
      this.fileoutput = String.fromCharCode.apply(
        null,
        new Uint8Array(encrypted)
      );
      this.status = "File encrypted";
      this.password = "";
    },
    async decrypt() {
      let decrypted = await window.crypto.subtle.decrypt(
        {
          name: "AES-GCM",
          iv: this.iv //The initialization vector you used to encrypt
        },
        this.key, //from generateKey or importKey above
        this.encrypted //ArrayBuffer of the data
      );
      this.status = "File decrypted";
      this.fileoutput = String.fromCharCode.apply(
        null,
        new Uint8Array(decrypted)
      );
    },
    async save() {
      await db
        .collection("webcrypto")
        .doc("secret-file")
        .set({
          name: document.getElementById("customFile").value,
          value: btoa(String.fromCharCode(...new Uint8Array(this.encrypted)))
        });

      this.fileoutput = "";
      this.status = "File saved to remote database";
    },
    async load() {
      let doc = await db
        .collection("webcrypto")
        .doc("secret-file")
        .get();
      console.log(doc.data());
      this.encrypted = Uint8Array.from(atob(doc.data().value), c =>
        c.charCodeAt(0)
      );
      this.fileoutput = String.fromCharCode.apply(
        null,
        new Uint8Array(this.encrypted)
      );
      this.status = "File loaded from remote database";
    },
    filechange(e) {
      let file = e.target.files[0];
      let textType = /text.*/;

      if (file.type.match(textType)) {
        reader.onload = () => {
          this.fileoutput = String.fromCharCode.apply(
            null,
            new Uint8Array(reader.result)
          );
        };
        reader.readAsArrayBuffer(file);
      } else {
        this.fileoutput = "File not supported!";
      }
    }
  }
};
</script>

<style>
#app {
}
</style>
