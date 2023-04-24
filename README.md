# Firebase Modular

En el Sprint del 10/04 al 21/04 avanzamos con la migración parcial de los servicios de Atom a la versión modular de Firebase.
Para ésto, se requiere investigar previamente la documentación de [Angular Fire](https://github.com/angular/angularfire#developer-guide).


## Detalles a tener en cuenta

Inicialmente migramos los archivos `*.service.ts` y sus archivos referenciados en el proyecto, los cuales incluyen funcionalidades como Firestore, Functions, Analytics, Performance y Cloud Storage.
Como breaking changes para algunos equipos por el impacto mayor de esta migración, vamos a explicar específicamente todo lo relacionado con `Firestore`.

En primera instancia, si visualizamos el archivo `app.module.ts`, podemos encontrar los providers de `AngularFire`:
```js
@NgModule({
	declarations: [AppComponent],
	imports: [
		...
		AngularFireModule.initializeApp(environment.firebase),
        	AngularFireFunctionsModule,
        	AngularFireDatabaseModule,
        	AngularFirestoreModule,
        	AngularFireAuthModule,
        	AngularFireStorageModule,
        	AngularFirePerformanceModule,
        	AngularFireAnalyticsModule,
        	AngularFireMessagingModule,
        	...
	],
	...
});
```
Como las versiones de compatibilidad importadas de `@angular/fire/compat` conviven correctamente junto a las versiones modulares importadas de `@angular/fire`, separamos los imports de Firebase para realizar la migración parcialmente e incluir las 2 versiones hasta que nos quedemos 100% con la modular:
```ts
const ANGULAR_FIRE_DEPS = [
    	provideFirebaseApp(() => initializeApp(environment.firebase)),
    	provideAuth(() => getAuth()),
    	providePerformance(() => getPerformance()),
    	provideAnalytics(() => getAnalytics()),
    	provideFirestore(() => getFirestore()),
    	provideMessaging(() => getMessaging()),
    	provideStorage(() => getStorage()),
    	AngularFireModule.initializeApp(environment.firebase),
    	AngularFireFunctionsModule,
    	AngularFireDatabaseModule,
    	AngularFirestoreModule,
    	AngularFireAuthModule,
    	AngularFireMessagingModule,
];

@NgModule({
	declarations: [AppComponent],
	imports: [
		...ANGULAR_FIRE_DEPS,
        	...
	],
	...
});
```
Como tarea entre todos, tenemos que reemplazar todos los imports de `@angular/fire/compat` en el proyecto y continuar con la versión modular.


## AngularFirestore

### Constructor de clase:
Anteriormente utilizamos la inyección de `AngularFirestore` que proviene de `@angular/fire/compat/firestore` para crear todas las queries y funcionalidades de Firestore.
Para ésto, si queríamos obtener la data de una colección, las implementaciones aproximadas eran las siguientes:
```ts
import { AngularFirestore } from "@angular/fire/compat/firestore";

export class ExampleComponent implements OnInit {
	private user: User; // Tomemos como ejemplo el usuario logueado en Atom
	private conversations: Conversation[] = [];
	
	constructor(private afs: AngularFirestore) {}
	ngOnInit(): void {
		this.afs
			.collection('companies')
			.doc(this.user.companyId)
			.collection<Conversation>('conversations')
			.get() // Nos trae los documentos de la colección con su metadata
			.subscribe(response: QuerySnapshot<Conversation> => {
				this.conversations = response.docs.map(doc => doc.data()); // Transformamos la data
			})
	}
}
```
--------------

### Cambios en tiempo real:
Ahora bien, si queríamos obtener todos los cambios en tiempo real de la colección, las implementaciones aproximadas eran las siguientes:
```ts
this.afs
	.collection('companies')
	.doc(this.user.companyId)
	.collection<Conversation>('conversations')
	.valueChanges() // En cada cambio que ocurra, nos trae la colección entera sin su metadata
	.subscribe(response: Conversation[] => {
		this.conversations = response // No es necesario transformar la data
	})
```
Y en el caso de que necesitáramos escuchar sólamente los últimos cambios sin necesidad de traernos toda la colección,  las implementaciones aproximadas eran las siguientes:
```ts
this.afs
	.collection('companies')
	.doc(this.user.companyId)
	.collection<Conversation>('conversations')
	.stateChanges() // En cada cambio que ocurra, nos trae las modificaciones con su tipo de estado.
	.subscribe(response: DocumentChangeAction<Conversation>[] => {
		response.forEach(change: DocumentChangeAction<Conversation> => {
			/*
			* Acá podemos acceder a las propiedades doc, newIndex, oldIndex y type de cada dato,
			* para realizar un recorrido y reconstruir los arreglos necesarios según el caso que nos 
			* devuelva la propiedad type.
			*/
			...
		})
	})
```
-------------

### Queries

En el caso de requerir una consulta avanzada con limitantes y algún orden específico, las implementaciones aproximadas eran las siguientes:
```ts
this.afs
	.collection('companies')
	.doc(this.user.companyId)
	.collection<Conversation>('conversations', (ref) =>
		ref
			.where('property', '==', 'someValue')
			.orderBy('otherProperty', 'desc')
			.limit(N)
	)
	.valueChanges() // En cada cambio que ocurra, nos trae la colección entera sin su metadata
	.subscribe(response: Conversation[] => {
		this.conversations = response // No es necesario transformar la data
	})
```
-------------

### Documentos simples:
Por último, mencionamos algunos ejemplos de operaciones para los documentos simples bajo promesas y observables, como agregar/actualizar/eliminar o escuchar un documento en particular:

```ts
async addDoc(data: Conversation, companyId: string): Promise<DocumentReference<Conversation>> {
	return this.afs
		.collection("companies")
		.doc(companyId)
		.collection<Conversation>("conversations")
		.add(data)
}

async setDoc(data: Conversation, companyId: string): Promise<void> {
	return this.afs
		.collection("companies")
		.doc(companyId)
		.collection("conversations")
		.doc(data.id)
		.set(data);
}

async updateDoc(conversationId: string, data: Partial<Conversation>, companyId: string): Promise<void> {
	return this.afs
		.collection("companies")
		.doc(companyId)
		.collection("conversations")
		.doc(conversationId)
		.update({
			...data
		});
}

async deleteDoc(conversationId: string, companyId: string): Promise<void> {
	return this.afs
		.collection("companies")
		.doc(companyId)
		.collection("conversations")
		.doc(conversationId)
		.delete()
}

// En el caso de que se requiera escuchar en tiempo real los cambios de un documento
listenDoc(conversationId: string, companyId: string): Observable<Conversation> {
	return this.afs
		.collection("companies")
		.doc(companyId)
		.collection("conversations")
		.doc<Conversation>(conversationId)
		.valueChanges()
}
```

# Firestore Modular
Luego de repasar las implementaciones anteriores, procedemos a explicar las nuevas!

### Constructor de clase:
Ahora utilizamos la inyección de `Firestore` que proviene de `@angular/fire/firestore` para crear todas las queries y funcionalidades de Firestore.
Para la construcción de la clase, tenemos 2 opciones (cualquiera está bien):

- Opción 1
```ts
import { Inject } from  "@angular/core";
import { Firestore } from "@angular/fire/firestore";

export class ExampleComponent {
	private firestore: Firestore = Inject(Firestore);
	constructor() {}
	ngOnInit(): void {...}
}
```

- Opción 2
```ts
import { Firestore } from "@angular/fire/firestore";

export class ExampleComponent {
	constructor(private firestore: Firestore) {}
	ngOnInit(): void {...}
}
```

Entonces, si queremos obtener la data de una colección, las implementaciones aproximadas serán las siguientes:
```ts
import { Firestore } from "@angular/fire/firestore";

export class ExampleComponent implements OnInit {
	private user: User; // Tomemos como ejemplo el usuario logueado en Atom
	private conversations: Conversation[] = [];
	
	constructor(private firestore: Firestore) {}
	async ngOnInit(): Promise<void> {
		const snapshots = await getDocs(
			collection(
				this.firestore, // Es la instancia inyectada de Firestore como primer parámetro
				`companies/${this.user.companyId}/conversations` // Es la ruta de la colección
			)  as CollectionReference<Conversation> // Casteo para tener los tipados correspondientes
		)
		this.conversations = snapshots.map(doc => doc.data()) // Es necesario transformar la data
	}
}
```

--------------

### Queries

En el caso de requerir una consulta avanzada con limitantes y algún orden específico, las implementaciones aproximadas serán las siguientes:
```ts
// Este ejemplo de imports lo hacemos explícito para que
// recuerden siempre importar TODO de @angular/fire/firestore

import {
    Firestore, // este import siempre tiene que estar, los otros son opcionales según la implementación
    getDocs,
    query,
    collection,
    CollectionReference,
    where,
    orderBy,
    limit,
} from "@angular/fire/firestore";

const snapshots = await getDocs(
	query(
		collection(
			this.firestore,
			`companies/${this.user.companyId}/conversations`
		) as CollectionReference<Conversation>,
		where('property', '==', 'someValue'),
		orderBy('otherProperty', 'desc'),
		limit(N)
	)
)
this.conversations = snapshots.map(doc => doc.data()); // Es necesario transformar la data
```
-------------

### Tiempo real - Colección entera:
Ahora bien, si queremos obtener todos los cambios en tiempo real de una colección, tenemos los métodos `collectionData`, `collectionSnapshots`, y `collectionChanges`.

Los primeros 2 mencionados siempre van a retornar la colección entera si ocurre un cambio en la colección en sí, a diferencia del tercero que nos devolverá sólo los cambios que ocurren en ese momento (similar a `stateChanges` de `AngularFirestore`).

Para tener en cuenta:
- `collectionData`: Devuelve un arreglo sin metadata.
- `collectionSnapshots`: Devuelve un arreglo con metadata.


```ts
collectionData(
	collection(
		this.firestore,
		`companies/${this.user.companyId}/conversations`
	) as CollectionReference<Conversation>
)
.subscribe(response => {
	this.conversations = response; // No es necesario transformar la data
})

collectionSnapshots(
	collection(
		this.firestore,
		`companies/${this.user.companyId}/conversations`
	) as CollectionReference<Conversation>
)
.subscribe(response => {
	this.conversations = response.map(doc => doc.data()); // Es necesario transformar la data
})
```
-----------

### Tiempo real - Últimos cambios:

Y en el caso de que necesitemos escuchar sólamente los últimos cambios sin necesidad de traernos toda la colección, creamos un método genérico en un archivo llamado `firebase.utils.ts` que nos permite reconstruir el arreglo que tenemos actualmente con datos, sin necesidad de crear un método aparte en cada componente o servicio donde implementemos el `collectionChanges`.
- Como primer parámetro, le pasamos el arreglo que queremos reconstruir.
- Como segundo parámetro, le pasamos el arreglo de cambios que nos retorna el observable de `collectionChanges`, sin transformar, ya que necesitamos saber el estado del documento que cambió.
- Retorna un nuevo arreglo que podemos asignar a la referencia de la propiedad que pasamos como primer parámetro.

```ts
// Mejorable en todo sentido jeje
static fullfillData<T>(array: T[], e: DocumentChange<T>[]): T[] {
    const arr: T[] = [...array];
    e.forEach(({ oldIndex, newIndex, ...docChange }) => {
        const t: T = docChange.doc.data();
        const index = arr.findIndex((i) => i["id"] === t["id"]);
        switch (docChange.type) {
            case "added":
                if (oldIndex === -1) {
                    arr.push(t);
                }
                break;
            case "removed": {
                if (index !== -1) {
                    arr.splice(index, 1);
                }
                break;
            }
            case "modified": {
                if (index !== -1) {
                    arr[index] = t;
                }
                break;
            }
        }
    });
    return arr;
}
```

Ejemplo de invocación:
```ts
import { fullFillData } from "src/app/utils/firebase.utils";

// En este caso incluímos un ejemplo utilizando una query también
collectionChanges(
	query(
		collection(
			this.firestore,
			`companies/${this.user.companyId}/conversations`
		) as CollectionReference<Conversation>,
		where('property', '==', 'someValue'),
		orderBy('otherProperty', 'desc'),
		limit(N)
	)
).subscribe(response => {
	this.conversations = fullFillData(this.conversations || [], response); // El OR se indica porque puede ser un arreglo vacío
})
```
-------------

### Documentos simples:
Ahora mencionamos algunos ejemplos de operaciones para los documentos simples bajo promesas, como agregar/actualizar/eliminar un documento en particular:

```ts
async addDocument(data: Conversation, companyId: string): Promise<DocumentReference<Conversation>> {
	return addDoc(
		collection(
			this.firestore,
			`companies/${companyId}/conversations`
		) as CollectionReference<Conversation>,
		data
	);
}

async setDocument(data: Conversation, companyId: string): Promise<void> {
	return setDoc(
		doc(
			this.firestore,
			`companies/${companyId}/conversations/${data.id}`
		),
		data
	);
}

async updateDocument(conversationId: string, data: Partial<Conversation>, companyId: string): Promise<void> {
	return updateDoc(
		doc(
			this.firestore,
			`companies/${companyId}/conversations/${conversationId}`
		),
		{ ...data }
	);
}

async deleteDocument(conversationId: string, companyId: string): Promise<void> {
	return deleteDoc(
		doc(
			this.firestore,
			`companies/${companyId}/conversations/${conversationId}`
		)
	);
}
```

### Documentos simples - Tiempo real

En el caso de que se requiera escuchar en tiempo real los cambios de un documento, tenemos que importar el método `onSnapshot` que nos devuelve un callback ejecutable de tipo `Unsubscribe`, similar a los observables de Angular pero se ejecuta directamente.

Para desubscribirse de un listener, acá tienen los ejemplos (haciendo referencia a listeners almacenados en la clase):
- Observable de Angular: `this.subscription.unsubscribe()`
- Callback de onSnapshot: `this.docSnapshotUnsubscribe()`
 
 Y este sería un ejemplo de un método de tipo `Unsubscribe` dentro de un componente o servicio:
```ts
private listenDocument(conversationId: string, companyId: string): void {
	this.docSnapshotUnsubscribe = onSnapshot(
		doc(
			this.firestore,
			`companies/${companyId}/conversations/${conversationId}`
		),
		(response) => {
			// Hacemos de cuenta que tenemos una propiedad en la clase para la conversacion
			this.conversation = response.data(); // Necesitamos transformar la data
		}
	)
}

// OBLIGATORIO implementar ngOnDestroy en los componentes
ngOnDestroy(): void {
	this.docSnapshotUnsubscribe();
}
```
