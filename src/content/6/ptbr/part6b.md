---
mainImage: ../../../images/part-6.svg
part: 6
letter: b
lang: ptbr
---

<div class="content">

Vamos continuar nosso trabalho com a simplificada [versão do Redux] (/ptbr/part6/flux_architecture_and_redux#Redux-notes) da nossa aplicação Notas.

Para facilitar nosso desenvolvimento, vamos mudar nosso reducer para que uma store seja inicializada com um estado que contenha algumas anotações:

```js
const initialState = [
  {
    content: 'reducer defines how redux store works',
    important: true,
    id: 1,
  },
  {
    content: 'state of store can contain any data',
    important: false,
    id: 2,
  },A
]

const noteReducer = (state = initialState, action) => {
  // ...
}

// ...
export default noteReducer
```

### Armazene com estado complexo

Vamos implementar a filtragem para as notas exibidas ao usuário. A interface do usuário para os filtros será implementada com os [botões radio] (https://developer.mozilla.org/en-us/docs/web/html/element/input/radio):

![navegador com os botões radio importantes/não importantes](../../images/6/01e.png)

Vamos começar com uma implementação muito simples e direta:
```js
import NewNote from './components/NewNote'
import Notes from './components/Notes'

const App = () => {
//highlight-start
  const filterSelected = (value) => {
    console.log(value)
  }
//highlight-end

  return (
    <div>
      <NewNote />
        //highlight-start
      <div>
        all          <input type="radio" name="filter"
          onChange={() => filterSelected('ALL')} />
        important    <input type="radio" name="filter"
          onChange={() => filterSelected('IMPORTANT')} />
        nonimportant <input type="radio" name="filter"
          onChange={() => filterSelected('NONIMPORTANT')} />
      </div>
      //highlight-end
      <Notes />
    </div>
  )
}
```

Como o atributo <i>name</i> de todos os botões radio é o mesmo, eles formam um <i>button group</i> onde apenas uma opção pode ser selecionada.

Os botões têm um gerenciador de alteração que atualmente imprime apenas a string associada ao botão clicado no console.

Decidimos implementar a funcionalidade do filtro armazenando <i>o valor do filtro</i> no Redux Store, além das próprias notas. O estado do store deve ficar assim depois de fazer essas alterações:

```js
{
  notes: [
    { content: 'reducer defines how redux store works', important: true, id: 1},
    { content: 'state of store can contain any data', important: false, id: 2}
  ],
  filter: 'IMPORTANT'
}
```

Somente a variedade de notas é armazenada no estado da implementação atual da nossa aplicação. Na nova implementação, o objeto de estado possui duas propriedades, <i>notas</i> que contém a matriz de notas e <i>filtro</i> que contém uma string indicando quais notas devem ser exibidas ao usuário.

### Reducers combinados

Poderíamos modificar nosso reducer atual para lidar com a nova forma do estado. No entanto, uma solução melhor nessa situação é definir um novo reducer separado para o estado do filtro:

```js
const filterReducer = (state = 'ALL', action) => {
  switch (action.type) {
    case 'SET_FILTER':
      return action.payload
    default:
      return state
  }
}
```

As ações para mudar o estado do filtro se parecem com isso:

```js
{
  type: 'SET_FILTER',
  payload: 'IMPORTANT'
}
```

Vamos também criar uma nova função _action creator_. Escreveremos o código para o action creator em um novo módulo <i>src/reducers/filterReducer.js</i>:

```js
const filterReducer = (state = 'ALL', action) => {
  // ...
}

export const filterChange = filter => {
  return {
    type: 'SET_FILTER',
    payload: filter,
  }
}

export default filterReducer
```

Podemos criar o reducer real para a nossa aplicação, combinando os dois reducers existentes com a função [combineReducers] (https://redux.js.org/api/combinereducers).

Vamos definir o reducer combinado no arquivo <i>index.js</i>:

```js
import React from 'react'
import ReactDOM from 'react-dom/client'
import { createStore, combineReducers } from 'redux' // highlight-line
import { Provider } from 'react-redux' 
import App from './App'

import noteReducer from './reducers/noteReducer'
import filterReducer from './reducers/filterReducer' // highlight-line

 // highlight-start
const reducer = combineReducers({
  notes: noteReducer,
  filter: filterReducer
})
 // highlight-end

const store = createStore(reducer) // highlight-line

console.log(store.getState())

/*
ReactDOM.createRoot(document.getElementById('root')).render(
  <Provider store={store}>
    <App />
  </Provider>
)*/

ReactDOM.createRoot(document.getElementById('root')).render(
  <Provider store={store}>
    <div />
  </Provider>
)
```

Como nossa aplicação quebra completamente neste momento, renderizamos um elemento vazio <i>div</i> em vez do componente <i>App</i>.

O estado da store é impresso no console:

![console devtools mostrando um array de dados de notas](../../images/6/4e.png)

Como podemos ver na saída do console, a store tem a forma exata que queríamos!

Vamos dar uma olhada em como o reducer combinado é criado:

```js
const reducer = combineReducers({
  notes: noteReducer,
  filter: filterReducer,
})
```

O estado da store definido pelo reducer acima é um objeto com duas propriedades: <i>notas</i> e <i>filtro</i>. O valor da propriedade <i>notas</i> é definido pelo <i>noteReducer</i>, que não precisa lidar com as outras propriedades do estado. Da mesma forma, a propriedade <i>filtro</i> é gerenciada pelo <i>filtroReducer</i>.

Antes de fazer mais alterações no código, vamos dar uma olhada em como diferentes ações mudam o estado da store definido pelo reducer combinado. Vamos adicionar o seguinte ao arquivo <i>index.js</i>:

```js
import { createNote } from './reducers/noteReducer'
import { filterChange } from './reducers/filterReducer'
//...
store.subscribe(() => console.log(store.getState()))
store.dispatch(filterChange('IMPORTANT'))
store.dispatch(createNote('combineReducers forms one reducer from many simple reducers'))
```

Ao simular a criação de uma nota e alterando o estado do filtro dessa maneira, o estado da store é registrado no console após cada alteração feita na store:

![Saída do console devtools mostrando filtro das notas e nova nota](../../images/6/5e.png)

Neste ponto, é bom tomar consciência de um detalhe minúsculo, mas importante. Se adicionarmos uma declaração de log no console <i> para o início de ambos os reducers</i>:

```js
const filterReducer = (state = 'ALL', action) => {
  console.log('ACTION: ', action)
  // ...
}
```

Com base na saída do console, pode-se ter a impressão de que toda ação é duplicada:

![Saída do console DevTools mostrando ações duplicadas nos reducers de nota e filtro](../../images/6/6.png)

Existe um bug no nosso código? Não. O reducer combinado trabalha de tal maneira que cada <i>ação</i> é tratada em <i>toda</i> parte do reducer combinado. Normalmente, apenas um reducer está interessado em qualquer ação, mas há situações em que vários reducers alteram suas respectivas partes do estado com base na mesma ação.

### Terminando os filtros

Vamos terminar a aplicação para que ele use o reducer combinado. Começamos alterando a renderização da aplicação e conectando a store à aplicação no arquivo <i>index.js</i>:

```js
ReactDOM.createRoot(document.getElementById('root')).render(
  <Provider store={store}>
    <App />
  </Provider>
)
```

Em seguida, vamos corrigir um bug causado pelo código que espera que o armazenamento da aplicação seja uma variedade de notas:

![Browser TypeError: Notes.map não é uma função](../../images/6/7ea.png)

É uma solução fácil. Como as notas estão no campo da store <i>notas</i>, só precisamos fazer uma pequena alteração na função seletor:

```js
const Notes = () => {
  const dispatch = useDispatch()
  const notes = useSelector(state => state.notes) // highlight-line

  return(
    <ul>
      {notes.map(note =>
        <Note
          key={note.id}
          note={note}
          handleClick={() => 
            dispatch(toggleImportanceOf(note.id))
          }
        />
      )}
    </ul>
  )
}
```

Anteriormente, a função seletora retornou todo o estado da store:

```js
const notes = useSelector(state => state)
```

E agora ele retorna apenas seu campo <i>notas</i>

```js
const notes = useSelector(state => state.notes)
```

Vamos extrair o filtro de visibilidade para o seu próprio componente <i>src/components/visibilityFilter.js</i>:

```js
import { filterChange } from '../reducers/filterReducer'
import { useDispatch } from 'react-redux'

const VisibilityFilter = (props) => {
  const dispatch = useDispatch()

  return (
    <div>
      all    
      <input 
        type="radio" 
        name="filter" 
        onChange={() => dispatch(filterChange('ALL'))}
      />
      important   
      <input
        type="radio"
        name="filter"
        onChange={() => dispatch(filterChange('IMPORTANT'))}
      />
      nonimportant 
      <input
        type="radio"
        name="filter"
        onChange={() => dispatch(filterChange('NONIMPORTANT'))}
      />
    </div>
  )
}

export default VisibilityFilter
```

Com o novo componente <i>App</i> pode ser simplificado da seguinte forma:

```js
import Notes from './components/Notes'
import NewNote from './components/NewNote'
import VisibilityFilter from './components/VisibilityFilter'

const App = () => {
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

A implementação é bastante direta. Clicar nos diferentes botões radio altera o estado da propriedade <i>filtro</i> da store.

Vamos alterar o componente <i>notas</i> para incorporar o filtro:

```js
const Notes = () => {
  const dispatch = useDispatch()
  // highlight-start
  const notes = useSelector(state => {
    if ( state.filter === 'ALL' ) {
      return state.notes
    }
    return state.filter  === 'IMPORTANT' 
      ? state.notes.filter(note => note.important)
      : state.notes.filter(note => !note.important)
  })
  // highlight-end

  return(
    <ul>
      {notes.map(note =>
        <Note
          key={note.id}
          note={note}
          handleClick={() => 
            dispatch(toggleImportanceOf(note.id))
          }
        />
      )}
    </ul>
  )
```

Só fazemos alterações na função seletor, que costumava ser

```js
useSelector(state => state.notes)
```

Vamos simplificar o seletor desestruturando os campos do estado que recebe como parâmetro:

```js
const notes = useSelector(({ filter, notes }) => {
  if ( filter === 'ALL' ) {
    return notes
  }
  return filter  === 'IMPORTANT' 
    ? notes.filter(note => note.important)
    : notes.filter(note => !note.important)
})
```

Há uma leve falha cosmética em nossa aplicação. Embora o filtro esteja definido como <i>ALL</i> por padrão, o botão radio associado não é selecionado. Naturalmente, esse problema pode ser corrigido, mas como este é um bug desagradável, porém um bug inofensivo, vamos salvar a correção para mais tarde.

A versão atual da aplicação pode ser encontrada no [GitHub](https://github.com/fullstack-hy2020/redux-notes/tree/part6-2), branch <i>part6-2</i>.

</div>

<div class="tasks">

### Exercício 6.9

#### 6.9 Melhores anedotas, step7

Implementar filtragem para as anedotas exibidas ao usuário.

![navegador mostrando filtragem de anedotas](../../images/6/9ea.png)

Armazene o estado do filtro na Redux store. Recomenda-se criar um novo reducer, action creators e um reducer combinado para a store usando a função <i>combineReducers</i>.

Crie um novo componente <i>filtro</i> para exibir o filtro. Você pode usar o seguinte código como modelo para o componente:

```js
const Filter = () => {
  const handleChange = (event) => {
    // input-field value is in variable event.target.value
  }
  const style = {
    marginBottom: 10
  }

  return (
    <div style={style}>
      filter <input onChange={handleChange} />
    </div>
  )
}

export default Filter
```

</div>

<div class="content">

### Redux Toolkit

Como vimos até agora, a implementação de configuração e gerenciamento de estado do Redux requer muito esforço. Isso se manifesta, por exemplo, no código relacionado ao reducer e ao action creator, que possui código padronizado um tanto repetitivo. [Redux Toolkit] (https://redux-toolkit.js.org/) é uma biblioteca que resolve esses problemas comuns relacionados ao Redux. A biblioteca, por exemplo, simplifica bastante a configuração do Redux Store e oferece uma grande variedade de ferramentas para facilitar o gerenciamento do estado.

Vamos começar a usar o Redux Toolkit em nossa aplicação, refatorando o código existente. Primeiro, precisaremos instalar a biblioteca:

```bash
npm install @reduxjs/toolkit
```

Em seguida, abra o arquivo <i>index.js</i>, que atualmente cria a Redux store. Em vez da função Redux <em>createStore</em>, vamos criar a store usando a função configureStore do [Redux Toolkit](https://redux-toolkit.js.org/api/configureStore):

```js
import React from 'react'
import ReactDOM from 'react-dom/client'
import { Provider } from 'react-redux'
import { configureStore } from '@reduxjs/toolkit' // highlight-line
import App from './App'

import noteReducer from './reducers/noteReducer'
import filterReducer from './reducers/filterReducer'

 // highlight-start
const store = configureStore({
  reducer: {
    notes: noteReducer,
    filter: filterReducer
  }
})
// highlight-end

console.log(store.getState())

ReactDOM.createRoot(document.getElementById('root')).render(
  <Provider store={store}>
    <App />
  </Provider>
)
```

Já nos livramos de algumas linhas de código agora que não precisamos da função <em>combineReducers</em> para criar o reducer para a store. Em breve, veremos que a função <em>configureStore </em> tem muitos benefícios adicionais, como a integração sem esforço das ferramentas de desenvolvimento e muitas bibliotecas comumente usadas sem a necessidade de configuração adicional.

Vamos refatorar os reducers, o que traz os benefícios do Redux Toolkit. Com o Redux Toolkit, podemos facilmente criar reducers e action creators relacionados usando a função [createSlice](https://redux-toolkit.js.org/api/createSlice). Podemos usar a função <em>createSlice</em> para refatorar o reducer e os action creators no arquivo <i>reducers/noteReducer.js</i> da seguinte maneira:

```js
import { createSlice } from '@reduxjs/toolkit' // highlight-line

const initialState = [
  {
    content: 'reducer define como redux store funciona',
    important: true,
    id: 1,
  },
  {
    content: 'estado da store pode conter qualquer valor',
    important: false,
    id: 2,
  },
]

const generateId = () =>
  Number((Math.random() * 1000000).toFixed(0))

// highlight-start
const noteSlice = createSlice({
  name: 'notes',
  initialState,
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
    }
  },
})
// highlight-end
```

O parâmetro <em>createSlice</em> da função <em>name</em> define o prefixo que é usado nos valores do tipo da ação. Por exemplo, a ação <em>createNote</em> definida posteriormente terá o valor do tipo de <em>notes/createNote</em>. É uma boa prática dar ao parâmetro um valor, que é único entre os reducers. Dessa forma, não haverá colisões inesperadas entre os valores do tipo de ação da aplicação. O parâmetro <em>InitialState</em> define o estado inicial do reducer. O parâmetro <em>reducers</em> toma o próprio reducer como um objeto, do qual as funções lidam com as alterações do estado causadas por certas ações. Observe que o <em>action.payload</em> na função contém o argumento fornecido chamando o action creator:

```js
dispatch(createNote('Redux Toolkit is awesome!'))
```

Esta chamada de envio responde ao envio do seguinte objeto:

```js
dispatch({ type: 'notes/createNote', payload: 'Redux Toolkit is awesome!' })
```

Se você acompanhou de perto, deve ter notado que dentro da ação <em>createNote</em>, parece acontecer algo que viola o princípio da imutabilidade dos reducers mencionado anteriormente:

```js
createNote(state, action) {
  const content = action.payload

  state.push({
    content,
    important: false,
    id: generateId(),
  })
}
```

Estamos transformando o array do argumento <em>state</em> chamando o método <em>push</em> em vez de retornar uma nova instância do array. Do que se trata?

O Redux Toolkit utiliza a biblioteca [imer] (https://immerjs.github.io/immer/) com reducers criados pela função <em>createSlice</em>, o que possibilita a transformação do argumento <em>state</em> dentro do reducer. Immer usa o estado mutado, transformado para produzir um estado novo e imutável e, portanto, as mudanças de estado permanecem imutáveis. Observe que o <em>state</em> pode ser alterado sem "mutá-lo", como fizemos com a <em>ToggleImportance of</em>. Nesse caso, a função <i>retorna</i> o novo estado. No entanto, a mutação do estado geralmente é útil, especialmente quando um estado complexo precisa ser atualizado.

A função <em>createSlice</em> retorna um objeto que contém o reducer, bem como os action creators definidos pelo parâmetro <em>reducer</em>. O reducer pode ser acessado pela propriedade <em>noteslice.reducer</em>, enquanto os action creators pela propriedade <em>noteSlice.actions</em>. Podemos produzir as exportações do arquivo da seguinte maneira:

```js
const noteSlice = createSlice(/* ... */)

// highlight-start
export const { createNote, toggleImportanceOf } = noteSlice.actions

export default noteSlice.reducer
// highlight-end
```

As importações em outros arquivos funcionarão exatamente como fizemos antes:

```js
import noteReducer, { createNote, toggleImportanceOf } from './reducers/noteReducer'
```

Precisamos alterar os nomes do tipo de ação nos testes devido às convenções do Redux Toolkit:

```js
import noteReducer from './noteReducer'
import deepFreeze from 'deep-freeze'

describe('noteReducer', () => {
  test('returns new state with action notes/createNote', () => {
    const state = []
    const action = {
      type: 'notes/createNote', // highlight-line
      payload: 'the app state is in redux store', // highlight-line
    }

    deepFreeze(state)
    const newState = noteReducer(state, action)

    expect(newState).toHaveLength(1)
    expect(newState.map(s => s.content)).toContainEqual(action.payload)
  })

  test('returns new state with action notes/toggleImportanceOf', () => {
    const state = [
      {
        content: 'the app state is in redux store',
        important: true,
        id: 1
      },
      {
        content: 'state changes are made with actions',
        important: false,
        id: 2
      }]
  
    const action = {
      type: 'notes/toggleImportanceOf', // highlight-line
      payload: 2
    }
  
    deepFreeze(state)
    const newState = noteReducer(state, action)
  
    expect(newState).toHaveLength(2)
  
    expect(newState).toContainEqual(state[0])
  
    expect(newState).toContainEqual({
      content: 'state changes are made with actions',
      important: true,
      id: 2
    })
  })
})
```

### Redux Toolkit e console.log

Como aprendemos, o console.log é uma ferramenta extremamente poderosa, geralmente sempre nos salva de problemas.

Vamos tentar imprimir o estado do Redux Store no console no meio do reducer criado com a função createSlice:

```js
const noteSlice = createSlice({
  name: 'notes',
  initialState,
  reducers: {
    // ...
    toggleImportanceOf(state, action) {
      const id = action.payload

      const noteToChange = state.find(n => n.id === id)

      const changedNote = { 
        ...noteToChange, 
        important: !noteToChange.important 
      }

      console.log(state) // highlight-line

      return state.map(note =>
        note.id !== id ? note : changedNote 
      )     
    }
  },
})
```

O seguinte é impresso no console

![console devtools mostrando Handler, Target como nulo, mas IsRevoked como verdadeiro](../../images/6/40new.png)

A saída é interessante, mas não muito útil. Trata-se da biblioteca Immer mencionada anteriormente usada pelo Redux Toolkit, que agora é usado internamente para salvar o estado da store.

O status pode ser convertido em um formato legível pelo homem, por exemplo ao convertê-lo em uma string e voltar a um objeto JavaScript da seguinte forma:

```js
console.log(JSON.parse(JSON.stringify(state))) // highlight-line
```

A saída do console agora está legível para humanos

![ferramentas de desenvolvimento mostrando o array de 2 notas](../../images/6/41new.png)

### Redux DevTools

[Redux DevTools](https://chrome.google.com/webstore/detail/redux-devtools/lmhkpmbekcpmknklioeibfkpmmfibljd) é um addon Chrome que oferece ferramentas de desenvolvimento úteis para o Redux. Pode ser usado, por exemplo, para inspecionar o estado da Redux store e enviar as ações através do console do navegador. Quando a store é criada usando a função <em>configureStore</em> do Redux Toolkit, nenhuma configuração adicional é necessária para que o Redux Devtools funcione.

Depois que o addon estiver instalado, clicar na guia <i>Redux</i> no console do navegador deve abrir as ferramentas de desenvolvimento:

![Navegador com addon Redux em Devtools](../../images/6/42new.png)

Você pode inspecionar como o envio de uma determinada ação muda o estado clicando na ação:

![DevTools inspecionando a árvore de notas no Redux](../../images/6/43new.png)

Também é possível enviar ações para a store usando as ferramentas de desenvolvimento:

![Redux Devtools enviando createNote com payload](../../images/6/44new.png)

Você pode encontrar o código da nossa aplicação atual na íntegra na <i>part6-3</i> branch [deste repositório do GitHub](https://github.com/fullstack-hy2020/redux-notes/tree/part6-3).

</div>

<div class="tasks">

### Exercícios 6.10.-6.13.

Vamos continuar trabalhando na aplicação de anedota usando o Redux que começamos no Exercício 6.3.

#### 6.10 Melhores anedotas, step8

Instale o Redux Toolkit para o projeto. Mova a Redux store creation para o arquivo <i>store.js</i> E use a função <em>configureStore</em> do Redux Toolkit para criar a store.

Altere a definição do reducer de <i>filtro e action creators</i> para usar a função <em>createSlice</em> do Redux Toolkit.

Além disso, comece a usar o Redux DevTools para depurar o estado da aplicação mais fácil.

#### 6.11 Melhores anedotas, step9

Altere também a definição do <i>Reducer de Anedota e action creators</i> para usar a função <em> CreateSlice <em> do Redux Toolkit.

#### 6.12 Melhores anedotas, step10

A aplicação possui um corpo pronto para o componente <i>notificação</i>:

```js
const Notification = () => {
  const style = {
    border: 'solid',
    padding: 10,
    borderWidth: 1
  }
  return (
    <div style={style}>
      render here notification...
    </div>
  )
}

export default Notification
```

Estenda o componente para que ele renderize a mensagem armazenada na Redux store, fazendo o componente assumir o seguinte formulário:

```js
import { useSelector } from 'react-redux' // highlight-line

const Notification = () => {
  const notification = useSelector(/* something here */) // highlight-line
  const style = {
    border: 'solid',
    padding: 10,
    borderWidth: 1
  }
  return (
    <div style={style}>
      {notification} // highlight-line
    </div>
  )
}
```

Você terá que fazer alterações no reducer existente na aplicação. Crie um reducer separado para a nova funcionalidade usando a função <em>createSlice</em> do Redux Toolkit.

A aplicação não precisa usar o componente <i>Notification</i> de forma inteligente neste momento dos exercícios. É suficiente para a aplicação exibir o valor inicial definido para a mensagem no <i>notificationReducer</i>.

#### 6.13 Melhores anedotas, step11

Estenda a aplicação para que ele use o componente <i>Notification</i> para exibir uma mensagem por cinco segundos quando o usuário vota em uma anedota ou cria uma nova anedota:

![navegador mostrando a mensagem por ter votado](../../images/6/8ea.png)

É recomendável criar separados os [action creators] (https://redux-toolkit.js.org/api/createslice#redcers) para definir e remover notificações.

</div>
