********************************************
Keycloak - Integration with Angular Frontend
********************************************

Overview
########

Nội dung này mô tả tích hợp một Application Client (Angular Frontend) với Keycloak. Application được ví dụ ở đây là CRM Web: 

CRM web được sử dụng bởi các user thuộc realm **etc-internal**; Application này yêu cầu user phải đăng nhập trước khi có thể sử dụng bất cứ chức năng nào.

+---------------------------+-----------------------------------------------------------------------------------------+
| KEY                       | VALUE                                                                                   |
+===========================+=========================================================================================+
| issuer                    | https://keycloak.127.0.0.1.nip.io/auth/realms/etc-internal                              |
+---------------------------+-----------------------------------------------------------------------------------------+
| authorization_endpoint    | https://keycloak.127.0.0.1.nip.io/auth/realms/etc-internal/protocol/openid-connect/auth |
+---------------------------+-----------------------------------------------------------------------------------------+
| clientId                  | crm                                                                                     |
+---------------------------+-----------------------------------------------------------------------------------------+
| Application ROOT URL      | http://localhost:4200                                                                   |
+---------------------------+-----------------------------------------------------------------------------------------+

OAuth Flow được áp dụng: :doc:`Code Flow with PKCE <oauth2-quick-review>`

Cài đặt
#######

Module sử dụng: angular-oauth2-oidc
***********************************

.. code-block:: bash

  npm i angular-oauth2-oidc --save
  npm i angular-oauth2-oidc-jwks --save


Các cài đặt chính 
*****************

.. code-block:: javascript

	const origin = environment.production ? 'https://crm.etc.vn/init-auth' : 'http://localhost:4200';
	const keycloakAuthConfig: AuthConfig = {
		issuer: 'https://keycloak.127.0.0.1.nip.io/auth/realms/etc-internal',
		clientId: 'crm',
		redirectUri: `${origin}`,
		silentRefreshRedirectUri: `${origin}/silent-refresh.html`,
		scope: 'openid profile email',
		responseType: 'code',

		disableAtHashCheck: true,
		clearHashAfterLogin: true,
		showDebugInformation: true,
	};
	keycloakAuthConfig.logoutUrl =
		`${keycloakAuthConfig.issuer}v2/logout?client_id=${keycloakAuthConfig.clientId}&returnTo=${encodeURIComponent(keycloakAuthConfig.redirectUri)}`;
		

.. code-block:: javascript

	import {Injectable, Injector, isDevMode} from '@angular/core';
	import {JwtHelperService} from '@auth0/angular-jwt';
	import {AuthConfig, OAuthService} from 'angular-oauth2-oidc';
	import {filter} from 'rxjs/operators';
	import {JwksValidationHandler} from 'angular-oauth2-oidc-jwks';

	@Injectable({
		providedIn: 'root'
	})
	export class InitialAuthService {
		private jwtHelper: JwtHelperService = new JwtHelperService();

		private _decodedAccessToken: any;
		private _decodedIDToken: any;

		get decodedAccessToken() {
			return this._decodedAccessToken;
		}

		get decodedIDToken() {
			return this._decodedIDToken;
		}

		constructor(private oauthService: OAuthService,
					private authConfig: AuthConfig,
					private injector: Injector) {
			if (isDevMode()) {
				console.log('[InitialAuthService] authConfig: ', this.authConfig);
			}
		}

		async initAuth(): Promise<any> {
			return new Promise((resolveFn, rejectFn) => {
				this.oauthService.configure(this.authConfig);
				this.oauthService.setStorage(localStorage);
				// this.oauthService.tokenValidationHandler = new NullValidationHandler();
				this.oauthService.tokenValidationHandler = new JwksValidationHandler();

				// subscribe to token events
				this.oauthService.events
					.pipe(filter((e: any) => e.type === 'token_received'))
					.subscribe(() => this.handleNewToken());


				// continue initializing app (provoking a token_received event) or redirect to login-page
				this.oauthService.loadDiscoveryDocumentAndTryLogin().then(isLoggedIn => {
					if (this.isAuthenticated()) {
						console.log('[InitialAuthService] token exist and valid');
						this.oauthService.setupAutomaticSilentRefresh();
						resolveFn();
						// if you don't use clearHashAfterLogin from angular-oauth2-oidc you can remove the #hash or route to some other URL manually:
						// const lazyPath = this.injector.get(LAZY_PATH) as string;
						// this.injector.get(Router).navigateByUrl(lazyPath + '/a') // remove login hash fragments from url
						//   .then(() => resolveFn()); // callback only once login state is resolved
					} else {
						if (isDevMode()) {
							console.log('[InitialAuthService] initLoginFlow');
						}
						this.oauthService.initLoginFlow();
						rejectFn();
					}
				});
			});
		}

		private handleNewToken() {
			this._decodedAccessToken = this.jwtHelper.decodeToken(this.oauthService.getAccessToken());
			this._decodedIDToken = this.jwtHelper.decodeToken(this.oauthService.getIdToken());
		}

		public isAuthenticated() {
			return this.oauthService.getAccessToken() && !this.jwtHelper.isTokenExpired(this.oauthService.getAccessToken());
		}
	}

Thực hiện initAuth() tại thời điểm ứng dụng được init, nếu chưa login sẽ chuyển hướng qua trang login của Keycloak

.. code-block:: javascript

	@NgModule({
		declarations: [
			AppComponent
		],
		imports: [
			BrowserModule,
			AppRoutingModule,
			CoreModule,
			OAuthModule.forRoot()

		],

		providers: [
			{provide: AuthConfig, useValue: keycloakAuthConfig},
			InitialAuthService,
			{
				provide: APP_INITIALIZER,
				useFactory: (initialAuthService: InitialAuthService) => () => initialAuthService.initAuth(),
				deps: [InitialAuthService],
				multi: true
			},

			AuthGuard
		],

		bootstrap: [AppComponent]
	})
	export class AppModule {
	}


Sample Project 
**************

`Download <../../_static/crm-angular-seed.rar>`_