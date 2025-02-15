<template>
	<div v-if="successMessage">
		<message :type="MessageType.Success" :text="successMessage" />
	</div>
	<div v-else>
		<template v-if="error">
			<message :type="MessageType.Error" :text="error" @close="onErrorClose" />
		</template>
		<form @submit.prevent="onSubmit">
			<!-- Email -->
			<div class="styled-form-group" :hasError="emailError">
				<label class="form-input-label" for="authorizer-sign-up-email"><span>* </span>Email</label>
				<input
					id="authorizer-sign-up-email"
					v-model="email"
					:class="`form-input-field ${emailError ? 'input-error-content' : null}`"
					placeholder="eg. foo@bar.com"
					type="email"
				/>
				<div v-if="emailError" class="form-input-error">{{ emailError }}</div>
			</div>

			<!-- Nickname -->
			<div class="styled-form-group" :hasError="nicknameError">
				<label class="form-input-label" for="authorizer-sign-up-email"
					><span>* </span>Nickname</label
				>
				<input
					id="authorizer-sign-up-email"
					v-model="nickname"
					:class="`form-input-field ${nicknameError ? 'input-error-content' : null}`"
					placeholder="eg. AzureDiamond"
					type="text"
				/>
				<div v-if="nicknameError" class="form-input-error">{{ nicknameError }}</div>
			</div>

			<!-- password -->
			<div class="styled-form-group" :hasError="passwordError">
				<label class="form-input-label" for="authorizer-sign-up-password"
					><span>* </span>Password</label
				>
				<input
					id="authorizer-sign-up-password"
					v-model="password"
					:class="`form-input-field ${passwordError ? 'input-error-content' : null}`"
					placeholder="********"
					type="password"
				/>
				<div v-if="passwordError" class="form-input-error">
					{{ passwordError }}
				</div>
			</div>

			<!-- confirm password -->
			<div class="styled-form-group" :hasError="confirmPasswordError">
				<label class="form-input-label" for="authorizer-sign-up-confirm-password"
					><span>* </span>Confirm Password</label
				>
				<input
					id="authorizer-sign-up-confirm-password"
					v-model="confirmPassword"
					:class="`form-input-field ${confirmPasswordError ? 'input-error-content' : null}`"
					placeholder="********"
					type="password"
				/>
				<div v-if="confirmPasswordError" class="form-input-error">
					{{ confirmPasswordError }}
				</div>
			</div>
			<template v-if="config.is_strong_password_enabled.value">
				<password-strength-indicator
					:value="password || undefined"
					:set-disable-button="setDisableButton"
				/>
				<br />
			</template>
			<styled-button
				:appearance="ButtonAppearance.Primary"
				:disabled="
					!!emailError ||
					!!passwordError ||
					!!confirmPasswordError ||
					!email ||
					!password ||
					!confirmPassword ||
					loading ||
					disableSignupButton
				"
			>
				<template v-if="loading">Processing ...</template>
				<template v-else>Sign Up</template>
			</styled-button>
		</form>
		<template v-if="setView">
			<styled-footer>
				<div>
					Already have an account?
					<styled-link @click="() => setView(Views.Login)">Log In</styled-link>
				</div>
			</styled-footer>
		</template>
	</div>
</template>

<script lang="ts">
import { reactive, toRefs, computed, type PropType } from 'vue';
import type { AuthToken, SignupInput } from '@authorizerdev/authorizer-js';
import globalConfig from '../state/globalConfig';
import globalContext from '../state/globalContext';
import { StyledButton, StyledFooter, StyledLink } from '../styledComponents/index';
import { Views, MessageType, ButtonAppearance } from '../constants/index';
import { isValidEmail } from '../utils/common';
import Message from './Message.vue';
import PasswordStrengthIndicator from './PasswordStrengthIndicator.vue';
import type { URLPropsType } from '../types';
export default {
	name: 'AuthorizerSignup',
	components: {
		'password-strength-indicator': PasswordStrengthIndicator,
		'styled-button': StyledButton,
		'styled-footer': StyledFooter,
		'styled-link': StyledLink,
		message: Message
	},
	props: {
		setView: {
			type: Function as PropType<(arg: Views) => void>,
			default: (_v: Views) => undefined
		},
		onSignup: {
			type: Function as PropType<(arg: AuthToken | void) => void>,
			default: undefined
		},
		urlProps: {
			type: Object as PropType<URLPropsType>,
			default: undefined
		},
		roles: {
			type: Object as PropType<string[]>,
			default: undefined
		}
	},
	setup(props) {
		const config = toRefs(globalConfig);
		const { setAuthData, authorizerRef } = toRefs(globalContext);
		const componentState: {
			error: null | string;
			successMessage: null | string;
			loading: boolean;
			disableSignupButton: boolean;
		} = reactive({
			error: null,
			successMessage: null,
			loading: false,
			disableSignupButton: false
		});
		const formData: {
			email: null | string;
			nickname: null | string;
			nickname: null | string;
			password: null | string;
			confirmPassword: null | string;
		} = reactive({
			email: null,
			nickname: null,
			password: null,
			confirmPassword: null
		});
		const emailError = computed((): string | null => {
			if (formData.email === '') {
				return 'Email is required';
			}
			if (formData.email && !isValidEmail(formData.email)) {
				return 'Please enter valid email';
			}
			return null;
		});
		const nicknameError = computed((): string | null => {
			if (formData.nickname === '') {
				return 'Nickname is required';
			}
			return null;
		});
		const passwordError = computed((): string | null => {
			if (formData.password === '') {
				return 'Password is required';
			}
			if (
				formData.password &&
				formData.confirmPassword &&
				formData.confirmPassword !== formData.password
			) {
				return `Password and confirm passwords don't match`;
			}
			return null;
		});
		const confirmPasswordError = computed((): string | null => {
			if (formData.confirmPassword === '') {
				return 'Confirm password is required';
			}
			if (
				formData.password &&
				formData.confirmPassword &&
				formData.confirmPassword !== formData.password
			) {
				return `Password and confirm passwords don't match`;
			}
			return null;
		});
		const onSubmit = async () => {
			try {
				componentState.loading = true;
				const data: SignupInput = {
					email: formData.email || '',
					password: formData.password || '',
					confirm_password: formData.confirmPassword || ''
				};
				if (props.urlProps?.scope) {
					data.scope = props.urlProps.scope;
				}
				if (props.urlProps?.roles) {
					data.roles = props.urlProps.roles;
				}
				if (props.urlProps?.redirect_uri) {
					data.redirect_uri = props.urlProps.redirect_uri;
				}
				if (props.urlProps?.state) {
					data.state = props.urlProps.state;
				}
				if (props.roles && props.roles.length) {
					data.roles = [...(data.roles || []), ...props.roles];
				}
				const res = await authorizerRef.value.signup(data);
				if (res) {
					componentState.error = null;
					if (res.access_token) {
						componentState.error = null;
						setAuthData.value({
							user: res.user || null,
							token: {
								access_token: res.access_token,
								expires_in: res.expires_in,
								refresh_token: res.refresh_token,
								id_token: res.id_token
							},
							config: globalConfig,
							loading: false
						});
					} else {
						componentState.error = null;
						componentState.successMessage = res?.message ? res.message : null;
					}
					if (props.onSignup) {
						props.onSignup(res);
					}
				}
			} catch (error: unknown) {
				componentState.loading = false;
				componentState.error = error instanceof Error ? error.message : 'Internal error!';
			}
		};
		const onErrorClose = () => {
			componentState.error = null;
		};
		const setDisableButton = (value: boolean) => {
			componentState.disableSignupButton = value;
		};
		return {
			...toRefs(componentState),
			...toRefs(formData),
			config,
			onSubmit,
			onErrorClose,
			MessageType,
			ButtonAppearance,
			Views,
			emailError,
			passwordError,
			nicknameError,
			confirmPasswordError,
			setDisableButton
		};
	}
};
</script>
<style scoped>
@import '../styles/default.css';
</style>
