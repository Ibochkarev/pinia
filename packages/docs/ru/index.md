---
layout: home

title: Pinia
titleTemplate: Интуитивно понятное хранилище для Vue.js

hero:
  name: Pinia
  text: Интуитивно понятное хранилище для Vue.js
  tagline: Безопасный, расширяемый и модульный дизайн. Забудьте о том, что вы вообще используете хранилище.
  image:
    src: /logo.svg
    alt: Pinia
  actions:
    - theme: brand
      text: Приступить к работе
      link: /introduction
    - theme: alt
      text: Демо
      link: https://stackblitz.com/github/piniajs/example-vue-3-vite
    - theme: cta vueschool
      text: Смотреть вступительное видео
      link: https://vueschool.io/lessons/introduction-to-pinia?friend=vuerouter&utm_source=pinia&utm_medium=link&utm_campaign=homepage
    - theme: cta vue-mastery
      text: Получить шпаргалку по Pinia
      link: https://www.vuemastery.com/pinia?coupon=PINIA-DOCS&via=eduardo

features:
  - title: 💡 Интуитивный
    details: Хранилища так же привычны, как и компоненты. API разработан для того, чтобы вы могли писать хорошо организованные хранилища.
  - title: 🔑 Безопасные типы
    details: Типы выводятся, что означает, что хранилища предоставляют вам автозаполнение даже в JavaScript!
  - title: ⚙️ Поддержка Devtools
    details: Pinia подключается к Vue devtools, чтобы предоставить вам расширенный опыт разработки как в Vue 2, так и в Vue 3.
  - title: 🔌 Расширяемый
    details: Реагируйте на изменения хранилища, чтобы расширить Pinia с помощью транзакций, синхронизации локального хранилища и т.д.
  - title: 🏗 Модульная конструкция
    details: Создайте несколько хранилищ и позвольте вашему коду bundler автоматически разделять их.
  - title: 📦 Чрезвычайно легкий
    details: Pinia весит ~ 1,5 кб, вы забудете, что она вообще есть!
---

<script setup>
import HomeSponsors from '../.vitepress/theme/components/HomeSponsors.vue'
import '../.vitepress/theme/styles/home-links.css'
</script>

<HomeSponsors />
