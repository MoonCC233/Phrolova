<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, shallowRef, ref, watch } from "vue";
import { useRoute } from "vue-router";
import { useI18n } from "vue-i18n";
import gsap from "gsap";

import { i18n } from "@/i18n";

import ModalOverlay from "@/components/shared/ModalOverlay.vue";
import { useAuthStore } from "@/stores/auth";
import { useMultiGameStore } from "@/stores/multiGame";
import type { CaptchaResponse } from "@/types";
import { errMsg } from "@/api/client";
import { fetchCaptcha, fetchScryptParams } from "@/api";
import { computeScryptHex } from "@/lib/scrypt-client";
import { getCookieConsent, setCookieConsent } from "@/composables/useStorage";
import { useOnlineCount } from "@/composables/useOnlineCount";

const route = useRoute();
const authStore = useAuthStore();
const multiGameStore = useMultiGameStore();
const { t, locale } = useI18n();
const { onlineCount, onlineConnected } = useOnlineCount();

const PORTRAIT_POOL = [
  { src: "/media/frolova.png", alt: "弗洛洛" },
  { src: "/media/jinhsi.png", alt: "今汐" },
  { src: "/media/phoebe.png", alt: "菲比" },
];

function randomPortrait() {
  return PORTRAIT_POOL[Math.floor(Math.random() * PORTRAIT_POOL.length)];
}

const currentPortrait = ref(randomPortrait());
const frolovaKey = ref(0);
const frolovaRef = shallowRef<HTMLElement | null>(null);
const menuRef = shallowRef<HTMLElement | null>(null);
let frolovaX: gsap.QuickToFunc | null = null;

let frolovaRX: gsap.QuickToFunc | null = null;
let frolovaRY: gsap.QuickToFunc | null = null;
let ctx: gsap.Context | null = null;

function onMouseMove(e: MouseEvent) {
  const cx = window.innerWidth / 2;
  const cy = window.innerHeight / 2;
  const tx = ((e.clientX - cx) / cx) * 30;
  const ty = ((e.clientY - cy) / cy) * 20;
  frolovaX?.(tx);

  frolovaRX?.(tx * 0.3);
  frolovaRY?.(-ty * 0.3);
}

function setupGsap() {
  ctx = gsap.context(() => {
    const portrait = frolovaRef.value;
    if (portrait) {
      frolovaX = gsap.quickTo(portrait, "x", { duration: 0.6, ease: "power2.out" });

      frolovaRX = gsap.quickTo(portrait, "rotateY", { duration: 0.6, ease: "power2.out" });
      frolovaRY = gsap.quickTo(portrait, "rotateX", { duration: 0.6, ease: "power2.out" });


    }

    const items = menuRef.value?.querySelectorAll(".menu-li");
    if (items) {
      gsap.from(items, {
        opacity: 0,
        x: 60,
        duration: 0.7,
        ease: "power2.out",
        stagger: 0.15,
        delay: 0.15,
        clearProps: "all",
      });
    }
  });
}

async function resetFrolova() {
  currentPortrait.value = randomPortrait();
  frolovaKey.value = 0;
  await nextTick();
  frolovaKey.value = Date.now();
  await nextTick();
  ctx?.revert();
  setupGsap();
}

watch(() => route.name, (name) => {
  if (name === "home") resetFrolova();
}, { immediate: true });

onMounted(() => {
  setupGsap();
});

onBeforeUnmount(() => {
  ctx?.revert();
});

const ANNOUNCE_KEY = "phrolova_announcement_v1";
const showAnnouncement = shallowRef(false);
const showCookieConsent = shallowRef(false);
const showAuthModal = shallowRef(false);

const authMode = shallowRef<"login" | "register">("login");
const resetMode = shallowRef(false);
const captchaImage = shallowRef("");
const captchaId = shallowRef("");
const localError = shallowRef("");
const form = reactive({
  username: "",
  password: "",
  captchaText: "",
});
const resetForm = reactive({
  oldPassword: "",
  newPassword: "",
  confirmPassword: "",
  captchaText: "",
});
const upgrading = shallowRef(false);

const menuItems = [
  { labelKey: "nav.singlePlayer", enKey: "nav.singlePlayer", to: "/single", icon: "ph:user-duotone" },
  { labelKey: "nav.multiplayer", enKey: "nav.multiplayer", to: "/multi", icon: "ph:users-three-duotone" },
  { labelKey: "nav.database", enKey: "nav.database", to: "/data", icon: "ph:database-duotone" },
  { labelKey: "nav.gameRules", enKey: "nav.gameRules", to: "/rules", icon: "ph:book-open-text-duotone" },
  { labelKey: "nav.leaderboard", enKey: "nav.leaderboard", to: "/leaderboard", icon: "ph:trophy-duotone" },
  { labelKey: "nav.acknowledgements", enKey: "nav.acknowledgements", to: "/acknowledgements", icon: "ph:hand-heart-duotone" },
] as const;

/** 获取指定 key 的英文翻译（不受当前 locale 影响） */
function enT(key: string) {
  return i18n.global.t(key, {}, { locale: "en" });
}

async function loadCaptcha() {
  const data = await fetchCaptcha();
  captchaImage.value = data.image || "";
  captchaId.value = data.captcha_id || "";
}

async function submitAuth() {
  localError.value = "";
  try {
    const payload = {
      username: form.username.trim(),
      password: form.password,
      captchaId: captchaId.value,
      captchaText: form.captchaText.trim(),
    };
    if (authMode.value === "login") {
      await authStore.login(payload);
    } else {
      await authStore.register(payload);
    }
    closeAuthModal();
  } catch (reason) {
    const errCode = String((reason as any)?.error_code ?? (reason as any)?.response?.data?.error_code ?? "");
    const httpStatus = (reason as any)?.response?.status ?? 0;
    if ((httpStatus === 426 || httpStatus === 400) && errCode === 'SCRYPT_UNAVAILABLE') {
      resetMode.value = true;
      resetForm.oldPassword = "";
      resetForm.newPassword = "";
      resetForm.confirmPassword = "";
      resetForm.captchaText = "";
      localError.value = t('auth.scryptDetected');
      loadCaptcha().catch(() => { /* ignore */ });
      return;
    }
    localError.value = errMsg(reason) || t('auth.accountFailed');
    await loadCaptcha();
    form.captchaText = "";
  }
}

async function submitResetPassword() {
  localError.value = "";
  if (!resetForm.oldPassword) {
    localError.value = t('auth.oldPasswordRequired');
    return;
  }
  if (resetForm.newPassword.length < 6) {
    localError.value = t('auth.newPasswordTooShort');
    return;
  }
  if (resetForm.newPassword !== resetForm.confirmPassword) {
    localError.value = t('auth.passwordMismatch');
    return;
  }

  upgrading.value = true;
  try {
    // 1. 获取 scrypt 参数（salt + N/r/p/dklen，不含哈希!）
    const params = await fetchScryptParams(form.username.trim());

    // 2. 浏览器端计算 scrypt(旧密码, salt, params) → derived_hex
    const oldPasswordHash = await computeScryptHex(resetForm.oldPassword, params);

    // 3. 发送 derived_hex 给服务端验证，验证通过则升级为 PBKDF2
    await authStore.upgradePassword({
      username: form.username.trim(),
      oldPasswordHash,
      newPassword: resetForm.newPassword,
      captchaId: captchaId.value,
      captchaText: resetForm.captchaText.trim(),
    });
    closeAuthModal();
  } catch (reason) {
    localError.value = errMsg(reason) || t('auth.upgradeFailed');
    await loadCaptcha();
    resetForm.captchaText = "";
  } finally {
    upgrading.value = false;
  }
}

function openAuthModal(mode: "login" | "register" = "login") {
  authMode.value = mode;
  resetMode.value = false;
  showAuthModal.value = true;
  localError.value = "";
  form.username = "";
  form.password = "";
  form.captchaText = "";
  resetForm.oldPassword = "";
  resetForm.newPassword = "";
  resetForm.confirmPassword = "";
  resetForm.captchaText = "";
  loadCaptcha().catch(() => {
    localError.value = t('auth.captchaLoadFailed');
  });
}

function closeAuthModal() {
  showAuthModal.value = false;
  resetMode.value = false;
}

function handleLogout() {
  multiGameStore.disconnect();
  authStore.logout();
}

onMounted(async () => {
  // 首次访问公告弹窗
  try {
    if (!localStorage.getItem(ANNOUNCE_KEY)) {
      showAnnouncement.value = true;
    }
  } catch { /* ignore */ }

  // 首次访问 Cookie 同意提示（仅未做选择时展示）
  try {
    if (getCookieConsent() === null) {
      showCookieConsent.value = true;
    }
  } catch { /* ignore */ }

  // hydrate() 已由 App.vue 初始化时调用，此处不再重复
  if (authStore.isAuthenticated) {
    await multiGameStore.resumeRoom().catch(() => undefined);
  }
});

function closeAnnouncement() {
  showAnnouncement.value = false;
  try { localStorage.setItem(ANNOUNCE_KEY, "1"); } catch { /* ignore */ }
}

function acceptCookies() {
  showCookieConsent.value = false;
  setCookieConsent("accepted");
}

function declineCookies() {
  showCookieConsent.value = false;
  setCookieConsent("declined");
}
</script>

<template>
  <section class="page-shell" @mousemove="onMouseMove">
    <div id="start-menu" class="home-stage">
      <div class="home-title-block">
        <h1 class="home-title">{{ t('home.title') }}</h1>
        <p class="home-subtitle">{{ t('home.subtitle') }}</p>
        <div v-if="onlineConnected" class="home-online-badge">
          <span class="home-online-dot home-online-dot--active"></span>
          <span class="home-online-text">{{ t('home.onlineCount', { count: onlineCount }) }}</span>
        </div>
        <div v-else class="home-online-badge">
          <span class="home-online-dot"></span>
          <span class="home-online-text">{{ t('home.connecting') }}</span>
        </div>
      </div>
      <div class="home-portrait" :key="frolovaKey" ref="frolovaRef">
        <div class="home-portrait-inner">
          <img :src="currentPortrait.src" :alt="currentPortrait.alt" />
        </div>
      </div>

      <div class="menu-container">
        <ul ref="menuRef" class="menu-list" aria-label="主菜单">
          <li v-for="(item, index) in menuItems" :key="item.to" class="menu-li">
            <RouterLink class="menu-item" :to="item.to">
              <Icon :icon="item.icon" class="menu-icon" aria-hidden="true" />
              <span class="menu-text">
                <span class="cn">{{ t(item.labelKey) }}</span>
                <span v-if="locale !== 'en'" class="en">{{ enT(item.enKey) }}</span>
              </span>
            </RouterLink>
          </li>
        </ul>
      </div>

      <div class="home-detail">
        <div v-if="!authStore.isAuthenticated" class="home-detail-card home-detail-card--ghost">
          <span class="feature-panel-kicker">{{ t('home.signInKicker') }}</span>
          <strong>{{ t('home.signInCta') }}</strong>
          <p>{{ t('home.signInDesc') }}</p>
          <button class="home-detail-link" type="button" @click="openAuthModal('login')">
            <Icon icon="ph:sign-in-duotone" class="home-detail-link-icon" aria-hidden="true" />
            {{ t('home.signInBtn') }}
          </button>
        </div>

        <div v-else class="home-detail-card home-identity-card">
          <span class="feature-panel-kicker">{{ t('home.playerKicker') }}</span>
          <div class="home-identity-header">
            <span class="home-identity-avatar">{{ authStore.playerId?.charAt(0)?.toUpperCase() }}</span>
            <strong class="home-identity-name">{{ authStore.playerId }}</strong>
            <span v-if="authStore.dbId != null" class="home-identity-dbid">#{{ authStore.dbId }}</span>
          </div>
          <div class="home-identity-stats">
            <div class="home-identity-stat">
              <span class="home-identity-stat-value">
                <Icon icon="ph:coin-duotone" class="home-stat-icon" aria-hidden="true" />
                {{ authStore.stats.score }}
              </span>
              <span class="home-identity-stat-label">{{ t('home.scoreLabel') }}</span>
            </div>
            <div class="home-identity-stat">
              <span class="home-identity-stat-value">
                <Icon icon="ph:crown-duotone" class="home-stat-icon" aria-hidden="true" />
                {{ authStore.stats.wins }}
              </span>
              <span class="home-identity-stat-label">{{ t('home.winsLabel') }}</span>
            </div>
            <div class="home-identity-stat">
              <span class="home-identity-stat-value">
                <Icon icon="ph:game-controller-duotone" class="home-stat-icon" aria-hidden="true" />
                {{ authStore.stats.matches }}
              </span>
              <span class="home-identity-stat-label">{{ t('home.matchesLabel') }}</span>
            </div>
          </div>
          <div class="home-identity-links">
            <RouterLink class="home-detail-link" to="/auth">{{ t('home.accountSettings') }}</RouterLink>
            <button class="home-detail-link" type="button" @click="handleLogout">{{ t('home.logout') }}</button>
          </div>
        </div>
      </div>

      <div class="home-social">
        <a href="https://github.com/xiaohuihui1022/Phrolova" target="_blank" rel="noopener" class="home-social-link">
          <Icon icon="ph:github-logo-duotone" /> GitHub
        </a>
        <span class="home-social-divider">·</span>
        <a href="https://phrolova.usotsuki-kaze.com/" target="_blank" rel="noopener" class="home-social-link">
          <Icon icon="ph:link-duotone" /> {{ t('home.friendLink') }}
        </a>
        <span class="home-social-divider">·</span>
        <a target="_blank"
          href="https://qm.qq.com/cgi-bin/qm/qr?k=hYVfe1ReAYkgLM71ZkQMKABUu8641H-B&jump_from=webapi&authKey=WP+actLjZvH3Q6/JHDiDjU1xj7HpjXNxPSoCB9skVZnH1w0I6jzVf2lvnCCP7I/B">
          <Icon icon="ph:chat-circle-dots-duotone" /> QQ群 457323277
        </a>
      </div>

      <div class="home-filings">
        <a href="https://beian.miit.gov.cn/" target="_blank" rel="noopener nofollow" class="home-filing-link">
          <Icon icon="ph:seal-check-duotone" class="home-filing-icon" aria-hidden="true" />
          苏ICP备2026062583号-1</a>
        <span class="home-social-divider">·</span>
        <a href="https://beian.mps.gov.cn/#/query/webSearch?code=32100102010396" target="_blank" rel="noopener nofollow" class="home-filing-link">
          <Icon icon="ph:seal-check-duotone" class="home-filing-icon" aria-hidden="true" />
          苏公网安备32100102010396号
        </a>
      </div>
    </div>

    <ModalOverlay v-if="showAnnouncement" panel-class="announce-modal" max-width="480px" @close="closeAnnouncement">
      <div class="announce-content">
        <div class="announce-icon"><Icon icon="ph:megaphone-duotone" /></div>
        <h2 class="announce-title">{{ t('home.announcementTitle') }}</h2>
        <div class="announce-body">
          <p>{{ t('home.announcementBody') }}</p>
          <div class="announce-section">
            <p class="announce-section-title"><Icon icon="ph:warning-circle-duotone" /> {{ t('home.announcementOldUserTitle') }}</p>
            <p>{{ t('home.announcementOldUserBody') }}</p>
          </div>
          <div class="announce-section">
            <p class="announce-section-title"><Icon icon="ph:chat-circle-dots-duotone" /> {{ t('home.announcementQQTitle') }}</p>
            <p>{{ t('home.announcementQQBody') }}</p>
          </div>
        </div>
        <button class="announce-btn" type="button" @click="closeAnnouncement">
          <Icon icon="ph:check-circle-duotone" /> {{ t('home.announcementBtn') }}
        </button>
      </div>
    </ModalOverlay>

    <ModalOverlay v-if="showAuthModal" panel-class="auth-modal" max-width="400px" @close="closeAuthModal">
      <header class="auth-modal-header">
        <p class="auth-modal-kicker">Account</p>
        <h2 class="auth-modal-title">{{ resetMode ? t('home.authTitleReset') : (authMode === "login" ? t('home.authTitle') : t('home.authTitleRegister')) }}</h2>
      </header>

      <div v-if="localError || authStore.error" class="auth-modal-error">
        {{ localError || authStore.error }}
      </div>

      <div class="auth-modal-body">
        <!-- 密码升级表单（需验证旧密码） -->
        <template v-if="resetMode">
          <p class="auth-modal-hint">{{ t('home.authResetHint', { username: form.username }) }}</p>

          <label class="auth-field">
            <span class="auth-field-label">{{ t('home.authOldPassword') }}</span>
            <input v-model="resetForm.oldPassword" class="auth-input" type="password" :placeholder="t('home.authOldPasswordPlaceholder')" />
          </label>

          <label class="auth-field">
            <span class="auth-field-label">{{ t('home.authNewPassword') }}</span>
            <input v-model="resetForm.newPassword" class="auth-input" type="password" :placeholder="t('home.authNewPasswordPlaceholder')" />
          </label>

          <label class="auth-field">
            <span class="auth-field-label">{{ t('home.authConfirmPassword') }}</span>
            <input v-model="resetForm.confirmPassword" class="auth-input" type="password" :placeholder="t('home.authConfirmPasswordPlaceholder')" />
          </label>

          <div class="auth-field">
            <span class="auth-field-label">{{ t('home.authCaptcha') }}</span>
            <div class="auth-captcha-row">
              <div class="auth-captcha-box" @click="loadCaptcha">
                <img v-if="captchaImage" class="auth-captcha-img" :src="captchaImage" :alt="t('home.authCaptcha')" />
                <span v-else class="auth-captcha-placeholder">{{ t('home.authCaptchaClickLoad') }}</span>
              </div>
              <input v-model="resetForm.captchaText" class="auth-input" type="text" maxlength="5" :placeholder="t('home.authCaptchaPlaceholder')" />
            </div>
          </div>

          <button class="auth-btn" type="button" :disabled="authStore.loading || upgrading" @click="submitResetPassword">
            {{ upgrading ? t('home.authVerifying') : t('home.authConfirmUpgrade') }}
          </button>

          <p class="auth-modal-foot">
            <button class="auth-switch-link" type="button" @click="resetMode = false">
              {{ t('home.authBackToLogin') }}
            </button>
          </p>
        </template>

        <!-- 登录/注册表单 -->
        <template v-else>
          <div class="auth-tabs">
            <button class="auth-tab" :class="{ 'auth-tab--active': authMode === 'login' }" type="button" @click="authMode = 'login'">{{ t('home.authTitle') }}</button>
            <button class="auth-tab" :class="{ 'auth-tab--active': authMode === 'register' }" type="button" @click="authMode = 'register'">{{ t('home.authTitleRegister') }}</button>
          </div>

          <label class="auth-field">
            <span class="auth-field-label">{{ t('home.authAccount') }}</span>
            <input v-model="form.username" class="auth-input" type="text" :placeholder="t('home.authAccountPlaceholder')" />
          </label>

          <label class="auth-field">
            <span class="auth-field-label">{{ t('home.authPassword') }}</span>
            <input v-model="form.password" class="auth-input" type="password" :placeholder="t('home.authPasswordPlaceholder')" />
          </label>

          <div class="auth-field">
            <span class="auth-field-label">{{ t('home.authCaptcha') }}</span>
            <div class="auth-captcha-row">
              <div class="auth-captcha-box" @click="loadCaptcha">
                <img v-if="captchaImage" class="auth-captcha-img" :src="captchaImage" :alt="t('home.authCaptcha')" />
                <span v-else class="auth-captcha-placeholder">{{ t('home.authCaptchaClickLoad') }}</span>
              </div>
              <input v-model="form.captchaText" class="auth-input" type="text" maxlength="5" :placeholder="t('home.authCaptchaPlaceholder')" />
            </div>
          </div>

          <button class="auth-btn" type="button" :disabled="authStore.loading" @click="submitAuth">
            {{ authMode === "login" ? t('home.authConfirmLogin') : t('home.authConfirmRegister') }}
          </button>

          <p class="auth-modal-foot">
            {{ authMode === "login" ? t('home.authNoAccount') : t('home.authHasAccount') }}
            <button class="auth-switch-link" type="button" @click="authMode = authMode === 'login' ? 'register' : 'login'">
              {{ authMode === "login" ? t('home.authGoRegister') : t('home.authGoLogin') }}
            </button>
          </p>
        </template>
      </div>
    </ModalOverlay>

    <Transition name="cookie-slide">
      <div v-if="showCookieConsent" class="cookie-consent" role="dialog" :aria-label="t('home.cookieTitle')">
        <div class="cookie-consent-icon"><Icon icon="ph:cookie-duotone" /></div>
        <div class="cookie-consent-body">
          <p class="cookie-consent-title">{{ t('home.cookieTitle') }}</p>
          <p class="cookie-consent-desc">{{ t('home.cookieDesc') }}</p>
          <div class="cookie-consent-actions">
            <button class="cookie-consent-btn cookie-consent-btn--primary" type="button" @click="acceptCookies">
              <Icon icon="ph:check-circle-duotone" /> {{ t('home.cookieAccept') }}
            </button>
            <button class="cookie-consent-btn cookie-consent-btn--ghost" type="button" @click="declineCookies">
              {{ t('home.cookieDecline') }}
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </section>
</template>

