<script setup lang="ts">
import { ref, computed } from 'vue';

const emit = defineEmits<{
  (e: 'create-successful'): void;
}>();

const props = defineProps<{
  token?: string;
}>();

const username = ref('');
const password = ref('');
const submitted = ref(false);
const isSubmitting = ref(false);
const apiError = ref<string | null>(null);

const activePasswordErrors = computed(() => {
  const errors: string[] = [];
  const val = password.value;

  if (val.length < 10) {
    errors.push('Password must be at least 10 characters long');
  }
  if (val.length > 24) {
    errors.push('Password must be at most 24 characters long');
  }
  if (/\s/.test(val)) {
    errors.push('Password cannot contain spaces');
  }
  if (!/\d/.test(val)) {
    errors.push('Password must contain at least one number');
  }
  if (!/[A-Z]/.test(val)) {
    errors.push('Password must contain at least one uppercase letter');
  }
  if (!/[a-z]/.test(val)) {
    errors.push('Password must contain at least one lowercase letter');
  }

  return errors;
});

const isPasswordInvalid = computed(() => activePasswordErrors.value.length > 0);
const isUsernameInvalid = computed(() => username.value.trim() === '');

function getAuthToken(): string {
  if (props.token) return props.token;
  if (typeof window === 'undefined') return '';

  const pathname = window.location.pathname;
  const match = pathname.match(/\/challenge-details\/([^/]+)/);
  if (match && match[1]) {
    return match[1];
  }

  const searchParams = new URLSearchParams(window.location.search);
  if (searchParams.has('token')) {
    return searchParams.get('token') || '';
  }

  const segments = pathname.split('/').filter(Boolean);
  return segments[segments.length - 1] || '';
}

async function handleSubmit(event: Event) {
  event.preventDefault();
  submitted.value = true;
  apiError.value = null;

  if (isUsernameInvalid.value || isPasswordInvalid.value) {
    return;
  }

  isSubmitting.value = true;

  try {
    const token = getAuthToken();
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
    };
    if (token) {
      headers['Authorization'] = `Bearer ${token}`;
    }

    const response = await fetch(
      'https://api.challenge.hennge.com/password-validation-challenge-api/001/challenge-signup',
      {
        method: 'POST',
        headers,
        body: JSON.stringify({
          username: username.value,
          password: password.value,
        }),
      }
    );

    if (response.ok) {
      emit('create-successful');
    } else if (response.status === 401 || response.status === 403) {
      apiError.value = 'Not authenticated to access this resource.';
    } else if (response.status === 500) {
      apiError.value = 'Something went wrong, please try again.';
    } else {
      try {
        const data = await response.json();
        if (data && Array.isArray(data.errors) && data.errors.includes('not_allowed')) {
          apiError.value = 'Sorry, the entered password is not allowed, please try a different one.';
        } else {
          apiError.value = 'Something went wrong, please try again.';
        }
      } catch {
        apiError.value = 'Something went wrong, please try again.';
      }
    }
  } catch {
    apiError.value = 'Something went wrong, please try again.';
  } finally {
    isSubmitting.value = false;
  }
}
</script>

<template>
  <div class="form-wrapper">
    <form class="form" @submit="handleSubmit">
      <label for="username">Username</label>
      <input
        id="username"
        name="username"
        v-model="username"
        type="text"
        aria-label="Username"
        :aria-invalid="submitted && isUsernameInvalid"
      />

      <label for="password">Password</label>
      <input
        id="password"
        name="password"
        v-model="password"
        type="password"
        aria-label="Password"
        :aria-invalid="isPasswordInvalid"
      />

      <ul
        v-if="activePasswordErrors.length > 0"
        class="validation-errors"
        aria-live="polite"
      >
        <li v-for="error in activePasswordErrors" :key="error">
          {{ error }}
        </li>
      </ul>

      <div
        v-if="apiError"
        class="api-error"
        role="alert"
      >
        {{ apiError }}
      </div>

      <button type="submit" class="submit-button" :disabled="isSubmitting">
        Create User
      </button>
    </form>
  </div>
</template>

<style scoped>
.form-wrapper {
  max-width: 500px;
  width: 80%;
  background-color: #efeef5;
  padding: 24px;
  margin: auto;
  border-radius: 8px;
}

.form {
  display: flex;
  gap: 8px;
  flex-direction: column;
}

label {
  font-weight: 700;
}

input {
  outline: none;
  padding: 8px 16px;
  height: 40px;
  font-size: 14px;
  background-color: #f8f7fa;
  border: 1px solid rgba(0, 0, 0, 0.12);
  border-radius: 4px;
}

.submit-button {
  outline: none;
  border-radius: 4px;
  border: 1px solid rgba(0, 0, 0, 0.12);
  background-color: #7135d2;
  color: white;
  font-size: 16px;
  font-weight: 500;
  height: 40px;
  padding: 0 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 8px;
  align-self: flex-end;
  cursor: pointer;
}

.validation-errors {
  color: #d93025;
  font-size: 13px;
  margin-top: 4px;
  margin-bottom: 4px;
  padding-left: 20px;
  list-style-type: disc;
}

.validation-errors li {
  margin-bottom: 2px;
}

.api-error {
  color: #d93025;
  font-size: 14px;
  font-weight: 500;
  margin-top: 8px;
  padding: 8px 12px;
  background-color: #fce8e6;
  border: 1px solid #f5c6cb;
  border-radius: 4px;
}
</style>

