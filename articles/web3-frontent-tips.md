---
title: "[wip] Web3フロントエンドTips"
emoji: "👾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [web3, react, ethereum, defi]
published: true
---

この記事は昔に書いた以下の記事の和訳&アップデートになります。
[https://dev.to/yujiym/web-30-frontend-stacks-in-2023-1i03](https://dev.to/yujiym/web-30-frontend-stacks-in-2023-1i03)

1.5年くらいDiFi(EVM)のフロントエンドを書いてきて、おすすめのスタック・ライブラリについてとWeb3ならではのTipsや工夫点についてまとめまして。他に何かおすすめがあったり、質問があれば教えてください！

# 1. システム構成
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tgc6dcgs9rm0dukfdiwo.png)

# 2. スタック・ライブラリ

## フォルダ構成 : [Turborepo](https://turbo.build/repo)

`monorepo (multiple apps in one repo)` 構成を使うことが多いです。
コードのほとんどは `packages` 配下の共通フォルダに置き、各アプリにはルーティングのための最小限のページコンポーネント構成とします。

以下のようなケースでアプリの拡張が柔軟に対応できます
- Chain毎に異なるページや関数を持つdApps
- 機能を限定したLite版を別途用意するケース
- ミドルウェア、APIやツールを切り出すケース

```sh:ファイル構成
.
├── apps
│   ├── dapp
│   │   └── pages (or app)  # each app has only pages dir
│   ├── sub-dapp
│   └── ...
├── packages
│   ├── assets
│   │   ├── img
│   │   └── styles
│   ├── config
│   ├── lib
│   │   ├── constants
│   │   │   └── abis  # abi json
│   │   ├── queries  # thegraph queries
│   │   ├── store  # state
│   │   ├── types
│   │   └── utils
│   ├── react-lib
│   │   ├── components  # components
│   │   └── hooks  # hooks
│   └── tsconfig
└── test
```

## Frontend Framework & ホスティング : [Next.js](https://nextjs.org/) + [TypeScript](https://www.typescriptlang.org/) + [Vercel](https://vercel.com/)

Web3向けのライブラリが充実しているので、Reactが無難な選択肢です。
Next.jsをTurborepoでデプロイする場合、Vercelが最も簡単なのですが、Vercelの便利な機能（Edge関数、ISR...）を使ってしまうと、ロックされてしまい、後々IPFSにデプロイしてdAppを完全分散化する場合に移行が難しくなってしまいます。
[Vite](https://vitejs.dev/)や[esbuild](https://esbuild.github.io/)も小規模なアプリケーションやツールには適していると思います。
最近はServerless frameworkを使ってもっと小さな構成でアプリを作れないか考えています。


#### ethereum接続 : [wagmi](https://wagmi.sh/) + [ethers.js](https://ethers.org/)


`wagmi`は`ethers.js`をwrapしたReact Hooks~~です~~でした。類似ライブラリは多いですが、機能も多く、テストも充実しているのでおすすめです。 -> [wagmiの他のライブラリとの比較](https://wagmi.sh/react/comparison)
また、はEthereum ABIに向けたTypeScript型を提供していて、これは`zod`スキーマとも連携して動作します。これにより、dAppsの型チェックがより厳密にできるようになります。-> [ABIType](https://github.com/wagmi-dev/abitype)
今はethers.jsの代替としてtypescript nativeで軽量で、ヒューマンリーダブルなエラーを返す（これが一番大事）、[viem](https://github.com/wagmi-dev/viem)を作っています。次のversionでethers.js依存はなくなります。


書込み系処理の例
```ts:react-lib/hooks/useApprove.ts
import { usePrepareContractWrite, useContractWrite, useWaitForTransaction, erc20ABI } from 'wagmi'
....
export default function useApprove(targetAddress: AddressType, owner: AddressType, spender: AddressType) {
  ....
  const args: [AddressType, BigNumber] = useDebounce([spender, BigNumber.from(amount)])
  const enabled: boolean = useDebounce(!!targetAddress && !!owner && !!spender)

  const prepareFn = usePrepareContractWrite({
    address: targetAddress,
    abi: erc20ABI,
    functionName: 'approve',
    args,
    enabled
  })

  const writeFn = useContractWrite({
    ...prepare.config,
    onSuccess(data) {
      ....
    }
  })

  const waitFn = useWaitForTransaction({
    chainId,
    hash: write.data?.hash,
    wait: write.data?.wait
  })

  return {
    prepareFn,
    writeFn,
    waitFn
  }
}
```

```tsx:SomeComponent.tsx
import useApprove from 'react-lib/hooks/contracts/useApprove'
....

export default function SomeComponent() {
  const { prepareFn, writeFn, waitFn } = useApprove(targetAddress, account, spenderAddress)

  // handle view with each hook's return value -> https://wagmi.sh/react/hooks/useContractWrite#return-value
  return (
    <div>
      {approve.prepare.isSuccess &&
        <button
          onClick={() => writeFn?.write}
          disabled={writeFn.isLoading || prepareFn.isError}
        >
          Approve
        </button>
      )}
      {writeFn.isLoading && <p>Loading...</p>}
      {writeFn.isSuccess && <p>Tx submitted!</p>}
      {waitFn.isSuccess && <p>Approved!</p>}
    </div>
  )
}
```


#### インデックス、集計処理 : [The Graph](https://thegraph.com/en/) + [urql](https://formidable.com/open-source/urql/)

> The Graph is an indexing protocol for querying networks like Ethereum and IPFS. Anyone can build and publish open APIs, called subgraphs, making data easily accessible.

Contract Callは遅く、1度に1つしか呼び出せないため、複数のコントラクトコール結果をインデックスしたり、集計値を計算するためのレイヤーが必要になってきます。（例：liquidity一覧をtableで取得する）
集計とインデックス処理は、Contractのeventをトリガーとするsubgraphでyml形式で指定でき、その結果をthegraphのdbに保存します。
クライアント側では、graphqlでデータを問い合わせることができます。`urql`は、軽量なgraphqlクライアントです。

ハマりポイントとしては、
- 同block内で実行順序が保証されないようなケースへ対応しておかないと、エラーになる
- localhost環境で開発・debugするには、Archive Nodeが必要（過去時点の状態に対してcontracl callかけるため）、HarfHatで環境をつくるのは辛いのでMock等をかかなければいけない

※[subsquid](https://subsquid.io/)が最近EVM Chainのサポートを追加したので、次はこれを試してみたいと思っています。


```ts
// define query
import { gql } from 'urql'

export const GET_MY_LIQUIDITIES = gql`
  query getMyLiquidities($account: String) {
    ${FRAGMENT}
    user(id: $account) {
      id
      liquidities {
        id
        protocols {
          ...Fragment
        }
      }
    }
  }
`
```

```tsx:SomeComponent.tsx
// call in component
import { useQuery } from 'urql'
....
export function SomeComponent() {
  ....
  const [{ data, fetching }] = useQuery({
    query: GET_MY_LIQUIDITIES,
    variables: {
      account: account.toLowerCase()
    },
    pause: !account
  })
  ....
}
```

## UI, CSS : [tailwindcss](https://tailwindcss.com/) + [PostCSS](https://postcss.org/) + [Radix UI](https://www.radix-ui.com/) + [UI components by shadcn](https://github.com/shadcn/ui)

シンプルで＆カスタマイズ可能なFrameworkが好みです。
Radix UIは[Headless UI](https://headlessui.com/)よりも機能が豊富ですが、スタイルを追加するのは難しいです。なので、shadcnによるコンポーネントはほど良い塩梅だと思います。

私はこの[CSS variables with Tailwind CSS setup](https://stackoverflow.com/a/66333605)を使っています。

## State管理 : [jotai](https://jotai.org/)

`jotai`は、シンプルで軽量な状態管理ライブラリです。`useState` + `ContextAPI` のようなシンプルな構成で、余計な再描画を防ぎ、多くのユーティリティを備えています。
類似のライブラリとして、[recoil](https://recoiljs.org/), [zustand](https://zustand-demo.pmnd.rs/), [valtio](https://valtio.pmnd.rs/)があるので、好きなものを選ぶのが良いと思います。


```ts
// packages/store/index.ts
import { atom } from "jotai";

type SessionType = {
  chainId?: number;
  account?: string;
};

export const sessionsAtom = atom<SessionType>({});
```

```tsx
// in component or hook
import { sessionAtom } from 'lib/store'
import { useAtom } from 'jotai'

....
const [session, setSession] = useAtom(sessionAtom)

....
setSession({ chainId: res.chainId, account: res.address })
```

## フォームライブラリ：[react-hook-form](https://react-hook-form.com/) & [zod](https://zod.dev)

前述したwagmi、typescriptと親和性の高いzodでスキーマ管理をして、[react-hook-form](https://react-hook-form.com/)を使うことが多いです。


# 3. Web3 frontend tips

#### トランザクションを非同期で処理する

BlockChainのレスポンス（特にMainnet）は、Web2のそれのレスポンスよりかなり遅いです。
ユーザーがトランザクションを送信した後、ディスプレイの読み込みやトランザクションのステータスの表示など、ユーザへの適切なフィードバックが必要です。

ユーザーのトランザクションが確認されたにもかかわらず、thegraphからのデータ更新が遅れ、ユーザーの操作がブラウザに反映されない場合が発生します。このケースは、最初のレンダリングにはthegraphのデータを使用し、後でcontract callの結果でその値を上書きするようにします。

また、トランザクションがconfirmする前にユーザーがページやサイトを離れてしまった場合に対応するため、ユーザーの未完了のトランザクションは一旦`localStorage`で永続化して、確定するまで保持するようにしています。


```tsx
// watching uncompleted transaction
import { useEffect } from 'react'
import { useBlockNumber, useProvider } from 'wagmi'
....

export default function useTxHandler() {
  const { chainId, account } = useWallet()
  const transactions = useTransactions() // get user's transactions from state
  const blockNumber = useBlockNumber({
    chainId,
    scopeKey: 'useTxHandler',
    watch: true
  })
  const provider = useProvider<any>()

  useEffect(() => {
    transactions.map(async (tx: Transaction) => {
      await provider
        .getTransactionReceipt(tx.hash)
        .then(async (transactionReceipt: any) => {
          if (transactionReceipt) {
            .... // update user's transaction status here
            })
          }
        })
        .catch((e: any) => {
          log(`failed to check transaction hash: ${tx.hash}`, e)
        })
    })
  }, [account, chainId, PROVIDER_URL, transactions, blockNumber.data])
}
```

## BigNumberの取り扱いについて

ERC20 には `decimals` フィールドがあり桁を意識して扱わなければなりません。
過去、外部ライブラリがいろいろ使われていたのですが、ethers.js v6から、ES2020ビルトインのBigIntが採用されました。これを使っていきましょう。 -> [Migrating from v5](https://docs.ethers.org/v6/migrating/#migrate-bigint) 


## [Uniswap tokenlist format](https://github.com/Uniswap/token-lists)

> This package includes a JSON schema for token lists, and TypeScript utilities for working with token lists.

OnChainにあるTokenの情報は限られており、OffChainのどこかにリスト形式で保持する必要があります。Uniswapが公開しているこの形式に従っておくと、取り扱いが便利です。

## 4. 気になっていること

### [Permit2](https://github.com/Uniswap/permit2)

> Permit2 introduces a low-overhead, next generation token approval/meta-tx system to make token approvals easier, more secure, and more consistent across applications.

各dAppの各コントラクトを承認する代わりに、一度`Permit2`を承認すれば、ERC20 ContractへのApproveをコントロールできるようになります。これにより、UX（walletの毎回のapproveがなくなる！）とセキュリティ（各Dappのコントラクトに対して大量のallowanceが残っていることによる問題）を大幅に改善することが期待できます。早くデファクトになってほしい。

### [Lit Protocol](https://litprotocol.com/)

> Lit is distributed key management for encryption, signing, and compute.

### [Mina](https://minaprotocol.com/)

> Mina is building the privacy and security layer for web3 with zero knowledge proofs.

## 5. 所感
- 1.5年前は雑なフロントエンド/ライブラリが多かったけど、ものすごい勢いでクオリティ高くなってきてる。動きも早い（2ヶ月前に書いた記事の和訳に伴う修正ですら結構あった）
- 最近だとERC-4337によるAA実装や、ZKP関係で追えないくらいの新情報が流れてくる
- UXをつきつめていくと、キモの台帳部分のセキュリティ以外については別の手段で代替するのが良いと思うようになった。OffChainで処理したり、Modular Blockchain (Lit Protocol, Mina Protocol, Ceramic Network)と組み合わせたり。そうなると、Web5やのnostrのやろうとしていることも筋は遠っているように思う。
  - ようやくDeFi以外のキラーユースケースが出てきそうな雰囲気で楽しみ
