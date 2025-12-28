# TypeScriptで作るシンプルSPA入門（ハンズオン）

## 導入
本記事は、カレントディレクトリにある実装コードを**そのまま引用**しながら、TypeScriptでSPA（Single Page Application）を組み立てる手順を丁寧に解説します。フレームワークなしで、**ルーティング・ページ構成・描画**を分離する最小構成を手を動かして理解できる内容です。

## ハンズオン全体像
このSPAは次の部品で構成されています。

- エントリ（`index.ts`）: アプリシェル生成とルーター初期化
- ルーター（`router.ts`）: ハッシュベースのページ切り替え
- ページ定義（`pages/types.ts`）: ページ契約の型
- ページ一覧（`pages/index.ts`）: ページ登録とデフォルトルート

まずは、**実際のコード**を読みながら構造をつかみ、その後に自分でページを追加できる流れで進めます。

## ステップ1: アプリの起点（`index.ts`）を理解する
エントリポイントでは、**アプリの土台となるDOM**を構築し、ルーターを起動します。次のコードがアプリ全体の骨格です。

```ts
import { initRouter } from './router.ts';
import { defaultRoute, pages } from './pages/index.ts';
import { applyTranslations, translate } from './utils/i18n.js';

const ensureAppRoot = (): HTMLElement => {
  const existing = document.getElementById('app');
  if (existing) {
    return existing;
  }

  const root = document.createElement('div');
  root.id = 'app';
  document.body.appendChild(root);
  return root;
};

const bootstrap = () => {
  const appRoot = ensureAppRoot();
  appRoot.className = 'app-shell';

  const header = document.createElement('header');
  header.className = 'app-shell__header';

  const brand = document.createElement('button');
  brand.type = 'button';
  brand.className = 'app-shell__brand';
  brand.dataset.i18n = 'appBrand';
  brand.textContent = translate('appBrand');
  header.appendChild(brand);

  const main = document.createElement('main');
  main.className = 'app-shell__main';

  appRoot.replaceChildren(header, main);
  applyTranslations(appRoot);

  const router = initRouter(main, pages, defaultRoute);

  brand.addEventListener('click', () => {
    router.navigate('#home');
  });

  document.addEventListener('click', (event) => {
    const target = (event.target as HTMLElement | null)?.closest<HTMLElement>('[data-navigate]');
    if (!target) {
      return;
    }

    const route = target.getAttribute('data-navigate');
    if (!route) {
      return;
    }

    event.preventDefault();

    const normalizedRoute = route.startsWith('#') ? route : `#${route}`;
    if (window.location.hash === normalizedRoute) {
      return;
    }

    router.navigate(route);
  });
};

bootstrap();
```

このコードのポイントは3つです。

- **`ensureAppRoot`**で`#app`を確実に用意する
- `header`と`main`を生成して**アプリシェル**を構築する
- `initRouter`で**SPAの切り替えを開始**する

> ここまでが「SPAの土台」。この先は、ページ定義とルーターが連携して画面を切り替えます。

## ステップ2: ページの契約を作る（`pages/types.ts`）
次に、各ページが守るべき「契約（インターフェース）」を確認します。

```ts
export interface Page {
  id: string;
  title: string;
  route: string;
  mount: (container: HTMLElement) => void;
  dispose?: () => void;
}
```

この`Page`型によって、ページは以下を備える必要があります。

- **id**: ページの一意な識別子
- **title**: `document.title`に使う画面名
- **route**: ルーティングで使うURLのハッシュ部分
- **mount**: DOMに描画する関数
- **dispose**: 必要なら後片付け（任意）

この契約があることで、ルーターは**ページの実装を知らなくても**安全に切り替えできます。

## ステップ3: ページ一覧を登録する（`pages/index.ts`）
次に、どのページを使うかを配列で登録します。

```ts
import { CreateTournamentPage } from './createTournament.js';
import { HistoryPage } from './history.js';
import { HomePage } from './home.js';
import { JoinTournamentPage } from './joinTournament.js';
import { GamePage } from './game.js';
import type { Page } from './types.js';
import { ProfilePage } from './profile.js';
import { SettingsPage } from './settings.js';
import { TournamentPage } from './tournament.js';
import { WaitingTournamentPage } from './waitingTournament.js';
import { BracketPage } from './bracket.js';
import { LoginPage } from './login.js';

export const pages: Page[] = [
  HomePage,
  GamePage,
  CreateTournamentPage,
  JoinTournamentPage,
  WaitingTournamentPage,
  BracketPage,
  TournamentPage,
  HistoryPage,
  ProfilePage,
  SettingsPage,
  LoginPage,
];

export const defaultRoute = HomePage.route;
```

ここでは**ページを配列で一括管理**し、`defaultRoute`をホーム画面に設定しています。この配列が、ルーターの「ページ辞書」になります。

## ステップ4: ルーターを実装する（`router.ts`）
SPAの心臓部であるルーターは、ハッシュ（`#`）を使ってページを切り替えます。実装は以下の通りです。

```ts
import type { Page } from './pages/types.ts';

export const initRouter = (
  container: HTMLElement,
  pages: Page[],
  defaultRoute: string,
) => {
  const normalizeRoute = (value: string): string => {
    const withHash = value.startsWith('#') ? value : `#${value}`;
    const [base] = withHash.split('?');
    return base;
  };

  const normalizedDefaultRoute = normalizeRoute(defaultRoute);

  const pageMap = new Map<string, Page>();
  pages.forEach((page) => {
    pageMap.set(normalizeRoute(page.route), page);
  });

  let currentPage: Page | undefined;

  const resolveRoute = (hash: string): Page => {
    const normalized = normalizeRoute(hash);
    return pageMap.get(normalized) ?? pageMap.get(normalizedDefaultRoute)!;
  };

  const mountPage = (nextPage: Page) => {
    if (currentPage?.dispose) {
      currentPage.dispose();
    }

    container.innerHTML = '';
    nextPage.mount(container);
    document.title = `Transcendence Pong | ${nextPage.title}`;
    document.body.dataset.page = nextPage.id;
    currentPage = nextPage;
  };

  const handleRouteChange = () => {
    const targetPage = resolveRoute(window.location.hash || normalizedDefaultRoute);
    mountPage(targetPage);
  };

  window.addEventListener('hashchange', handleRouteChange);
  handleRouteChange();

  return {
    navigate: (route: string) => {
      window.location.hash = route.startsWith('#') ? route : `#${route}`;
    },
    destroy: () => {
      window.removeEventListener('hashchange', handleRouteChange);
    },
  };
};
```

このルーターがやっていることを整理すると、以下の通りです。

- **ルートの正規化**: `#`を付与し、クエリを除去
- **ページ辞書の作成**: `Map`に登録して高速参照
- **遷移時処理**: `dispose`で後片付け→`mount`で描画
- **メタ情報更新**: `document.title`と`data-page`を更新

> ハッシュルーティングはサーバー設定不要で、静的環境に強いのがメリットです。

## ステップ5: ナビゲーションを有効にする
`index.ts`の後半では、`data-navigate`属性を使ったクリック遷移を実装しています。

```ts
document.addEventListener('click', (event) => {
  const target = (event.target as HTMLElement | null)?.closest<HTMLElement>('[data-navigate]');
  if (!target) {
    return;
  }

  const route = target.getAttribute('data-navigate');
  if (!route) {
    return;
  }

  event.preventDefault();

  const normalizedRoute = route.startsWith('#') ? route : `#${route}`;
  if (window.location.hash === normalizedRoute) {
    return;
  }

  router.navigate(route);
});
```

この方式を使うと、ページ内のボタンやリンクに`data-navigate`を付けるだけで遷移できます。

- 例: `<a data-navigate="home">Home</a>`
- 例: `<button data-navigate="#settings">設定</button>`

## ステップ6: i18nの仕込み（翻訳対応）
`index.ts`ではi18n用の関数が呼ばれています。

```ts
import { applyTranslations, translate } from './utils/i18n.js';

brand.dataset.i18n = 'appBrand';
brand.textContent = translate('appBrand');
applyTranslations(appRoot);
```

ここでは、`data-i18n`属性を使って翻訳対象を示し、`applyTranslations`で一括適用しています。具体的な翻訳辞書は`utils/i18n.js`に委ねられており、**UI構成と文言管理を分離**する設計です。

## ステップ7: 新しいページを追加する手順
実際にページを追加する場合、流れは次の通りです。

1. **ページオブジェクトを作成**（`Page`型に合わせる）
2. **`pages/index.ts`に登録**
3. **`data-navigate`で遷移リンクを追加**

ページ追加のポイントは、`route`と`id`をユニークにすることです。`mount`内でDOMを構築すれば、ルーターが自動的に画面を切り替えてくれます。

## まとめ
このSPAは、TypeScriptで**最小構成のSPA**を構築するための要素が揃っています。`index.ts`でアプリシェルを作り、`router.ts`でハッシュルーティングを管理し、`Page`型で各ページの責務を明確化しています。ハンズオンとしては、**ページ追加やUI拡張**を試すことで、SPAの基本構造を確実に理解できます。
