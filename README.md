# `fp-ts` & co. : programmation fonctionnelle en TypeScript

Dans certains projet, en TypeScript, on utilise `io-ts` pour valider des types à l'execution. Pourquoi ne pas aussi utiliser `fp-ts` (et son écosystème), inclu avec `io-ts`, comme bibliothèque de types fonctionnels ?

Toutes les bibliothèques présentées sont du même mainteneur : [Giulio Canti](https://github.com/gcanti)


## `io-ts`

* [doc](https://gcanti.github.io/io-ts/) / [github](https://github.com/gcanti/io-ts)

Validation de types avec `io-ts` :

```ts
import * as t from 'io-ts'
import * as E from 'fp-ts/lib/Either'
import { pipe } from 'fp-ts/lib/pipeable'

const CatType = t.strict({
    name: t.string,
    laser: t.union([t.null, t.number]),
    weapons: t.array(t.string)
})

type Cat = t.TypeOf<typeof CatType>
// type Cat = { 
//     name: string
//     laser: null | number
//     weapons: string[] 
// }

pipe(
    CatType.decode(rawCat),
    E.fold(errors => ..., cat => doStuff(cat))
)
```

> * `t.strict({ name: A })` est l'alias de `t.exact(t.type({ name: A }))` (voir [ici](https://github.com/gcanti/io-ts#implemented-types--combinators))
> * `Type.decode` retourne une `Either<t.Errors, Cat>`, sur laquelle on peut `fold`.


## `fp-ts`

* [doc](https://gcanti.github.io/fp-ts/) / [github](https://github.com/gcanti/fp-ts)

### `Option`

Pour mieux gérer le fait que `laser` est `Nullable`, on peut le transformer en `Option` :

```ts
import * as O from 'fp-ts/lib/Option'

const laser: O.Option<number> = O.fromNullable(cat.laser)
const laserOrNull: null | number = O.toNullable(laser)
```

> `Option.toNullable` (ou `Option.toUndefined`) pour repasser en `null | A` (ou `undefined | A`), par exemple pour intégrer la valeur dans un template `jsx`

Ce qui permet de faire tout plein de choses avec notre `Option` :

```ts
const laserValue: number = pipe(
    laser,
    O.map(_ => _ + 1),
    O.chain(_ => O.some(_ - 1)),
    O.filter(_ => _ > 2),
    O.getOrElse(() => -1)
)
```

> * Pour un `None` : `O.none`.
> * `chain` correspond au `flatMap` en Scala


### `Array` et `Record`

`fp-ts` ajoute également des fonction pour manipuler les `array` et les `object` de manière typesafe.

#### `Array`

```ts
import * as A from 'fp-ts/lib/Array'

const weapon: O.Option<string> = 
    A.findFirst<string>(_ => _.startsWith('AK'))(cat.weapons)

const weaponBis: O.Option<string> = pipe(
    cat.weapons,
    A.findFirst(_ => _.startsWith('AK'))
)
```

> * Les deux syntaxes font la même chose, mais la première :
>   * ne permet pas de chaîner les appels
>   * ne permet pas d'inférer les types paramétriques de la fonction, qui doivent donc être précisés
> * L'arité max de `pipe` est de 10.


#### `Record`

```ts
import * as R from 'fp-ts/lib/Record'

const items: Record<string, number> = { shield: 54268, sword: 4236 }

const over9000: Record<string, number> = pipe(
    items,
    R.filter(_ => _ > 9000),
    R.mapWithIndex((k, n) => k === 'bow' ? n + 1 : n)
)

const ammo: O.Option<number> = R.lookup('ammo', items)
```


### `Task`

Un wrapper de `Promise`.

```ts
import * as T from 'fp-ts/lib/Task'

const task1: T.Task<number> = () => Promise.resolve(18)
const task2: T.Task<number> = () => Promise.resolve(24)

const task: Promise<number> = pipe(
    task1,
    T.map(_ => _ + 1),
    T.chain(a => pipe(task2, T.map(b => a + b)))
)()
```

> Ne pas oublier de faire `()` sur la `Task` pour récupérer la `Promise`.


### Autres

Comme la bibliothèque est largement inspirée de Haskell, on y trouve d'autres types fonctionnels : 
* `NonEmptyArray`
* `Validation`
* `IO` (et `IOEither`) 
* `OptionT`, `EitherT` et autres transformers
* ...


## `fp-ts-contrib`

* [doc](https://gcanti.github.io/fp-ts-contrib) / [github](https://github.com/gcanti/fp-ts-contrib)

Ajoute encore plus de fonctions, sans doute très pratiques, mais avec très peu de doc. La seule fonctionnalité que je connais est celle qui permet de faire des for-comprehension.

```ts
import { Do } from 'fp-ts-contrib/lib/Do'

const task: T.Task<number> = Do(T.task)
    .bind('a', task1)
    .bindL('b', ({ a }) => pipe(task2, T.map(_ => _ + a)))
    .bindL('c', ({ a, b }) => () => Promise.resolve(a + b))
    .return(({ a, b, c }) => a * b + c)
```

> Et cela fonctionne avec n'importe quelle monade comme paramètre du `Do`.


## `io-ts-types`

* [doc](https://gcanti.github.io/io-ts-types) / [github](https://github.com/gcanti/io-ts-types)

`io-ts-types` ajoute d'autres (dé)sérialiseurs pour `io-ts` : `Option`, `Either`, `Date`, `UUID` et d'autres.

Plus haut, on a écrit notre `Type` nullable comme cela :

```ts
const NullableNumberType = t.union([t.null, t.number])
type NullableNumber = t.TypeOf<typeof NullableNumberType>
// type NullableNumber = null | number
```

Pour ensuite le transformer en `Option` :

```ts
const maybeNumber: O.Option<number> = O.fromNullable(nullableNumber)
```

Avec `io-ts-types`, pour transformer un `null | number` en `Option<number>`, on peut directement écrire :

```ts
import { optionFromNullable } from 'io-ts-types/lib/optionFromNullable'

const OptionNumberType = optionFromNullable(t.number)
type OptionNumber = t.TypeOf<typeof OptionNumberType>
// type OptionNumber = O.Option<number>
```


## `monocle-ts`

* [doc](https://gcanti.github.io/monocle-ts) / [github](https://github.com/gcanti/monocle-ts)

> A (partial) porting of Scala monocle.
