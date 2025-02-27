<template>
  <div class="home">
    <b-modal v-model="isError" id="error-modal" @ok="redirectWrapper" ok-only hide-header no-close-on-esc no-close-on-backdrop>
      <div>You are logged out. Returning to login...</div>
    </b-modal>
    <CountdownModal v-if="!start"/>
    <div v-show="!isError">
      <div class="progressBar">
        <ProgressBar />
      </div>
      <div class="container">
        <div class="paragraph" id="paragraph"></div>
        <textarea class="userinput" id="userinput" autofocus></textarea>
      </div>
    </div>
    <PostGame v-if="isFinished" :winner='result'/>
  </div>
</template>

<script>
  import db from '@/firebase'
  import CountdownModal from '../components/CountdownModal.vue'
  import PostGame from '../components/PostGame.vue'
  import ProgressBar from '../components/ProgressBar.vue'
  import { 
    checkIfUserIsNull, 
    redirectToHome,
    deleteCurrentUser,
    deleteUserRoom
  } from '@/resources/errorHandling.js'
  
  export default{
    Name: 'Gameplay',
    components: {
      CountdownModal,
      PostGame,
      ProgressBar,
    },
    data() {
      return {
        challenge: String,
        challengelength: -1,
        start: true,
        isPlayer1: true,
        isFinished: false,
        checkOpponentProgress: null,
        result: false,
        isError: false
      }
    },
    mounted() {
      const roomID = this.$store.getters['auth/getCurrentRoomID']
      const inputElement = document.getElementById('userinput')
      userinput.addEventListener('input', this.checkInput)

      if(roomID !== null) {
        this.checkOpponentProgress = db.collection('Rooms')
          .doc(roomID)
          .onSnapshot(async doc => {
            if(doc.exists && this.isPlayer1){
              if(doc.data().progress2 === this.challengelength){
                this.isFinished = true
                inputElement.disabled = true;
                this.unsubscribe()
                userinput.removeEventListener('input', this.checkInput)
              }
            }
            else if (doc.exists && !this.isPlayer1){
              if(doc.data().progress1 === this.challengelength){
                this.isFinished = true
                inputElement.disabled = true;
                this.unsubscribe()
                userinput.removeEventListener('input', this.checkInput)
              }
            }
            else if(!doc.exists){
              this.isFinished = true
              inputElement.disabled = true;
              this.result = true
              this.unsubscribe()
              userinput.removeEventListener('input', this.checkInput)
            }
          })
      }
    },
    methods: {
      unsubscribe: async function() {
        if(this.checkOpponentProgress !== null)
          this.checkOpponentProgress()
      },

      redirectWrapper: function() {
        redirectToHome()
      },

      checkInput: function() {
        const roomID = this.$store.getters['auth/getCurrentRoomID']
        const paragraphElement = document.getElementById('paragraph')
        const inputElement = document.getElementById('userinput')

        const arrayParagraph = paragraphElement.querySelectorAll('span')
        const arrayInput = inputElement.value.split('')

        let correct = true
        var numCorrect = 0
        arrayParagraph.forEach((characterSpan, index) => {
          const character = arrayInput[index]
          if (character == null) {
            characterSpan.style.color = "black"
            correct = false
          }
          else if (character === characterSpan.innerText) {
            characterSpan.style.color = "green"
            numCorrect += 1;
          }
          else {
            characterSpan.style.color = "red"
            correct = false
          }
        })

        if (this.isPlayer1){
          const gameRoom = db.collection('Rooms')
            .doc(roomID)
            .update({
            progress1: numCorrect
            })
        }
        else {
          const gameRoom = db.collection('Rooms')
            .doc(roomID)
            .update({
            progress2: numCorrect
            })
        }

        if(numCorrect === this.challengelength){
          this.isFinished = true
          inputElement.disabled = true;
          this.result = true
          this.unsubscribe()
          deleteUserRoom()
          userinput.removeEventListener('input', this.checkInput)
        }
      },

      handleExit: function() {
        userinput.removeEventListener('input', this.checkInput)
        this.unsubscribe()
        deleteUserRoom()
        deleteCurrentUser()
      }
    },
    async created() {
      //handles bad exits
      window.addEventListener('beforeunload', this.handleExit)

      //bad logins/refreshes
      const userIsNull = checkIfUserIsNull()
      if(userIsNull) {
        window.removeEventListener('beforeunload', this.handleExit)
        this.isError = true
      }
      else {
        this.start = false

        const roomID = this.$store.getters['auth/getCurrentRoomID']
        const queryroom = db.collection('Rooms').doc(roomID)
        const room = await queryroom.get()
        
        const paragraphID = room.data().challengeString.id
        
        const query = db.collection('Paragraphs').doc(paragraphID)
        const doc = await query.get()
        this.challenge = doc.data().Bank
        this.challengelength = doc.data().Length
        
        if (room.data().user1.id === this.$store.getters['auth/getCurrentUserID'])
          this.isPlayer1 = true
        else 
          this.isPlayer1 = false

        const paragraphElement = document.getElementById('paragraph')
        const inputElement = document.getElementById('userinput')
        const copy = this.challenge
        // paragraphElement.innerHTML = ''
        copy.split('').forEach(character => {
          const characterSpan = document.createElement('span')
          characterSpan.innerText = character
          paragraphElement.appendChild(characterSpan)
        })

        inputElement.value = null
      }
    },

    beforeDestroy() {
      window.removeEventListener('beforeunload', this.handleExit)
    }
    
  }
</script>

<style scoped>
  *{
    box-sizing: border-box;
  }

  .home{
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100vh;
    background: linear-gradient(0deg, rgb(255, 99, 71), rgba(255, 99, 71, 0.25)), url("./../../public/keyboardbg.png");
    background-size: cover;
    background-color: tomato;
    font-family: 'Montserrat';
    font-weight: bold;
  }

  body, .input{
    font-family: 'Gill Sans', 'Gill Sans MT', Calibri, 'Trebuchet MS', sans-serif;
  }

  .container{
    background-color: white;
    padding: 1rem;
    border-radius: .5rem;
    width: 700px;
    max-width: 90%;
  }

  .paragraph{
    margin-bottom: 1rem;
    margin-left: calc(1rem + 2px);
    margin-right: calc(1rem + 2px);
  }

  .userinput{
    background-color: transparent;
    border: 2px solid black;
    outline: none;
    width: 100%;
    height: 8rem;
    margin: auto;
    resize: none;
    padding: .5rem 1rem;
    font-size: 1rem;
    border-radius: .5rem;
  }

  .userinput:focus{
    border-color: black;
  }

  .progressBar{
    margin: 5vh;
  }

</style>
