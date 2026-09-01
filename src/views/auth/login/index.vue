<!-- 登录页面 -->
<template>
  <div class="flex w-full h-screen">
    <LoginLeftView />

    <div class="relative flex-1">
      <AuthTopBar />

      <div class="auth-right-wrap">
        <div class="form">
          <h3 class="title">{{ $t('login.title') }}</h3>
          <p class="sub-title">{{ $t('login.subTitle') }}</p>
          <ElForm
            ref="formRef"
            :model="loginForm"
            :rules="rules"
            :key="formKey"
            @keyup.enter="handleSubmit"
            style="margin-top: 25px"
          >
            <ElFormItem v-if="tenantEnabled" prop="tenantId">
              <ElSelect v-model="loginForm.tenantId" filterable>
                <ElOption
                  v-for="tenant in tenantList"
                  :key="tenant.tenantId"
                  :label="tenant.companyName"
                  :value="tenant.tenantId"
                >
                  <span>{{ tenant.companyName }}</span>
                </ElOption>
                <template #prefix>
                  <ArtSvgIcon icon="ri-building-line" class="el-input__icon input-icon" />
                </template>
              </ElSelect>
            </ElFormItem>
            <ElFormItem prop="username">
              <ElInput
                class="custom-height"
                :placeholder="$t('login.placeholder.username')"
                v-model.trim="loginForm.username"
              >
                <template #prefix>
                  <ArtSvgIcon icon="ri-user-line" class="el-input__icon input-icon" />
                </template>
              </ElInput>
            </ElFormItem>
            <ElFormItem prop="password">
              <ElInput
                class="custom-height"
                :placeholder="$t('login.placeholder.password')"
                v-model.trim="loginForm.password"
                type="password"
                autocomplete="off"
                show-password
              >
                <template #prefix>
                  <ArtSvgIcon icon="ri-lock-password-line" class="el-input__icon input-icon" />
                </template>
              </ElInput>
            </ElFormItem>
            <ElFormItem v-if="captchaEnabled" prop="code">
              <ElInput
                class="custom-height"
                :placeholder="$t('login.placeholder.code')"
                v-model="loginForm.code"
                auto-complete="off"
                style="width: 63%"
                @keyup.enter="handleSubmit"
              >
                <template #prefix>
                  <!-- <svgIcon icon-class="validCode" class="el-input__icon input-icon" /> -->
                  <ArtSvgIcon icon="ri:shield-keyhole-line" class="el-input__icon input-icon" />
                </template>
              </ElInput>
              <div class="login-code">
                <img :src="codeUrl" class="login-code-img" @click="getCode" />
              </div>
            </ElFormItem>

            <!-- 推拽验证 -->
            <div class="relative pb-5 mt-6" v-if="dragVerifyEnable">
              <div
                class="relative z-[2] overflow-hidden select-none rounded-lg border border-transparent tad-300"
                :class="{ '!border-[#FF4E4F]': !isPassing && isClickPass }"
              >
                <ArtDragVerify
                  ref="dragVerify"
                  v-model:value="isPassing"
                  :text="$t('login.sliderText')"
                  textColor="var(--art-gray-700)"
                  :successText="$t('login.sliderSuccessText')"
                  progressBarBg="var(--main-color)"
                  :background="isDark ? '#26272F' : '#F1F1F4'"
                  handlerBg="var(--default-box-color)"
                />
              </div>
              <p
                class="absolute top-0 z-[1] px-px mt-2 text-xs text-[#f56c6c] tad-300"
                :class="{ 'translate-y-10': !isPassing && isClickPass }"
              >
                {{ $t('login.placeholder.slider') }}
              </p>
            </div>

            <div class="flex-cb mt-2 text-sm">
              <ElCheckbox v-model="loginForm.rememberMe">{{ $t('login.rememberPwd') }}</ElCheckbox>
              <RouterLink class="text-theme" :to="{ name: 'ForgetPassword' }">{{
                $t('login.forgetPwd')
              }}</RouterLink>
            </div>

            <div style="margin-top: 30px">
              <ElButton
                class="w-full custom-height"
                type="primary"
                @click="handleSubmit"
                :loading="loading"
                v-ripple
              >
                {{ $t('login.btnText') }}
              </ElButton>
            </div>

            <div class="mt-5 text-sm text-gray-600">
              <span>{{ $t('login.noAccount') }}</span>
              <RouterLink class="text-theme" :to="{ name: 'Register' }">{{
                $t('login.register')
              }}</RouterLink>
            </div>
          </ElForm>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
  import AppConfig from '@/config'
  import { getCodeImg, getTenantList } from '@/api/login'
  import { LoginData, TenantVO } from '@/api/types'
  import { useUserStore } from '@/store/modules/user'
  import { useI18n } from 'vue-i18n'
  import { HttpError } from '@/utils/http/error'
  import { fetchLogin } from '@/api/auth'
  import { ElNotification, type FormInstance, type FormRules } from 'element-plus'
  import { useSettingStore } from '@/store/modules/setting'

  // const { proxy } = getCurrentInstance() as ComponentInternalInstance;

  defineOptions({ name: 'Login' })

  const settingStore = useSettingStore()
  const { isDark } = storeToRefs(settingStore)
  const { t, locale } = useI18n()
  const formKey = ref(0)

  // 监听语言切换，重置表单
  watch(locale, () => {
    formKey.value++
  })

  type AccountKey = 'super' | 'admin' | 'user'

  export interface Account {
    key: AccountKey
    label: string
    userName: string
    password: string
    roles: string[]
  }

  const dragVerifyEnable = ref(false)
  const dragVerify = ref()

  const userStore = useUserStore()
  const router = useRouter()
  const route = useRoute()
  const isPassing = ref(false)
  const isClickPass = ref(false)

  const codeUrl = ref('')
  // 验证码开关
  const captchaEnabled = ref(false)
  // 租户开关
  const tenantEnabled = ref(true)
  // 租户列表
  const tenantList = ref<TenantVO[]>([])

  const systemName = AppConfig.systemInfo.name
  const formRef = ref<FormInstance>()

  const loginForm = ref<LoginData>({
    tenantId: '000000',
    username: '',
    password: '',
    rememberMe: false,
    code: '',
    uuid: ''
  } as LoginData)

  const rules = computed<FormRules>(() => ({
    tenantId: [{ required: true, trigger: 'blur', message: t('login.placeholder.tenantId') }],
    username: [{ required: true, trigger: 'blur', message: t('login.placeholder.username') }],
    password: [{ required: true, trigger: 'blur', message: t('login.placeholder.password') }],
    code: [{ required: true, trigger: 'change', message: t('login.placeholder.code') }]
  }))

  const loading = ref(false)

  /**
   * 获取验证码
   */
  const getCode = async () => {
    const res = await getCodeImg()
    const { data } = res
    captchaEnabled.value = data.captchaEnabled === undefined ? true : data.captchaEnabled
    if (captchaEnabled.value) {
      // 刷新验证码时清空输入框
      loginForm.value.code = ''
      codeUrl.value = 'data:image/gif;base64,' + data.img
      loginForm.value.uuid = data.uuid
    }
  }

  /**
   * 获取租户列表
   */
  const initTenantList = async () => {
    const { data } = await getTenantList(false)
    tenantEnabled.value = data.tenantEnabled === undefined ? true : data.tenantEnabled
    if (tenantEnabled.value) {
      tenantList.value = data.voList
      if (tenantList.value != null && tenantList.value.length !== 0) {
        loginForm.value.tenantId = tenantList.value[0].tenantId
      }
    }
  }

  onMounted(() => {
    getCode()
    initTenantList()
  })

  // 登录
  const handleSubmit = async () => {
    if (loading.value) {
      return
    }
    if (!formRef.value) return

    // try {
    // 表单验证
    const valid = await formRef.value.validate()
    if (!valid) {
      return
    }
    // } catch (err) {
    // return
    // }

    // 拖拽验证
    if (dragVerifyEnable.value && !isPassing.value) {
      isClickPass.value = true
      return
    }

    try {
      loading.value = true

      // 勾选了需要记住密码设置在 localStorage 中设置记住用户名和密码
      if (loginForm.value.rememberMe) {
        localStorage.setItem('tenantId', String(loginForm.value.tenantId))
        localStorage.setItem('username', String(loginForm.value.username))
        localStorage.setItem('password', String(loginForm.value.password))
        localStorage.setItem('rememberMe', String(loginForm.value.rememberMe))
      } else {
        // 否则移除
        localStorage.removeItem('tenantId')
        localStorage.removeItem('username')
        localStorage.removeItem('password')
        localStorage.removeItem('rememberMe')
      }

      // 调用action的登录方法
      await userStore.login(loginForm.value)

      // if (!err) {
      //   const redirectUrl = redirect.value || '/';
      //   await router.push(redirectUrl);
      //   loading.value = false;
      // } else {
      //   loading.value = false;
      //   // 重新获取验证码
      //   if (captchaEnabled.value) {
      //     await getCode();
      //   }
      // }

      // 登录请求
      const { token, refreshToken } = await fetchLogin({
        // userName: 'Super',
        // userName: 'Admin',
        userName: 'User',
        password: '123456'
      })

      // 验证token
      if (!token) {
        throw new Error('Login failed - no token received')
      }

      // 存储 token 和登录状态
      userStore.setToken(token, refreshToken)
      userStore.setLoginStatus(true)

      // 登录成功处理
      setTimeout(() => {
        ElNotification({
          title: t('login.success.title'),
          type: 'success',
          duration: 2500,
          zIndex: 10000,
          message: `${t('login.success.message')}, ${systemName}!`
        })
      }, 1000)

      // 获取 redirect 参数，如果存在则跳转到指定页面，否则跳转到首页
      const redirect = route.query.redirect as string
      router.push(redirect || '/')
    } catch (error) {
      // 处理 HttpError
      if (error instanceof HttpError) {
        // console.log(error.code)
      } else {
        // 处理非 HttpError
        // ElMessage.error('登录失败，请稍后重试')
        await getCode()
        console.error('[Login] Unexpected error:', error)
      }
    } finally {
      loading.value = false
      resetDragVerify()
    }
  }

  // 重置拖拽验证
  const resetDragVerify = () => {
    if (dragVerifyEnable.value) {
      dragVerify.value.reset()
    }
  }
</script>

<style scoped>
  @import './style.css';
</style>

<style lang="scss" scoped>
  :deep(.el-select__wrapper) {
    height: 40px !important;
  }
</style>
