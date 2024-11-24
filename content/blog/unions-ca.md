+++
title = "Dissenyant una representació en memòria eficient per als valors d'un full de càlcul"
date = 2024-10-21
+++

## Introducció

Primer de tot, aquest article s'adreça a un públic ja familiaritzat amb el llenguatge de programació *Rust*. A més, és recomanable tenir coneixements previs de com les dades es troben a la memòria de l'ordinador és certament recomanable, tot i que molts detalls s'explicaran sempre que calgui.

Aquest és el primer article d'una nova sèrie on es detallarà com crear un programa de fulls de càlculs, sent la motivació de l'autor les deficiències de tots els que ha fet servir. Concretament, es començarà a escriure aquest nou programa dissenyant la distribució en memòria dels valors que trobem a cada cel·la, i per tant cal enumerar els valors possibles: nombres, caràcters, matrius i fórmules.

## Un primer intent: la delegació dinàmica

Malgrat que la manera més idiomàtica de representar aquestes dades són les enumeracions, es començarà utilitzant aquest altre mètode a causa de la seva eficiència (128 bits constants; un punter per les dades i altre per les implementacions). Aquesta tècnica requereix definir els trets comuns de totes les dades a una *trait*, i permetrà tractar-les totes com una mateixa; a continuació, es defineix una que requereix la implementació de `Display`, per poder representar les dades gràficament:
```rust
trait Valor: Display {}
```

D'aquesta manera, es pot definir cada dada concreta i fer-les implementar-la:
```rust
type ValorDinàmic = Box<dyn Valor>;
struct Nombre(f64);
struct Caràcters(String);
struct Fòrmula(Rc<Fòrmula>);
struct Matriu(Box<dyn Iterator<Item = ValorDinàmic>);

// implementació omesa
```

Cal mencionar que tots els valors ocupen només 64 bits (de manera equivalent a un punter) si es redueixen els `Caràcters` a un `ThinVec<u8>`, i d'aquesta manera no escauria cap abstracció per emmagatzemar-les. Per tant, es pot concloure que aquest mètode és ineficient i impedeix optimitzacions significatives.

## La delegació d'enumeració

Aquesta solució és la més idiomàtica al llenguatge Rust, i generalment ofereix un rendiment molt superior, ja que no defineix cap abstracció. Es pot exemplificar amb les següents dades i implementacions:
```rust
enum Valor {
    Nombre(f64),
    Caràcters(ThinVec<u8>),
    Fòrmula(Rc<Formula>),
    Matriu(Box<dyn Iterator<Item = CellValue>>),
}

impl Display for CellValue {
    fn fmt(&self, f: ...) -> ... {
        match self {
            Self::Nombre(x) => x.fmt(f),
            Self::Caràcters(x) => x.fmt(f),
            Self::Fòrmula(x) => x.fmt(f),
            Self::Matriu(x) => x.fmt(f),
        }
    }
}
```

A continuació, es mostra la representació en memòria:
```txt
[ 64 bits                | 64 bits               ]
|________________________|_______________________|
 Etiqueta; indica          Element en qüestió.
 l'element representat.    
```

Tanmateix, s'ha d'esmentar que per motius de precisió, els nombres emprats al programa final no seran de 64 bits, sinó de 128, i per tant l'enumeració s'haurà d'allargar 64 bits, incomplint l'objectiu de mantenir-la en 128:

```txt
[ 64 bits ... | 64 bits               | 64 bits         ... ]
|_________..._|_______________________|_________________..._|
 Etiqueta       Meitat d'un nombre, o   L'altra meitat d'un
                qualsevol altre         nombre, o res en
                element sencer.         els altres casos.
```

## Implementació amb unions i punters etiquetats

Atès al fet que els 15 bits menys significants del nombre en qüestió sempre són zero, i els punters a les dades tenen una alineació major de dos, es pot fer servir el mètode de punters etiquetats per desar l'etiquetes de dos bits a les dades dins d'aquest espai buit. Per aconseguir-ho escaurà definir les dades a una arcaica unió, pel fet que s'explicarà tot seguit:

```rust
union Valor {
    nombre: Decimal,
    caràcters: ThinVec<u8>,
    fòrmula: Rc<Fòrmula>,
    matriu: Box<dyn Iterator<Item = CellValue>>,
    etiqueta: TaggedPtr<2, ()>
}
```

Com les unions defineixen un espai de memòria que pot estar ocupat per múltiples elements lliurement, també permeten reinterpretar dades, ja que es pot interpretar la dada emmagatzemada *A* com a una dada possible *B* a la unió *A∪B*. Per tant, es pot obtenir l'etiqueta de les dades llegint-la per a després seleccionar la dada correcta, i per acabar només s'ha de definir el valor concret de l'etiqueta a cada cas i tornar a implementar la funcionalitat desitjada:
```rust
const ETIQUETA_NOMBRE: u8 = 0b00;
const ETIQUETA_CARÀCTERS: u8 = 0b10;
const ETIQUETA_FÒRMULA: u8 = 0b11;
const ETIQUETA_MATRIU: u8 = 0b01;

impl Display for Valor {
    fn fmt(&self, f: ...) -> ... {
        unsafe {
            let etiqueta = self.etiqueta.get_tag();
            self.etiqueta.set_tag(0);
            let fmt = match etiqueta {
                ETIQUETA_NOMBRE => self.nombre.fmt()
                ETIQUETA_CARÀCTERS => self.caràcters.fmt()
                ETIQUETA_FÒRMULA => self.fòrmula.fmt()
                ETIQUETA_MATRIU => self.matriu.fmt()
            }
            self.etiqueta.set_tag(etiqueta);
            fmt
        }
    }
}
```

D'aquesta manera, s'aconsegueix el màxim rendiment i es preserva l'estructura següent:
```txt
[ 2 bits |       126 bits    ...   ] 
|________|___________________...___|
 Etiqueta  Dades emmagatzemades
```

## Conclusió

Les unions són eines increïblement versàtils, i no pas un arcaisme de l'era de Dennis Ritchie. Aquestes ens permeten modelar dades *a priori* tan simples com una enumeració, i a vegades cal emprar-les si escau el màxim de rendiment, especialment quan s'ha d'interactuar manualment amb els bits de les dades. Tot i això, com s'ha vist a la seva implementació, aquestes contenen riscos a la seguretat de la memòria i s'han d'utilitzar amb cura.
