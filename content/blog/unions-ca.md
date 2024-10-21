+++
title = "Dissenyant una representació en memòria eficient per als valors d'un full de càlcul"
date = 2024-10-21
+++

## Introducció
Primer de tot, aquest article s'adreça a un públic ja familiaritzat amb el llenguatge de programació Rust, i tenir coneixements previs de com les dades es troben a la memòria de l'ordinador és certament recomanable, tot i que els detalls de com funcionen les coses s'explicaran sempre que calgui.

Aquest és el primer article d'una nova sèrie on es detallarà com escriure un programa de fulls de càlculs d'interfície *CLI*, sent la motivació de l'autor les deficiències de tots els que ha fet servir. Concretament, es començarà a escriure aquest nou programa dissenyant la distribució en memòria dels valors que trobem a cada cel·la, i per tant cal saber quins valors pot ser que hi hagi a aquestes.
* Un nombre? Potser.
* Una cadena de caràcters? Potser.
* Una fórmula matemàtica? Potser.

Tanmateix, això és només el començament. Altres programes de fulls de càlcul--com ara Google Docs--suporten fórmules que produeixen matrius (és a dir, diferents cel·les), i  trobem que els valors d'una cel·la es poden trobar sobreescrits pels d'una altra; així doncs, el valor d'una cel·la pot ser el que se l'ha assignat, o un produït per una matriu. El programa esmentat anteriorment, Google Docs, els distingeix, però per simplicitat, això no s'implementarà pas. A més a més, l'autor vol recalcar que el disseny principal d'aquest programa es fonamentarà en formules que treballin amb matrius, de manera contrària a altres fulls de càlcul.

## Un primer intent: delegació dinàmica

Tot i que la manera més idiomàtica de representar les dades d'una cel·la en memòria seria emprar les enumeracions de Rust, per raons educacionals, es començarà amb la tècnica de delegació dinàmica o *dynamic dispatch*, ja que com es podrà observar a continuació, la seva representació en memòria és increïblement eficient (dues paraules, o 128 bits). Aquesta tècnica requereix definir els trets comuns de totes les dades dins d'una *trait*, i permetrà tractar-les totes com una mateixa.

A més a més, com el propòsit d'aquest article és arribar a una implementació millor que la delegació dinàmica, és convenient definir-hi un punt de comparació. Per començar, es pot proposar la següent *trait*.
```rs
trait Valor: Display + Any {}
```

Com es pot apreciar, encara no s'ha definit cap mètode comú necessari; només que cal que implementin alguna manera de representar-se gràficament a la pantalla mitjançant `Display`. A més a més, es requereix que també compleixin els requisits d'`Any`, que els permet convertir-se d'un objecte de delegació dinàmica al seu tipus subjacent; emprant la tècnica d'abatiment o *downcasting*, es pot desar un identificador únic de cada tipus de dada amb la resta d'implementacions dinàmiques, i si es verifica de manera estàtica que aquest identificador és el mateix que un d'un tipus de dades ja conegut, és no hi ha cap risc en convertir-ho al seu homònim estàtic i concret.

D'aquesta manera, es pot definir cada tipus concret:
```rust
type ValorDinàmic = Box<dyn Valor>;
struct Nombre(f64);
struct Caràcters(String);
struct Fòrmula; // To-do
struct Iterador(Box<dyn Iterator<Item = ValorDinàmic>);

// Les implementacions manuals de `Display` i `Valor` es
// deixen com un exercici per al lector.
```

D'aquesta manera, la conclusió més important que es pot fer és el fet que l'`Iterador` requereix dos punters dinàmics, un per ell mateix i un altre pels valors que produeix. Abans de discutir perquè això no és pas una bona arquitectura, s'analitzarà la representació d'un objecte de delegació dinàmica d'un `Valor`:
```txt
[ 64 bits              | 64 bits               ]
|______________________|_______________________|
 Punter a memòria heap, Punter a la taula dinàmica
 on hi ha les dades     on es troben els punters
 concretes.             a les implementacions
                        concretes de les dades.
```

Per tant, la delegació dinàmica sempre ocupa dues paraules o 128 bits, i redueix la quantitat de trànsit de memòria sempre que les dades ocupin menys de 64 bits. Tot i això, els nombres són exactament una paraula, i es guanyaria molt rendiment si es poguessin emmagatzemar sense cap indirecta^[1]. A més a més, tot i que els caràcters ocupen tres paraules, la implementació de vectors prims o *`ThinVec`* les redueix a una (es recorda que una `String` és equivalent a un `Vec<u8>`). Això només deixa les fórmules com una possible dada que sobrepassi aquests 64 bits, però com veurem a articles posteriors, aquesta s'emmagatzemarà dins de la indirecta d'un contenidor compta-referències o `Rc`, que també ocupa sempre 64 bits.

Així doncs, es pot concloure que totes les indirectes de delegació dinàmica serien innecessàries i lentes, però, tot i això, com els fulls de càlcul poden arribar a treballar amb milers de milions de cel·les, escau que els seus valors ocupin el menys possible. Per tant, l'objectiu d'aquest article a partir d'ara serà aconseguir millors resultats en rendiment que la delegació dinàmica sense ocupar més memòria.

## Delegació d'enumeració
Aquest mètode és el que s'hauria d'emprar en comptes de la delegació dinàmica, ja que no només és molt més idiomàtic al llenguatge Rust, sinó que, a més a més, ofereix un rendiment molt superior. Llibreries com `enum_dispatch` permeten automatitzar aquest procediment, que cal recordar que només es pot utilitzar quan tots els tipus que han d'implementar la `trait` es coneixen--cosa que aquí és ben certa. Concretament, `enum_dispatch` generaria la implementació de `Display` per cadascun dels elements de `CellValue`:
```rust
enum CellValue {
    Num(f64),
    Str(ThinVec<u8>),
    Formula(Rc<Formula>),
    Iter(Box<dyn Iterator<Item = CellValue>>),
}

impl Display for CellValue {
    fn fmt(&self, f: ...) -> ... {
        match self {
            Self::Num(x) => x.fmt(f),
            Self::Str(s) => s.fmt(f),
            Self::Formula(o) => o.fmt(f),
            Self::Iter(i) => todo!(),
        }
    }
}
```

El `todo!()` que es troba a la branca d'`Iter`, que en angles significa *per fer*, es necessari perquè encara no s'ha desenvolupat pas la seva implementació concreta. Tanmateix, ara cal analitzar la representació en memòria dels valors de `CellValue` en memòria:
```txt
[ 64 bits              | 64 bits               ]
|______________________|_______________________|
 Etiqueta; indica        Un f64, un ThinVec<u8>
 l'element representat.  o un Rc<Formula>.
```

Com es pot apreciar, aquesta enumeració preserva l'objectiu de no sobrepassar els 128 bits, i permet una execució d'ordres de magnitud més veloç atès al fet que no empra punters opacs per les funcions de la `trait` i les dades són a memòria immediata; les execucions de `Display::fmt()` ara són estàtiques, només cal escollir quina escau depenent de l'etiqueta de l'enumeració, cosa que fa l'expressió de `match`.

Tot i això, cal esmentar que, per motius que es veuran a continuació, els tipus de nombres emprats al programa final no seran de 64 bits, sinó de 128, i de moment s'anomenaran `Long` (llarg en angles). D'aquesta manera, caldrà fer que l'enumeració sigui 64 bits més llarga, incomplint l'objectiu de mantenir-la a 128, i a més a més aquest espai només s'emprarà en el cas d'un nombre:

```txt
[ 64bits              | 64bits                | 64bits                   ]
|_____________________|_______________________|__________________________|
 Etiqueta               Meitat d'un Long, o     L'altra meitat d'un Long,
                        un ThinVec<u8>,         o res sinó.
                        ThinVec<u8> o
                        Rc<Formula> sencers.
```

Per tant, s'està desaprofitant 64 bits per cada valor que no és un nombre, i això s'ha de millorar, cosa que podem fer per un simple motiu: `Long` no empra tots els seus bits.

## El nombre decimal i punters etiquetats.
**Renúncia de responsabilitat: A partir d'aquest punt, el codi pot ser invàlid a certes arquitectures.**

El nom correcte pel `Long` emprat fins ara és `Decimal`, un nombre com els de punt flotant--com ara `f64`--però molt més precís i que es pot emprar a càlculs financers. La seva representació en memòria és la següent:
```txt
[ 15bits | 113 bits                                             ]
|________|______________________________________________________|
 Buit.     Dades internes del nombre, irrelevant.
```

Encara que aquests primers 15 bits no s'empren pas, escau que sempre siguin zero perquè el moment que no ho siguin, les implementacions internes del nombre poden arribar a comportaments invàlids.

Si el lector alguna vegada ha fet servir punters etiquetats, segurament ja n'és conscient de quina cosa es mostrarà: per a qualsevol punter, si l'element al qual apunta té una alineació superior a 1, podem desar-hi tanta informació com el valor d'aquesta, ja que els bits menys significats coincidents amb l'alineació sempre seran 0. Cal anotar que si s'altera aquesta informació. Si es considera que totes les dades tindran una alineació d'almenys 2, i es troba que les dades són un punter o un nombre decimal, es troba que sempre es complirà la següent relació:

```txt
[ 2bits |  ...  ]
|_______|_______|
 Zero.    Dades.
 ```

Per tant, es poden explotar aquests bits per desar-hi l'etiqueta de l'enumeració, en comptes d'emprar una paraula sencera (64 bits). A l'hora de fer-ho, s'emprarà la llibreria de `tagged_pointer`, ja que el funcionament d'això es fora de l'abast d'aquest escrit.

## Ara amb unions, les enumeracions sense etiquetes de C.
Les unions són l'eina emprada per implementar enumeracions estàndards, i permeten definir un espai de memòria que pot estar ocupat per múltiples elements, però només un a la vegada. D'aquesta manera, també poden transmutar-hi dades, ja que es permet llegir una dada A com a una de B.

Per començar, es redefinirà l'enumeració anterior com una unió, cal dir que s'ha d'afegir `ManuallyDrop<T>` si escau. A més a més, s'afegirà una definició de l'etiqueta emprant `TaggedPtr<N, T>`, perquè ens permet transmutar la memòria i llegir l'etiqueta directament.
```rust
union CellValue {
    num: Decimal,
    str: ManuallyDrop<ThinVec<u8>>,
    formula: ManuallyDrop<Rc<Formula>>,
    iter: ManuallyDrop<Box<dyn Iterator<Item = CellValue>>>,
}
```

### Implementant manualment la delegació dinàmica d'`Iter`.

Abans de continuar, cal desxifrar el significat de `Box<dyn T>`, perquè així es trobarà que es pot eliminar una indirecta de 64 bits: la taula de funcions dinàmiques. Tot i que un iterador consta de més de vuitanta funcions que segurament es faran servir més endavant, totes aquestes també tenen una definició general, excepte per una: `Iterator::next()`, que avança els elements dins d'aquest. Per tant, es desarà el punter opac a aquesta funció en comptes de la taula dinàmica sencera, i s'emprarà el compilador per a generar estàticament la resta de funcions a partir d'aquesta. Primer, es definiran manualment les dades:

```rust
struct DynIter {
    dades: TaggedPtr<Aligned, 2>,
    next: NonNull<()>, // El punter mai sera nul
}
```

Ara s'utilitzaran les eines del compilador per generar-hi la resta de funcions:

```rust
impl Iterator for DynIter {
    type Item = CellValue;

    fn next(&mut self) -> Option<Self::Item> {
        type NextFn = for<'a> fn(&'a mut Aligned) -> CellValue;
        unsafe { transmute::<_, NextFn>(self.next)(self.data.ptr().as_mut()) }
    }
}
```

Prenent avantatge d'això, es reestructurarà les unions anteriors per a limitar quan un valor serà un iterador, cosa que serà útil a articles posteriors:
```rust
const MASK_BITS: usize = 2;

union Value {
    tag: TaggedPtr<Aligned, MASK_BITS>,
    num: Decimal,
    str: ManuallyDrop<ThinVec<u8>>,
    formula: ManuallyDrop<Rc<Formula>>,
}

struct DynIter {
    data: TaggedPtr<Aligned, MASK_BITS>,
    next: NonNull<()>,
}

union CellValue {
    tag: TaggedPtr<Aligned, MASK_BITS>,
    value: ManuallyDrop<Value>,
    iter: ManuallyDrop<DynIter>,
}
```

### Configurant TaggedPtr

Cal recordar que s'ha d'escollir les màscares de bits emprades pels diferents valors de la unió. Com `0b00` i `0b01` són les més ràpides a quasi tots els processadors, s'assignaran als nombres i als iteradors, els tipus de dades més comunes. A més a més, s'assignarà la càrrega nul·la `0b00` al nombre decimal, ja que reduirà els possibles riscos d'errors amb els seus bits interns:
```rust
const MÀSCARA_NOMBRE: usize = 0b00;
const MÀSCARA_ITER: usize = 0b01;
const MÀSCARA_STR: usize = 0b10;
const MÀSCARA_FÒRMULA: usize = 0b11;
```

Com la manera de construir-hi un `TaggedPtr<N, T>` és la funció `TaggedPtr::new(ptr, tag)`, s'hauran d'aconseguir els 64 bits menys significants del `CellValue`, i per aquest propòsit es pot emprar una còpia transmutativa via `std::mem::transmute_copy(t)`; concretament, aquesta funció permet copiar-hi dades de llocs de mides diferents reinterpretar-ne els bits, essent aquesta la manera més *ad hoc* de fer-ho en aquest cas:
```rust
impl Value {
    unsafe fn tag(mut self, tag: usize) -> Self {
        self.tag = TaggedPtr::new(transmute_copy(&self), tag);
        self
    }

    fn num(dec: Decimal) -> Self { Self { num: dec } }

    fn str(str: ThinVec<u8>) -> Self {
        unsafe { Self { str: ManuallyDrop::new(str) }.tag(STR_MASK) }
    }

    fn formula(f: Rc<Aligned>) -> Self {
        unsafe { Self { formula: ManuallyDrop::new(f) }.tag(FORMULA_MASK) }
    }
}
```
L'última cosa que es mostrarà en aquest article és com trobar-hi el valor real dins de les unions, i això es pot aconseguir verificant l'etiqueta:
```rust
impl CellValue {
    pub fn get_value(&mut self) -> Option<&mut Value> {
        if self.downcast_iter().is_none() {
            Some(unsafe { transmute(&mut self.value) })
        } else {
            None
        }
    }

    pub fn get_iter(&mut self) -> Option<&mut DynIter> {
        if unsafe { self.tag }.tag() == ITER_MASK {
            Some(unsafe { transmute(&mut self.iter) })
        } else {
            None
        }
    }
}

impl Value {
    fn get_num(&mut self) -> Option<&mut Decimal> {
        if unsafe { self.tag }.tag() == NUM_MASK {
            Some(unsafe { transmute(&mut *self) })
        } else {
            None
        }
    }

    // the rest are almost the same, so I leave as an
    // exercise to the reader creating a macro for them.
}
```

La implementació del codi de destrucció i la *macro* per automatitzar-hi els accessos es deixen com un exercici per al lector.

## Conclusió

Les unions són eines increïblement versàtils, i no pas un arcaisme de l'era perduda de Dennis Ritchie. Aquestes ens permeten modelar dades *a priori* tan simples com una enumeració, i a vegades cal emprar-les si escau el màxim de rendiment. Tanmateix, s'agrairia una ergonomia similar a les que es troben a les enumeracions, ja que si no cal definir-hi nombroses *macros*.

---
[1]: Cal dir que Niko Matsakis ho va proposar al seu [escrit sobre dyn*](https://smallcultfollowing.com/babysteps/blog/2022/03/29/dyn-can-we-make-dyn-sized/).
