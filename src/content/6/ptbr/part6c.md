---
mainImage: ../../../images/part-6.svg
part: 6
letter: c
lang: ptbr
---

<div class="content">

Vamos expandir a aplicação de modo que as notas sejam armazenadas no backend. Usaremos o [json-server](/ptbr/part2/obtendo_dados_do_servidor), que nos é familiar desde a parte 2.

O estado inicial do banco de dados é armazenado no arquivo <i>db.json</i>, que está localizado na raiz do projeto:

```json
{
  "notes": [
    {
      "content": "the app state is in redux store",
      "important": true,
      "id": 1
    },
    {
      "content": "state changes are made with actions",
      "important": false,
      "id": 2
    }
  ]
}
```

Vamos instalar o json-server para o projeto...

```js
npm install json-server --save-dev
```

e adicione a linha seguinte à parte <i>scripts</i> do arquivo <i>package.json</i>

```js
"scripts": {
  "server": "json-server -p3001 --watch db.json",
  // ...
}
```

Agora, vamos iniciar o json-server com o comando _npm run server_.

### Obtendo dados do backend

Em seguida, vamos criar um método no arquivo <i>services/notes.js</i>, que usa o <i>axios</i> para buscar dados do backend

```js
import axios from 'axios'

const baseUrl = 'http://localhost:3001/notes'

const getAll = async () => {
  const response = await axios.get(baseUrl)
  return response.data
}

export default { getAll }
```

Vamos adicionar o axios ao projeto

```bash
npm install axios
```

Vamos alterar a inicialização do estado em <i>noteReducer</i>, para que, por padrão, não haja notas:

```js
const noteSlice = createSlice({
  name: 'notes',
  initialState: [], // highlight-line
  // ...
})
```

Também vamos adicionar uma action <em>appendNote</em>, para adicionar um objeto nota:

```js
const noteSlice = createSlice({
  name: 'notes',
  initialState: [],
  reducers: {
    createNote(state, action) {
      const content = action.payload

      state.push({
        content,
        important: false,
        id: generateId(),
      })
    },
    toggleImportanceOf(state, action) {
      const id = action.payload

      const noteToChange = state.find(n => n.id === id)

      const changedNote = { 
        ...noteToChange, 
        important: !noteToChange.important 
      }

      return state.map(note =>
        note.id !== id ? note : changedNote 
      )     
    },
    // highlight-start
    appendNote(state, action) {
      state.push(action.payload)
    }
    // highlight-end
  },
})

export const { createNote, toggleImportanceOf, appendNote } = noteSlice.actions // highlight-line

export default noteSlice.reducer
```

Uma maneira rápida de inicializar o estado das notas, com base nos dados recebidos do servidor, é buscar as notas no arquivo <i>index.js</i> e despachar uma action, usando o action creator <em>appendNote</em>, para cada objeto nota individual:

```js
// ...
import noteService from './services/notes' // highlight-line
import noteReducer, { appendNote } from './reducers/noteReducer' // highlight-line

const store = configureStore({
  reducer: {
    notes: noteReducer,
    filter: filterReducer,
  }
})

// highlight-start
noteService.getAll().then(notes =>
  notes.forEach(note => {
    store.dispatch(appendNote(note))
  })
)
// highlight-end

// ...
```

Despachar várias actions parece um pouco impraticável. Vamos adicionar um action creator <em>setNotes</em> que pode ser usado para substituir diretamente o array de notas. Vamos obter o action creator, da função <em>createSlice</em>, implementando a action <em>setNotes</em>:

```js
// ...

const noteSlice = createSlice({
  name: 'notes',
  initialState: [],
  reducers: {
    createNote(state, action) {
      const content = action.payload

      state.push({
        content,
        important: false,
        id: generateId(),
      })
    },
    toggleImportanceOf(state, action) {
      const id = action.payload

      const noteToChange = state.find(n => n.id === id)

      const changedNote = { 
        ...noteToChange, 
        important: !noteToChange.important 
      }

      return state.map(note =>
        note.id !== id ? note : changedNote 
      )     
    },
    appendNote(state, action) {
      state.push(action.payload)
    },
    // highlight-start
    setNotes(state, action) {
      return action.payload
    }
    // highlight-end
  },
})

export const { createNote, toggleImportanceOf, appendNote, setNotes } = noteSlice.actions // highlight-line

export default noteSlice.reducer
```

Agora, o código no arquivo <i>index.js</i> está muito melhor:

```js
// ...
import noteService from './services/notes'
import noteReducer, { setNotes } from './reducers/noteReducer' // highlight-line

const store = configureStore({
  reducer: {
    notes: noteReducer,
    filter: filterReducer,
  }
})

noteService.getAll().then(notes =>
  store.dispatch(setNotes(notes)) // highlight-line
)
```

> **Obs:** Por que não usamos await em vez de promessas e manipuladores de
> eventos registrados nos métodos then?
>
> O await só funciona dentro de funções <i>async</i>, e o código em <i>index.js</i> não
> está dentro de uma função. Portanto, devido à natureza simples da operação,
> abstemos-nos de usar <i>async</i> desta vez.

No entanto, decidimos mover a inicialização das notas para o componente <i>App</i> e, como de costume, ao buscar dados de um servidor, usaremos o <i>hook effect</i>.

```js
import { useEffect } from 'react' // highlight-line
import NewNote from './components/NewNote'
import Notes from './components/Notes'
import VisibilityFilter from './components/VisibilityFilter'
import noteService from './services/notes'  // highlight-line
import { setNotes } from './reducers/noteReducer' // highlight-line
import { useDispatch } from 'react-redux' // highlight-line

const App = () => {
    // highlight-start
  const dispatch = useDispatch()
  useEffect(() => {
    noteService
      .getAll().then(notes => dispatch(setNotes(notes)))
  }, [])
  // highlight-end

  return (
    <div>
      <NewNote />
      <VisibilityFilter />
      <Notes />
    </div>
  )
}

export default App
```

O uso do hook useEffect  causa um aviso do eslint:

![vscode warning useEffect missing dispatch dependency](../../images/6/26ea.png)

Podemos nos livrar desse aviso fazendo o seguinte:

```js
const App = () => {
  const dispatch = useDispatch()
  useEffect(() => {
    noteService
      .getAll().then(notes => dispatch(setNotes(notes)))
  }, [dispatch]) // highlight-line

  // ...
}
```

Agora a variável <i>dispatch</i>, que definimos no componente _App_, que é praticamente a função de despacho da store do Redux, foi adicionada ao array que o useEffect recebe como parâmetro.
**Se** o valor da variável dispatch mudar durante a execução, o effect seria executado novamente. No entanto, isso não pode ocorrer em nossa aplicação, portanto, o aviso é desnecessário.

Outra maneira de se livrar do aviso seria desabilitar o ESlint nessa linha:

```js
const App = () => {
  const dispatch = useDispatch()
  useEffect(() => {
    noteService
      .getAll().then(notes => dispatch(setNotes(notes)))   
      // highlight-start
  }, []) // eslint-disable-line react-hooks/exhaustive-deps  
  // highlight-end

  // ...
}
```

Geralmente, desabilitar o ESlint quando ele gera um aviso não é uma boa ideia. Embora a regra do ESlint em questão tenha gerado algumas [discussões](https://github.com/facebook/create-react-app/issues/6880), vamos utilizar a primeira solução.

Mais informações sobre a necessidade de definir as dependências dos hooks podem ser encontradas na [documentação do React](https://reactjs.org/docs/hooks-faq.html#is-it-safe-to-omit-functions-from-the-list-of-dependencies).

### Enviando dados para o backend

Podemos fazer a mesma coisa quando se trata de criar uma nova nota. Vamos expandir o código de comunicação com o servidor da seguinte forma:

```js
const baseUrl = 'http://localhost:3001/notes'

const getAll = async () => {
  const response = await axios.get(baseUrl)
  return response.data
}

// highlight-start
const createNew = async (content) => {
  const object = { content, important: false }
  const response = await axios.post(baseUrl, object)
  return response.data
}
// highlight-end

export default {
  getAll,
  createNew, // highlight-line
}
```

O método _addNote_, do componente <i>NewNote</i>, sofre algumas alterações:

```js
import { useDispatch } from 'react-redux'
import { createNote } from '../reducers/noteReducer'
import noteService from '../services/notes' // highlight-line

const NewNote = (props) => {
  const dispatch = useDispatch()
  
  const addNote = async (event) => { // highlight-line
    event.preventDefault()
    const content = event.target.note.value
    event.target.note.value = ''
    const newNote = await noteService.createNew(content) // highlight-line
    dispatch(createNote(newNote)) // highlight-line
  }

  return (
    <form onSubmit={addNote}>
      <input name="note" />
      <button type="submit">add</button>
    </form>
  )
}

export default NewNote
```

Como o backend gera IDs para as notas, vamos alterar o action creator <em>createNote</em> no arquivo <i>noteReducer.js</i> de acordo:

```js
const noteSlice = createSlice({
  name: 'notes',
  initialState: [],
  reducers: {
    createNote(state, action) {
      state.push(action.payload) // highlight-line
    },
    // ..
  },
})
```

A alteração da importância das notas pode ser implementada utilizando o mesmo princípio, fazendo uma chamada assíncrona ao servidor e, em seguida, despachando uma action apropriada.

O estado atual do código da aplicação pode ser encontrado no [GitHub](https://github.com/fullstack-hy2020/redux-notes/tree/part6-3), na branch <i>part6-3</i>.

</div>

<div class="tasks">

### Exercícios 6.14.-6.15

#### 6.14 Anedotas e o backend, passo 1

Quando a aplicação for iniciada, busque as anedotas do backend implementado usando json-server.

Você pode usar, por exemplo, [este](https://github.com/fullstack-hy2020/misc/blob/master/anecdotes.json) arquivo como dados iniciais do backend.

#### 6.15 Anedotas e o backend, passo 2

Modifique a criação de novas anedotas para que as anedotas sejam armazenadas no backend.

</div>

<div class="content">

### Actions assíncronas e Redux Thunk

Nossa abordagem é até boa, mas não é ideal que a comunicação com o servidor aconteça dentro das funções dos componentes. Seria melhor se a comunicação pudesse ser abstraída dos componentes para que eles não precisem fazer mais nada além de chamar o <i>action creator</i> apropriado. Como exemplo, o componente <i>App</i> inicializaria o estado da aplicação da seguinte forma:

```js
const App = () => {
  const dispatch = useDispatch()

  useEffect(() => {
    dispatch(initializeNotes())  
  }, [dispatch]) 

  // ...
}
```

e o <i>NewNote</i> criaria uma nova anotação da seguinte forma:

```js
const NewNote = () => {
  const dispatch = useDispatch()
  
  const addNote = async (event) => {
    event.preventDefault()
    const content = event.target.note.value
    event.target.note.value = ''
    dispatch(createNote(content))
  }

  // ...
}
```

Nessa implementação, ambos os componentes despachariam uma action sem precisar saber sobre a comunicação com o servidor que acontece nos bastidores. Esse tipo de <i>async actions</i> pode ser implementado usando a biblioteca [Redux Thunk](https://github.com/reduxjs/redux-thunk). O uso dessa biblioteca não requer nenhuma configuração adicional, ou mesmo instalação, quando a store Redux é criada usando a função <em>configureStore</em>, do Redux Toolkit.

Com o Redux Thunk, é possível implementar <i>action creators</i> que retornam uma função em vez de um objeto. A função recebe os métodos <em>dispatch</em> e <em>getState</em> da store do Redux como parâmetros. Isso permite, por exemplo, a implementação de <i>action creators</i> assíncronos, que aguardam a conclusão de uma determinada operação assíncrona e, em seguida, despacham uma ação que altera o estado da store.

Podemos definir um action creator chamado <em>initializeNotes</em> que inicializa as notas com base nos dados recebidos do servidor:

```js
// ...
import noteService from '../services/notes' // highlight-line

const noteSlice = createSlice(/* ... */)

export const { createNote, toggleImportanceOf, setNotes, appendNote } = noteSlice.actions

// highlight-start
export const initializeNotes = () => {
  return async dispatch => {
    const notes = await noteService.getAll()
    dispatch(setNotes(notes))
  }
}
// highlight-end

export default noteSlice.reducer
```

Na função interna, ou seja, na <i>action assíncrona</i>, primeiro a operação busca todas as notas do servidor e, em seguida, <i>despacha</i> a action <em>setNotes</em>, que as adiciona à store.

O componente <i>App</i> agora pode ser definido da seguinte forma:

```js
// ...
import { initializeNotes } from './reducers/noteReducer' // highlight-line

const App = () => {
  const dispatch = useDispatch()

  // highlight-start
  useEffect(() => {
    dispatch(initializeNotes()) 
  }, [dispatch]) 
  // highlight-end

  return (
    <div>
      <NewNote />
      <VisibilityFilter />
      <Notes />
    </div>
  )
}
```

A solução é elegante. A lógica de inicialização das notas foi completamente separada do componente React.

A seguir, vamos substituir o action creator <em>createNote</em> criado pela função <em>createSlice</em> por um action creator assíncrono:

```js
// ...
import noteService from '../services/notes'

const noteSlice = createSlice({
  name: 'notes',
  initialState: [],
  reducers: {
    toggleImportanceOf(state, action) {
      const id = action.payload

      const noteToChange = state.find(n => n.id === id)

      const changedNote = { 
        ...noteToChange, 
        important: !noteToChange.important 
      }

      return state.map(note =>
        note.id !== id ? note : changedNote 
      )     
    },
    appendNote(state, action) {
      state.push(action.payload)
    },
    setNotes(state, action) {
      return action.payload
    }
    // createNote definition removed from here!
  },
})

export const { toggleImportanceOf, appendNote, setNotes } = noteSlice.actions // highlight-line

export const initializeNotes = () => {
  return async dispatch => {
    const notes = await noteService.getAll()
    dispatch(setNotes(notes))
  }
}

// highlight-start
export const createNote = content => {
  return async dispatch => {
    const newNote = await noteService.createNew(content)
    dispatch(appendNote(newNote))
  }
}
// highlight-end

export default noteSlice.reducer
```

O princípio aqui é o mesmo: primeiro, uma operação assíncrona é executada e, em seguida, a ação que altera o estado da store é <i>despachada</i>.

O componente <i>NewNote</i> sofre as seguintes alterações:

```js
// ...
import { createNote } from '../reducers/noteReducer' // highlight-line

const NewNote = () => {
  const dispatch = useDispatch()
  
  const addNote = async (event) => {
    event.preventDefault()
    const content = event.target.note.value
    event.target.note.value = ''
    dispatch(createNote(content)) //highlight-line
  }

  return (
    <form onSubmit={addNote}>
      <input name="note" />
      <button type="submit">add</button>
    </form>
  )
}
```

Por fim, vamos limpar um pouco o arquivo <i>index.js</i> movendo o código relacionado à criação da store do Redux para seu próprio arquivo, <i>store.js</i>:

```js
import { configureStore } from '@reduxjs/toolkit'

import noteReducer from './reducers/noteReducer'
import filterReducer from './reducers/filterReducer'

const store = configureStore({
  reducer: {
    notes: noteReducer,
    filter: filterReducer
  }
})

export default store
```

Após as alterações, o conteúdo do arquivo <i>index.js</i> fica da seguinte forma:

```js
import React from 'react'
import ReactDOM from 'react-dom/client'
import { Provider } from 'react-redux' 
import store from './store' // highlight-line
import App from './App'

ReactDOM.createRoot(document.getElementById('root')).render(
  <Provider store={store}>
    <App />
  </Provider>
)
```

O estado atual do código da aplicação pode ser encontrado no [GitHub](https://github.com/fullstack-hy2020/redux-notes/tree/part6-5) na branch <i>part6-5</i>.

O Redux Toolkit oferece uma série de ferramentas para simplificar o gerenciamento assíncrono de estado. Algumas ferramentas adequadas para esse caso de uso são, por exemplo, a função [createAsyncThunk](https://redux-toolkit.js.org/api/createAsyncThunk) e a API [RTK Query](https://redux-toolkit.js.org/rtk-query/overview).

</div>

<div class="tasks">

### Exercises 6.16.-6.19

#### 6.16 Anedotas e o Backend, parte 3

Modifique a inicialização da store do Redux para acontecer usando action creators assíncronos, que são possíveis graças à biblioteca Redux Thunk.

#### 6.17 Anedotas e o Backend, parte 4

Também modifique a criação de uma nova anedota para acontecer usando action creators assíncronos, possibilitados pela biblioteca Redux Thunk.

#### 6.18 Anedotas e o Backend, parte 5

A votação ainda não salva as alterações no backend. Corrija a situação com a ajuda da biblioteca Redux Thunk.

#### 6.19 Anedotas e o Backend, parte 6

A criação de notificações ainda é um pouco tediosa, pois é necessário executar duas ações e usar a função _setTimeout_:

```js
dispatch(setNotification(`new anecdote '${content}'`))
setTimeout(() => {
  dispatch(clearNotification())
}, 5000)
```

Crie um action creator que permita fornecer a notificação da seguinte forma:

```js
dispatch(setNotification(`you voted '${anecdote.content}'`, 10))
```

O primeiro parâmetro é o texto a ser exibido e o segundo parâmetro é o tempo de exibição da notificação, em segundos.

Implemente o uso dessa notificação aprimorada em sua aplicação.

</div>
