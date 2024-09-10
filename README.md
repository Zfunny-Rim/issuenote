# issuenote

## Project setup
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Lints and fixes files
```
npm run lint
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).

### Project Outline
Qlik 개발서버 (https://dev.jsct.site)의 jwt2 virtual proxy를 이용한 sso 연결과 Qlik APP간 변수 통신 (setStringValue, getContent) 예제 포함
 - JWT 토큰은 JAM으로 가져옴 (https://223.130.139.187:8891) ,  현재 user02로 Hardcoding 되어있음.
 - 개발서버의 ISSUE NOTE Extension과 통신하여, 호출 APP ID와 값을 읽어갈 변수명을 GET방식으로 전달받아 getContent를 호출하여 변수값을 읽어옴
 - Status와 Issue Summary를 입력 후, extension과 상호합의 된 형태의 값으로 Qlik 변수에 값을 입력하고 reload를 통해 Partial Reload 하여 변수값을 저장함.
