<template>
  <div>
    <div class="modal-body">
      <form class="signup-form" @submit.prevent>
        <div class="upload-img-section">
          <img
            class="upload-img"
            v-if="currentUserInfo.icon_image_url !== undefined"
            :src="currentUserInfo.icon_image_url" />
          <img
            class="upload-img"
            v-else-if="uploadedImage"
            :src="uploadedImage" />
          <div class="upload-img-dammy" v-else />
          <label class="upload-btn">
            <img class="btn-pic" src="~/assets/images/pc/common/btn_pic.png" alt="upload">
            <input class="upload-img-input" type="file" accept="image/*" @change="onFileChange">
          </label>
        </div>
        <div class="signup-form-group" :class="{ 'error': hasUserDisplayNameError }">
          <label class="signup-form-label">表示名</label>
          <input
            class="signup-form-input"
            type="text"
            minlength="1"
            maxlength="30"
            v-model="userDisplayName"
            @input="setUserDisplayName($event.target.value)"
            @blur="showError('userDisplayName')"
            @focus="resetError('userDisplayName')">
          <label class="signup-form-label">自己紹介</label>
        </div>
        <div class="signup-form-group">
          <textarea
            class="signup-form-textarea"
            type="text"
            placeholder="自己紹介を入力してください"
            maxlength="100"
            v-model="selfIntroduction"
            @input="setSelfIntroduction($event.target.value)"/>
        </div>
      </form>
    </div>
    <div class="modal-footer">
      <app-button class="to-next-step-button" :disabled="invalidSubmit" @click="onSubmit">
        完了する
      </app-button>
    </div>
  </div>
</template>

<script>
import { mapActions, mapGetters } from 'vuex'
import { required } from 'vuelidate/lib/validators'
import { ADD_TOAST_MESSAGE } from 'vuex-toast'
import AppButton from '../atoms/AppButton'
import { htmlDecode } from '~/utils/article'

export default {
  data() {
    return {
      userDisplayName: '',
      selfIntroduction: '',
      uploadedImage: ''
    }
  },
  components: {
    AppButton
  },
  async created() {
    await this.setCurrentUserInfo()
    this.userDisplayName =
      htmlDecode(this.currentUserInfo.user_display_name) || this.currentUser.userId
    this.selfIntroduction = htmlDecode(this.currentUserInfo.self_introduction) || ''
    this.setUserDisplayName(this.userDisplayName)
    this.setSelfIntroduction(this.selfIntroduction)
  },
  computed: {
    invalidSubmit() {
      return this.$v.profileSettingsModal.formData.$invalid
    },
    hasUserDisplayNameError() {
      return (
        this.profileSettingsModal.formError.userDisplayName &&
        this.$v.profileSettingsModal.formData.userDisplayName.$error
      )
    },
    ...mapGetters('user', [
      'currentUser',
      'signUpAuthFlowModal',
      'showSignUpAuthFlowModal',
      'profileSettingsModal',
      'currentUserInfo'
    ])
  },
  validations: {
    profileSettingsModal: {
      formData: {
        userDisplayName: {
          required
        }
      }
    }
  },
  methods: {
    onFileChange(e) {
      const files = e.target.files || e.dataTransfer.files
      this.createImage(files[0])
    },
    createImage(file) {
      if (file.size > 4.5 * 1024 * 1024) {
        alert('アイコン画像は4.5MBまでアップロード可能です')
        return
      }
      const reader = new FileReader()
      reader.onload = async (e) => {
        try {
          const base64Image = e.target.result
          const base64Hash = base64Image.substring(base64Image.match(',').index + 1)
          const imageContentType = base64Image.substring(
            base64Image.match(':').index + 1,
            base64Image.match(';').index
          )
          await this.postUserIcon({ iconImage: base64Hash, imageContentType })
          await this.setCurrentUserInfo()
          this.uploadedImage = base64Image
        } catch (error) {
          const toastMessage = document.querySelector('.toast')
          const modalMask = document.querySelector('.modal-mask')
          const modalMaskZIndex = Number(
            window.getComputedStyle(modalMask).getPropertyValue('z-index')
          )
          const originalToastZIndex = toastMessage.style.zIndex
          toastMessage.style.zIndex = modalMaskZIndex + 1
          this.sendNotification({
            text: 'アイコン画像をアップロードできませんでした。',
            type: 'warning'
          })
          setTimeout(() => {
            toastMessage.style.zIndex = originalToastZIndex
          }, 2500)
          console.error(error)
        }
      }
      reader.readAsDataURL(file)
    },
    setUserDisplayName(userDisplayName) {
      this.setProfileSettingsUserDisplayName({ userDisplayName })
    },
    setSelfIntroduction(selfIntroduction) {
      this.setProfileSettingsSelfIntroduction({ selfIntroduction })
    },
    showError(type) {
      this.$v.profileSettingsModal.formData[type].$touch()
      this.showProfileSettingsError({ type })
    },
    resetError(type) {
      this.$v.profileSettingsModal.formData[type].$reset()
      this.hideProfileSettingsError({ type })
    },
    async onSubmit() {
      if (this.invalidSubmit) return
      const { userDisplayName, selfIntroduction } = this.profileSettingsModal.formData
      const formattedSelfIntroduction = selfIntroduction.replace(/\r?\n/g, '')
      try {
        await this.putUserInfo({ userDisplayName, selfIntroduction: formattedSelfIntroduction })
        await this.setCurrentUserInfo()

        document.querySelector('html,body').style.overflow = ''
        if (this.showSignUpAuthFlowModal) {
          this.setSignUpAuthFlowModal({
            showSignUpAuthFlowModal: false
          })
          this.setSignUpAuthFlowProfileSettingsModal({
            isSignUpAuthFlowProfileSettingsModal: false
          })
        } else {
          this.setProfileSettingsModal({
            showProfileSettingsModal: false
          })
        }
        this.sendNotification({ text: 'プロフィールを変更しました' })
      } catch (error) {
        this.sendNotification({ text: 'プロフィールを変更できませんでした。', type: 'warning' })
        console.error(error)
      }
    },
    ...mapActions({
      sendNotification: ADD_TOAST_MESSAGE
    }),
    ...mapActions('user', [
      'setSignUpAuthFlowProfileSettingsModal',
      'setProfileSettingsUserDisplayName',
      'setProfileSettingsSelfIntroduction',
      'showProfileSettingsError',
      'hideProfileSettingsError',
      'setSignUpAuthFlowModal',
      'setProfileSettingsModal',
      'putUserInfo',
      'postUserIcon',
      'setCurrentUserInfo'
    ])
  }
}
</script>

<style lang="scss" scoped>
.modal-body {
  margin: 0 auto;

  .signup-form {
    margin: 30px auto 0;
    max-width: 400px;
    width: 80%;

    .upload-img-section {
      position: relative;
      width: 120px;
      margin: 0 auto 30px;

      .upload-img {
        width: 120px;
        height: 120px;
        border-radius: 50%;
        object-fit: cover;
      }

      .upload-img-dammy {
        width: 120px;
        height: 120px;
        border-radius: 50%;
        background: #d8d8d8;
      }

      .upload-btn {
        cursor: pointer;
        position: absolute;
        bottom: -20px;
        right: -20px;

        .btn-pic {
          width: 65px;
        }
      }

      .upload-img-input {
        display: none;
      }
    }

    &-label {
      color: #030303;
      font-size: 14px;
      line-height: 20px;
    }

    &-input,
    &-textarea {
      border: none;
      border-radius: 0;
      border-bottom: 1px dotted #232538;
      margin-bottom: 30px;
      padding: 5px 0;
      width: 100%;
      resize: none;
      overflow: hidden;

      &::-webkit-input-placeholder {
        color: #cecece;
        font-size: 14px;
        letter-spacing: 0.05em;
      }

      &:focus {
        outline: 0;
      }
    }

    &-textarea {
      height: 4em;
    }

    .error {
      .signup-form {
        &-label {
          color: #f06273;
        }

        &-input {
          border-bottom: 1px dotted #f06273;
        }
      }
    }
  }
}

.modal-footer {
  width: 270px;
  margin: 90px auto 40px;

  .to-next-step-button {
    margin: 20px auto 0;
  }

  .error-message {
    color: #f06273;
    font-size: 12px;
    width: 100%;
  }
}

@media screen and (max-width: 320px) {
  .modal-footer {
    width: 250px;
    margin: 40px auto;
  }
}
</style>
