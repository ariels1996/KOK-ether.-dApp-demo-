<template>
  <div id="app" style = "background: #fbfdd8;">
    <div style="display: flex; flex-direction: column;">
      <!-- Upload Interface -->
      <div id="upload">
        <div v-if="this.$root.$data.loading === false">

          <div>

            <img style="margin-left: 30px;max-width: 300px;"
              src="/logo.png"
            >

          </div>

          <h1 class = "title"><b>콕</b></h1>
          <h5 class = "title">호랑이는 죽어서 가죽을 남기고, 사람은 죽어서 낙서를 남긴다.</h5>

          <!-- Form for file choose, caption text and submission -->
          <form
            class="margin-sm"
            @submit.stop.prevent="handleSubmit"
          >
            <div>
              <b-form-file
                @change="captureFile"
                v-model="file" 
                ref="file-input" 
                class="mb-2"
              />
            </div>
            <b-form-textarea
              v-model="caption"
              placeholder="Post description"
              :rows="3"
              :max-rows="6"
              class="margin-xs"
            />
            <b-button
              class="margin-xs"
              variant="secondary"
              @click="handleOk"
              style="float:right; margin-top : 2%; background-color : #70addc; border-color : #70addc;"
            >
              Submit
            </b-button>
          </form>
        </div>
        <div
          v-if="this.$root.$data.loading === true"
          style="margin-top: 10%; margin-bottom: 5%"
        >
          <img
            class="upload-load"
            src="https://media.giphy.com/media/2A6xoqXc9qML9gzBUE/giphy.gif"
          >
        </div>
      </div>

      <!-- Posts Interface -->
      <ul class="home-list">
        <li
          v-for="item in this.$root.$data.currentPosts"
          :key="item.key"
          :item="item"
        >
          <!-- Card UI for post's image & caption text -->
          <b-card
            border-variant="secondary"
            :img-src="item.src"
          >
            <p class="home-card-text">
              {{ item.caption }}
            </p>
          </b-card>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import ipfs from './contracts/ipfs';

export default {
  name: 'App',
  // data variables
  data() {
    return {
      buffer: '',
      caption: '',
    };
  },
  methods: {
    /* used to catch chosen image &
     * convert it to ArrayBuffer.
     */
    captureFile(file) {
      const reader = new FileReader();
      if (typeof file !== 'undefined') {
        reader.readAsArrayBuffer(file.target.files[0]);
        reader.onloadend = async () => {
          this.buffer = await this.convertToBuffer(reader.result);
        };
      } else this.buffer = '';
    },
    /**
     * converts ArrayBuffer to
     * Buffer for IPFS upload.
     */
    async convertToBuffer(reader) {
      return Buffer.from(reader);
    },
    /**
     * submits buffered image & text to IPFS
     * and retrieves the hashes, then store
     * it in the Contract via sendHash().
     */
    onSubmit() {
      alert('Uploading on IPFS...');
      this.$root.loading = true;
      let imgHash;

      ipfs.add(this.buffer)
        .then((hashedImg) => {
          imgHash = hashedImg[0].hash;
          return this.convertToBuffer(this.caption);
        }).then(bufferDesc => ipfs.add(bufferDesc)
          .then(hashedText => hashedText[0].hash)).then((textHash) => {
          this.$root.contract.methods
            .sendHash(imgHash, textHash)
            .send({ from: this.$root.currentAccount },
              (error, transactionHash) => {
                if (typeof transactionHash !== 'undefined') {
                  alert('Storing on Ethereum...');
                  this.$root.contract.once('NewPost',
                    { from: this.$root.currentAccount },
                    () => {
                      this.$root.getPosts();
                      alert('Operation Finished! Refetching...');
                    });
                } else this.$root.loading = false;
              });
        });
    },
    /**
     * validates if image & captions
     * are filled before submission.
     */
    handleOk() {
      if (!this.buffer || !this.caption) {
        alert('Please fill in the information.');
      } else {
        this.onSubmit();
      }
    },

  },
};
</script>

<style>

@font-face { font-family: 'KimNamyun'; src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/noonfonts_one@1.0/KimNamyun.woff') format('woff'); font-weight: normal; font-style: normal; }
#app {
  font-family: "KimNamyun", "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  display: flex;
  justify-content: center;
  color: #2c3e50;
  margin-top: 3%;
}
.title {
  font-family: "KimNamyun", "Avenir", Helvetica, Arial, sans-serif;

}

.home-load {
  width: 50px;
  height: 50px;
}

.card img {
  object-fit: cover;
  max-height: 400px;
  /*max-width: 500px;*/
}


.card {
  text-align: left;
  width: 600px;
  margin-bottom: 20px;
}


.home-list{
  padding: 0;
  list-style: none;
}

.home-card-text {
  text-align: justify;
  font-size: 20px;
}

#upload {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-bottom: 5%;
  width: 600px;
}

.upload-load {
  width: 50px;
  height: 50px;
}
/*
.margin-xs {
  margin-top: 3%;
}
*/
.margin-sm {
  margin-top: 7%;
}

.border-style {
  border: 1px solid #ced4da;
}



</style>
