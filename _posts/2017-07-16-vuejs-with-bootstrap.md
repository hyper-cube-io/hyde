---
layout: post
title: Vuejs with Bootstrap
author: kwSeo
tags: [vuejs, bootstrap]
---

# Vue.js with Bootstrap

Vue를 공부하면서 Vue에서 Bootstrap을 편하게 사용하기 위해서 알아봤던 방법에 대해서 공유하고자 한다.

## Vue.js

최근 FE 프레임워크 진영은 마치 삼국지를 연상하게 한다. React, Angular2 그리고 Vue. 비록 Vue가 아직 React와 Angular 시리즈에 비하면 역사가 오래되지 않았지만, 점점 자신의 영역을 넓혀가는 중이다. 한국에서도 점점 많은 개발자들이 관심을 보이고 있으며 한국 포럼도 존재하고 기술 문서도 한글 번역이 이루어 졌다. 또한, Vue는 앞에서 언급한 두 개의 프레임워크보다 더 사용하기 쉬우며, 성능도 꿀리지 않는다. 그리고 그 특유의 유연함을 통해서 규모가 작은 앱부터 큰 앱까지 아주 편하게 사용할 수 있다. 

## 도구없는 환경에서 Bootstrap을 함께 사용하기

사실 도구(webpack, Browserify 등)가 없는 환경은 너무 간단해서 더이상 설명할 필요도 없어보인다. 하지만 간단히 경량 앱에서 Vue를 사용하는 모습을 소개하기 위해서 간단하게 예제를 보고 가겠다. 자세한 사항은 여기보다 
물론 아무런 도구를 사용하지 않기 때문에 ES6이상의 문법은 전혀 사용되지 않는다.

```html
...
<head>
  ...
  <link rel="stylesheet" type="text/css" href="bootstrap/css/bootstrap.min.css">
</head>
<body>
  <div id="container">
    <form class="form-inline">
      <div class="form-group">
        <input type="text" class="form-control" v-model="message">
        <p>{{message}}</p>
        <button type="button" class="btn btn-default" @click="alertMessage">Alert</button>
      </div>
    </form>
  <div>

  <script src="jquery/js/jquery.min.js"></script>
  <script src="bootstrap/js/bootstrap.min.js"></script>
  <script src="vue/js/vue.min.js"></script>
  <script src="main.js"></script>
</body>
...
```

```javascript
  new Vue({
    el: '#container',
    data: {
      message: ''
    },
    methods: {
      alertMessage: function () {
        alert(this.message);
      }
    }
  });
```

## Vue 컴포넌트와 함께 Bootstrap 적용하기

Vue를 컴포넌트([Single File Component](https://vuejs.org/v2/guide/single-file-components.html))로 나누어 본격적으로 Vue를 사용하기 시작하면 Bootstrap을 사용하는데 고려할 사항이 늘어난다. 앱이 컴포넌트 단위로 분리되며 모듈화를 위해서 각 컴포넌트는 별도의 파일로 분리될 것이다. 또한, 각 컴포넌트는 자신의 JavaScript 로직 및 디자인을 위한 CSS가 적용된다. 이러한 컴포넌트들(vue파일들)을 빌드하여 브라우저에서 돌아가는 JavaScript로 바꾸기 위해서 Webpack이라는 도구를 사용한다. 그리고 앱의 의존성 관리를 위해서 NPM을 사용할 것이고 배포 환경별로 프로파일을 분리할 것이다. 개발 환경에서는 사용하는 라이브러리들의 일반 버전을 사용하고 서비스 환경에서는 라이브러리들의 min 버전을 사용하고 난독화 및 압축을 해야할테니 말이다. 
여기서는 단순히 많이 사용되는 Bootstrap을 Vue로 개발하는 앱에 적용하는 방법에 대해서 몇가지를 정리할 것이다.
개발 환경은 [vue-cli](https://github.com/vuejs/vue-cli)를 통해 생성된 프로젝트를 기준으로 한다.

### static resource로 적용하기

위 '도구없는 환경에서 Bootstrap을 함께 사용하기'에서 설명한 방법이다. Vue Project의 index.html에 직접 Bootstrap의 CSS와 JavaScript를 추가해주는 방법이다.
처음 웹을 공부할때부터 사용하던 방법이니 매우 간단해보이지만 이 방법으로는 Webpack이 주는 해택을 이용할 수 없다.

### 앱 전역으로 Bootstrap을 적용하기

일단 NPM을 통해서 Boostrap을 설치한다.

```bash
$ npm install --save jquery bootsrap
```

위 명령을 실행하고 package.json을 확인하면 의존성에 JQuery와 Bootstrap이 추가된 것을 볼 수 있다. 이제 실제 Vue 로직내에서 JQuery의 기능을 사용하기 위해서 `$` 키워드를 전역으로 적용을 할 것이다. 그러기 위해서는 expose-loader가 필요하다(loader가 뭔지는 webpack 문서 참고). npm을 통해서 expose-loader도 설치한다.

```bash
$ npm install --save-dev expose-loader
```

expose-loader를 사용하면 자신이 원하는 키워드로 라이브러리를 바인딩할 수 있다. 그러기 위해서 main.js에 아래의 코드를 추가한다.

```javascript
import 'expose-loader?$!expose-loader?jQuery!jquery'
```

expose-loader의 `expose-loader?libraryName!./file.js` 같은 형식으로 정의하여 사요할 수 있다. '?' 다음에 작성하는 libraryName은 바인딩할 키워드이며 '!' 다음에 실제 라이브러리 경로를 작성한다. '!' 다음에 expose-loader를 반복하여 작성해서 동시에 여러개를 바인딩할 수 도 있다.
위 코드에 경우 JQuery를 $와 jQuery에 바인딩하는 것이다. 이를 통해 모든 컴포넌트가 JQuery를 사용할 수 있다. 이제 Bootstrap을 추가할 차례이다.

Bootstrap을 추가하는 방법은 매우 간단하다. 앞에서 NPM을 통해서 Bootstrap을 설치하였기 때문에 Bootstrap을 import해주기만 하면 된다.

```javascript
import 'bootstrap'
```

문제는 CSS이다. vue-cli를 통해서 프로젝트를 생성하였다면 css-loader가 이미 설정되어 있을텐데 추가적으로 style-loader를 추가해주어야 한다.

```bash
$ npm install --save-dev style-loader
```

**Before**

```javascript
// in webpack.config.js
...
{
  test: /\.css$/,
  loader: ['css-loader']
}
...
```

**After**

```javascript
// in webpack.config.js
...
{
  test: /\.css$/,
  loader: ['style-loader', 'css-loader']
}
...
```

그리고 Bootstrap의 몇몇 리소스들을 로드하기 위해서 추가적으로 file-loader에 파일 확장자를 추가해야 한다.

```javascript
...
{
  test: /\.(png|jpg|gif|svg|eot|woff|woff2|ttf)$/,
  loader: 'file-loader',
  options: {
    name: '[name].[ext]?[hash]'
  }
}
...
```

그후 Bootstrap CSS를 import해주어야 한다.

```javascript
// in main.js
import 'bootstrap/dist/css/bootstrap.css'
```

이로서 Bootstrap을 사용하기 위한 준비가 완료되었다. 이제 어떤 컴포넌트든 Bootstrap을 사용할 수 있다.

```javascript
// in some vue component
<template>
  <div>
    <h1 class="title">Hello!!!</h1>
    <input class="form-control" v-model="message">
    <button type="button" class="btn btn-default" @click="show">
    <some-component :value="message"></some-component>
  </div>
</template>

<script>
  import someComponent from './some/component.vue'

  export default {
    data () {
      message: ''
    },
    methods: {
      show () {
        alert(this.message)
      }
    },
    components: {
      someComponent
    }
  }
</script>

<style scoped>
  .title {
    font-size: 20pt;
    font-weight: bolder;
    border-bottom: 1px solid #999;
  }
</style>
```

### Boostrap-vue 플러그인 이용하기

Bootstrap을 사용하는 또다른 방법은 Bootstrap-vue 플러그인을 사용하는 것이다. Bootstrap-vue는 Vue에서 사용할 수 있도록 Bootstrap을 컴포넌트로 만든 플러그인이다(참고로 Bootstrap-vue는 Bootstrap4를 바탕으로 한다). 

```bash
$ npm install --save bootstrap-vue
```

```javascript
// in main.js
import Vue from 'vue'
import BootstrapVue from 'bootstrap-vue';

Vue.use(BootstrapVue);

// using style-loader
import 'bootstrap/dist/css/bootstrap.css'
import 'bootstrap-vue/dist/bootstrap-vue.css'
```

위 코드처럼 필요한 설치과정을 마무리하면 Vue 컴포넌트 어디서든 Bootsrap-vue에서 만든 컴포넌트를 사용할 수 있다. 사용 예제는 아래와 같다.

```javascript
<template>
  <div>
    <b-card header="Todo List" :no-block="tasksExist">
      <b-list-group v-if="tasksExist">
        <b-list-group-item v-for="task in tasks" :key="task.id" @click.native="moveToDoingList">
          <task :data="task"></task>
        </b-list-group-item>
      </b-list-group>
      <h5 v-else>
        해야할 일이 없습니다.
      </h5>
    </b-card>
  </div>
</template>

<script>
import task from './Task.vue';

export default {
  props: ['tasks'],
  components: {
    task
  },
  computed: {
    tasksExist () {
      return this.tasks && this.tasks.length > 0
    }
  },
  methods: {
    moveToDoingList () {
      alert('아직 구현이 안됬습니다!!')
    }
  }
};
</script>

<style scoped>
  .no-padding {
    padding: 0;
    margin: 0;
  }
</style>
```


