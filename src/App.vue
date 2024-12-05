<script setup>
import axios from 'axios';
import { onMounted, ref, useTemplateRef, nextTick } from 'vue';

// 定数
const BATCH_SIZE = 10; // 1回で取得・表示する件数

// 変数
const posts = ref([]);
const isLoading = ref(false);
const hasMore = ref(true);
const list = useTemplateRef("ref-list"); // useTemplateRefでテンプレート内のDOM要素を取得
let observer = null;

const fetchPosts = async () => {
  if (isLoading.value || !hasMore.value) return; // isLoadingを含めて重複リクエスト防止

  isLoading.value = true;

  try {
    const response = await axios.get('https://vool.jp/wp-json/wp/v2/posts', {
      params: {
        offset: posts.value.length, // 既に取得済みのデータ数から開始
        per_page: BATCH_SIZE, // 1回のリクエストで返すレコードの数を指定
      },
    });

    console.log(response.data);

    if (response.data.length === 0) {
      hasMore.value = false;

      // 既存のオブザーバーを切断
      observer?.disconnect();
    } else {
      // 新しく取得したデータを追加
      posts.value = [...posts.value, ...response.data];

      // データ取得/追加後、DOM更新を待ってからobserveLastItem()を実行
      await nextTick();
      observeLastItem();
    }
  } catch (error) {
    console.error('データの取得中にエラーが発生しました:', error.message);
  } finally {
    isLoading.value = false;
  }
};

const observeLastItem = () => {
  // 既存のオブザーバーを切断
  observer?.disconnect();

  // 新しいオブザーバーを作成
  observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          fetchPosts();
        }
      });
    },
    {
      root: null,
      rootMargin: "0px 0px 0px 0px", // ビューポートの下端を基準
      threshold: 1 // 対象の要素の最下部を基準
    }
  );

  // リスト内の最後の子要素を監視
  const lastItem = list.value.lastElementChild;
  if (lastItem) {
    observer.observe(lastItem);
  }
};

onMounted(() => {
  fetchPosts();
});
</script>

<template>
  <div class="l-inner">
    <ul ref="ref-list" class="list">
      <TransitionGroup name="fade">
        <li v-for="post in posts" :key="post.id" class="item">
          <a :href="post.link" class="link" target="_blank" rel="noopener noreferrer">
            <p>ID：{{ post.id }}</p>
            <p>スラッグ：{{ post.slug }}</p>
            <p>タイトル：{{ post.title.rendered }}</p>
            <p v-html="post.excerpt.rendered"></p>
          </a>
        </li>
      </TransitionGroup>
    </ul>
    <div v-if="isLoading">
      <span class="loader"></span>
    </div>
  </div>
</template>

<style scoped>
.l-inner {
  margin: 100px auto;
  width: min(1000px, 100%);
  padding: 0 25px;
}

.list {
  display: grid;
  grid-template-columns: minmax(0, 1fr);
  row-gap: 50px;
  list-style: none;
  padding: 0;
}

.link {
  display: block;
  text-decoration: none;
  color: inherit;
  box-shadow: 5px 5px 8px rgb(0 0 0 / 0.2);
  border: 1px solid;
  border-radius: 8px;
  padding: 20px;
}

.fade-enter-from {
  opacity: 0;
}

.fade-enter-active {
  transition: opacity 0.75s ease;
}

.fade-enter-to {
  opacity: 1;
}

.loader {
  margin: 20px 0;
  display: block;
  width: 16px;
  padding: 2px;
  aspect-ratio: 1;
  border-radius: 50%;
  background: currentColor;
  --_m:
    conic-gradient(#0000 10%, currentColor),
    linear-gradient(#000 0 0) content-box;
  -webkit-mask: var(--_m);
  mask: var(--_m);
  -webkit-mask-composite: source-out;
  mask-composite: subtract;
  animation: l3 1s infinite linear;
}

@keyframes l3 {
  to {
    transform: rotate(1turn)
  }
}
</style>
