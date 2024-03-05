---
title: "번들러 마이그레이션(webpack to vite) 트러블 슈팅 노트"
seoTitle: "bundler migration webpack to vite"
seoDescription: "development log note during bundler migration webpack to vite"
datePublished: Mon Jan 29 2024 07:18:43 GMT+0000 (Coordinated Universal Time)
cuid: clrylm4u6000u0ajtbfmxf4h2
slug: webpack-to-vite
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1706512593340/09e4640d-73a9-4f9d-a111-fc44e16c0975.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1706512690660/9c28964f-5f41-41cc-8c7a-76dc94be188d.png
tags: webpack, migration, vite

---

# 마이그레이션 이유

* 번들링 속도 개선: 개발 환경에서 개발시, 배포 빌드시 속도를 개선해줍니다.
    
    * 자세한 내용은 [Vite 공식 홈페이지 문서](https://vitejs-kr.github.io/guide/why.html)를 참고해주세요.
        

# 개발 환경에서 설정과 트러블 슈팅

개발 환경에서의 트러블슈팅은 주로 소스 코드와 config, 패키지 변경에서 이뤄졌습니다.

### index.html 관련 설정

* index.html 위치 변경: %root\_dir%/public/index.html → %root\_dir%/index.html
    
* 허용되지 않는 html 파일 내 `%env_variable%` 제거 (참고: 4.2.0 버전부터 지원됩니다)
    
* 기본 tsx 파일(main.tsx, or index.tsx)의 script module import 추가
    
    * `<script type="module" src="/src/index.tsx"></script>`
        

### css import: 허용되지 않는 nodemodules nested `@import ~`

* javascript import로 nodemodules css 파일들 가져오기로 변경
    

### component tsx 파일 설정

* useScript 시 상대경로 → 절대경로(리액트 페이지 루트에서 시작)
    

### vite config 설정하기(vite.config.ts)

* vite 리액트 리프레시 지원을 위해 react() → reactRefresh() 로 변경 `@vitejs/plugin-react-refresh`
    
* 환경 변수 이식
    

```javascript
	import { loadEnv } from 'vite';
	
	const env = loadEnv(mode, process.cwd(), '');
	
	// defineConfig 내 json 설정
	define: {
	  __APP_ENV__: env.APP_ENV,
      'process.env': env,
	},
```

* CJS 지원을 위한 플러그인 설정
    

```typescript
	import { viteCommonjs } from '@originjs/vite-plugin-commonjs';
	
	// defineConfig 내 json 설정
	plugins: [viteCommonjs()]
```

# 배포 환경에서 설정과 트러블 슈팅

* CJS 방식 require가 코드 내에서 사용됐을 때 변경하지 못하는 이슈
    
    * component inline require를 다 파일 import로 대체하여 해결
        

# 최종 config 파일들 상태

* 회사 도메인 연관 코드는 삭제했습니다.
    

### 변경 전 webpack 설정

* webpack.config.js
    

```typescript
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const ReactRefreshWebpackPlugin = require('@pmmmwh/react-refresh-webpack-plugin');
const webpackDevClientEntry = require.resolve('react-dev-utils/webpackHotDevClient');
const reactRefreshOverlayEntry = require.resolve('react-dev-utils/refreshOverlayInterop');

module.exports = {
  entry: './src/index.js',
  output: {
    path: path.join(__dirname, '/dist'),
    filename: 'index_bundle.js',
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_module/,
        use: {
          loader: 'babel-loader',
        },
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
      {
        test: /\.(svg|png|jpe?g|gif)$/i,
        loader: 'file-loader',
        options: {
          name: '[name].[ext]',
          outputhPath: 'imgs',
        },
      },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: 'public/index.html',
    }),
    new ReactRefreshWebpackPlugin({
      overlay: {
        entry: webpackDevClientEntry,
        module: reactRefreshOverlayEntry,
      },
    }),
  ],
};
```

### 변경 후 vite 설정

* vue.config.ts
    

```ts
import { defineConfig, loadEnv } from 'vite';
import reactRefresh from '@vitejs/plugin-react-refresh';
import viteCommonjs from 'vite-plugin-commonjs';

export default defineConfig(({ command, mode }) => {
  // Set the third parameter to '' to load all env regardless of the `VITE_` prefix.
  const env = loadEnv(mode, process.cwd(), '');
  return {
    plugins: [reactRefresh(), viteCommonjs()],
    root: './',
    define: {
      __APP_ENV__: env.APP_ENV,
      'process.env': env,
    },
    base: '/',
    build: {
      outDir: './build',
      manifest: true,
      rollupOptions: {
        input: 'index.html',
        output: {
          dir: './build',
          format: 'esm',
        },
      },
    },
    css: {
      preprocessorOptions: {
        less: {
          javascriptEnabled: true,
        },
      },
    }
  };
});
```

# 최종 결과

* 개발 환경 속도 대폭 개선
    
* 번들링 속도 약 55프로 개선: 기존 평균 1분 → 33초
    
* 웹팩 관련 패키지 및 복잡한 설정(보일러 플레이트 설정..?) 제거
    

글 작성을 염두에 두지 못하고 뒤늦게 정리하다보니 내용이 미흡해서 아쉽네요. 누군가에게는 도움이 되길 바랍니다.