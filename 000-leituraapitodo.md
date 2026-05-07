
```typescript
//-----------------------------------------------------------------
// core.ts
// Lógica central da lista de tarefas (todo list)
// Responsável por: ler, salvar e manipular os dados em arquivo

// Caminho do arquivo onde os dados ficam salvos (na mesma pasta do script)
const jsonFilePath = __dirname + '/data.temp.json';

// Lista em memória — carregada do arquivo ao iniciar o servidor
// Todas as operações mexem aqui primeiro, e depois salvam no arquivo
const list: string[] = await loadFromFile();

// loadFromFile()
// Lê o arquivo JSON e retorna o array de strings salvo nele.
// Se o arquivo não existir ainda (ENOENT), retorna lista vazia.
async function loadFromFile() {
  try {
    const file = Bun.file(jsonFilePath);       // Abre o arquivo com a API do Bun
    const content = await file.text();          // Lê o conteúdo como texto
    return JSON.parse(content) as string[];     // Converte de JSON para array JS
  } catch (error: any) {
    if (error.code === 'ENOENT')               // ENOENT = arquivo não encontrado
      return [];                               // Primeira vez rodando: começa do zero
    throw error;                               // Qualquer outro erro, deixa estourar
  }
}


// saveToFile()
// Serializa a lista em memória para JSON e grava no arquivo.
// Chamada toda vez que a lista é modificada (add/update/remove).
async function saveToFile() {
  try {
    await Bun.write(jsonFilePath, JSON.stringify(list)); // Escreve o array como JSON
  } catch (error: any) {
    throw new Error("Erro ao salvar os dados no arquivo: " + error.message);
  }
}

// addItem(item)
// Adiciona um novo item ao final da lista e salva.
async function addItem(item: string) {
  list.push(item);       // Empurra o novo item para o array em memória
  await saveToFile();    // Persiste a mudança no arquivo
}

// getItems()
// Retorna uma cópia do estado atual da lista em memória.
// (Não precisa ler o arquivo de novo — já está carregado)
async function getItems() {
  return list;
}

// updateItem(index, newItem)
// Substitui o item em `index` pelo valor `newItem`.
// Lança erro se o índice estiver fora do range da lista.
async function updateItem(index: number, newItem: string) {
  if (index < 0 || index >= list.length)
    throw new Error("Index fora dos limites");   // Validação: índice deve existir
  list[index] = newItem;                          // Substitui no array em memória
  await saveToFile();                             // Persiste
}

// removeItem(index)
// Remove o item na posição `index` da lista.
// splice(index, 1) remove 1 elemento a partir daquele índice.
// Lança erro se o índice estiver fora do range.
async function removeItem(index: number) {
  if (index < 0 || index >= list.length)
    throw new Error("Index fora dos limites");   // Validação: índice deve existir
  list.splice(index, 1);                          // Remove 1 item naquela posição
  await saveToFile();                             // Persiste
}

// Exporta as funções para serem usadas no api.ts
// Só expõe o necessário — saveToFile e loadFromFile são internos
export default { addItem, getItems, updateItem, removeItem };

//-----------------------------------------------------

// api.ts
// Servidor HTTP com Bun
// Responsável por: expor as rotas REST que o frontend consome
// Porta padrão: 3000  →  http://localhost:3000

import todo from "./core.ts";   // Importa a lógica do core (add, get, update, remove)

const server = Bun.serve({
  port: 3000,

  // routes: objeto onde cada chave é um path da URL
  // Cada path pode ter sub-chaves com os métodos HTTP (GET, POST, PUT, DELETE...)
  routes: {

    // Serve o HTML estático quando o usuário abre o site no browser
    "/": new Response(Bun.file("./public/index.html")),

    // ROTAS DA TODO LIST
    "/api/todo": {

      // GET /api/todo → Retorna todos os itens da lista como JSON
      GET: async () => {
        const items = await todo.getItems()
        return Response.json(items)       // Ex: ["Comprar pão", "Estudar TypeScript"]
      },

      // POST /api/todo → Adiciona um novo item
      // Body esperado: { "item": "texto do novo item" }
      POST: async (req) => {
        const data = await req.json() as any;  // Lê o body da requisição como JSON
        const item = data.item || null;

        // Validação: se não veio "item" no body, rejeita com 400
        if (!item)
          return Response.json('Por favor, forneça um item para adicionar.', { status: 400 });

        await todo.addItem(item);         // Adiciona na lista e salva no arquivo
        return Response.json(data);       // Devolve o que foi recebido como confirmação
      },
    },

    // Rotas com parâmetro dinâmico (:index)
    // O `:index` é substituído pelo número na URL real
    // Ex: PUT /api/todo/2  →  req.params.index === "2"
    "/api/todo/:index": {

      // PUT /api/todo/:index → Atualiza o item na posição `index`
      // Body esperado: { "newItem": "novo texto" }
      PUT: async (req) => {
        const index = parseInt(req.params.index);   // Pega o índice da URL e converte para número

        // Validação: se não for um número, rejeita
        if (isNaN(index))
          return Response.json('Índice inválido. um número inteiro é esperado.', { status: 400 });

        const data = await req.json() as any;
        const newItem = data.newItem || null;

        // Validação: se não veio "newItem" no body, rejeita
        if (!newItem)
          return Response.json('Por favor, forneça um novo item para atualizar.', { status: 400 });

        try {
          await todo.updateItem(index, newItem);     // Atualiza no core
          return Response.json(`Item no índice ${index} atualizado para "${newItem}".`);
        } catch (error: any) {
          return Response.json(error.message, { status: 400 });  // Ex: "Index fora dos limites"
        }
      },

      // DELETE /api/todo/:index → Remove o item na posição `index`
      DELETE: async (req) => {
        const index = parseInt(req.params.index);

        if (isNaN(index))
          return Response.json('Índice inválido.', { status: 400 });

        try {
          await todo.removeItem(index);              // Remove no core
          return Response.json(`Item no índice ${index} removido com sucesso.`);
        } catch (error: any) {
          return Response.json(error.message, { status: 400 });
        }
      },
    },
    
    // EXEMPLO BÁSICO — Garatuja de como estruturar rotas no Bun
    // Pode copiar e adaptar este bloco para criar novas rotas
    "/api/exemplo": {

      // GET simples — sem body, só retorna uma string
      GET: () => {
        return new Response(`Esse é o exemplo: ${Date.now()}`)
                                        // Date.now() = timestamp atual em ms
      },

      // POST com body JSON — lê os dados e adiciona informações extras antes de devolver
      POST: async (req) => {
        const data = await req.json() as any;
        data.recebidoEm = new Date().toLocaleDateString("pt-BR");  // Adiciona data no retorno
        return Response.json(data);
      },
    },

    // Rota com parâmetro :id — template para recursos únicos
    "/api/exemplo/:id": {

      // PUT — Atualiza completamente o recurso (manda o objeto todo)
      PUT: async (req, params) => {
        const { id } = req.params;               // Pega o :id da URL
        const data = await req.json() as any;
        data.id = id;                            // Injeta o id no retorno
        data.recebidoEm = new Date().toLocaleDateString("pt-BR");
        return Response.json(data);
      },

      // PATCH — Atualiza parcialmente (só os campos enviados)
      // Diferença do PUT: PATCH recebe só o que mudou, PUT recebe o objeto completo
      PATCH: async (req, params) => {
        const { id } = req.params;
        const data = await req.json() as any;
        data.chavesAtualizadas = Object.keys(data);   // Lista quais campos foram enviados
        data.id = id;
        data.atualizadoEm = new Date().toLocaleDateString("pt-BR");
        return Response.json(data);
      },

      // DELETE — Remove o recurso pelo id
      DELETE: (req, params) => {
        const { id } = req.params;
        return new Response(`Recurso com id ${id} deletado`, { status: 200 });
      }
    }
    // FIM DO EXEMPLO BÁSICO
  },

  // fetch() — Fallback para qualquer rota não mapeada acima
  // Se o cliente bater em /qualquer-coisa-desconhecida → 404
  async fetch(req) {
    return new Response(`Not Found`, { status: 404 });
  },
});

// Confirma no terminal que o servidor subiu corretamente
console.log(`Server running at http://localhost:${server.port}`);
```