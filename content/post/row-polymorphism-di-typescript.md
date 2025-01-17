---
title: "Row Polymorphism di Typescript"
date: 2019-07-28T16:35:16+02:00
description: "Fitur yang sangat penting bagi bahasa pemrograman yang banyak berinteraksi dengan record, seperti Typescript"
images: ["/uploads/dna.jpg"]
image:
  src: "/uploads/dna.jpg"
  caption: Image by <a href="https://pixabay.com/users/qimono-1962238/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1811955">Arek Socha</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1811955">Pixabay</a>
tags: ["typescript", "polymorphism", "types"]
categories: ["programming", "type system"]
draft: false
---

## Intro
Pada suatu hari ada sebuah object koordinat 3 dimensi yang memiliki attribute X, Y, dan Z

```ts
interface Coordinate3d {
  x: number;
  y: number;
  z: number;
}

const coord3d: Coordinate3d = {
  x: 17,
  y: 8,
  z: 45
}
```

dan object kooridinat tersebut dapat di-flip secara horizontal seolah-olah memberikan efek bercermin. Dandan sekalian kalo bisa bro.

```ts
const flipX = (coordinate: Coordinate3d) => ({
  ...coordinate,
  x: coordinate.x * -1
})

const flippedCoord = flipX(coord3d)
// { x: -17, y: 8, z: 45 }
```

Function `flipX` tersebut menerima object bertipe `Coordinate3d` yang memiliki attribute x, y, dan z. Artinya compiler akan complain ketika data seperti `Coordinate2d` dimasukkan ke dalam function tersebut.

```ts
interface Coordinate2d {
  x: number;
  y: number;
}

const coord2d: Coordinate2d = {
  x: 178,
  y: 45,
}

const flippedCoord = flipX(coord2d)
                           ^^^^^^^
// Argument of type 'Coordinate2d' is not assignable to parameter of type 'Coordinate3d'.
//  Property 'z' is missing in type 'Coordinate2d' but required in type 'Coordinate3d'.
```

Sedangkan kalau dipikir baik-baik, fungsi tersebut hanya melakukan perubahan pada row x saja. Kalau yang dibutuhkan hanyalah x, function `flipX` harusnya bisa dibuat lebih "minimal" dengan mengubahnya menjadi

```ts
const flipX = <A extends { x: number }>(hasX: A): A => ({
  ...hasX,
  x: hasX.x * -1
})
```

Dengan begini, `flipX` sekarang hanya memiliki satu attribute sebagai _constraint_, yaitu x yang bertipe number. Seolah-olah ia menyeru, "Barangsiapa memiliki row x bertipe number, niscaya aku akan dapat melakukan manipulasi data terhadapnya. Sesungguhnya, aku adalah row polymorphism".

> Yang berarti: function `flipX` ini **polymorphic terhadap row `x` yang bertipe number**

```js
flipX({ x: 17, y: 8, z: 45 }) // { x: -17, y: 8, z: 45 } ✅
flipX({ x: 178, y: 45 }) // { x: -178, y: 45 } ✅
flipX({ x: 99 }) // { x: -99 } ✅
```

🎉🎉🎉

NOTE: Perlu diperhatikan bahwa penggunaan keyword `extends` di sini sangat penting untuk mengindikasikan bahwa type parameter `A` **bisa jadi memiliki row lain selain x**. Jika kita tuliskan code di atas seperti ini

```ts
type X = { x: number }

const flipX = (hasX: X): X => ({
  ...hasX,
  x: hasX.x * -1
})
```

Maka `flipX` tidak lagi sepenuhnya row-polymorphic. Akan terlihat ketika sebuah subset dari X dimasukkan

```js
const coord3d = { x: 17, y: 8, z: 45 }
const res = flipX(coord3d)
res.y
    ^
// Property 'y' does not exist on type 'X'.
```

Compiler sekarang kehilangan informasi row y (dan row z!) karena notasi dari fungsi `flipX` yang hanya menyatakan X sebagai return type-nya.

Inilah dasar dari definisi Row Polymorphism, yang memungkinkan kita untuk membuat program yang polymorphic terhadap rows **tanpa kehilangan informasi row sama sekali** (kayak barusan).

Tapi jangan salah, kalau mau dibuat lebih constrained juga boleh kok. Misal kita ingin menulis sebuah function yang hanya menerima row `x` saja, tidak lebih tidak kurang. Otherwise compiler will complain.

```ts
type Exact<O, E> = E & Record<Exclude<keyof O, keyof E>, never>

const exact = <O extends Exact<O, { x: number }>>(obj: O) => {}

exact({ x: 1 }) // typecheck ✅

exact({ x: 1, y: 2 }) // complain ✅
              ^
exact({ y: 2, z: 3 }) // complain ✅
        ^     ^
// Type 'number' is not assignable to type 'never'.
```

Plis jangan muntah liat syntax-nya ya!

### Adding
Row Polymorphism nggak sebatas bisa baca row aja, tapi harusnya juga bisa melakukan manipulasi terhadap rows: seperti penambahan, pengurangan, dan penamaan ulang.

Anggap kita punya function yang polymorphic dengan row `firstName` bertipe string, yang menambahkan row `lastName` (jika belum ada) sebagai return value-nya.

```js
const addLastName = obj => ({
  lastName: obj.firstName,
  ...obj
})

addLastName({ firstName: 'jihad' })
// { firstName: 'jihad', lastName: 'jihad' }

addLastName({ firstName: 'jihad', lastName: 'waspada' })
// { firstName: 'jihad', lastName: 'waspada' }
```

Bagaimana informasi penambahan row ini dapat ditangkap oleh compiler?

Kebetulan Typescript memiliki operasi intersection yang dinotasikan menggunakan symbol `&` sehingga operasi penambahan row ini relatif terlihat mudah.

```ts {hl_lines=[5]}
type FName = { firstName: string }

const addLastName = <A extends FName>(
  obj: A
): A & { lastName: string } => ({
  lastName: obj.firstName,
  ...obj
})
```

```js
const person = { firstName: 'jihad', email: 'email@email.email' }
const res = addLastName(person)
// { firstName: 'jihad', lastName: 'jihad', email: 'email@email.email' }
```

### Deleting
Sekarang kita balik: kita ingin membuat sebuah function yang polymorphic terhadap row `firstName` dan `lastName` (keduanya bertipe string), dan ingin menghilangkan `lastName` dari object tersebut.

Typescript sudah menyediakan helper type untuk operasi ini dengan menggunakan [Exclude](https://www.typescriptlang.org/docs/handbook/advanced-types.html).

```ts
type Names = { firstName: string; lastName: string }
type NoLastName<A> = { [K in Exclude<keyof A, "lastName">]: A[K] }

const removeLastName = <A extends Names>(
  obj: A
): NoLastName<A> => {
  const newObj = { ...obj }
  delete newObj.lastName

  return newObj
}
```

### Renaming
Menurut saya renaming ini adalah operasi gabungan dari `adding` dan `deleting`. Anggap kita ingin mengubah label (key) row y menjadi z. Operasi ini dapat dilakukan dengan dua cara yang identik:

1. Tambah row `z` kemudian hapus row `y`, atau
2. Hapus row `y` dulu baru tambah row `z`

```ts
type Remove<A, K1> = { [P in Exclude<keyof A, K1>]: A[P] }

const rename = <A, K1 extends keyof A, K2 extends string>(
  a: A,
  k1: K1,
  k2: K2
): Remove<A, K1> & { [P in K2]: A[K1] } => {
   ^^^^^^^^^^^^^   ^^^^^^^^^^^^^^^^^^^^
//    removal           addition
  ...
}

const x2 = rename({ x: 17, y: 8 }, 'y', 'z')
// { x: 17, z: 8 }
```

## Kesimpulan
Sebuah fungsi yang row-polymorphic adalah fungsi yang reusable, dapat digunakan oleh berbagai macam record selama memenuhi constraint-nya. Row Polymorphism di Typescript lumrahnya ditandai dengan keyword `extends` agar compiler tidak kehilangan informasi tipe rows yang sedang dimanipulasi. Terjaganya informasi ini sangat dibutuhkan ketika kita ingin mengembalikan object tersebut kembali (_the row type parameter appears in the return type_).

Typescript sendiri sudah menyediakan sekumpulan _utlity types_ (seperti Union, Intersection, Exclude, dsb) yang bisa digunakan untuk mendukung Row Polymorphism dengan cukup mudah. Yah, menurut saya sih lebih mudah dibanding [RP nya Purescript](https://github.com/purescript/purescript-record/blob/master/src/Record.purs) yang... sudahlah 😄