# 트러블슈팅

자세한 내용은 [Rollup의 문제 해결 가이드](https://rollupjs.org/troubleshooting/)를 참조하시기를 바랍니다.

이 글이 제안하는 해결책이 잘 동작하지 않는다면 [GitHub Discussions](https://github.com/vitejs/vite/discussions)이나 [Vite Land Discord](https://chat.vitejs.dev)의 `#help` 채널에 질문을 게시해 보세요.

## CLI

### `Error: Cannot find module 'C:\foo\bar&baz\vite\bin\vite.js'`

프로젝트 폴더의 경로에 `&` 가 포함되어있을 가능성이 있습니다. 이는 Windows의 `npm`에서 동작시키지 않습니다([npm/cmd-shim#45](https://github.com/npm/cmd-shim/issues/45)).

다음 중 하나를 수행해야 합니다:

- 다른 패키지 매니저로 교체하세요 (e.g. `pnpm`, `yarn`)
- 프로젝트의 경로에서 `&` 을 제거하세요 

## 개발 서버

### 요청이 영원히 중단됨

Linux를 사용하는 경우  file descriptor 및 inotify 제한으로 인해 문제가 발생할 수 있습니다. Vite는 대부분의 파일을 번들로 제공하지 않기 때문에 브라우저는 한도를 초과하여 많은 file descriptor가 필요한 많은 파일을 요청할 수 있습니다.

이 문제를 해결하기 위해서:

- `ulimit`로 file descriptor 제한 늘리기

  ```shell
  # Check current limit
  $ ulimit -Sn
  # Change limit (temporary)
  $ ulimit -Sn 10000 # You might need to change the hard limit too
  # Restart your browser
  ```

- `sysctl`로 inotify 관련 제한 늘리기.

  ```shell
  # Check current limits
  $ sysctl fs.inotify
  # Change limits (temporary)
  $ sudo sysctl fs.inotify.max_queued_events=16384
  $ sudo sysctl fs.inotify.max_user_instances=8192
  $ sudo sysctl fs.inotify.max_user_watches=524288
  ```

위의 단계가 작동하지 않으면 `DefaultLimitNOFILE=65536`을 주석 처리되지 않도록 하여 다음 파일에 추가하세요:

- /etc/systemd/system.conf
- /etc/systemd/user.conf

Ubuntu Linux의 경우 systemd 구성 파일을 업데이트하는 대신 `/etc/security/limits.conf` 파일에 `* - nofile 65536` 줄을 추가해야 할 수 있습니다.

이 설정은 유지되지만, **다시 시작해야 합니다**.

### 네트워크 요청 로딩 중지

자체 서명된 SSL 인증서를 사용하는 경우 Chrome은 모든 캐싱 지시문을 무시하고 콘텐츠를 다시 로드합니다. Vite는 이러한 캐싱 지시문에 의존합니다.

문제를 해결하려면 신뢰할 수 있는 SSL 인증서를 사용하십시오.

참조: [Cache problems](https://helpx.adobe.com/mt/experience-manager/kb/cache-problems-on-chrome-with-SSL-certificate-errors.html), [Chrome issue](https://bugs.chromium.org/p/chromium/issues/detail?id=110649#c8)

#### macOS

이 명령으로 CLI를 통해 신뢰할 수 있는 인증서를 설치할 수 있습니다:

```
security add-trusted-cert -d -r trustRoot -k ~/Library/Keychains/login.keychain-db your-cert.cer
```

또는 인증서를 키체인 액세스 앱으로 가져오고 인증서에 대한 신뢰도를 "항상 신뢰"로 변경하세요.

### 431 Request Header Fields Too Large

서버/WebSocket 서버가 큰 HTTP 헤더를 수신하면 요청이 삭제되고 다음 경고가 표시됩니다.

> Server responded with status code 431. See https://vitejs.dev/guide/troubleshooting.html#_431-request-header-fields-too-large.

이는 Node.js가 [CVE-2018-12121](https://www.cve.org/CVERecord?id=CVE-2018-12121) 완화를 위해 요청 헤더 크기를 제한하기 때문입니다..

이를 방지하려면 요청 헤더 크기를 줄이세요. 예를 들어 길이가 긴 쿠키를 삭제합니다. 또는 최대 헤더 크기를 변경하기 위해서 [`--max-http-header-size`](https://nodejs.org/api/cli.html#--max-http-header-sizesize)를 사용할 수 있습니다.

## HMR

### Vite가 파일 변경을 감지했지만 HMR이 작동하지 않음

대소문자가 다른 파일을 가져왔을지도 모릅니다. 예를 들어 `src/foo.js`가 존재하고 `src/bar.js`에는 다음이 포함됩니다:

```js
import './Foo.js' // './foo.js' 이어야 합니다.
```

관련 이슈: [#964](https://github.com/vitejs/vite/issues/964)

### Vite가 파일 변경을 감지하지 못함

WSL2로 Vite를 실행하는 경우 일부 조건에서는 파일 변경 사항을 볼 수 없습니다. [`server.watch` option](/config/server-options.md#server-watch)를 참고하세요.

### HMR 대신 전체 리 로드가 발생합니다.

Vite 또는 플러그인에서 HMR을 처리하지 않으면 전체 리 로드가 발생합니다.

또한 의존성 루프가 있는 경우 전체 리 로드가 발생합니다. 이 문제를 해결하려면 루프를 제거해 보세요.

### 콘솔에서 많은 수의 HMR 업데이트

이는 상호참조 때문에 발생할 수 있습니다. 이 문제를 해결하려면 상호참조를 끊어보세요.

## 빌드

### CORS 오류로 인해 빌드된 파일이 작동하지 않음

HTML 파일 출력이 `file` 프로토콜로 열린 경우 다음 오류와 함께 스크립트가 실행되지 않습니다.

> Access to script at 'file:///foo/bar.js' from origin 'null' has been blocked by CORS policy: Cross origin requests are only supported for protocol schemes: http, data, isolated-app, chrome-extension, chrome, https, chrome-untrusted.

> Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at file:///foo/bar.js. (Reason: CORS request not http).

이러한 상황이 나타나는 이유에 대한 자세한 정보는 [Reason: CORS request not HTTP - HTTP | MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/Errors/CORSRequestNotHttp)을 참조하세요.

`http` 프로토콜로 파일에 액세스해야 합니다. 이를 달성하는 가장 쉬운 방법은 `npx vite preview`를 실행하는 것입니다.

## Others


### 브라우저 호환성을 위해 외부화된 모듈

브라우저에서 Node.js 모듈을 사용할 때 Vite는 다음 경고를 출력합니다.

> Module "fs" has been externalized for browser compatibility. Cannot access "fs.readFile" in client code.
This is because Vite does not automatically polyfill Node.js modules.

polyfill을 수동으로 추가할 수 있지만 번들 크기를 줄이기 위해서 브라우저 코드에 Node.js 모듈을 사용하지 않는 것이 좋습니다. 외부 라이브러리(브라우저에서 사용됨)에서 모듈을 가져온 경우 해당 라이브러리에 문제를 보고하는 것이 좋습니다.

### Syntax Error / Type Error 발생

Vite는 엄격하지 않은 모드(느슨한 모드)에서만 실행되는 코드를 처리할 수 없으며 지원하지 않습니다. 이는 Vite가 ESM을 사용하고 ESM은 내부적으로 항상 [엄격 모드](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode)이기 때문입니다.

예를 들어 이러한 오류가 표시될 수 있습니다.

> [ERROR] With statements cannot be used with the "esm" output format due to strict mode

> TypeError: Cannot create property 'foo' on boolean 'false'

이러한 코드들이 의존성 모듈 내부에서 사용된다면, 문제를 해결하기 위해[`patch-package`](https://github.com/ds300/patch-package) (또는 [`yarn patch`](https://yarnpkg.com/cli/patch) 또는 [`pnpm patch`](https://pnpm.io/cli/patch))를 사용할 수 있습니다.