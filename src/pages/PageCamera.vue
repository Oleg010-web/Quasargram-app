<template>
  <q-page class="constrain-more q-pa-md">
    <div class="camera-frame q-pa-md">
      <video
        v-show="!imageCaptured"
        ref="video"
        class="full-width"
        autoplay
        playsinline
      />
      <canvas
        v-show="imageCaptured"
        ref="canvas"
        class="full-width"
        height="200"
      />
    </div>
    <div class="text-center q-pa-md">
      <q-btn
        v-if="hasCameraSupport"
        @click="captureImage"
        :disable="imageCaptured"
        round
        color="grey-10"
        size="lg"
        icon="eva-camera"
      />
      <q-file
        v-else
        @update:model-value="captureImageFallBack"
        type="file"
        v-model="imageUpload"
        label="Choose an image"
        accept="image/*"
        outlined
      >
        <template v-slot:prepend>
          <q-icon name="eva-attach-outline" />
        </template>
      </q-file>
      <div class="row justify-center q-ma-md">
        <q-input
          class="col col-sm-6"
          v-model="post.caption"
          label="Caption *"
          dense
        />
      </div>
      <div class="row justify-center q-ma-md">
        <q-input
          class="col col-sm-6"
          :loading="locationLoading"
          v-model="post.location"
          label="Location"
          dense
        >
          <template v-slot:append>
            <q-btn
              v-if="!locationLoading"
              @click="getLocation"
              round
              dense
              flat
              icon="eva-navigation-2-outline"
            />
          </template>
        </q-input>
      </div>
      <div class="row justify-center q-mt-lg">
        <q-btn
          @click="addPost"
          :disable="!post.caption || post.photo"
          unelevated
          rounded
          color="primary"
          label="Post Image"
        />
      </div>
    </div>
  </q-page>
</template>

<script setup>
//imports
import { uid } from 'quasar'
import { useQuasar } from 'quasar'
import { reactive, onMounted, ref, onBeforeUnmount, watch } from 'vue'
import * as locales from 'md-gum-polyfill'
import { dataUrItoBlob } from 'src/composable/useFileLikeObject'
import {
  disableCamera,
  video,
  hasCameraSupport,
  initCamera
} from 'src/state/camera'
import {
  locationError,
  locationLoading,
  postLocation
} from 'src/state/location'
import { getSityandCountry } from 'src/api/posts/getSityandCountry'
import { sendPost } from 'src/api/posts/sendPost'
import { useRouter } from 'vue-router'
//data
let post = reactive({
  id: uid(),
  caption: '',
  location: '',
  photo: null,
  date: Date.now()
})

const imageCaptured = ref(false)
const imageUpload = ref([])
const $q = useQuasar()
const router = useRouter()

let canvas = ref()

//methods

const captureImage = () => {
  canvas.value.width = video.value.getBoundingClientRect().width
  canvas.value.height = video.value.getBoundingClientRect().height
  let context = canvas.value.getContext('2d')
  context.drawImage(video.value, 0, 0, canvas.value.width, canvas.value.height)
  imageCaptured.value = true
  post.photo = dataUrItoBlob(canvas.value.toDataURL())
  disableCamera()
}

const captureImageFallBack = file => {
  post.photo = file

  let context = canvas.value.getContext('2d')
  let reader = new FileReader()
  reader.onload = event => {
    let img = new Image()
    img.onload = () => {
      canvas.value.width = img.width
      canvas.value.height = img.height
      context.drawImage(img, 0, 0)
      imageCaptured.value = true
    }
    img.src = event.target.result
  }
  reader.readAsDataURL(file)
}

const getLocation = () => {
  locationLoading.value = true
  navigator.geolocation.getCurrentPosition(
    position => {
      getSityandCountry(position)
      post.location = postLocation
    },
    err => {
      locationError()
    },
    { timeout: 7000 }
  )
}

const addPost = async () => {
  let formData = new FormData()
  formData.append('id', post.id)
  formData.append('caption', post.caption)
  formData.append('location', post.location)
  formData.append('date', post.date)
  formData.append('file', post.photo, post.id + '.png')

  post = await sendPost(formData)
  console.log(post)
  if (post) {
    router.push('/')
  }
}

//hooks
onMounted(() => {
  initCamera()
})

onBeforeUnmount(() => {
  if (hasCameraSupport.value) {
    disableCamera()
  }
})
</script>

<style lang="sass">
.camera-frame
  border: 2px solid $grey-10
  border-radius: 10px
</style>
